---
layout: post
title: "Bastion floating on Tide - Part 2"
author: Written by Jeremy o0Ignition0o Lempereur
---

## Part 2: Building a Bastion

### Introduction

This post is Part two of a series on how to build a highly fault tolerant system using bastion-rs, and expose an async web api with Tide.

- [Part one](/2020/02/23/bastion-floating-on-tide-part-1.html) is about setting up a simple Tide server, and covers initial steps such as adding dependencies, some basic routing and the request/reply model.

- In Part two we will first do a bit of refactoring, and then transform our function into a bastion child.

The complete example file is available [in this github gist](https://gist.github.com/o0Ignition0o/3b5dd124d9cfb6b3189e942548db79ad) and you can reach out to me anytime if you have a question.

### Before we dive into the bastion

I can't remember where I read this sentence. (If anyone knows please let me know so I can give due credit) But it goes something like this:

`Software architecture is like sewing, you want to intertwine threads two by two at most, otherwise you ll end up with a ball of knots.`

This is a principle I tend to follow as much as possible, which allows me to read the code I wrote a long time ago. So before we dive into setting up a bastion, let's refactor the code we wrote [before](/2020/02/23/bastion-floating-on-tide-part-1.html), so we can only expose one function, that will serve prime numbers.

```rust
mod prime_number {
        pub fn get(num_digits: usize) -> u128 {
            get_prime(num_digits)
        }
        // the previous code
}
```

I put it in a separate module here, but It would probably be in an other file if it wasn't intended to live in a github gist. 
As you can see, we just wrapped all of the logic [from our previous code](https://gist.github.com/o0Ignition0o/22c86b382166ba6ce1669a9e4dd37062) in a module, and exposed only one `prime_number::get()`. We don't expose the other functions anymore (number_or_panic, is_prime etc.), except our Tide handler.

The tide request handler will now call `prime_number::get()`:

```rust
async fn prime_number(req: tide::Request<()>) -> String {
    use std::time::Instant;
    let d: usize = req.param("digits").unwrap_or(1);
    // Start a stopwatch
    let start = Instant::now();
    // Get a prime number
    let prime_number = prime_number::get(d);
    // Stop the stopwatch
    let elapsed = Instant::now().duration_since(start).as_secs();
    format!(
        "{} is a prime number with {} digits.\nIt was computed in {} seconds.\n",
        prime_number, d, elapsed
    )
}
```

I think we could do better with the name of this function, but let's keep it that way for now.

We now have a single entry point to receive prime_numbers, and our example is still working fine. Time to build a bastion!

## Let's wrap our business logic in a bastion child

Now there's a lot of features we can use in bastion, so we'll keep it as simple as possible, and only use what we need. There's now a couple of examples in the [bastion examples folder](https://github.com/bastion-rs/bastion/tree/master/src/bastion/examples) as well as a couple of cool [showcases](https://github.com/bastion-rs/showcase) if you would like do dig deeper.

If you have any question or just wanna hang out with us, please [join our discord](https://discord.gg/gVpJt7) and let us know what you're working on! :D

### Step one, our response structure

We are now going to wrap our prime number generator in a bastion child, that will receive requests for Prime numbers, and reply with the generated prime number, and a couple of informations as well.

We will first define a Response struct that will allow prime_number::get() to give us the prime number we asked for, as well as how long it took it to find it.
An impl Display for Response will allow us to simplify the tide request handler even further:

```rust
mod prime_number {
    use std::time::{Duration, Instant};

    #[derive(Debug)]
    pub struct Response {
        prime_number: u128,
        num_digits: usize,
        compute_time: Duration,
    }

    impl std::fmt::Display for Response {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(
                f,
                "{} is a prime number that has {} digits.\nIt was found in {}s and {}ms",
                self.prime_number,
                self.num_digits,
                self.compute_time.as_secs(),
                self.compute_time.as_millis() % 1000
            )
        }
    }

    // our prime number getter now returns a Response
    pub fn get(num_digits: usize) -> Response {
        let start = Instant::now();
        // Get a prime number
        let prime_number = get_prime(num_digits);
        // Stop the stopwatch
        let elapsed = Instant::now().duration_since(start);

        Response {
            prime_number,
            num_digits,
            compute_time: elapsed,
        }
    }
}
```

We `#[derive(Debug)]` here because currently, in Bastion, structures that serve to communicate need to implement the `Debug` trait. This may however change in the future.

Our request handler now looks like this:

```rust
async fn prime_number(req: tide::Request<()>) -> String {
    let d: usize = req.param("digits").unwrap_or(1);
    let prime_number_response = prime_number::get(d);
    format!(
        "{}\n",
        prime_number_response
    )
}
```

### Step two, create a message processor

A bastion child is an autonomous being, that will wait for messages and handle them if it can. We will now build an async function that loops for ever, and processes ouf prime_numbers requests.

```rust
// `serve_prime_numbers` is the bastion child's behavior.
async fn serve_prime_numbers(ctx: BastionContext) -> Result<(), ()> {
    // let's put the context in an arc, so we can pass it to other threads
    let arc_ctx = std::sync::Arc::new(ctx);
    // a child will keep processing messages until it crashes
    // (or until it gets told to shutdown)
    loop {
        // msg! is our message receiver helper.
        // we will only use one variant here
        // =!> means messages that can be replied to
        // usize means it will only match against messages that are a usize
        msg! { arc_ctx.clone().recv().await?,
            nb_digits: usize =!> {
                // clone the context to send it to a thread
                let ctx2 = arc_ctx.clone();
                // answer! takes a context and will automagically figure out whom to reply to
                blocking!(answer!(ctx2, prime_number::prime_number(nb_digits)).expect("couldn't reply :("));
            };
            // this is a catch all for any other message we might receive
            unknown:_ => {
                println!("uh oh, I received a message I didn't understand\n {:?}", unknown);
            };
        }
    }
}
```

Now there's a lot of information here, so let's walk trough it together:

#### BastionContext

A child processor is an async function that takes a `BastionContext` as a parameter, and returns a Result<(),()>. In our case it will hopefully never return, because we are running it in a `loop{}`.

Refer to the [BastionContext documentation](https://docs.rs/bastion/0.3.5-alpha/bastion/context/struct.BastionContext.html) if you would like to dig deeper.


#### msg!

The `msg!` macro is where most of the magic is happening. We are not using all of the features available in this match statement on steroids, but [the fibonacci example](https://github.com/bastion-rs/bastion/blob/master/src/bastion/examples/fibonacci.rs#L31) dives deeper on the topic.

Refer to the [msg! documentation](https://docs.rs/bastion/0.3.5-alpha/bastion/macro.msg.html#example) if you would like to dig deeper.

It used to be one of my biggest struggles as I started using Bastion, so we are trying to find ways to express this behavior withoug macros. If you have ideas you would like to share, please let me know in the [corresponding issue](https://github.com/bastion-rs/bastion/issues/214).


#### answer!

The `answer!` macro does a bit of magic. It uses the BastionContext to deduce who to reply to, and sends the reply.

Refer to the [answer! documentation](https://docs.rs/bastion/0.3.5-alpha/bastion/macro.answer.html#example) if you would like to dig deeper.

#### blocking!

Let's talk about my favorite macros here, the "runners". There are two preferred ways to run an async function on bastion:

[blocking!](https://docs.rs/bastion/0.3.5-alpha/bastion/macro.blocking.html) puts a task in our blocking threadpool, which focuses on CPU bound tasks.

[spawn!](https://docs.rs/bastion/0.3.5-alpha/bastion/macro.spawn.html) does the same, but tasks aren't put in a separate threadpool, so you want to use this for I/O bound tasks.

[run!](https://docs.rs/bastion/0.3.5-alpha/bastion/macro.run.html) allows you to wait until a task is complete.

If you're not familiar with CPU bound and I/O bound, [this blog post is an excellent resource](https://www.hellsoft.se/understanding-cpu-and-i-o-bound-for-asynchronous-operations/) to get started and to figure out what kind of tasks are CPU or I/O bound. Bonus points for one of my favourite illustrations on multi-threaded code.

Now that we have our runner, let's spawn a child!

### Step three, create a child

Let's now head bback to our main function, and build our bastion:

```rust

fn main() {

    // We need a bastion in order to run everything
    Bastion::init();
    Bastion::start();

    // Spawn 1 child that will serve prime numbers
    let prime_number_children =
        Bastion::children(|children| children.with_redundancy(1).with_exec(serve_prime_numbers))
            .expect("couldn't create children");

    let nb_digits = 6;

    // Ask our child for a prime number
    let reply = prime_number_children.elems().first().expect("no child?")
        .ask_anonymously(nb_digits)
        .expect("couldn't ask for a prime number");
    
    Bastion::stop();
    Bastion::block_until_stopped();
}
```

A bastion needs to be initialized and started before it can do anything.
stop() and block_until_stopped() allow the bastion to gracefully shut down.

The [bastion struct documentation](https://docs.rs/bastion/0.3.5-alpha/bastion/struct.Bastion.html) is very exhaustive and will allow you to dig deeper.

Bastion::children allows us to set up redundancy (1 for now), and the message processing function (`serve_prime_numbers` in our case)

We can then pick our first (and only) whild and `ask_anonymously` for a prime number.

Time for our last explanation before we can tie bastion and tide (this one sounds funny doesn't it?).

#### Ask/Tell anonymously or not ?

Messages can either expect a reply or not.

`ask()` and `ask_anonymously()` will allow a child to use the `answer!` macro we saw earlier.

`tell()` and `tell_anonymously()` won't.

The `anonymously` variants are used from "outside the bastion". They add a mechanism that will allow to identify message senders even if they aren't other children, which could come in handy when say... you want to `answer!` to a tide request handler :) 

[The ChildRef documentation](https://docs.rs/bastion/0.3.5-alpha/bastion/child_ref/struct.ChildRef.html) shows examples on how to use each function, if you would like to dig deeper.

## Let's give it a run

At that point, we should be able to run it, so let's try it out:

```bash
$ cargo run 
    Finished dev [unoptimized + debuginfo] target(s) in 0.11s
     Running `target/debug/floating_on_tide_2`
$ # nothing happened
```

Wait what? how comes nothing happened? Hmmm it turns out something happened, but we didn't catch it! We just sent a request, and we didn't await a reply. let's do it now

### Writing our request/reply function

Our request/reply function will:
- make a request
- wait for a reply
- return the reply

In order to to this, we need to have a handle to our bastion children, so we can `ask_anonymously` for prime numbers. Let's write this structure:

```rust
// Our PrimeClient will keep a children_handle
// to be able to send messages to the bastion
struct PrimeClient {
    children_handle: ChildrenRef,
}

impl PrimeClient {
    pub fn new(children_handle: ChildrenRef) -> Self {
        Self { children_handle }
    }

    // request_prime will allow us to perform a request,
    // and reply when we receive an answer
    pub async fn request_prime(&self, nb_digits: usize) -> Result<prime_number::Response, ()> {
        // Ask our child for a prime number
        let reply = self
            .children_handle
            .elems()
            .first()
            .expect("no child?")
            .ask_anonymously(nb_digits)
            .expect("couldn't ask for a prime number");

        // wait for the reply
        msg! { reply.await?,
            response: prime_number::Response => {
                // This is a prime number Response, let's return it
                Ok(response)
            };
            unknown:_ => {
                // Something wrong happened
                println!("uh oh, I received a message I didn't understand\n {:?}", unknown);
                Err(())
            };
        }
    }
}
```

Ok we now have a structure that will allow us to perform a request, and hopefully get a reply, let's edit our main function so it uses our new structure.

```rust
fn main() {
    // We need a bastion in order to run everything
    Bastion::init();
    Bastion::start();

    // Spawn 1 child that will serve prime numbers
    let prime_number_children =
        Bastion::children(|children| children.with_redundancy(1).with_exec(serve_prime_numbers))
            .expect("couldn't create children");

    // Create a prime client...
    let prime_client = PrimeClient::new(prime_number_children);
    let nb_digits = 6;

    use bastion::run;
    let response = run!(async { prime_client.request_prime(nb_digits).await })
        .expect("couldn't perform request");

    println!("{}", response);

    Bastion::stop();
    Bastion::block_until_stopped();
}
```

Let's see how it goes...

```bash
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 1.59s
     Running `target/debug/floating_on_tide_2`
483991 is a prime number that has 6 digits.
It was found in 0s and 0ms
```

Yay \o/ it works, it's now time to plug it to our tide server!

## Tide and Bastion

As an aside, I _really_ want to let you know how amazing [tide's documentation](https://docs.rs/tide/0.11.0/tide/) is. It was a pleasure to briefly skim the doc, and find exactly what it was looking for, with super nice examples, and no bloat around it. Kudos to the team, you're doing an amazing job!

Tide has two ways to create an app, [new()](https://docs.rs/tide/0.11.0/tide/fn.new.html) and [with_state()](https://docs.rs/tide/0.11.0/tide/fn.with_state.html). We are now going to change our new() to a with_state() call so we can pass our prime_client to the tide app:

```rust
// Create a prime client...
let prime_client = PrimeClient::new(children_handle);

task::block_on(async move {
    // ...That will populate our tide state
    let mut app = tide::with_state(prime_client);
    app.at("/prime/:digits").get(prime_number);
app.listen("127.0.0.1:8080").await
})?;
```

In order to receive this state, we will update our prime_number async fn so it can use it:

```rust
// tide::Request is now parameterized over PrimeClient, which is our state
async fn prime_number(req: tide::Request<PrimeClient>) -> Result<String, tide::Error> {
    let d: usize = req.param("digits").unwrap_or(1);

    // Use the PrimeClient to ask for a prime number
    // The state is available under req.state()
    let response = req.state().request_prime(d).await.map_err(|_| {
        tide::Error::from_str(
            tide::StatusCode::InternalServerError,
            "I'm sorry, I couldn't get a prime number",
        )
    })?;

    Ok(format!("{}\n", response))
}
```

## Let's give it a run!

```bash
$ cargo run               
   Compiling floating_on_tide_2 v0.1.0 (/Users/ignition/Projects/oss/bastion/floating_on_tide_2)
    Finished dev [unoptimized + debuginfo] target(s) in 3.37s
     Running `target/debug/floating_on_tide_2`
```

We can now spawn an other shell and start performing requests:

```bash
$ curl http://127.0.0.1:8080/prime/15
623361288611851 is a prime number that has 15 digits.
It was found in 0s and 350ms

$ curl http://127.0.0.1:8080/prime/15
947499788349449 is a prime number that has 15 digits.
It was found in 0s and 410ms

$ curl http://127.0.0.1:8080/prime/15
510096850144577 is a prime number that has 15 digits.
It was found in 0s and 327ms

$ curl http://127.0.0.1:8080/prime/15
826290052329481 is a prime number that has 15 digits.
It was found in 0s and 427ms
```

Congratulations! We have successfully plugged Bastion to our tide server!

### Conclusion

What do you mean by conclusion? Where's the Supervisor? What is a dispatch mechanism? How can we add fault tolerance to our system? Our system still isn't panic! proof yet is it?

You're correct, we needed to do all of this work so we can learn about all of these amazing things in part 3 of this series, which will probably be the last one!

Again, I hope you enjoyed reading this post as much as I enjoyed writing it, and I can't wait to share part three with you!

The complete example file is available [in this Github gist](https://gist.github.com/o0Ignition0o/3b5dd124d9cfb6b3189e942548db79ad) and you can reach out to me anytime if you have a question.

I you have any questions or would like to have a chat with us, don't hesitate to reach out to the Bastion team on [Github](https://github.com/bastion-rs), [Twitter](https://twitter.com/bastion_rs), and [Discord](https://discord.gg/HvdUXX)!

