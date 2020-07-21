---
layout: post
title: "Celebrate the 1st Bastioniversary with the 0.4 release! :tada: :confetti_ball: :rocket:"
author: Written by the Bastion team
---

### The Excitement of a 1st birthday!

Exactly 1 year passed since the initial release of Bastion. In this one year we’ve done so many amazing things together as Bastion team.

We are proud to announce our 0.4 release on our 1st birthday! :tada: :confetti_ball: :rocket:


### What is Bastion?

Bastion is a distributed runtime which is inspired from the design principles of Erlang and adapts these into Rust ecosystem. 


### What is new?

This release is coming with exciting features:

- **Agnostik** executor runtime
    - With Agnostik, Bastion can run on any runtime. Head over to the \[agnostik project\](https://github.com/bastion-rs/agnostik/) for more information.


- **Nuclei, an agnostic proactive IO system**
    - With nuclei and agnostik, IO and executors are separated.
    - You can mix and match any executor with agnostik and use the same IO system.
    - Nuclei is based on Proactive IO.
    - Nuclei supports io_uring and works well with completion based evented IO. Windows support is currently in the works.
    - Completely async, can be independently used from Bastion.
        - It will power Bastion’s IO system and ecosystem.


- **Autoscaling feature** for actors
    - Right now with the `scaling` feature enabled, you can create actor groups in Bastion that will adapt to incoming workload according to given resizer parameters.
    - Example construction is like:
```rust
    children
        // Start with 3 actors
        .with_redundancy(3)
        // Do heartbeat every 5 seconds
        .with_heartbeat_tick(Duration::from_secs(5))
        .with_resizer(
            OptimalSizeExploringResizer::default()
                // A minimal acceptable size of group
                .with_lower_bound(1)
                // Max 10 actors in runtime
                .with_upper_bound(UpperBound::Limit(10))
                // Scale up when a half of actors have more than 3 messages
                .with_upscale_strategy(UpscaleStrategy::MailboxSizeThreshold(3))
                // Increase the size of group on 10%, if necessary to scale up
                .with_upscale_rate(0.1)
                // Decrease the size of group on 20%, if there are too many idle actors
                .with_downscale_rate(0.2)
        )
```

- **Cluster/distributed actors**

    - After some extensive testing of the Artillery protocol, we are super thrilled to show SWIM based epidemic cluster for Bastion. 
    (SWIM stands for Scalable Weakly-consistent Infection-style Membership, we know it’s a mouthful, read [this blog post](https://asafdav2.github.io/2017/swim-protocol/) if you would like an introduction).
    Here is an [example code for cluster execution](https://github.com/bastion-rs/showcase/tree/master/bastion-cluster).
    - [Here's a live example of a ping pong cluster](https://twitter.com/vertexclique/status/1273945945126973440).

- **Named child in children groups**
    - Right now you can get the names of the child in children groups. This is very handy when logging for example.
    - With this enabled everything is user readable content addressable in Bastion runtime.

- **Heartbeat** for children
    - The bastion runtime continuously watches children with heartbeat to check their status and do sampling over their internals.
    - This feature doesn’t require any feature flags.
    - It is shown in the scaling example seen above:
```rust    
    children
        // Do heartbeat each 5 seconds
        .with_heartbeat_tick(Duration::from_secs(5))
```

- **Dispatchers** allow you to route specific messages to specific groups and collect messages from them. 
    - This enables creating various fan-outs and fan-ins, and streaming data processing implementations.
    - Here's [how to use a round robin dispatcher](https://github.com/bastion-rs/bastion/blob/master/src/bastion/examples/round_robin_dispatcher.rs#L4).

- **Dead letters mailbox** so undelivered messages can be processed or monitored later on.

- **Lockfree runtime statistics** sampling for run queue offloading. 

- **Actor registry** support

- **Restart strategy for actors**:
    - We have written a restart strategy for actors, so they can be restarted with various backoff strategies like linear, exponential etc.
    
    Moreover, if panic or error occurs in your actors, you can define a max restart policy to give up after n attempts, or maybe [you're never going to give up ?](https://url.pet/r9GGm) Maybe you don’t want to restart at all:
```rust
    // At the beginning we're creating a new instance of RestartStrategy
    // we then provide a policy and a back-off strategy.
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
```

- **Adapted NUMA allocator** to latest Alloc API to make use of it easily with Bastion.


### What’s on our roadmap?
- **Bastion** 
    - Management subtrees of actors in runtime
    - Local and shared state API for actors
    - Link and Monitor implementations
    - Refactoring the msg! macro
- **Alcazar**
    - Implement an early prototype of web framework based on top of the existing Bastion codebase. The goal is to expose a simple web api to send / receive messages to a Bastion.


### Wanna join us?

We are working a lot towards letting Bastion become a prominent runtime, and help rustaceans build scalable and reliable distributed applications.

We are looking for maintainers who will:

- Simplify the code and guide the development
- Develop distributed operations
- Triage and help Prioritizing bugs/feature requests
- Develop new features or help with optimizations and refactoring
- Make documentation and infrastructure shine
- Join the fun and hang out with us!

If you are interested in these topics above and want to deep dive into Rust. This is a good start.
This project and other crates that will be developed around with this project will reside under [Bastion RS](https://github.com/bastion-rs/bastion).
Looking forward to hearing from future contributors!


### We need your support!

We would love to invest into better testing on different platforms and architectures, verification, and constant improvement in our infrastructure and nourish the Rust ecosystem with a new approach for applications.

All contributors are dedicating their free time and trying to get the most of it at a continuous pace. But good mood and a cheerful team only got us so far. We unfortunately don't have enough funding at the Bastion project and all projects under it at the moment. 

Since the beginning of the project we have collected 35$ total. This is why kindly ask you to consider supporting us on [https://opencollective.com/bastion](https://opencollective.com/bastion).

We are also considering other approaches like providing consulting and integration advice, and advanced support features that will allow us to get feedback and spend more time making Bastion the best framework for distributed systems out there. 

If you are currently using bastion in your company, or if you’re considering using it in production, please reach out to us!

If you (use Bastion and project ecosystem/considering using Bastion/want to help to project to get going), please consider donating over our OpenCollective. All these features mentioned above prepped in a long time, and we want to do more together with you!

If we start getting enough funding, we will begin writing posts about where the money goes and what we built with it!

If you have any questions or would like to have a chat with us, don't hesitate to reach out to the Bastion team on [Github](https://github.com/bastion-rs), [Twitter](https://twitter.com/bastion_rs), and [Discord](https://discord.gg/HvdUXX)!

