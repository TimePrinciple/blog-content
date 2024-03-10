---
# Must present
author: Ruoqing He
title: Rust Tokio
# Optional
date: 2024-02-28
summary: An exploration of Rust Tokio.
---

## Preface

When it comes to multi-threaded programming, there are times we need to deliberately trigger *task-switching*, for the newly spawned threads (synchronous or not) to possess CPU. Generally we are taught to invoke `std::thread::sleep` for a estimated time interval, a interval is expected to cover the execution of other spawned threads. Additionally, the reason of calling sleep here are two:

1. Force the current thread (the thread who spawned other threads through `std::thread::spawn`) to give up its dominance on CPU.
2. Prevents the thread comes back too quickly (e.g. the next round of scheduling), it would result either *busy waiting* (if a loop is set to query the status of spawned threads) or early exiting of main thread (if there is not even a loop, the spawned threads may well not have sufficient time to complete its work but got killed).

The *busy waiting* here suggest it would be a wast if we just `sleep` for a very short interval, and the tasks remained in the *ready queue* of scheduler are few, the `sleep` would get called many times, the wasteful time used for *task-switching* would grow exponentially with respect to the time interval you have chosen. An extreme example is `std::thread::yield_now` which gives up a time slice and immediately moves the task to the back of *ready queue*, it would be *busy waiting* literally.

Then what happens if we `sleep` for a very long time, maybe tenfold of the sum of execution time need by spawned threads. This time, these threads may surely have their jobs finished, but the time takes for the main thread to return is too long, it would not be acceptable for the user, and pointless for we programmers. What `sleep` does is putting the thread to a *blocking queue* for a certain period of time, the tasks in *blocking queue* are not options for CPU until the event they expect (time is up in this case) has happened.

All in all, using `sleep` for places like *task-switching* is not good for readability and even produces cognitive load. Needless to say that the `Duration` is hard to estimate, the word `sleep` seems not related in this context. Methods like `yield_now`, `join`, communication through `channel`, or synchronization by `Mutex` are way better than `sleep`.

## tokio Runtime

To build an asynchronous application, a runtime with following supports are required:
- A **I/O event loop**, called the driver, which drives I/O resources and dispatches I/O events to *tasks* that depend on them.
- A **scheduler** to execute tasks that use these I/O resources.
- A **timer** for scheduling work to run after a set period of time.

### Create tokio Runtime

To use tokio, the asynchronous runtime has to be created first, for the asynchronous tasks to be executed.

The `Runtime` structure si defined in `rt` (abbreviation of `runtime`), so the `rt` feature has to be enabled (either with `cargo add tokio --features rt` or a modification in `Cargo.toml`).

```rust
use tokio::runtime;

fn main() {
    let rt = runtime::Builder::new_multi_thread()
        // Set number of worker threads, 10 is the default value used by macro tokio::main
        .worker_threads(10)
        // enable_all() is an equivalence of enable_io() and enable_time()
        // Enable I/O driver, requires either `net`, `process`, `signal` feature
        .enable_io()
        // Enable time driver, requires `time` feature
        .enable_time()
        .build()
        .unwrap();
    // block the main thread for 10 seconds
    // which keeps workers alive in this period
    std::thread::sleep(std::time::Duration::from_secs(10));
}
```

with this program running, the number of threads could be examined by issuing

```bash
# The [t] here excludes the grep process itself
$ ps -eLf | grep 'targe[t]'
```

which might produce the following output or something similar:

```bash
TimePri+   16101   14864   16101  2   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16103  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16104  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16105  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16106  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16107  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16108  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16109  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16110  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16111  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
TimePri+   16101   14864   16112  0   11 16:49 pts/7    00:00:00 target/debug/explore_tokio
```

here, the number of threads amounts to 11, with 1 thread being the main thread.

Additionally, the `Runtime`s needs a thread to manage them (for their nature of being structures in Rust, which implies that a `drop` on them will lead to shutdown), so they can naturally be used with `std::thread` to create multiple `Runtime`s.

### Execute tasks in Runtime

When `Runtime` is set, the asynchronous tasks now have environments to get executed. Most of the time, the async tasks are network associated, like async http requests.

#### Context Switching

To execute tasks, we'll need to be in an async context.

## tokio Task Communication

*Message passing* is the primarily used strategy between the asynchronous tasks of tokio. It could be generally understood as there is a sender task and an according receiver task during the communication. The benefits of message passing are avoid data sharing between concurrent tasks which eliminates data racing.

`channel` is used for message passing. tokio offers different kinds of `channel`s:

- `oneshot`: One to one. Only one message is allow to be sent.
- `mpsc`: Many to one. There could be multiple senders (producer), but only one receiver (consumer).
- `broadcast`: Many to many. There could be multiple senders, and multiple receivers.
- `watch`: One to many. There could be only one sender, but multiple receivers (subscriber).

## tokio Task Synchronization

When writing asynchronous tasks, there are times we need to inspect the status between tasks. tokio offers the following synchronization primitives:

- Mutex: operates on a guaranteed FIFO basis. This means that the order in which tasks call the `lock` method is the exact order in which they will acquire the lock.
- RwLock: allows a number of readers or at most one writer at any point in time. The priority policy is *fair* or (*write-preferring*), in order to ensure that readers cannot starve writes. Fairness is ensured using a **first-in, first-out** queue for the tasks awaiting the lock; if a task that wishes to acquire the write lock is at the head of the queue, read locks will not be given out until the write lock has been released. This is in contrast to the Rust standard library's `std::sync::RwLock`, where the priority policy is dependent on the operating system's implementation.
- Notify: provides a basic mechanism to notify a single task of an event. `Notify` itself does not carry any data. Instead, it it to be used to signal another task to perform an operation.
- Barrier: A barrier enables multiple tasks to synchronize the beginning of some computation.
- Semaphore: maintains a set of permits. Permits are used to synchronize access to a shared resource. A semaphore differs from a mutex in that it can allow more than one concurrent caller to access the shared resource at a time.
