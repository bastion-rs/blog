---
layout: post
title: "Bastion floating on Tide - Part 1"
author: Written by Jeremy o0Ignition0o Lempereur
---

## Part 1: Building a Tide server

### Introduction

This post is Part one of a series on how to build a highly fault tolerant system using bastion-rs, and expose an async web api with Tide.

- Part one is about setting up a simple Tide server, and covers initial steps such as adding dependencies, some basic routing and the request/reply model.

The complete example file is available [in this Github gist](https://gist.github.com/o0Ignition0o/22c86b382166ba6ce1669a9e4dd37062) and you can reach out to me anytime if you have a question.

### Creating a simple Tide project

If you haven’t yet, I highly recommend you to install [cargo-edit](https://github.com/killercup/cargo-edit), a tool that will allow you to add dependencies to a project without the need to edit Cargo.toml manually (among other really cool features). If you don’t want to install it, you need to skip the `cargo add` statements in the example, and edit Cargo.toml to add the relevant crates as we move along.

```bash
$ cargo install cargo-edit
```

Let’s start with a Tide example:

```bash
$ cargo new bastion_with_tide # Create a new project
$ cd bastion_with_tide # Go to the project folder
$ cargo add async-std # Add async-std
$ cargo add tide # Add tide
```

Now that we created a project and added the required dependencies, let's edit the `src/main.rs` file:

```rust
fn main() -> Result<(), std::io::Error> {
    use async_std::task;
    task::block_on(async {
        let mut app = tide::new();
        app.at("/").get(|_| async move { "Hello, world!" });
        app.listen("127.0.0.1:8080").await?;
        Ok(())
    })
}
```
And run the application:

```bash
cargo run
[…]
   Compiling bastion_with_tide v0.1.0 (/Users/ignition/Projects/oss/bastion_with_tide)
    Finished dev [unoptimized + debuginfo] target(s) in 35.37s
     Running `target/debug/bastion_with_tide`
```

Now let’s open a browser and go to [http://127.0.0.1:8080](http://127.0.0.1:8080)

![Tide hello world!](https://raw.githubusercontent.com/bastion-rs/blog/master/images/tide-hello-world.png){: .center }

Ok we now have a Tide web server up and running, but we’re not doing too much for now. Let’s try to build an application that will do some heavy computing behind the scenes.

### Let's use some CPU cycles!

<s>For the purpose of this example, we will create a fibonacci adder.</s>

Florian skade Gilcher suggested an amazing example, that some of you might have read about already.
For the purpose of this example, we will create an amazing service for delivering prime numbers to the internet!

If this exercise doesn't ring a bell, don't worry, I didn't know about it either, and I had an amazing journey discovering "Programming Erlang", written by [Joe Armstrong](https://en.wikipedia.org/wiki/Joe_Armstrong_(programmer)).

If you were interested simple examples on how to use Tide, head over to [the Tide examples](https://github.com/http-rs/tide/tree/master/examples). If you would like to know how a Fibonacci example would look like, I have submitted [a pull request](https://github.com/http-rs/tide/pull/406) to add it to the Tide examples.

Our prime number service will, given a number of digits, return a prime number with the expected number of digits.

In pseudocode it would look like this:

```
GET: /prime/:digits => return prime_number(digits)
```

First of all, we want to extract some data from the route we create. We can leverage the power of Tide and get named parameters by specifying a route and a handler function:
```rust
app.at("/prime/:digits").get(prime_number);
```

The Tide router will extract `d` as a parameter, which will be queryable in our `prime_number` function:

```rust
async fn prime_number(req: tide::Request<()>) -> String {
    let d: usize = req.param("digits").unwrap_or(1);
    format!("you asked me to compute a prime number that has {} digits.\n", d)
}
```

Go for a `cargo run` and give it a try with different numbers:
```bash
➜ curl "127.0.0.1:8080/prime/10"
you asked me to compute a prime number that has 10 digits.
```

Let’s now do some heavy computation!

Well maybe not… We're going to create a couple of functions that will allow us to compute a prime number with the correct width.
Long story short we will iterate over numbers with the correct number of digits starting from a random one, and use [a test to find out if the random number is a prime number](https://en.wikipedia.org/wiki/Primality_test#Pseudocode). It will look like this:

```rust
fn get_prime(num_digits: usize) -> u128 {
    let min_bound = get_min_bound(num_digits);
    // with num_digits = 4, max_bound == 10000
    let max_bound = get_max_bound(num_digits);
    // maybe_prime is a number in range [1000, 10000)
    // the closing parentheses means it won't reach the number.
    // the maximum allowed value for maybe_prime is 9999.
    use rand::Rng;
    let mut maybe_prime = rand::thread_rng().gen_range(min_bound, max_bound);
    loop {
        if is_prime(maybe_prime) {
            return maybe_prime;
        }
        // for any integer n > 3,
        // there always exists at least one prime number p
        // with n < p < 2n - 2
        maybe_prime += 1;
        // We don't want to return a number
        // that doesn't have the right number of digits
        if maybe_prime == max_bound {
            maybe_prime = min_bound;
        }
    }
}
```

If you want to follow along and be able to do cargo runs, here's what you want to do:

Add a dependency to the rand crate, so we can generate random numbers:
```
$ cargo add rand
```

In `src/main.rs`, we want to add a missing use statement:

```rust
use std::iter;
```
Here's the contents of the missing functions:
```rust
// in order to determine if n is prime
// we will use a primality test.
// https://en.wikipedia.org/wiki/Primality_test#Pseudocode
fn is_prime(n: u128) -> bool {
    if n <= 3 {
        n > 1
    } else if n % 2 == 0 || n % 3 == 0 {
        false
    } else {
        for i in (5..=(n as f64).sqrt() as u128).step_by(6) {
            if n % i == 0 || n % (i + 2) == 0 {
                return false;
            }
        }
        true
    }
}

// given a sequence of digits, return the corresponding number
// eg: assert_eq!(1234, digits_to_number(vec![1,2,3,4]))
fn digits_to_number(iter: impl Iterator<Item = usize>) -> u128 {
    iter.fold(0, |acc, b| acc * 10 + b as u128)
}

fn get_min_bound(num_digits: usize) -> u128 {
    let lower_bound_iter =
        iter::once(1usize).chain(iter::repeat(0usize).take(num_digits - 1 as usize));
    digits_to_number(lower_bound_iter)
}

fn get_max_bound(num_digits: usize) -> u128 {
    let lower_bound_iter = iter::once(1usize).chain(iter::repeat(0usize).take(num_digits));
    digits_to_number(lower_bound_iter)
}
```


I won't go too much into the implementation details here, but the whole example is available [in this Github gist](https://gist.github.com/o0Ignition0o/22c86b382166ba6ce1669a9e4dd37062) and you can reach out to me anytime if you have a question.

If you would like to know more about the theory behind prime numbers generation, I will instead urge you to dig more into Joe Armstrong's book, and have a look at [Bertrand's postulate](https://en.wikipedia.org/wiki/Bertrand%27s_postulate), and [Fermat primality test](https://en.wikipedia.org/wiki/Fermat_primality_test). If you (like me) are still amazed by what you just learnt about prime numbers, have a look at a special subset of prime numbers, called [Mersenne prime numbers](https://en.wikipedia.org/wiki/Mersenne_prime).

Ok enough with this rabbit hole, let's go for a cargo run.

We need to edit our `get_prime` function get a prime number, and return it among a couple of information on how long it took us to compute it:

```rust
async fn prime_number(req: tide::Request<()>) -> String {
    use std::time::Instant;
    let d: usize = req.param("digits").unwrap_or(1);
    // Start a stopwatch
    let start = Instant::now();
    // Get a prime number
    let prime_number = get_prime(d);
    // Stop the stopwatch
    let elapsed = Instant::now().duration_since(start).as_secs();
    format!(
        "{} is a prime number with {} digits.\nIt was computed in {} seconds.\n",
        prime_number, d, elapsed
    )
}
```

Go ahead and give it a run, and notice how the computation time grows depending on the number of digits (and luck, mostly).

For example computing this sequence on my machine for `d = 15` took me less than a second. It took me 9 seconds for `d = 18`, and 237 seconds for `d = 20` !

```bash
➜ curl "http://127.0.0.1:8080/prime/15"
729994968290557 is a prime number with 15 digits.
It was computed in 0 seconds.
➜ curl "http://127.0.0.1:8080/prime/16"
7717274638523017 is a prime number with 16 digits.
It was computed in 1 seconds.
➜ curl "http://127.0.0.1:8080/prime/17"
76559666666708933 is a prime number with 17 digits.
It was computed in 3 seconds.
➜ curl "http://127.0.0.1:8080/prime/18"
654693728093206181 is a prime number with 18 digits.
It was computed in 9 seconds.
➜ curl "http://127.0.0.1:8080/prime/19"
1832975536808651843 is a prime number with 19 digits.
It was computed in 17 seconds.
➜ curl "http://127.0.0.1:8080/prime/20"
34003387272791828929 is a prime number with 20 digits.
It was computed in 237 seconds.
```

Depending on your computer, the duration might vary a little bit. Find a sweet spot you’re comfortable with, we will keep using it for the rest of the tutorial.



### Add a little twist

The more complex a system becomes, the more likely it is to fail. In order to introduce Bastion and a couple of its basic features, we will introduce a failure mode that will happen randomly in our example, and see how Bastion can help us reason about failures and complex systems. But first, a disclaimer.

### Disclaimer

THIS EXAMPLE COMES AS IS... Yeah no that's not what I meant by a disclaimer.

async-rs and the async-std library allow you to achieve a lot, and by a lot I really mean A LOT. The twist I'm about to introduce (leading to a panic) is `not` idiomatic at all, and you're much better off leveraging Result, Option and other awesome Rust structures that were built to do this. The purpose of this example is not to make you think Bastion can solve this problem and async-rs cannot. async-rs can totally handle failures (and generating prime numbers for that matter) in a graceful way.

I would also like to let you know that the async-rs team has been doing an amazing job, and that they are an amazing team, always friendly, super helpful, and they helped me a lot while I was preparing the examples and this blog post.

When it comes to choosing a runtime, I encourage you to first define your goals and what matters the most, and then pick the best tool for the job. Sometimes an actor based framework is not necessary because your system will never reach a high level of complexity. Sometimes you should rather start with a very simple web server, and let the tools evolve as your system grows.

As [John Gall ](https://en.wikipedia.org/wiki/John_Gall_(author)#Gall's_law) states it:

```
A complex system that works is invariably found
to have evolved from a simple system that worked.
A complex system designed from scratch never works
and cannot be patched up to make it work.
You have to start over with a working simple system.
```

As a more general thought, when it comes to libraries and frameworks focusing on topics in the same area, I really believe having different approaches and being helpful to each other will improve the quality of each framework, the ecosystem, and the community as a whole.

With that in mind, let's introduce a bit of chaos to our server!

### Chaos !

We will create a `number_or_panic` function that will well... either return the prime number or panic :)
```rust
fn number_or_panic(number_to_return: u128) -> u128 {
    // Let's roll a dice
    if rand::random::<u8>() % 6 == 1 {
        panic!(format!(
            "I was about to return {} but I chose to panic instead!",
            number_to_return
        ))
    };
    number_to_return
}
```

We can now edit our `get_prime` function to sometimes panic, and watch the server fail:

```rust
loop {
    if is_prime(maybe_prime) {
        return number_or_panic(maybe_prime);
    }
    // [...]
}
```

Let's take our example for a cargo run and watch it fail after a couple of calls:

```bash
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/bastion_with_tide`
thread 'async-std/executor' panicked at 'I was about to return 2940675629 but I chose to panic instead!', src/
main.rs:41:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
[1]    26676 abort      cargo run
```

And here's what happened on the client's side:

```bash
$ curl "http://127.0.0.1:8080/prime/10"
2864551423 is a prime number with 10 digits.
It was computed in 0 seconds.
$ curl "http://127.0.0.1:8080/prime/10"
3718438259 is a prime number with 10 digits.
It was computed in 0 seconds.
$ curl "http://127.0.0.1:8080/prime/10"
8959500691 is a prime number with 10 digits.
It was computed in 0 seconds.
$ curl "http://127.0.0.1:8080/prime/10"
curl: (52) Empty reply from server
$ curl "http://127.0.0.1:8080/prime/10"
curl: (7) Failed to connect to 127.0.0.1 port 8080: Connection refused
```

A couple of prime numbers were generated, then the server sent an empty reply. What's even more unfortunate is that once an empty reply was returned, further connections are simply refused. In hindsight it makes sense. Because the panic caused the server to crash. It can't serve requests anymore.

### Conclusion

As said in the disclaimer, there are several ways to recover from this, and in part two of this series, we will explore one way to escape this prime numbers russian roulette. We will add fault tolerance, and dive a bit into the actor model by building a Bastion on Tide!

I hope you enjoyed reading this blog post as much as I enjoyed writing it, and I can't wait to share part two with you!

The complete example file is available [in this Github gist](https://gist.github.com/o0Ignition0o/22c86b382166ba6ce1669a9e4dd37062) and you can reach out to me anytime if you have a question.

I you have any questions or would like to have a chat with us, don't hesitate to reach out to the Bastion team on [Github](https://github.com/bastion-rs), [Twitter](https://twitter.com/bastion_rs), and [Discord](https://discord.gg/HvdUXX)!
