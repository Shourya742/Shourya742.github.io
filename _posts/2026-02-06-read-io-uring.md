---
layout: "post"
title:  "Implement asynchronous random file reading based on io_using in Rust"
date:   "2050-02-06 14:23:35 +0530"
categories: io_uring
--- 


This article introduces `io_uring` the basic usage, then describes the implementation of the asynchronous file reading library.

## Introduction to io_uring


`io_uring`: It is an asynchronous I/O interface provided by the linux kernel. It was introduced in Linux 5.1 in May 2019 and is now used in various project. For example:

* RocksDB's MultiRead currently performs `io_uring` concurrent file reads.
* Tokio `io_uring` wraps an API. With the release of Tokio 1.0, the developers stated that true asynchronous file operations would be provided via io_uring. Currently, Tokio's asynchronous file operations are implemented by calling synchronous APIs from a separate I/O thread.
* QEMU 5.0 is already in use io_uring.

Most current `io_uring` tests on Direct I/O compare its performance with Linux AIO. `io_uring` can usually achieve twice the performance of AIO.

## Random file reading scenarios

In database systems, we often need to read content from arbitrary positions in a file using multiple threads (<fid>, <offset>, <size>). Commonly used `read/write` APIs cannot accomplish this (because they require seek operation, which necessitates exclusive access to the file handle). The following method can achieve random file reads.

* By `mmap` directly mapping the file into memory, reading the file becomes directly reading memory, allowing concurrent reading in multiple threads.
* `pread offset`: It can read `count` bytes starting from a certain position, and also supports multi-threaded concurrent reading.

However, both of these approaches will block the current thread. For example, if `mmap` a page fault occurs while reading a block of memory, the current thread will be blocked; `pread` is inherently a blocking API. Asynchronous APIs (such as Linux AIO, io_uring) can reduce context switching, thereby improving throughput in certain scenarios.

## Basic usage of io_uring

`Liburing` provides a more user-friendly API. Tokio's io_uring_crate, build upon this, provides rust `io_uring` API. The following example demonstrates `io_uring` its usage.

To use it `io_uring`, you need to create a ring first. Here, we use the `tokio-rs/io-uring` provided concurrent API, which supports multiple threads using the same ring.

```
use io_uring::IoUring;
let ring = IoUring::new(256)?;
let ring = ring.concurrent();
```
Each ring corresponds to a submission queue and a completion queue. Here, the queues are set to hold a maximum of 256 elements.

The process of `io_uring` performing I/O operations consist of three steps: adding the task to the submission queue, submitting the task to the kernel and retrieving the task from the completion queue. This section uses reading a file as an example to illustrate the entire process.

A file reading task can be constructed using `opcode::Read` and `ring.submission().push(entry)` the task can be added to a queue using `opcode::Read`.

```rust
use io_uring::{opcode, types::Fixed};
let read_op = opcode::Read::new(Fixed(fid), ptr, len).offset(offset);
let entry = read_op.build().user_data(user_data);
unsafe {ring.submission().push(entry)?;}
```

Once the task is added, submit it to the kernel.

```rust
assert_eq!(ring.submit()?, 1);
```

The last polling task has been completed.

```rust
loop {
    if let Some(entry) = ring.completion().pop() {
        // do something
    }
}
```

In this way, we have achieved random file reading based on `io_uring`.

Note 1: There are currently three execution modes: default mode, poll mode, and kernel poll mode. If using kernel poll mode, it is not necessary to call the function to submit the task.

## Implement asynchronous file reading interface using io_uring

Our goal is to implement an interface like this, `io_uring` wrapping it up and exposing only a simple `read` function to the developer.

```rust
ctx.read(fid, offset, &mut buf).await?
```

After referencing tokio-linux-aio's asynchronous wrapper for Linux AIO, I adopted the following method to implement `io_uring` asynchronous reads based on 

* Developers `io_uring` need to create one before using it `UringContext`.
* `UringContext` At the same time it is created, one (or more) processes will run in the background to submit tasks and poll for task completion `UringPoolFuture` (this corresponds to the second and third steps of reading files).
* Developers can create a file by `ctx` calling the file reading interface. After calling: `ctx.read -> UringReadFuture -> ctx.read.await`
1. `UringReadFuture`: It creates an object that is permanently stored in memory `UringTask`, then puts the file read task into a queue, using `UringTask` the address of the object as the user data for the read operation.  `UringTask` this involves a channel.
2. `UringPollFuture`: Submit the task in the background.
3. `UringPollFuture`: Poll the background for completed tasks.
4. `UringPollFuture`: Extract the user data, restore it to an object, and notify the I/O operation is complete `UringTask` via a channel `UringReadFuture`.


This allows us to easily implement `io_uring` asynchronous file reading. It also brings the added benefit of automatic task batching. Normally, a single I/O operation generates one syscall. However, since we use a separate Future to submit and poll tasks, multiple unsubmitted tasks may exist in the queue at the time of submission, which can be submitted all at once. This reduces the overhead of syscall context switching (though it also increases latency). 