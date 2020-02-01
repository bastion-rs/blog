---
layout: post
title: "Writing AWS Lambda Functions with Bastion"
author: Written by Mahmut Bulut
---

![Bastion and AWS Lambdas](https://github.com/bastion-rs/blog/blob/master/images/bastion_and_lambda.png)

In this post, we will look how AWS Lambda Functions can be implemented using Bastion. AWS Lambda functions are well known serverless platform used for tasks varying from infrastructure management, serving APIs, data processing to orchestration tasks.

Let's start! We are going to use the [Bastion AWS Lambda Example](https://github.com/bastion-rs/showcase/tree/master/bastion-aws-lambda) in our showcase repo.

First let's clone the Bastion showcase repository:
```bash
$ git clone git@github.com:bastion-rs/showcase.git
```

Get into the containing directory of our example lambda declaration:
```bash
$ cd showcase-zero/bastion-aws-lambda
```

We are using `serverless` utility to leverage compilation and configuration of our lambda. In this directory you will see how our lambda is configured by `serverless.yml`. For installing `serverless` and its dependencies for Rust environment we will do:
```bash
$ npm i -g serverless
$ npm i -D serverless-rust
```

Now we have installed both dependencies for compiling and configuring our lambda. Our lambda is taking a list of sites as input and concurrently make GET requests. And returns their body strings whenever they are available. Main code for lambda is in `page_fetcher/src/main.rs`. Let's take a look at the Lambda code and walk through it:

```rust
/// This is the JSON payload we expect to be passed to us by the client accessing our lambda.
#[derive(Deserialize, Debug)]
struct InputPayload {
    sites: Vec<String>
}
```

This is our input payload that contains sites that will be requested when lambda is triggered.

```rust
/// This is the JSON payload we will return back to the client if the request was successful.
#[derive(Serialize, Debug)]
struct OutputPayload {
    status: String
}
```

When we process sites and receive body of the sites we are going to return the lambda status. Think that these are the cluster internal applications which we concurrently trigger their endpoints. Instead of sites you can use your cluster internal naming.

-------

### Message Handler (Dispatcher)

```rust
fn dispatcher(
    payload: InputPayload,
    _c: Context,
) -> Result<OutputPayload, HandlerError> {
    let (p, mut c) = unbounded::<bool>();

    Bastion::children(|children: Children| {
        children.with_exec(move |_ctx: BastionContext| {
            let sites = payload.sites.clone();
            let workers = worker_pool(payload.sites.len());
            let p = p.clone();

            async move {
                info!("Dispatching started");

                for (worker, site) in workers.elems().iter().zip(sites) {
                    info!("Site sent for processing!");
                    let answer = worker.ask_anonymously(site).unwrap();
                    // Or use the returned body
                    let _ = answer.await.unwrap();
                }

                let _ = p.unbounded_send(true);

                Ok(())
            }
        })
    }).unwrap();

    // Wait for completion signal OR data itself
    while let Err(_) = c.try_next() {}

    Ok(OutputPayload { status: "OK".into() })
}
```

Let's break this dispatcher down. Lambda runtime unfortunately doesn't allow us to write direct runtimeless lambda application so we need to write a `dispatcher` that will be registered to underlying runtime. So this is our dispatcher that will use the underlying runtime to dispatch requests to Bastion for processing.

We created a unbounded channel for receiving completion that we made requests to all given sites and received a response from each other:

```rust
let (p, mut c) = unbounded::<bool>();
```

We are creating a children group to setup workers and fan out requests to workers. So we will make our workers run individually:
```rust
let workers = worker_pool(payload.sites.len());
```

Above line sets up workers using `worker_pool` method with given amount of sites.

```rust
async move {
    info!("Dispatching started");

    for (worker, site) in workers.elems().iter().zip(sites) {
        info!("Site sent for processing!");
        let answer = worker.ask_anonymously(site).unwrap();
        // Or use the returned body
        let _ = answer.await.unwrap();
    }

    let _ = p.unbounded_send(true);

    Ok(())
}
```

We link each worker and the site. Using `ask` we are sending site to the workers. Here we need to know the difference between `ask` and `tell`. `tell` method works like a `fire and forget` messaging system. Message passed with tell won't expect response from receiver actor. In Bastion we have `ask` and `ask_anonymously`. Latter is usable across `BastionContext`'s. In Bastion `context` is a hierarchy branch management which allows granular microconcurrency between children, child and supervisors. `ask` is usable within the same context which allows us to directly identify children inside the same hierarchy. We are using `ask_anonymously` in here because `workers` and `dispatcher` are separate branches.

Our workers are sending back the body. Even though we are not using it we wait for the response from all the workers. If we don't wait inside the for loop order will be different across responses. In Bastion order of messages are belong to user's code.

After these explanation for the `dispatcher`. Let's take a look at the workers:

-------

### Worker Actors

```rust
fn worker_pool(pool_size: usize) -> ChildrenRef {
    Bastion::children(|children: Children| {
        children
            .with_redundancy(pool_size)
            .with_exec(move |ctx: BastionContext| {
                async move {
                    info!("Worker started!");

                    // Start receiving work
                    loop {
                        msg! { ctx.recv().await?,
                            site: String =!> {
                                info!("Received site: {}!", site.clone());
                                let body = surf::get(site.clone()).recv_string().await.unwrap();
                                warn!("Site: {} Body: {}", site, body);
                                let _ = answer!(ctx, body);
                            };
                            _: _ => ();
                        }
                    }
                }
            })
    })
    .expect("Couldn't start a new children group.")
}
```

We are passing the given `pool_size` as redundancy:

```rust
children
    .with_redundancy(pool_size)
```

Then we are giving the actor body:
```rust
.with_exec(move |ctx: BastionContext| {
    async move {
        info!("Worker started!");

        // Start receiving work
        loop {
            msg! { ctx.recv().await?,
                site: String =!> {
                    info!("Received site: {}!", site.clone());
                    let body = surf::get(site.clone()).recv_string().await.unwrap();
                    warn!("Site: {} Body: {}", site, body);
                    let _ = answer!(ctx, body);
                };
                _: _ => ();
            }
        }
    }
})
```

In this body you might realize that we receive messages with **[msg!](https://docs.rs/bastion/0.3.4/bastion/macro.msg.html)** macro. And it has an interesting syntax. While designing Bastion we've thought about `tell`, `ask` and `broadcast` scenarios between actors. `=!>` corresponds to asked messages. So as you know we've asked our workers to fetch a site. We are receiving this match from `ask` arm of our mailbox.

Oh yes, we are using [surf](https://docs.rs/surf/1.0.3/surf/) to issue our call. and returning the response with our `answer!` macro.

-------

### Application's Main

These were the methods that were doing the processing. It's time to take a look at the application protected against crashes. Our main entry:

```rust
#[fort::root]
async fn main(_: BastionContext) -> Result<(), ()> {
    let _ = simple_logger::init_with_level(log::Level::Info);
    lambda!(dispatcher);

    Ok(())
}
```

**[Fort](https://github.com/bastion-rs/fort#fort)** is a proc macro attribute that supplies fault tolerance to main method and wraps your main method into Bastion. It gives you root context of the Bastion system as argument for further hierarchy and let you run your application in panic handler context. Rest is basically initializing our logging system, and lambda handler registration.

That's all we have inside the code. For more information take a look at to our **[Documentation](https://docs.rs/bastion/0.3.4/bastion/struct.Bastion.html)**.

### Testing locally

For testing locally what we need to do is:
```bash
$ serverless invoke local -f page_fetcher -d \
    '{
      "sites": [
        "https://bastion.rs",
        "https://blog.bastion.rs",
        "http://google.com",
        "https://docs.rs/",
        "https://crates.io/",
        "https://twitter.com/",
        "https://news.ycombinator.com/",
        "http://play.rust-lang.org/",
        "http://catb.org/jargon/html/hates.html"
      ]
    }'
```

Then wait for container build finish, artefact extracted, and run inside the container. Then hopefully you will see the lambda logs.

### Conclusion

In this post, we introduced Bastion to AWS Lambda runtime. For your own applications feel free to explore more advanced methods, use message passing with different approaches mentioned above, design your own hierarchy and concurrency nodes. Don't forget to share your suggestions and ideas with us. Join our [Discord server](https://discord.gg/DqRqtRT), open an [issue/feature request/support ticket](https://github.com/bastion-rs/bastion/issues/new/choose) in our GitHub.
