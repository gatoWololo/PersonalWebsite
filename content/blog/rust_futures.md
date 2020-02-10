+++
date = 2019-05-29
title = "Implement your own Rust: Future, Reactor, Executor, And More (For beginners, by beginners)"
description = "Using Rust Futures without Tokio or Mio"
+++

# Preface and Goals
I am by no means a expert on Rust Futures. This is just what I learned trying to implement a Future for my use-case. Specifically, we will see how to use Rust's latest[^1] future API to implement our own reactor and executor (No Tokio or Mio here!). In this blog I attempt to provide resources, history of futures, and a working implementation. Including the upcoming **async** and **await** syntax.

# Difficulties Learning Futures
Like many other Rust developers on _the internet_, I had a general idea of what futures are and their purpose in Rust. However, when trying to grasp the implementation details, or implement your own future, finding information on the internet is quite frustrating. Specifically, futures are still very much in flux, so people have held off from creating documentation that might soon be outdated. Existing [[1]](#r1) blogs were outdated with messages like "This document is now out of date. I recommend learning about Rust futures another way." . These outdated blogs may still be useful! But starting with no experience with futures, it was hard to tell apart the useful from outdated. The biggest challenge for the curious: it is _hard_ to find resources that explain futures without using Tokio or Mio.

# Assumptions and Goals
This post assumes intermediate Rust knowledge (e.g `impl trait`, `associated types`, `closures` , etc) and a high level understanding of futures and async IO. For the uninitialized, I found Snoyman's blog [[2]](#r2) post quite helpful.

The full implementation we will develop in this post can found here [[3]](#r3).

You do not need any systems programming knowledge even though I'll be implementing a future and runtime for waiting on child exit events.

# Background
For my current research project we're interested in implementing handlers for Linux ptrace events. We want to have a coroutine-style program structure: where we have one coroutine handler per process/thread we are tracing. A big design constraint is that ptrace is not very friendly to threaded executions, that is you cannot (easily) have a multi-threaded tracer.

Instead, we want to have all our coroutines live in the same main process (thread), and merely "stack switch" between our futures.

We will be implementing a simpler example here, where a future may wait on a child exit event.

# Existing Resources
I found the following resources invaluable when learning the finer details of futures. You don't have to read all these, as I will (attempt) to  go over everything you need to know to get started with futures.

Farenheit [[4]](#r4) has the same goal as this blog post (no tokio or mio). I used Farenheit as my main blueprint with some changes.

The Rust Async Book [[5]](#r5) is still in the works, I found their chapter two: "Under the Hood: Executing Futures and Tasks" most helpful.

The embedded systems executor series [[6]](#r6) was very informative.

Tokio's docs [[7]](#r7)[[8]](#r8) have a lot of useful information.

The _Understanding the Waker_ series [[9]](#r9) from the source itself.

TODO Add Local Executor (Naive, loop on all futures)

Here [[10]](#r10)[[11]](#r11) are other confused souls like myself, which helped me piece together part of the story.

# Background Terminology
There are a lot of terms used around futures. Many blogs and documentation expect the reader to already know what they are. I tried to give definition here. All the details will come later!

- Future: represents a computation or value that is not yet available, like a closure, declaring a future does not execute it.
- Task: the runtime instantiation of a future. The actual computation being executed.
- Executor: responsible for running and scheduling our tasks.
- Reactor: the driver for the IO, usually by communicating with the OS directly through a low-level system call, for Linux this is usually `epoll`.
- Waker: represents a handle that will inform the executor that a task is ready to be scheduled for running again.
- Polling: polling generally refers to a non-blocking check for some IO resource to see if it is ready.

# Getting started
With our new terminology we can now talk about exactly why we will implement. When I say "future runtime", I mean specifically the _executor_ that will schedule our _tasks_, and the _reactor_ which will do the low-level communication with the OS. We will see how the executor runs the reactor, schedules tasks, and generates wakers. We will also implement a future, i.e we will create our own async computation which we can wait on.

# Tokio and Mio (What are they?)
Tokio[[12]](#r12) is a futures framework/runtime as it provides the executor as well as the futures for many asynchronous tasks. Tokyo defers the work of doing the low-level IO with the OS to Mio [[13]](#r13). In Linux, Mio runs an `epoll` loop to poll for events. The previous description is an understatement: both libraries do _a lot_ and I don't want to sell them short. Similarly, due to the complexity, it was not obvious to me if they could fit our use-case, most examples I found in the Tokio docs had to do with networks and severs. Lastly, our team hesitated to introduce such heavy weight dependencies into our project.

# Our First Piece of Rust Code: What is a Future?
A future is simply any type implements the future trait[[14]](#r14).
```rust
pub trait Future {
type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```
## The `Output` Associated Type
`Output`, aptly named, represents the type our future will produce. Our future for waiting on child exiting `Output` will be a Nix [[15]](#r15) `WaitStatus` [[16]](#r16).

## The `poll` Method

`poll` is the one method which we have to implement for our future. There is a lot to unpack in the type signature, but it also provides a lot of information (types FTW!):

Notice the `self` argument has type `Pin<&mut Self>` (instead of regular old `&mut Self`). This was my first time seeing an arbitrary self type [[17]](#r17). That is we cannot simply call `myFuture.poll(...)`, unless our future is wrapped in a Pin _and_ the data inside is a mutable reference.

`Pin` is magical and mysterious, I defer `Pin` details to the documentation [[18]](#r18). However, a one sentence summary quoting the docs: "A `Pin<P>` ensures that the pointee of any pointer type `P` has a stable location in memory, meaning it cannot be moved elsewhere and its memory cannot be deallocated until it gets dropped". More information on the API design of Pin can be found here [[19]](#r19). Why do Rust Futures need to be pinned? Well it's complicated: The idea is that in the implementation details, futures end up having some self-referential data, Rust lifetime checker generally does not like this. `Pin` therefore asks as a guarantee that the future will not be moved/dropped. The master **@withoutboats** has a throughout blog post series [[20]](#r20) describing the problem in detail. As an implementor of the `Future` trait, this is not as important. When implementing the executor and reactor, we have to care.

We will go into details about the `Context` later.

The return type `Poll<Self::Output>` is fairly straightforward:
```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```
It either returns the data if it's ready. Or simply informs us that it is still waiting. Question! Who _exactly_ is "us" in the last sentence? It is the exectutor! Which will be calling the `poll` method directly on the future.

# Implementing Future, First Try
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

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {

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
The `waitpid()` system call [[21]](#r21) takes a `Pid` and blocks until that process' status has changed. This is a _key_ difference between traditional blocking IO and asynchronous IO: we should not block on an event. Therefore we use the `WaitPidFlag::WNOHANG` _with no hang_ flag. This turns `waitpid` from a blocking call to polling. In general, **your poll implementation should not block** and should return quickly. Otherwise, other tasks sharing the same thread will be blocked as well.

The idea of polling is very general. For example, we can poll on sockets to see if data is available from network, or we can wait for a timer to run out, or in our example we will wait for a child process to exit. The method or IO system call needed for polling is very much case-specific[^2].

# Async and await.
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
    let child = WaitStatusEvent { pid };
    let status = child.await;
    println!("Return Value {:?}!", status);
    status
}
```
Now all mentions of `Future<_>` are gone from our function signature. This gives us insight into the compiler transformation that `async` performs. It is useful to remember that **async functions are returning a `-> impl Future<_>`.** This means the entire function `wait_for_child` _is_ a future. In general we build bigger futures out of smaller futures, the future executes its code until a `.await` operation where we may yield execution.

# Calling Futures.
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

# Implementing an Executor

Conceptually, the executor has a few jobs.
- It accepts new futures to spawn as tasks.
- It keeps track of all tasks, scheduling those which events have arrived.

Given a future `f: impl Future<Ouput=T>` calling its `poll()` method will automatically start executing the future. Any calls to `.await` will _actually_ start the `poll()` method as defined by us. Adding some print statements to our functions let's us observe this behavior:
```rust
async fn wait_for_child(pid: Pid) -> WaitStatus {
    println!("Beginning of async function.");
    ...
}

impl Future for PtraceEvent {
    type Output = WaitStatus;

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
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

We define our executor  as follows:

```rust
pub struct WaitidExecutor<'a> {
  ... // Ignore fields for now.
}
```

Fields, and method will come later, but note our executor doesn't need to implement any fancy traits.

## Adding Futures to Our Executor
We define a method to add futures to our executor for running:
```rust
pub fn add_future<F>(&mut self, added_future: F)
where
    F: Future<Output = WaitStatus> + Send + 'a,
{
   ...
}
```
(**Note:** This is probably not the best function signature to add a futures. As Rust will specialize the function for every concrete type used for `F`. A dynamic trait object is probably more appropriate here.)

The idea is simple: we _poll_ the `added_future` future once. If polling returns `Poll::Pending`, we add our future to a queue and try again later, otherwise it runs to completion.

### Calling poll(...)
We want to call `fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>`. So first we need to wrap our future in a `Pin`. However, calling Pin's `fn new(pointer: P) -> Pin<P>` [[22]](#r22) does not work:

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

 I don't understand pinning in detail, however it seems that pinning only makes sense for types that act like pointers. Thus, we can only pin types that implements `Deref`, the easiest way I found to create a pinned future is using the (TODO hyperlink) `Box` method `Box::pin()`:

 ```rust
// Pin it, and box it up for storing.
let mut added_future: BoxFuture<'a, WaitStatus> = Box::pin(added_future);
 ```

Where `BoxFuture` is defined (TODO add hyperlink) as:
```rust
/// An owned dynamically typed Future for use in cases where you can't statically type your result or need to add some indirection.
type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + 'a + Send>>;
```
That is, a pinned, boxed (heap allocated), `Future` trait object, of `Output` type `T` with lifetime `'a`, which implements `Send` (safe to send across threads), phew!

We are almost there... but not quite. Calling `poll(...)` on our `added_future: BoxFuture<'a, WaitStatus>` still won't work. `poll` demands the calling object be of the form: `self: Pin<&mut Self>`. Notice the mutable reference `&mut` is "inside" the `Pin<...>`. The `Pin` API has the exact right method (TODO add hyperlink) for our purposes:
```rust
pub fn as_mut(&mut self) -> Pin<&mut <P as Deref>::Target>
```
And finally we have the right type set up to call `poll`!

### Creating a Context and Waker
`poll` takes an argument `cx: &mut Context`. Checking out `Context` [[23]](#r23). We see a `Context` just holds a reference to the waker via the method `pub fn waker(&self) -> &'a Waker`. In the future `Context` may expand to include other things.

The only way I have to create a `Context` is via the associated function:
```rust
pub fn from_waker(waker: &'a Waker) -> Context<'a>
```

TODO add hyperlinks!
So first we need a `Waker`. The only way I found to create a `Waker` is by implementing `ArcWake` and calling `fn into_waker(self: Arc<Self>) -> Waker`. Roughly, we know a waker is in charge of informing the executor when a task is ready to be polled again, but up to this point. It is not clear why we need this. So we will implement a "null" waker for now:
```rust
struct WaitidWaker {
    // none for now...
}

impl ArcWake for WaitidWaker {
    fn wake_by_ref(arc_self: &Arc<Self>) {
      // blank!
    }
}
```
### Putting It All Together
We have now a fully working function to add futures:
```rust
pub fn add_future<F>(&mut self, added_future: F)
 where
     F: Future<Output = WaitStatus> + Send + 'a,
 {
     // Create waker.
     let task_id = self.get_next_task_id();
     let waker = Arc::new(WaitidWaker { }).into_waker();

     // Pin it, and box it up for storing.
     let mut added_future: BoxFuture<'a, WaitStatus> = Box::pin(added_future);

     match added_future.as_mut().poll(&mut Context::from_waker(&waker)) {
         Poll::Pending => {
             println!("New future returned Pending!");
             self.waiting_tasks.insert(task_id, added_future);
         }
         Poll::Ready(_) => {
             // Task is done, no need to add it to queue.
             // get dropped
         }
     }
}
```
Where `waiting_tasks: BTreeMap<TaskId, BoxFuture<'a, WaitStatus>>`, and `TaskId` is a unique `u32` assigned to every task. So all tasks that are still _pending_ on some IO event go to our `waiting_tasks` queue.

## Running our queued futures.
Next we create a function to run all our queued futures to completion:
```rust
pub fn run_all(&mut self) {
  ...
}
```
The calling thread will block on `run_all` until all added futures finish[^4]. A naive implementation could simply iterate over our `waiting_tasks`, polling all futures seeing if any can make progress. This type of busy waiting is unnecessary, we can do better! This is where the _reactor_ comes into play. Instead of busy waiting by looping on `poll`, the executor will call reactor to block until an event comes from any future

# The Polling, registering, blocking, waking, loop
Let's have a look at the high level overview of how our entire future infratructure makes progress:
1) Futures are added to the executor via `add_future`.
2) `add_future` calls the `poll` method on a future.
3) Inside a `poll` future registers it's IO event with the reactor.
3) Futures that return Pending are queued away.
4) The executor calls the reactor to block until IO events arive.
TODO Clean Up!

### Reactor: Waitid
`waitid` (TODO hyperlink) is a Linux system call for blocking on events from a PID. We use `waitid` with `P_ALL` to wait for events for any PID. The only way we have to execute a future is by calling its `poll` method. Recall the `poll` implementation for `WaitStatusEvent` calls `waitpid(self.pid, , Some(WaitPidFlag::WNOHANG))`. Importantly, `waitpid` **consumes the event** as a side effect of returning successfully. `WaitStatusEvent::poll()` must do the consuming. So to avoid our reactor consuming the event we use `WNOWAIT` "Leave the child in a waitable state; a later wait call can be  used  to  again retrieve the child status information." For this reason the reactor's waiting method should be _side effect free_. As an example, consider a `SocketEvent` future, its reactor would check if the socket is available for reading, but `SocketEvent::poll` would do the (side effect computation:) _actual_ reading.

### Blocking Executor
We can now properly implement `pub fn run_all(&mut self)`. The implementation gets bogged down with low level details. So I present a high level overview here:
```rust
pub fn run_all(&mut self) {
    // Keep looping until all waiting tasks are done.
    loop {
        reactor::wait_for_event();
        // Use task_id to `poll` correct future.
        let task_id = ???;

        match self.waiting_tasks.entry(task_id) {
            Entry::Occupied(mut task) => {
                // ...
                match task.get_mut().as_mut().poll(context) {
                    // Made progress but still pending.
                    Poll::Pending => continue,
                    // All done!
                    Poll::Ready(_) => task.remove_entry(),
                }
            }
            Entry::Vacant(_) => {
                panic!("No task waiting for this PID");
            }
        }
    }
}
```
The executor calls the reactor's `fn wait_for_event()`, which blocks until an event is ready. After calling `wait_for_event`, how does the executor know which future (represented by a `task_id`) is ready to run next?

Observe `wait_for_event()` neither takes or returns any arguments. This is weird, but on purpose. One of the features of futures is the **decoupling** of executor and reactor: we can use different executors with different reactors and vice versa (e.g. use a single-threaded executor versus a multithreaded one).

Even with our partial implementation we already see the decoupling: Our executor makes no mention of `waitid`, `waitpid`, or child event futures. _It is completely future agnostic_. Similarly, as we are about to see, the reactor knows nothing about how the executor is implemented.

# Implementing a Reactor
While the executor and reactor know nothing of each other, the reactor and future are intimately tied. Which makes sense: the reactor needs to know what IO event a future is waiting on, and how to wait for this event.

## Revisiting `poll()`
Let's implement a reactor aware `poll()` for `WaitStatusEvent`:
```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
    match waitpid(self.pid, Some(WaitPidFlag::WNOHANG)).
        expect("Unable to waitpid from poll") {
            WaitStatus::StillAlive => {
                reactor::register(self.pid, cx.waker().clone());
                Poll::Pending
            }
            w => Poll::Ready(w),
        }
}
```
We added but a single line `reactor::register(self.pid, cx.waker().clone());`. The future informs the reactor that it is waiting on `waitpid()` events from `self.pid`, and the future passes the reactor a copy of its waker. Notice the future does nothing with the waker except pass it along to the reactor. The why will become clear soon.

## Reactor implementation
First and foremost, a reactor blocks for IO events by interacting with the OS (low-level details are changed to make the concept clear):
```rust
fn wait_for_event() {
    // Block here for actual events to come.
    let ready_pid = libc::waitid(libc::P_ALL, ..., libc::WNOWAIT ...);

    // Call waker of ready_pid here.
    PID_WAKER.with(|hashtable| {
        hashtable
            .borrow_mut()
            .get(&ready_pid)
            .expect("No such entry, should have been there.")
            .wake_by_ref();
    });
}
```
We simply call the `wake_by_ref()` method of the waker. Which will inform the executor of what future to poll next. Let's take a look at `PID_WAKER`:
```rust
thread_local! {
    pub static PID_WAKER: RefCell<HashMap<Pid, Waker>> = RefCell::new(HashMap::new());
}
```
It is simply a `HashMap` living in TLS from PIDs to wakers. Entries are added to this map via:
```rust
fn register(pid: Pid, waker: Waker) {
    PID_WAKER.with(|pid_waker| {
        pid_waker.borrow_mut().insert(pid, waker);
    });
}
```

Why does `PID_WAKER` live in TLS? Recall `WaitStatusEvent` needs to call `register()` to 

# All About the Waker
The `Waker` documentation [TODO] states "A Waker is a handle for waking up a task by notifying its executor that it is ready to be run." Okay, but how? The documentation on the `Waker` methods is equally as vague:

```rust
/// Wake up the task associated with this Waker.
pub fn wake(self)
```
Thanks documentation. This is intentionally vague because _waking_ is entirely implementation dependent, and will be different for every executor implementation. For an in depth dive of wakers see **@withoutboats** excellent blog series [[9]](#r9). However, in general Thread Local State (TSL) TODO REF accessible to both the waker and the executor is used to inform the executor which `TaskID`(s) should run next.

So in `executor.rs` we define:
```rust
thread_local! {
    pub static NEXT_TASKID: RefCell<TaskId> = RefCell::new(0);
}
```
The waker will place 

Our current goal is straightforward: given a PID (returned from `waitid`) select the correct future from our `waiting_tasks: BTreeMap<TaskId, BoxFuture<'a, WaitStatus>>` to run. 

# Lasting Thoughts
As mentioned, this is an **introduction** to these concepts, and I myself am not an expert on futures. Here I list remaining questions and thoughts.

- **@withoutboats** mentions on [their Waker API blog](https://boats.gitlab.io/blog/post/wakers-i/),"One of the great things about Rust’s design is that you can combine different executors with different reactors, instead of being tied down to one runtime library for both pieces." The design presented tightly couples (for the worse), the executor and reactor. It is unclear to me what a refactored, uncoupled design would like.
- I have two "poll twice"

# Footnotes
[^1] I am using `futures-preview = "0.3.0-alpha.16"` found [here](https://github.com/rust-lang-nursery/futures-rs). It seems eventually this will become the default std implementation?

[^2] Linux's [`epoll` system call](http://man7.org/linux/man-pages/man7/epoll.7.html) covers polling on many IO sources, and is therefore general enough to be useful in most cases.

[^3] Rust futures are implemented as coroutines, for background on Rust coroutines check out [this](https://github.com/rust-lang/rust/issues/43122). See [generators](https://doc.rust-lang.org/beta/unstable-book/language-features/generators.html) for a brief introduction to `yield`.

[^4] A real implementation probably wants to allow for dynamically adding new futures _from_ running futures, this is certainly a possible extension, but beyond the scope of this tutorial.

## References
#### [1] [Future by example](https://paulkernfeld.com/2018/01/20/future-by-example.html) {#r1}
#### [2] [Async, futures, and tokio - Rust Crash Course lesson 7](https://www.snoyman.com/blog/2018/12/rust-crash-course-07-async-futures-tokio) {#r2}
#### [3] [_This_ blog's code repository](https://github.com/gatoWololo/RustFutureExample){#r3}
#### [4] [Farenheit](https://github.com/polachok/fahrenheit) {#r4}
#### [5] [Rust Async Book](https://rust-lang.github.io/async-book/) {#r5}
#### [6] [Embedded Executor](https://josh.robsonchase.com/embedded-executor/) {#r6}
#### [7] [Tokio Docs: Going Deeper: futures](https://tokio.rs/docs/going-deeper/futures/) {#r7}
#### [8] [Tokio Docs: Runtime Model](https://tokio.rs/docs/going-deeper/runtime-model/) {#r8}
#### [9] [Wakers](https://boats.gitlab.io/blog/post/wakers-i/) {#r9}
#### [10] [Futures 0.3: how does Waker and Executor connect?](https://www.reddit.com/r/rust/comments/anu8w4/futures_03_how_does_waker_and_executor_connect/) {#r10}
#### [11] [TODO](https://www.reddit.com/r/rust/comments/90g9ln/asyncawaitfuturesexecutor_without_external_crates/) {#r11}
#### [12] [Tokio](https://tokio.rs/) {#r12}
#### [13] [Mio]((https://github.com/tokio-rs/mio) {#r13}
#### [14] [The future trait](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/prelude/trait.Future.html) {#r14}
#### [15] [Nix Crate](https://docs.rs/nix/0.14.1/nix/) {#r15}
#### [16] [Wait Status](https://docs.rs/nix/0.14.1/nix/sys/wait/enum.WaitStatus.html) {#r16}
#### [17] [Arbitrary Self Type](https://github.com/rust-lang/rust/issues/44874) {#r17}
#### [18] [Pin Documentation](https://doc.rust-lang.org/std/pin/index.html) {#r18}
#### [19] [TODO Pin RFC?](https://github.com/rust-lang/rust/issues/49150) {#r19}
#### [20] [Withoutboats: Pin](https://boats.gitlab.io/blog/post/2018-01-25-async-i-self-referential-structs/) {#r20}
#### [21] [Nix: Waitpid](https://docs.rs/nix/0.14.1/nix/sys/wait/fn.waitpid.html) {#r21}
#### [22] [Pin::new()](https://doc.rust-lang.org/nightly/std/pin/struct.Pin.html#method.new) {#r22}
#### [23] [Futures Context](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/task/struct.Context.html) {#r23}
#### [] []() {#r}
#### [] []() {#r}
#### [] []() {#r}
#### [] []() {#r}

#### [] []() {#r}
