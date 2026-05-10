# Fearless Concurrency

By this point in the book, you've seen how hard Rust works to keep your memory safe. Handling concurrent programming safely and efficiently is one of Rust's core goals — and it approaches it the same way.

Rust calls this **Fearless Concurrency**. The idea is simple: concurrency bugs should be compile-time errors, not runtime. Once your code compiles, you can be confident it's free from data races and invalid memory access, free from logical errors.

Rust achieves this using two tools you already know: **ownership** and the **type system**.

---

## Key Terms

**Process** — a running instance of a program. The OS manages multiple processes at once.

**Thread** — an independent unit of execution *within* a process. A single program can have multiple threads running simultaneously.

---

## Threads

### Why threads?

Running tasks concurrently using threads can improve performance — a web server handling multiple requests at once, for example. But threads come with well-known problems:

- **Race conditions** — two threads access shared data in an unpredictable order
- **Deadlocks** — two threads wait on each other forever, neither making progress
- **Heisenbugs** — bugs that only appear under very specific timing conditions, nearly impossible to reproduce

### The 1:1 model

Rust's standard library uses a **1:1 threading model**: one OS-level thread per language thread.

- An **OS-level thread** is a real thread managed by the kernel — it's what actually runs on the CPU.
- A **language thread** is what you create in code. In Rust's case, each one maps directly to one OS thread.

Some crates implement other threading models (M:N, green threads, etc.), each with different trade-offs. Rust's async system, covered in the next chapter, is another approach to concurrency built on a different model.

---

## Spawning Threads

### `thread::spawn`

To create a new thread, use `thread::spawn`. It takes a closure containing the code the new thread should run.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }
}
```

### `thread::sleep`

`thread::sleep` pauses the current thread for a given duration, giving other threads a chance to run. This is the basic mechanism behind concurrent execution.

> **Important:** When the main thread exits, all spawned threads are shut down immediately — even if they haven't finished running.

In the example above, the spawned thread tries to count to 9, but gets cut off when the main thread finishes at 5.

---

## Waiting for Threads to Finish

To make sure a spawned thread runs to completion, save the return value of `thread::spawn` — a `JoinHandle` — and call `.join()` on it.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap(); // waits here until the spawned thread finishes
}
```

Calling `.join()` blocks the current thread until the target thread finishes. Where you place it matters — if you call it before the main thread's work, the main thread just waits and the two don't run concurrently at all.

---

## Using `move` Closures with Threads

If a spawned thread needs data from the main thread, you have to use the `move` keyword.

Why?
 When you capture a variable from the outer scope inside a thread's closure, Rust infers that you want to borrow its reference by default. But Rust can't guarantee how long the spawned thread will live — the main thread could drop that value while the spawned thread is still using a reference to it. So Rust refuses to compile it.

```rust
// This won't compile
let v = vec![1, 2, 3];
let handle = thread::spawn(|| {
    println!("{v:?}"); // borrows v — Rust rejects this
});
```

The fix is `move`, it transfers ownership of `v` into the closure itself. Now the thread owns the data, and there's no risk of it being dropped out from under the thread.

```rust
let v = vec![1, 2, 3];
let handle = thread::spawn(move || {
    println!("{v:?}"); // v is now owned by this closure
});
handle.join().unwrap();
```

> **Note:** `move` copies the value *into* the closure. Changes made to `v` inside the spawned thread won't affect the original in the main thread. And since ownership has moved, you can't use `v` in the main thread at all after the `move`.

This is Rust's ownership system doing its job: the compiler ensures both threads can never have conflicting access to the same data.

---

## Message Passing
 
Another way to handle concurrency is **message passing** — threads communicate by sending data to each other instead of sharing it.
 
The core primitive for this in Rust is a **channel**.
 
### Channels
 
A channel is a one-way pipe between two threads: data goes in one end and comes out the other.
 
It has two halves:
- **Transmitter (`tx`)** — the sending end. A thread calls `.send()` on it to push data into the channel.
- **Receiver (`rx`)** — the receiving end. Another thread reads from it to get the data.
Rust's standard library implements channels via `std::sync::mpsc`.
> **mpsc** stands for *multiple producer, single consumer* — you can have multiple transmitters sending into the same channel, but only one receiver on the other end.
 
```rust
use std::sync::mpsc;
 
fn main() {
    let (tx, rx) = mpsc::channel();
}
```
 
`mpsc::channel()` returns a tuple — the first element is the transmitter, the second is the receiver.
 
### Sending and Receiving
 
Move the transmitter into a spawned thread and call `.send()` to pass a value across:
 
```rust
use std::sync::mpsc;
use std::thread;
 
fn main() {
    let (tx, rx) = mpsc::channel();
 
    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });
 
    let received = rx.recv().unwrap();
    println!("Got: {received}");
}
```
 
`tx.send(val)` returns `Result<T, E>` — it errors if the receiver has already been dropped.
 
On the receiving side, you have two options:
 
**`rx.recv()`** — blocks the current thread until a value arrives, then returns it as `Result<T, E>`. Returns an error when the channel closes (i.e. the transmitter is dropped).
 
**`rx.try_recv()`** — returns immediately regardless of whether a value is available. Returns `Ok(value)` if something is there, `Err` if not. Useful when the receiving thread has other work to do — you can call it periodically in a loop instead of sitting idle.
 
### Ownership Through Channels
 
When you send a value through a channel, ownership transfers to the receiver. Using the value after sending it won't compile:
 
```rust
thread::spawn(move || {
    let val = String::from("hi");
    tx.send(val).unwrap();
    println!("{val}"); // error: val was moved into the channel
});
```
 
This is intentional. Once a value is sent, the receiving thread owns it and may modify or drop it at any time. Using it on the sending side after that point would be unsafe. Rust catches this at compile time.
 
### Sending Multiple Values
 
A thread can send multiple values through a channel. On the receiving end, treating `rx` as an iterator will yield each value as it arrives and stop when the channel closes:
 
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
 
fn main() {
    let (tx, rx) = mpsc::channel();
 
    thread::spawn(move || {
        let vals = vec!["hi", "from", "the", "thread"];
        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });
 
    for received in rx {  // iterates until the channel closes
        println!("Got: {received}");
    }
}
```
 
### Multiple Producers
 
Since `mpsc` allows multiple producers, you can clone the transmitter and give each spawned thread its own copy — all feeding into the same receiver:
 
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
 
fn main() {
    let (tx, rx) = mpsc::channel();
    let tx2 = tx.clone();
 
    thread::spawn(move || {
        let vals = vec!["hi", "from", "the", "thread"];
        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });
 
    thread::spawn(move || {
        let vals = vec!["more", "messages", "for", "you"];
        for val in vals {
            tx2.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });
 
    for received in rx {
        println!("Got: {received}");
    }
}
```
 
Each thread gets its own transmitter clone. All messages converge at the single receiver. The order of output isn't guaranteed — that's concurrency.
 
