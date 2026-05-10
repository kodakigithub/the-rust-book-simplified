
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
