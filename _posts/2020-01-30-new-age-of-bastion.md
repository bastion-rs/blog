---
layout: post
title: "New age of Bastion"
author: Written by Mahmut Bulut, and edited by Jeremy Lempereur
---

**Disclaimer:** This post is not only an announcement of new features of Bastion, which comes with 0.3.4; it also declares our future perspective for the project.

So we had a long way since our project had started in June 2019. Right now, we are evolving from a runtime that enables fault-recovery for your server-side applications to an agnostic runtime, which has its executor but also works with other executors. Since the beginning we can interoperate with [async-std](https://async.rs) without the hassle.

We have started with an idea of runtime, which can recover from failures in the application domain, no matter what happens. Async/await landed, and we got stronger and well known in the community. Then Bastion deployed to AWS lambdas, did, and still doing its job in data processing pipelines. Benchmarks of sparse queries outperformed plenty of sparse queries and aggregations over NYC taxi data in various inmem benchmarks.

Where futures-executor performs:
```
test count_aggregation      ... bench: 108,916,527 ns/iter (+/- 11,909,528)
```
Bastion performed:
```
test count_aggregation      ... bench: 87,096,485 ns/iter (+/- 3,874,812)
```
Everything started with an idea of replicating what Erlang does in its runtime. Having the idea of using the runtime for data processing came right after it. Bastion's purpose is to provide a generic runtime for fault-tolerant middlewares, distributed systems, and data processing pipelines.

I didn't aim for faster runtime, but I aimed to exploit the whole resources whenever it's possible. Bastion also doesn't aim to replace every other existing runtimes. It just unfolds a different approach to design systems. This system design is very graspable for the people who come from a functional background like Scala, Erlang, or Haskell. When we released Bastion 0.1 in June 2019 with Tokio runtime, I realized that it's not the way that this project can continue. With premature design assumptions of the runtime, I struggled with developing the project with Tokio runtime. Having to implement a monolithic runtime with several lack of reduction based concurrency principles made me write an executor for Bastion. So we end up having a multiple run queue based, redundancy statistics sampled, async executor which is a primer of the runtime. Throughout time, people wanted to use blocking tasks also. So we came with an idea of using an adaptive blocking pool, which can [automatically scale the resources based on the throughput](https://docs.rs/bastion-executor/0.3.4/bastion_executor/blocking/index.html) and prevents IO hogs.

## What is next with Bastion?

* We are currently working on being runtime agnostic. Working in everywhere but still supplying our own executor to the people who want to have fault-tolerant systems in their projects.

* We are exploring HCS(Hot code swap) case. And currently, we have [design discussions](https://github.com/bastion-rs/bastion/issues/103) going on. We would be really happy to gather your feedback and ideas on HCS!

* WASM is a rising star in the ecosystem. We would like to support it, first we want to make everything `#[no_std]`, then we want to move towards to make Bastion projects compile for WASM.

* We are working on Bastion Streams! A streaming framework implementation which is backpressure aware and works with a push interface like API. Many people are asking about how this will be implemented, and I have pretty concrete ideas about this. Especially from the perspective of Rust! So now I am quite bit busy enabling parallel streams in Rust. **Not with [this](https://docs.rs/futures/0.3.1/futures/stream/trait.Stream.html), or [this](https://rust-lang.github.io/async-book/05_streams/01_chapter.html), or via [this](https://rust-lang.github.io/async-book/06_multiple_futures/03_select.html)**. Come join us if you find this interesting!

* Distributed properties are still needs to be solved. We are slow at that side but wrote initial epidemic protocol implementation
We aren't moving too fast on this, but an initial epidemic protocol implementation can be seen and used in our [Artillery](http://github.com/bastion-rs/artillery) repository.

## New version is in here! Tell me more!

We have added features to `0.3.4`, you can see it in our [CHANGELOG](https://github.com/bastion-rs/bastion/blob/master/CHANGELOG.md). But I will showcase a bunch of excellent features of the new Bastion version!

------

**At Bastion itself, at the surface API level, here are our changes:**

**0-** We added restart strategies, linear and exponential backoff between restarts. And maximum restarts for the failed children. Check out our [example in the repo](https://github.com/bastion-rs/bastion/blob/master/bastion/examples/restart_strategy.rs). Thanks to [@Relrin](https://github.com/Relrin) and for the clean cut of their PR!
```rust
// Here we are specifying the used restart strategy for our supervisor.
// By default the bastion's supervisors are always trying to restart
// failed actors with unlimited amount of tries.
//
// At the beginning we're creating a new instance of RestartStrategy
// and then provides a policy and a back-off strategy.
let restart_strategy = RestartStrategy::default()
    // Set the limits for supervisor, so that it could stop
    // after 3 attempts. If the actor can't be started, the supervisor
    // will remove the failed actor from tracking.
    .with_restart_policy(RestartPolicy::Tries(3))
    // Set the desired restart strategy. By default supervisor will
    // try to restore the failed actor as soon as possible. However,
    // in our case we want to restart with a small delay between the
    // tries. Let's say that we want a regular time interval between the
    // attempts which is equal to 1 second.
    .with_actor_restart_strategy(ActorRestartStrategy::LinearBackOff {
      timeout: Duration::from_secs(1),
    });

// Define the supervisor
supervisor
	 // That uses our restart strategy defined earlier
	 .with_restart_strategy(restart_strategy)
	 // And tracks the child group, defined in the following function
	 .children(|children| failed_actors_group(children))
```
**1-** Now we have a signature method to address all the actors in the system. Check our context addressing system [here](https://docs.rs/bastion/0.3.4/bastion/context/struct.BastionContext.html#method.signature)

```rust
Bastion::children(|children| {
    children.with_exec(|ctx: BastionContext| {
        async move {
            ctx.tell(&ctx.signature(), "Hello to myself");

            Ok(())
        }
    })
}).expect("Couldn't create the children group.");
```

**2-** We've added meta-architecture instantiation, so you can have a boilerplate free hierarchy for your actors and/or workers.
```rust
let children = children! {
	// the default redundancy is 1
	redundancy: 100,
	action: |msg| {
	   // do something with the message here
	},
};

// simpler supervisor declaration.
let sp = supervisor! {};

let children = children! {
  	supervisor: sp,
  	redundancy: 10,
  	action: |msg| {
    	// do something with the message here
    },
};
```

**3-** You can now mark your blocking code and execute it inside Bastion's blocking adaptive thread pool.
```rust
let task = blocking! {
	thread::sleep(time::Duration::from_millis(3000));
};
```

**4-** If you want to run a process and wait for it's completion, you can call `run!`. `run` is equivalent to what other Rust frameworks call `block_on`. We decided to call it run because that's the name implementations in other languages use. We think sticking to cross languages conventions will avoid confusion among users, regardless of their favorite language.

With the blocking task written above, we can use:
```rust
run!(task);
```
And that's it! This is equivalent of [zio running effects](https://zio.dev/docs/overview/overview_running_effects) in Rust environment. Now we have it!

-------

**The Executor, our huge steampunk engine**

This is the core, the piece everything is running on top of.

It is way more diverged and different from any runtime out there.
The approach we took is very different from the other frameworks in the ecosystem.

What are the differences ? Let's have a look!

**0-** When under contention we skip work-stealing. We directly force queue consumers to handle messages coming from the run queue and continue working without work-stealing for a small fraction of time. Think of it like a congestion unblocking mechanism. It works pretty well so far! Works pretty well.

**1-** Bastion executor relies mostly on interconnect messages and tries to do single clock advancement without OS migration of threads. We are pinning our threads which means we need to be aware of how many vCPU Bastion is running on, and we maintain thread pinning on the fly. A lot of efforts were put into our pinning API in this release.

**2-** We have a blocking pool based on the paper [A Novel Predictive and Self–Adaptive Dynamic Thread Pool Management](https://ieeexplore.ieee.org/document/5951889) and you can see this implementation documentation [here](https://docs.rs/bastion-executor/0.3.4/bastion_executor/blocking/index.html). That's our blocking congestion prevention mechanism and it enables high throughput for the blocking operation under various cases ranging from long running, spike oriented, bursting, decaying task timing behaviors.
Moreover, this thread pool is also [configurable with an environment variable](https://github.com/bastion-rs/bastion/pull/141/files#diff-d7ca8f060f95b61604b5bd4036546196R334-R349) for constrained environments.

**3-** We improved the performance by making less syscalls when yielding. We are now passively looking for an opportunity to run tasks, and yield only once, as soon as possible.

**4-** Added convenience methods for blocking pool with [spawn_blocking](https://docs.rs/bastion-executor/0.3.4/bastion_executor/blocking/fn.spawn_blocking.html) in addition to existing [spawn](https://docs.rs/bastion-executor/0.3.4/bastion_executor/pool/fn.spawn.html), [run](https://docs.rs/bastion-executor/0.3.4/bastion_executor/run/fn.run.html). If you don't want any supervision and restart mechanisms, and are just looking for a quick dive into bastion, add `bastion-executor` to your `Cargo.toml` file and you can start spawning sync and async tasks in no time !

------

Bastion uses [LightProc](https://docs.rs/lightproc/0.3.4/lightproc/lightproc/index.html), and it's the crucial implementation that makes fault tolerance easier for the applications.

**In LightProc we have added:**

An interface for storing state! This will be used to persist messages or data in memory and can be backed by anything that you can define.
```rust
use lightproc::proc_stack::ProcStack;

pub struct GlobalState {
   pub amount: usize
}

ProcStack::default()
    .with_pid(1)
    .with_state(GlobalState { amount: 1 });
```
From Rust HashMap's to your NoSQL backend etc. [Wire up anything](https://docs.rs/lightproc/0.3.4/lightproc/proc_stack/struct.ProcStack.html#method.with_state) to your processes. Soon we will expose convenience APIs for mailbox preservation across restarts. [@onsails](https://github.com/onsails) is working on it [here](https://github.com/bastion-rs/bastion/pull/143).

## A Big Thanks

Yes, a big thanks, to everyone who enabled this project to run, supported, wrote code and made it production grade in plenty of aspects. Thanks for everything and thanks for paving the road with us! We are getting bigger and I really like how passionate people are about what are we trying to do here! I can't thank enough everyone who shared their ideas on [our Discord channel](https://discord.gg/DqRqtRT), on issues and over the PRs! Thank you all, let's continue building together! 2020 is a year that we (as Rust community) will unite over plenty of subjects, and topics. I like to close my thanks with these words (though this was a famous US Marshall Plan propaganda, this sentence pretty much sums up all of us as people working on Rust): **Whatever the weather we must move together!**

## Conclusion
In the upcoming blog posts, we will talk about how the Bastion runtime is designed, which design decisions were made (and why) and our newest initiatives and next goals. Meanwhile we are expanding our ecosystem with various projects and would love to welcome maintainers who want to work on distributed systems and data processing. We have pretty high aims for our projects, and would love to help and mentor you. Come join us! Join our [Discord](https://discordapp.com/invite/DqRqtRT), have a look at our issues and projects [our org's GitHub](https://github.com/bastion-rs)!
