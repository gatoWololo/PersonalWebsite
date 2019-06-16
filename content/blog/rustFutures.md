+++
date = 2019-05-29
title = "Rust Futures From Scratch: History, Resources, and Implementation (For beginners, by beginners)"
description = "Using Rust Futures without Tokio or Mio"
+++

## Preface and Goals
I am by no means a expert on Rust Futures. This is just what I learned trying to implement a Future for my use-case. Specifically, we will see how to use Rust's latest[^1] future API to implement our own reactor and executor (No Tokio or Mio here!). In this blog I attempt to provide resources, history of futures, and a working implementation. Including the upcoming **async** and **await** syntax.

## Difficulties Learning Futures
Like many other Rust developers on _the internet_, I had a general idea of what futures are and their purpose in Rust. However, when trying to grasp the implementation details, or implement your own future, finding information on the internet is quite frustrating. Specifically, futures are still very much in flux, so people have held off from creating documentation that might soon be outdated. [Existing](https://paulkernfeld.com/2018/01/20/future-by-example.html) blogs were outdated with messages like "This document is now out of date. I recommend learning about Rust futures another way." . These outdated blogs may still be useful! But starting with no experience with futures, it was hard to tell apart the useful from outdated. The biggest challenge for the curious: it is _hard_ to find resources that explain futures without using Tokio or Mio.

## Assumptions and Goals
This post assumes intermediate Rust knowledge (e.g `impl trait`, `associated types`, `closures` , etc) and a high level understanding of futures and async IO. For the uninitialized, I found [Snoyman's blog](https://www.snoyman.com/blog/2018/12/rust-crash-course-07-async-futures-tokio) post quite helpful.

The full implementation we will develop in this post can found [here](https://github.com/gatoWololo/RustFutureExample).

You do not need any systems programming knowledge even though I'll be implementing a future and runtime for waiting on child exit events.

## Background
For my current research project we're interested in implementing handlers for Linux ptrace events. We want to have a coroutine-style program structure: where we have one coroutine handler per process/thread we are tracing. A big design constraint is that ptrace is not very friendly to threaded executions, that is you cannot (easily) have a multi-threaded tracer.

Instead, we want to have all our coroutines live in the same main process (thread), and merely "stack switch" between our futures.

We will be implementing a simpler example here, where a future may wait on a child exit event.

## Existing Resources
I found the following resources invaluable when learning the finer details of futures. You don't have to read all these, as I will (attempt) to  go over everything you need to know to get started with futures.

[Farenheit](https://github.com/polachok/fahrenheit) has the same goal as this blog post (no tokio or mio). I used Farenheit as my main blueprint with some changes.

[The Rust Async Book](https://rust-lang.github.io/async-book/) is still in the works, I found their chapter two: "Under the Hood: Executing Futures and Tasks" most helpful.

[This blog post](https://josh.robsonchase.com/embedded-executor/) was very helpful as well, and has similar goals.

[Tokio's docs](https://tokio.rs/docs/going-deeper/futures/) and [here](https://tokio.rs/docs/going-deeper/runtime-model/) have a lot of useful information.

[Understanding the Waker](https://boats.gitlab.io/blog/post/wakers-i/) from the source itself.

[Here](https://www.reddit.com/r/rust/comments/anu8w4/futures_03_how_does_waker_and_executor_connect/) were [other](https://www.reddit.com/r/rust/comments/90g9ln/asyncawaitfuturesexecutor_without_external_crates/) confused souls like myself which helped me piece together part of the story.

## Background Terminology
There are a lot of terms used around futures. Many blogs and documentation expect the reader to already know what they are. I tried to give definition here. All the details will come later!

- Future: represents a computation or value that is not yet available, like a closure, declaring a future does not execute it.
- Task: the runtime instantiation of a future. The actual computation being executed.
- Executor: responsible for running and scheduling our tasks.
- Reactor: the driver for the IO, usually by communicating with the OS directly through a low-level system call, for Linux this is usually `epoll`.
- Waker: represents a handle that will inform the executor that a task is ready to be scheduled for running again.
- Polling: polling generally refers to a non-blocking check for some IO resource to see if it is ready.

## Getting started
With our new terminology we can now talk about exactly why we will implement. When I say "future runtime", I mean specifically the _executor_ that will schedule our _tasks_, and the _reactor_ which will do the low-level communication with the OS. For our project, the executor and reactor will be one component. We will see how the executor runs the reactor, schedules tasks, and generates wakers. We will also implement a future, i.e we will create our own async computation which we can wait on.

## Tokio and Mio (What are they?)
[Tokio](https://tokio.rs/) is a futures framework/runtime as it provides the executor as well as the futures for many asynchronous tasks. Tokyo defers the work of doing the low-level IO with the OS to [Mio](https://github.com/tokio-rs/mio). In Linux, Mio runs an `epoll` loop to poll for events. The previous description is an understatement: both libraries do _a lot_ and I don't want to sell them short. Similarly, due to the complexity, it was not obvious to me if they could fit our use-case, most examples I found in the Tokio docs had to do with networks and severs. Lastly, our team hesitated to introduce such heavy weight dependencies into our project.

## Our First Piece of Rust Code: What is a Future?
A future is simply any type implements the [future trait](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/prelude/trait.Future.html).
```rust
pub trait Future {
type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```
### The Output Associated Type
`Output`, aptly named, represents the type our future will produce. Our future for waiting on child exiting `Output` will be a [Nix](https://docs.rs/nix/0.14.1/nix/) [WaitStatus](https://docs.rs/nix/0.14.1/nix/sys/wait/enum.WaitStatus.html).

### The Poll Method

`poll` is the one method which we have to implement for our future. There is a lot to unpack in the type signature, but it also provides a lot of information (types FTW!):

Notice the `self` argument has type `Pin<&mut Self>` (instead of regular old `&mut Self`). This was my first time seeing an [arbitrary self type](https://github.com/rust-lang/rust/issues/44874). That is we cannot simply call `myFuture.poll(...)`, unless our future is wrapped in a Pin _and_ the data inside is a mutable reference.

`Pin` is magical and mysterious, I defer `Pin` details to the [documentation](https://doc.rust-lang.org/std/pin/index.html). However, a one sentence summary quoting the docs: "A `Pin<P>` ensures that the pointee of any pointer type `P` has a stable location in memory, meaning it cannot be moved elsewhere and its memory cannot be deallocated until it gets dropped". More information on the API design of Pin [here](https://github.com/rust-lang/rust/issues/49150). Why do Rust Futures need to be pinned? Well it's complicated and I don't fully understand: The idea is that in the implementation details, futures end up having some self-referential data, Rust lifetime checker generally does not like this. `Pin` therefore asks as a guarantee that the future will not be moved/dropped. The master **@withoutboats** has a throughout [blog post series](https://boats.gitlab.io/blog/post/2018-01-25-async-i-self-referential-structs/) describing the problem in detail. As an implementor of the `Future` trait, this is not as important. When implementing the executor and reactor, we have to care.

The `cs: &mut Context` currently holds a reference to the waker via the method `pub fn waker(&self) -> &'a Waker`. In the future `cx` may expand to include other data while maintaining backwards compatibly. Check out `Context` [here](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/task/struct.Context.html).

The return type `Poll<Self::Output>` is fairly straightforward:
```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```
It either returns the data if it's ready. Or simply informs us that it is still waiting. Question! Who _exactly_ is "us" in the last sentence? It is the exectutor/reactor! Which will be calling the `poll` method directly on the future.

### Implementing Future, First Try
We will start with a simple future, which is **not** actually what we want, but it's a first good step. We will define our own type `WaitStatusEvent` which will implement future:
```rust
/// Future representing calling waitpid() on a pid.
pub struct WaitStatusEvent {
    /// Pid we're waiting on.
    pub pid: Pid,
}

impl Future for WaitStatusEvent {
    /// Produces a WaitStatus when done.
    type Output = WaitStatus;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
       -> Poll<Self::Output> {

        match waitpid(self.pid, Some(WaitPidFlag::WNOHANG)).
          expect("Unable to waitpid from poll") {
            WaitStatus::StillAlive => {
                Poll::Pending
            }
            w => Poll::Ready(w),
        }
    }
}
```
The [system call `waitpid()`](https://docs.rs/nix/0.14.1/nix/sys/wait/fn.waitpid.html) takes a `Pid` and blocks until that process' status has changed. This is a _key_ difference between traditional blocking IO and asynchronous IO: we should not block on an event. Therefore we use the `WaitPidFlag::WNOHANG` _with no hang_ flag. This turns `waitpid` from a blocking call to polling. In general, **your poll implementation should not block** and should return quickly. Otherwise, other tasks sharing the same thread will be blocked as well.

The idea of polling is very general. For example, we can poll on sockets to see if data is available from network, or we can wait for a timer to run out, or in our example we will wait for a child process to exit. The method or IO system call needed for polling is very much case-specific[^2].

### Async and await.
We can now write a function which uses our future:
```rust
fn wait_for_child(pid: Pid) -> impl Future<Output=WaitStatus> {
    let child = WaitStatusEvent { pid };
    async {
        let status = child.await;
        println!("Return Value {:?}!", status);
        status
    }
}
```
Here we see the use of Rust's new `.await` and `async` syntax for working ergonomically with futures. `.await` acts like a _yield point_. In `child.await` we will implicitly call `poll`. If `poll` returns with `Ready(WaitStatus)` we continue, executing this function. If `poll` returns `Pending` the function `yields` execution back to the executor[^3].

We can _only_ call `.await` inside a `async` block. Async blocks implicitly create a future which we see as the return type of this function. Notice `wait_for_child` return a anonymous type implementing `Future<_>` as its return type.

We can use `async` at the function level as well!
```rust
async fn wait_for_child(pid: Pid) -> WaitStatus {
    let child = PtraceEvent { pid };
    let status = child.await;
    println!("Return Value {:?}!", status);
    status
}
```
Now all mentions of `Future<_>` are gone from our function signature. This gives us insight into the compiler transformation that `async` performs. It is useful to remember that **async functions are returning a `-> impl Future<_>`.** This means the entire function `wait_for_child` _is_ a future. In general we build bigger futures out of smaller futures, the future executes its code until a `.await` operation where we may yield execution.

## Calling Futures.
Great let's create a child process and call our future!
```rust
fn main() -> nix::Result<()> {
    match fork()? {
        ForkResult::Parent { child: child } => {
            wait_for_child(child);
        }
        ForkResult::Child => {
            nix::unistd::sleep(1);
            println!("Child done!");
        }
    }
    Ok(())
}
```
The child waits one second and exits. The parent calls our async `wait_for_child()`. Neat, it compiles! But this **does not** do what we want:
```rust
warning: unused implementer of `core::future::future::Future` that must be used
  --> src/main.rs:24:13
   |
24 |             wait_for_child(child);
   |             ^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: #[warn(unused_must_use)] on by default
   = note: futures do nothing unless you `.await` or poll them
```
So this "do nothing"... We cannot call `.await` as the error suggests:
```
error[E0728]: `await` is only allowed inside `async` functions and blocks
  --> src/main.rs:24:13
   |
14 | fn main() -> nix::Result<()> {
   |    ---- this is not `async`
...
24 |             wait_for_child(child).await;
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^ only allowed inside `async` functions and blocks
```
Let's go down the rabbit hole and make `main()` async?
```rust
error[E0277]: `main` has invalid return type `impl core::future::future::Future`
  --> src/main.rs:14:20
   |
14 | async fn main() -> nix::Result<()> {
   |                    ^^^^^^^^^^^^^^^ `main` can only return types that implement `std::process::Termination`
   |
   = help: consider using `()`, or a `Result`
```
The compiler hates it. We have reached the very top of the program, and we cannot add any more asyncs or awaits. What now? Well this is where the executor comes in! Instead of `wait_for_child(child);` or `wait_for_child(child).await;`, we will pass this function into our executor.

## Implementing an Executor
### Executor overview:
Conceptually, the executor has a few jobs.
- It accepts new futures to spawn as tasks.
- It keeps track of all tasks, scheduling those which events have arrived.

Given a future `f: impl Future<Ouput=T>` calling its `poll()` method will automatically start executing the future. Any calls to `.await` will _actually_ start the `poll()` method as defined by us. Adding some print statements to our
functions let's us observe this behavior:
```rust
async fn wait_for_child(pid: Pid) -> WaitStatus {
    println!("Beginning of async function.");
    ...
}

impl Future for PtraceEvent {
    type Output = WaitStatus;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
      -> Poll<Self::Output> {
        println!("future polling!");
        ...
    }
}
```

We see the following sequence of events:
```
New future added! // From executor function (comming soon!)
Beginning of async function.
future polling!
New future returned Pending!
```

### Executor Implementation
We define our executor + reactor as follows:

```rust
pub struct WaitidExecutor<'a> {
  ... // Ignore fields for now.
}
```

Next we define a method to add futures to our executor for running:
```rust
pub fn add_future<F>(&mut self, future: F)
where
    F: Future<Output = WaitStatus> + Send + 'a,
{

}
```
This is probably not the best function signature to add a futures. As Rust will specialize the function for every concrete type used for `F`. A dynamic trait object is probably more appropriate here.

We want to call `fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>`. So first we need to wrap our future in a Pin. However, calling the `new()` `Pin` [method](https://doc.rust-lang.org/nightly/std/pin/struct.Pin.html#method.new): `pub fn new(pointer: P) -> Pin<P>` does not work:

```rust
error[E0277]: the trait bound `F: std::ops::Deref` is not satisfied
  --> src/executor.rs:89:9
   |
89 |         Pin::new(future);
   |         ^^^^^^^^ the trait `std::ops::Deref` is not implemented for `F`
   |
   = help: consider adding a `where F: std::ops::Deref` bound
   = note: required by `std::pin::Pin::<P>::new`
```

 I don't understand painting in detail,  however it seems that idea is to wrap pointers,  EG types that implements drf, the easiest way I found 2 create a pinned future s using the box method box::pin, 
 
 future.as_mut.

## Footnotes
[^1] I am using `futures-preview = "0.3.0-alpha.16"` found [here](https://github.com/rust-lang-nursery/futures-rs). It seems eventually this will become the default std implementation?

[^2] Linux's [`epoll` system call](http://man7.org/linux/man-pages/man7/epoll.7.html) covers polling on many IO sources, and is therefore general enough to be useful in most cases.

[^3] Rust futures are implemented as coroutines, for background on Rust coroutines check out [this](https://github.com/rust-lang/rust/issues/43122). See [generators](https://doc.rust-lang.org/beta/unstable-book/language-features/generators.html) for a brief introduction to `yield`.
