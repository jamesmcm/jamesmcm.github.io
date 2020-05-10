---
layout: post
title: "A practical introduction to async programming in Rust"
titleen: "A practical introduction to async programming in Rust"
titlesp: "A practical introduction to async programming in Rust"
date: 2020-05-06 23:40:04 +0200
comments: true
categories: [Rust]
meta: false
---

In this post we will explore a brief example of asynchronous programming
in Rust with the Tokio runtime, demonstrating different execution scenarios. This post is aimed
at beginners to asynchronous programming.

The source code for this example is [available on Github](https://github.com/jamesmcm/async-rust-example).
A [branch](https://github.com/jamesmcm/async-rust-example/tree/async-std) using the async-std runtime is also available (contributed by
[@BartMassey](https://github.com/BartMassey)).

<!--more-->

## What is asynchronous programming?

Asynchronous programming allows you to continue carrying out computations
whilst waiting for results from IO operations
(often network requests or responses), even on a single OS thread.

This is achieved via the use of an async runtime which handles the
assignment of async tasks (i.e. [green threads](https://en.wikipedia.org/wiki/Green_threads))
to actual OS threads.

Unlike OS threads, green threads are not expensive to create and so we
needn't worry about hitting a hard limit. Whereas OS threads need to
hold their own stack, leading to high memory usage when dealing with
many threads. On Linux you can check your thread limit per process
with: `cat /proc/sys/kernel/threads-max`, mine is 127,162.


This is a significant issue if we need a separate OS
thread to handle each request on a web server for example, and is the
origin of the [C10k problem](https://en.wikipedia.org/wiki/C10k_problem)
\- how to handle 10,000 concurrent connections to a web server. 

Early web servers did indeed use a separate OS thread per request, in order to
handle requests in parallel. The key insight is that those threads spend
most of their time waiting for network responses, rather than doing any
computation.

### Async and await

Rust has adopted the
[async/await](https://en.wikipedia.org/wiki/Async/await) syntax for
defining asynchronous code blocks and functions.

The `async` keyword defines an async code block or function. Specifying
that it will return a `Future`, a value that will need to be `.await`ed
elsewhere in order to trigger the execution of the task (note the lazy
execution) and wait for the return value to be available.

The `.await` keyword (which must be inside an `async` block/function)
is used to wait asynchronously for an async task to finish, and get the return value.
Note that while the task itself cannot progress until the `Future` is ready,
the actual OS thread can have other tasks assigned to it by the runtime,
and so continue to do work. 

Effectively the task is informing the
runtime that at this point it may yield the execution to another task
(eventually, that other task will also `await` something, and if the
Future in this task is ready, execution of this task may continue) -
this is an implementation of [co-operative multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking).

This syntax is very elegant, and allows us to write asynchronous
programs that maintain a structure similar to simple, synchronous
programs.


## When should I use async?

Asynchronous programming is useful when the thread would otherwise just be 
waiting on IO operations - for example when making network requests or
responses, reading or writing to disk, or waiting for user input.

It is not useful if you are always doing computations and there is no
waiting on IO operations, even if those computations could run in
parallel - for example in a ray tracer. In this case it would be best to
parallelise the computations over OS threads directly (taking advantage
of the multiple cores in your CPU), for example using parallel iterators in the [rayon](https://crates.io/crates/rayon)
crate (or if you want thread-level control, then with the [crossbeam](https://crates.io/crates/crossbeam)
and [threadpool](https://crates.io/crates/threadpool) crates). 
However, remember [Amdahl's Law](https://en.wikipedia.org/wiki/Amdahl%27s_law) and consider that
algorithmic improvements might yield a better return than focusing on 
parallelisation in this cases.

It is also not useful if there are no other tasks to do whilst waiting 
for the IO operations. For example in the [previous blog post](/blog/2020/04/19/data-engineering-with-rust-and-aws-lambda/#en), Rusoto
actually returns a `RusotoFuture` object when we request the DB
credentials from AWS Secrets Manager, however in this case one
invocation of our Lambda function corresponds to one request - there is
no work to be done whilst waiting for the DB credentials to arrive. So
we can just program synchronously (fortunately `RusotoFuture` provides the
`.sync()` method to do exactly that).

## Example

In this example we will simulate three very slow network requests,
consisting of three stages:
* Connection - an asynchronous delay of 2 seconds
* Waiting for a response - an asynchronous delay of 8 seconds (the delay
  is on the server side)
* Computation - a synchronous delay of 4 seconds (i.e. it has to block
  the current OS thread to do the computation).

We will use [Tokio](https://docs.rs/tokio/0.2.20/tokio/) as our async
runtime for this example, as it is currently the most popular.
The other main alternative is the [async_std](https://docs.rs/async-std/1.5.0/async_std/) runtime
\- code for using this runtime is available in the [async-std branch of the Github repo](https://github.com/jamesmcm/async-rust-example/tree/async-std) (this code
was contributed by [@BartMassey](https://github.com/BartMassey)).

Note that both use the common [futures](https://docs.rs/futures/0.3.4/futures/) crate 
so you can swap the async runtime whilst keeping mostly the same API.

### Server

The server used in this example is adapted from the [Tokio tutorial](https://tokio.rs/docs/getting-started/echo/).
It echoes back the bytes received, after a delay of 8 seconds.

The full adapted code is as follows (and [available in the source code
for this post](https://github.com/jamesmcm/async-rust-example)):

```rust
use futures::stream::StreamExt;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let addr = "127.0.0.1:6142";
    let mut listener = TcpListener::bind(addr).await.unwrap();

    let server = {
        async move {
            let mut incoming = listener.incoming();
            while let Some(conn) = incoming.next().await {
                match conn {
                    Err(e) => eprintln!("accept failed = {:?}", e),
                    Ok(mut sock) => {
                        tokio::spawn(async move {
                            let (mut reader, mut writer) = sock.split();
                            tokio::time::delay_for(tokio::time::Duration::from_secs(8)).await;
                            match tokio::io::copy(&mut reader, &mut writer).await {
                                Ok(amt) => {
                                    println!("wrote {} bytes", amt);
                                }
                                Err(err) => {
                                    eprintln!("IO error {:?}", err);
                                }
                            }
                        });
                    }
                }
            }
        }
    };
    println!("Server running on localhost:6142");
    server.await;
}
```

### Synchronous requests

In the synchronous case, we simply run each request in series, one after
another. Therefore we would expect a total execution time of `3*(2+8+4)
= 42` seconds to finish all 3 tasks.

We can visualise this case with a diagram:
![Synchronous execution](/images/synchronous.svg "Synchronous execution")

We can implement this using only the standard library:

```rust
use std::io::prelude::*;
use std::net::TcpStream;
use std::thread::sleep;
use std::time::Instant;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let now = Instant::now();

    task("task1", now.clone())?;
    task("task2", now.clone())?;
    task("task3", now.clone())?;
    Ok(())
}

fn task(label: &str, now: std::time::Instant) -> Result<(), Box<dyn std::error::Error>> {
    // Simulate network delay using thread sleep for 2 seconds
    println!(
        "OS Thread {:?} - {} started: {:?}",
        std::thread::current().id(),
        label,
        now.elapsed(),
    );
    sleep(std::time::Duration::from_secs(2));

    // Write to server - server will echo this back to us with 8 second delay
    let mut stream = TcpStream::connect("127.0.0.1:6142")?;
    stream.write_all(label.as_bytes())?;
    println!(
        "OS Thread {:?} - {} written: {:?}",
        std::thread::current().id(),
        label,
        now.elapsed()
    );

    // Read 5 chars we expect (to avoid dealing with EOF, etc.)
    let mut buffer = [0; 5];
    stream.read_exact(&mut buffer)?;
    stream.shutdown(std::net::Shutdown::Both)?;
    println!(
        "OS Thread {:?} - {} read: {:?}",
        std::thread::current().id(),
        label,
        now.elapsed()
    );

    // Simulate computation work by sleeping actual thread for 4 seconds
    sleep(std::time::Duration::from_secs(4));
    println!(
        "OS Thread {:?} - {} finished: {:?}",
        std::thread::current().id(),
        std::str::from_utf8(&buffer)?,
        now.elapsed()
    );
    Ok(())
}
```

Running this (see [the repo for this workspace](https://github.com/jamesmcm/async-rust-example)):
```bash
$ cargo run --release --bin server
$ cargo run --release --bin client_synchronous
```

```
OS Thread ThreadId(1) - task1 started: 578ns
OS Thread ThreadId(1) - task1 written: 2.000346788s
OS Thread ThreadId(1) - task1 read: 10.002177173s
OS Thread ThreadId(1) - task1 finished: 14.002328699s
OS Thread ThreadId(1) - task2 started: 14.002387112s
OS Thread ThreadId(1) - task2 written: 16.002673602s
OS Thread ThreadId(1) - task2 read: 24.006071003s
OS Thread ThreadId(1) - task2 finished: 28.006204147s
OS Thread ThreadId(1) - task3 started: 28.006263855s
OS Thread ThreadId(1) - task3 written: 30.00652763s
OS Thread ThreadId(1) - task3 read: 38.008234993s
OS Thread ThreadId(1) - task3 finished: 42.008389223s
```

Gives exactly the 42 seconds total execution time that we calculated above.

### Synchronous requests (Tokio)

Note that it is possible to get synchronous behaviour from async
functions when using Tokio (sometimes unintentionally). Implementing the
above with Tokio:

```rust
use futures::stream::StreamExt;
use std::error::Error;
use std::thread::sleep;
use std::time::Instant;
use tokio::join;
use tokio::net::TcpStream;
use tokio::prelude::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
    let now = Instant::now();

    // Synchronous
    task("task1", now.clone()).await?;
    task("task2", now.clone()).await?;
    task("task3", now.clone()).await?;
    Ok(())
}

async fn task(label: &str, now: std::time::Instant) -> Result<(), Box<dyn Error + Send + Sync>> {
    // Simulate network delay using Tokio async delay for 2 seconds
    println!(
        "OS Thread {:?} - {} started: {:?}",
        std::thread::current().id(),
        label,
        now.elapsed(),
    );
    tokio::time::delay_for(tokio::time::Duration::from_secs(2)).await;

    // Write to server - server will echo this back to us with 8 second delay
    let mut stream = TcpStream::connect("127.0.0.1:6142").await?;
    stream.write_all(label.as_bytes()).await?;
    println!(
        "OS Thread {:?} - {} written: {:?}",
        std::thread::current().id(),
        label,
        now.elapsed()
    );

    // Read 5 chars we expect (to avoid dealing with EOF, etc.)
    let mut buffer = [0; 5];
    stream.read_exact(&mut buffer).await?;
    stream.shutdown(std::net::Shutdown::Both)?;
    println!(
        "OS Thread {:?} - {} read: {:?}",
        std::thread::current().id(),
        label,
        now.elapsed()
    );

    // Simulate computation work by sleeping actual thread for 4 seconds
    sleep(std::time::Duration::from_secs(4));
    println!(
        "OS Thread {:?} - {} finished: {:?}",
        std::thread::current().id(),
        std::str::from_utf8(&buffer)?,
        now.elapsed()
    );
    Ok(())
}
```

Running this produces the same output as before:

```bash
$ cargo run --release --bin client_async
```
```
OS Thread ThreadId(1) - task1 started: 333ns
OS Thread ThreadId(1) - task1 written: 2.001476012s
OS Thread ThreadId(1) - task1 read: 10.003284491s
OS Thread ThreadId(1) - task1 finished: 14.003404307s
OS Thread ThreadId(1) - task2 started: 14.003476979s
OS Thread ThreadId(1) - task2 written: 16.005013941s
OS Thread ThreadId(1) - task2 read: 24.005471439s
OS Thread ThreadId(1) - task2 finished: 28.005575307s
OS Thread ThreadId(1) - task3 started: 28.005615372s
OS Thread ThreadId(1) - task3 written: 30.007082377s
OS Thread ThreadId(1) - task3 read: 38.009223127s
OS Thread ThreadId(1) - task3 finished: 42.009349576s
```

It is the `.await`ing of the tasks in series which causes this to be
synchronous. The main function is async, but the use of `.await` causes
it to wait for the result of that Future before continuing. There is
nothing special about the main function compared to other async
functions in this regard. No other tasks exist at
that time to yield execution to, causing the execution to be
synchronous in practice.

Note the Send + Sync bounds are not required for the above
implementation (since it runs on only one OS thread),
but we will need them in the last example. This is also why we clone
`now` instead of borrowing it in `task()` (alternatively we could wrap
it in an `Arc`).

We will use the same `async fn task()` definition for the following
examples, where it will be omitted.

### Asynchronous requests (one OS thread)

In the asynchronous, single OS thread case we start the waiting
steps (connection and getting the server response) concurrently.
However, the final computation step will still need to be done in series
for each task. Therefore we expect a total execution time of
`8+2+(3*4)=22` seconds.

In a diagram:
![Asynchronous execution on a single OS thread](/images/asynchronous_single.svg "Asynchronous execution on a single OS thread")


Using the same definition of `async fn task()` as before:

```rust
use futures::stream::futures_unordered::FuturesUnordered;
use futures::stream::StreamExt;
use std::error::Error;
use std::thread::sleep;
use std::time::Instant;
use tokio::net::TcpStream;
use tokio::prelude::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
    let now = Instant::now();

    // Asynchronous single-thread
    let mut futs = FuturesUnordered::new();

    futs.push(task("task1", now.clone()));
    futs.push(task("task2", now.clone()));
    futs.push(task("task3", now.clone()));

    while let Some(_handled) = futs.next().await {}
    Ok(())
}
```

Running this gives our expected total execution time of 22 seconds:

```
OS Thread ThreadId(1) - task1 started: 3.994µs
OS Thread ThreadId(1) - task2 started: 21.174µs
OS Thread ThreadId(1) - task3 started: 25.511µs
OS Thread ThreadId(1) - task3 written: 2.002221984s
OS Thread ThreadId(1) - task2 written: 2.002406898s
OS Thread ThreadId(1) - task1 written: 2.002483563s
OS Thread ThreadId(1) - task3 read: 10.003326999s
OS Thread ThreadId(1) - task3 finished: 14.003478669s
OS Thread ThreadId(1) - task2 read: 14.00365763s
OS Thread ThreadId(1) - task2 finished: 18.00379238s
OS Thread ThreadId(1) - task1 read: 18.003951713s
OS Thread ThreadId(1) - task1 finished: 22.004094444s
```

In this case we use a `FuturesUnordered` collection to allow us to
repeatedly await the different Futures. However, we never use
`tokio::spawn()` so it is only running on the single OS thread
(as we never allow the creation of more).

Note that we could use the `join!` macro here instead of allocating a
`FuturesUnordered`, we will see an example of this later. However, that
is only practical for a small number of Futures.

We can also force Tokio to only use one thread by putting arguments in
the attribute for our main function:

```rust
#[tokio::main(core_threads = 1, max_threads = 1)]
async fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
...
}
```

### Asynchronous requests (multiple OS threads)

In the case of asynchronous requests across multiple OS threads, we can
do every step concurrently (and the OS threads could complete other tasks
when the tasks await). This means that we can do the final computation step in
parallel on different OS threads.


![Asynchronous execution on multiple OS threads](/images/asynchronous_multi.svg "Asynchronous execution on multiple OS threads")

Therefore we would expect a total execution time of `2+8+4=14` seconds
for all 3 requests. This is the best we can achieve - the same time as
completing a single request.

Note that this requires the types we send across threads to be
thread-safe, i.e. implementing Send or Sync - just as we would if we were
using OS threads directly.

The implementation is very similar to the previous example, but instead
of awaiting on the futures directly, we `tokio::spawn` the tasks and
await their handles. This allows Tokio to execute them on different OS
threads.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
    let now = Instant::now();

    let mut futs = FuturesUnordered::new();
    futs.push(tokio::spawn(task("task1", now.clone())));
    futs.push(tokio::spawn(task("task2", now.clone())));
    futs.push(tokio::spawn(task("task3", now.clone())));
    while let Some(_handled) = futs.next().await {}
    Ok(())
}
```

And we observe the 14 second execution time (note we don't care about
the order of execution):

```
OS Thread ThreadId(2) - task1 started: 17.055µs
OS Thread ThreadId(3) - task2 started: 30.227µs
OS Thread ThreadId(2) - task3 started: 32.513µs
OS Thread ThreadId(2) - task3 written: 2.001499145s
OS Thread ThreadId(3) - task1 written: 2.00153689s
OS Thread ThreadId(5) - task2 written: 2.001721878s
OS Thread ThreadId(3) - task3 read: 10.003403756s
OS Thread ThreadId(2) - task1 read: 10.003501s
OS Thread ThreadId(5) - task2 read: 10.003417328s
OS Thread ThreadId(3) - task3 finished: 14.003584085s
OS Thread ThreadId(2) - task1 finished: 14.003664981s
OS Thread ThreadId(5) - task2 finished: 14.003698375s
```

The different OS Thread IDs demonstrate that the tasks are being
executed on separate OS threads.

For completeness, here is the same implementation using the `join!`
macro instead of allocating a `FuturesUnordered`:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
    let now = Instant::now();
    // Asynchronous multi-threaded

    match join!(
        tokio::spawn(task("task1", now.clone())),
        tokio::spawn(task("task2", now.clone())),
        tokio::spawn(task("task3", now.clone()))
    ) {
        (x, y, z) => {
            (x.ok(), y.ok(), z.ok())
        }
    };
    Ok(())
}
```

This saves the allocation of the `FuturesUnordered` but it can be
awkward to work with the tuple of Results that is returned (especially
for many Futures).

#### How does this differ from OS thread parallelisation?

You could implement something similar for the synchronous case using OS threads directly, for
example with the [rayon](https://crates.io/crates/rayon) crate mentioned previously.

However, in this case each request would need its own OS thread, and if
we had to carry out 10,000 concurrent requests, we might hit the thread
limit. 

That is, the execution diagram would look like this:
![Synchronous execution on multiple OS threads](/images/synchronous_multios.svg "Synchronous execution on multiple OS threads")

Note that the OS threads would spend a lot of time just waiting on IO operations,
unable to start other requests.

We can modify our first synchronous example above to do this with rayon:

```rust
use rayon::prelude::*;

fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let now = Instant::now();

    ["task1", "task2", "task3"]
        .par_iter()
        .map(|x| task(x, now.clone()))
        .collect::<Result<Vec<_>, _>>()?;
    Ok(())
}
```

And it terminates in 14 seconds as expected (note that each task is
allocated to one OS thread):

```
OS Thread ThreadId(3) - task1 started: 280.871µs
OS Thread ThreadId(6) - task2 started: 281.03µs
OS Thread ThreadId(7) - task3 started: 283.838µs
OS Thread ThreadId(6) - task2 written: 2.000605562s
OS Thread ThreadId(7) - task3 written: 2.000619598s
OS Thread ThreadId(3) - task1 written: 2.000679853s
OS Thread ThreadId(3) - task1 read: 10.002321036s
OS Thread ThreadId(6) - task2 read: 10.00233185s
OS Thread ThreadId(7) - task3 read: 10.002384653s
OS Thread ThreadId(3) - task1 finished: 14.002447762s
OS Thread ThreadId(6) - task2 finished: 14.002540969s
OS Thread ThreadId(7) - task3 finished: 14.002589621s
```

However, this would not be resource-efficient when handling a large
number of tasks since one task corresponds to one OS thread.

Whereas in the async case we only need the additional threads during the
computation step (which is synchronous). This means that we could use a
fixed OS threadpool size and still benefit from some parallelisation in
the computation step whilst having guarantees about our resource usage
(i.e. we can limit the maximum number of OS threads and still be able to
start new requests).

## Conclusion

I hope this blog post has helped you to better understand when and how
to apply asynchronous programming in Rust.

The elegant async/await syntax allows for clear and concise asynchronous
programming. However, it may take some getting used to if you do not
have prior experience with asynchronous programming.

These examples also demonstrate the difference between concurrency and
parallelism. In the asynchronous single OS thread case we are handling
tasks concurrently, but nothing is running in parallel (i.e. we use only
one OS thread).

### Current Limitations

Note that currently you cannot use [async methods in traits](https://rust-lang.github.io/async-book/07_workarounds/06_async_in_traits.html), and nor
can you create [async destructors](https://boats.gitlab.io/blog/post/poll-drop/). This is a problem if you want a struct
to make a network request when it is dropped, but you don't want to
block the OS thread whilst it does that - you cannot `.await` because
`drop()` (from [Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html)) is not async.

I hit the latter issue in my recent [s3rename](https://github.com/jamesmcm/s3rename) crate, and am still
attempting to work around it. See [this issue](https://github.com/jamesmcm/s3rename/issues/16) for details.


