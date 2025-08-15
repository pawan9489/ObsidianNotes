## 2x2 Concurrency Matrix

To capture the interplay between the two most important pivots (Task Nature and Memory Model), we can use a 2x2 matrix. This shows how different tools and patterns fit into each quadrant.

| .                                | **Shared State**  <br>(Communicate by sharing memory)                                                                                                                                                                                                                                                | **Message Passing**  <br>(Share memory by communicating)                                                                                                                                                                                                                                                                   |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CPU-Bound**  <br>(Parallelism) | **Quadrant 1: Parallel Computation**  <br>Multiple threads work on the same complex data structure.  <br>**Tools:** `std::thread` + `Arc<Mutex<T>>` for complex data, `rayon` with `Mutex`es. **`Arc<AtomicUsize>`** (for simple counters/flags),<br> **Use Case:** In-memory scientific simulation. | **Quadrant 2: Data Flow Pipelines**  <br>Raw data is fed to a pool of worker threads; results are collected.  <br>**Tools:** `std::thread` + `std::sync::mpsc::channel`. <br> **Use Case:** A multi-threaded video encoder.                                                                                                |
| **I/O-Bound**  <br>(Async)       | **Quadrant 3: Cached/Shared Resources**  <br>Many async tasks need access to a shared resource, like a connection pool or in-memory cache.  <br>**Tools:** `tokio::spawn` + `Arc<Mutex<T>>`. <br> **Use Case:** A web server with a shared state object.                                             | **Quadrant 4: Service-Oriented / Actor Systems**  <br>Independent services or actors handle requests without sharing state. The "purest" async model.  <br>**Tools:** `tokio::spawn` + `tokio::sync::channel`, Actor frameworks like `actix`. <br> **Use Case:** A microservices architecture, a high-traffic chat server. |
This matrix provides a much more nuanced view. It shows that `Mutex` isn't just for threads, and channels aren't just for async. It forces you to think about both the **nature of the work** and the **architecture of your data flow** to choose the right combination of tools for the job.
### The Concurrency Toolbox

#### Locks (For Shared State)

- **`std::sync::Mutex<T>`**
    - **What:** A standard, blocking mutual exclusion lock.
    - **Use Case:** The general-purpose lock for protecting data in multi-threaded, CPU-bound contexts. The thread will sleep until the lock is available.
- **`std::sync::RwLock<T>`**
    - **What:** A standard, blocking Read-Write lock.
    - **Use Case:** For the "Many Readers" problem in CPU-bound contexts. Allows unlimited concurrent readers OR one exclusive writer.
- **std::sync::atomic Module (AtomicUsize, AtomicBool, etc.)**
    - **What:** Primitive types that can be safely modified across threads without a mutex. Operations are performed via indivisible hardware instructions.
    - **Use Case:** High-performance, low-contention scenarios for simple data. Perfect for shared counters, flags, or sequence numbers. Use when the overhead of a Mutex is too high for a very simple, frequent operation. Avoid for complex logic, which can lead to inefficient spin-locking.
- **`tokio::sync::Mutex<T>`**
    - **What:** An async-aware mutex.
    - **Use Case:** Must be used in async code when a lock might be held across an `.await` point. It yields control back to the async runtime instead of blocking the OS thread.
- **`tokio::sync::RwLock<T>`**
    - **What:** An async-aware Read-Write lock.
    - **Use Case:** The async solution to the "Many Readers" problem. Ideal for a shared cache in a web server.
---
#### Channels (For Message Passing)
- **`std::sync::mpsc::channel`**
    - **What:** A standard, blocking, Multi-Producer, **Single-Consumer** (MPSC) channel.
    - **Use Case:** The basic channel included with Rust. Good for simple cases where one thread collects all results.
- **`crossbeam_channel::unbounded` or `::bounded`**
    - **What:** A high-performance, blocking, **Multi-Producer, Multi-Consumer** (MPMC) channel.
    - **Use Case:** A powerful replacement for `std::sync::mpsc`. Use this in CPU-bound contexts when you need multiple workers to both send and receive from the same channel, or when you need higher performance.
- **`tokio::sync::mpsc::channel`**
    - **What:** An async, bounded, Multi-Producer, Single-Consumer (MPSC) channel.
    - **Use Case:** A work queue for a specific async task that processes items one by one.
- **`tokio::sync::broadcast::channel`**
    - **What:** An async, bounded, Multi-Producer, Multi-Consumer (MPMC) channel where every consumer sees every message.
    - **Use Case:** Broadcasting events to all interested listeners, like a shutdown signal or live state updates.
- **`tokio::sync::oneshot::channel`**
    - **What:** An async, Single-Producer, Single-Consumer channel for sending a **single value**.
    - **Use Case:** The perfect tool for getting a result back from a spawned task. Essential for the "CPU Offload" pattern.
---
### The Decision-Making Concurrency Questionnaire 🤔

#### Step 1: What is the nature of your task?
- **Question:** Is your code's performance limited by **CPU calculations** or by waiting for **external I/O**?
    - **A) CPU-Bound** → You need **Parallelism**. Your path starts with `std::thread`. Proceed to Step 2.
    - **B) I/O-Bound** → You need **Asynchronous Concurrency**. Your path starts with `tokio::spawn`. Proceed to Step 2.
---
#### Step 2: How will your tasks access data?
- **Question:** Do your tasks need to operate on one large, central piece of data, or can the work be broken down into independent, self-contained chunks?
    - **A) Central Data** → Your approach is **Shared State**. Proceed to Step 3.
    - **B) Independent Chunks** → Your approach is **Message Passing**. Proceed to Step 3.
---
#### Step 3: How will your tasks interact?
- _(If you chose Shared State in Step 2)_
	- **Question:** What kind of data are you protecting?
	    - **A) A simple primitive type (u64, bool, etc.) for simple operations (incrementing, setting a flag)?** → Your best tool is an **Atomic** type (e.g., std::sync::atomic::AtomicUsize).
	    - **B) A complex data structure (Vec, HashMap, a custom struct)?** → Proceed to the next question.
	        - **Question:** Is this shared data read much more often than it is written to?
	            - **Yes (Read-Mostly)** → Your primary tool is a **RwLock**.
	            - **No (Frequent Writes)** → Your primary tool is a **Mutex**.
- _(If you chose Message Passing in Step 2)_
    - **Question:** What is your communication pattern?
        - **A) Distributing many jobs to one or more workers?** → Use an **`mpsc`** or **`crossbeam`** channel.
        - **B) Getting a single result back from one task?** → Use a **`oneshot`** channel.
        - **C) Sending the same event to all listeners?** → Use a **`broadcast`** channel.
---
### Final Recommendation ✅
Based on your answers, here is your recommended starting point:
- **1A + 2A (CPU-Bound + Shared State) → Quadrant 1**
    - **Tools:** `std::thread` combined with...
	    - `Arc<std::sync::Mutex<T>> or Arc<std::sync::RwLock<T>>` for complex data.
		- `Arc<std::sync::atomic::AtomicUsize>` for simple, high-performance counters or flags.
- **1A + 2B (CPU-Bound + Message Passing) → Quadrant 2**
    - **Tools:** `std::thread` combined with **`crossbeam_channel`** (for MPMC) or **`std::sync::mpsc`** (for MPSC).
- **1B + 2A (I/O-Bound + Shared State) → Quadrant 3**
    - **Tools:** `tokio::spawn` combined with `Arc<tokio::sync::Mutex<T>>` or `Arc<tokio::sync::RwLock<T>>`.
- **1B + 2B (I/O-Bound + Message Passing) → Quadrant 4**
    - **Tools:** `tokio::spawn` combined with `tokio::sync` channels (`mpsc`, `oneshot`, `broadcast`).

## Sample codes

### **Notes: CPU-Bound Quadrants**

This section covers parallel processing for computationally intensive tasks that are limited by CPU speed, not by waiting for external resources like networks or disks.

#### **Core Philosophy: The Two Models**
1. **Quadrant 2: Message Passing ("Share Memory by Communicating")**
    - **Concept:** Threads are isolated and do not touch the same data directly. They transfer ownership of data by sending messages over channels. This is often easier to reason about and less prone to deadlocks.
    - **Analogy:** A team of people sending emails. Once you send the email (the data), you don't edit it anymore; it belongs to the receiver.
2. **Quadrant 1: Shared State ("Communicate by Sharing Memory")**
    - **Concept:** Multiple threads have access to the same piece of memory. To prevent chaos and data races, access is controlled via synchronization primitives like Mutexes or Atomics.
    - **Analogy:** A team of people editing the same document on a shared whiteboard. Only one person can write at a time (holding the marker/lock).
### **Quadrant 1 - Shared State in Practice**
- **Primary Tools:**
    - `std::sync::Mutex<T>`: A "sleeping" lock that uses the OS to ensure mutual exclusion. It's the general-purpose tool for protecting any complex data T.
    - `std::sync::atomic`: "Lock-free" primitives (AtomicU32, AtomicBool, etc.) that use special CPU instructions for unparalleled performance on simple types.
    - `std::sync::Arc<T>`: Atomic Reference Counting. Essential for giving multiple threads shared ownership of a Mutex or Atomic.
- **Key Challenge & Solution (Shared Counter):**
    - **Problem:** Have 10 threads each increment a counter 100 times. Final result must be 1000.
    - **Mutex Solution:**
        1. Create state: let counter = Arc::new(Mutex::new(0));
        2. In each thread, loop 100 times.
        3. Inside the loop, the "critical section" is:
		    ```rust
            let mut guard = counter.lock().unwrap(); // Acquire lock
            *guard += 1;                           // Mutate data
            // `guard` is dropped here, automatically releasing the lock
            ```
    - **Atomic Solution (Better for this case):**
        1. Create state: let counter = Arc::new(AtomicU32::new(0));
        2. In each thread, call the atomic operation:
            ```rust
            counter.fetch_add(1, Ordering::SeqCst);
            ```
        3. Read the final result with counter.load(Ordering::SeqCst).
- **My Questions & The Answers:**
    1. **Why does Mutex have "overhead"?**
        - It involves **system calls** to the OS kernel to manage the lock state and put waiting threads to sleep. This process (context switching) is expensive compared to a simple CPU instruction.
    2. **What does "lock-free" (Atomic) mean?**
        - It means free of OS-level locks. The operation is guaranteed to be indivisible ("atomic") by a **special CPU hardware instruction**. It's much faster but only works for primitive types that fit in a CPU word (like u32, bool, pointers). Threads don't "sleep"; they may "spin" (busy-wait) for a nanosecond if there's contention, which is handled by the hardware memory bus.
    3. **Why no generic `Atomic<T>`?**
        - Because CPUs don't have instructions to atomically modify arbitrary, complex data structures (T). They only have instructions for simple, word-sized primitives. Mutex solves this by not touching the data, but rather guarding the gate to the data.
    4. **Can Mutex be used without Arc?**
        - Yes. Arc is for shared ownership across threads that may outlive the parent function. If you can guarantee the Mutex will live long enough (e.g., using std::thread::scope), you can just borrow it (`&Mutex<T>`) instead.
    5. **Why do compilers/CPUs reorder instructions?**
        - For **performance**. They hide the latency of slow operations (like memory access) by executing other independent instructions in the meantime. Memory Ordering is our way to tell them when this reordering is not safe because another thread depends on the order. SeqCst is the strictest, safest "don't reorder" rule.
```rust
use std::thread;
use std::sync::{Arc, Mutex, MutexGuard};

fn main() {
    let counter: Arc<Mutex<u32>> = Arc::new(Mutex::new(0));
    let mut join_handles: Vec<thread::JoinHandle<()>> = vec![];
    
    for _ in 0..10 {
        let counter_clone = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..100 {
                let mut n: MutexGuard<u32> = counter_clone.lock().unwrap();
                *n += 1;
            }
        });
        join_handles.push(handle);
    }
    
    for handle in join_handles {
        let _ = handle.join();
    }
    
    println!("{}", counter.lock().unwrap()); // 1000
}

//// Atomics for primitives types sharing, locks have overhead
use std::thread;
use std::sync::{Arc};
use std::sync::atomic::{AtomicU32, Ordering};

fn main() {
    let counter: Arc<AtomicU32> = Arc::new(AtomicU32::new(0));
    let mut join_handles: Vec<thread::JoinHandle<()>> = vec![];
    
    for _ in 0..10 {
        let counter_clone = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..100 {
                counter_clone.fetch_add(1, Ordering::SeqCst);
            }
        });
        join_handles.push(handle);
    }
    
    for handle in join_handles {
        let _ = handle.join();
    }
    
    println!("{}", counter.load(Ordering::SeqCst)); // 1000
}
```

### **Quadrant 2 - Message Passing in Practice**
- **Primary Tools:**
    - `std::thread::spawn`: Creates a new OS thread. The closure passed to it **must** be move to take ownership of its environment.
    - `std::sync::mpsc::channel`: Creates a **M**ultiple **P**roducer, **S**ingle **C**onsumer channel. Returns a (Sender, Receiver) tuple.
	    - `let (sender, receiver) = channel();`
	    - General terminology `let (tx, rx) = channel();` - `tx - Transmission`, `rx - Receive`.
- **Key Challenge & Solution (Worker Pool Pattern):**
    - **Problem:** An mpsc channel has only one Receiver. How can multiple worker threads all receive jobs from the same source?
    - **Solution:** The single Receiver must be shared. We wrap it in `Arc<Mutex<T>>`.
        - Arc: Allows multiple threads to have a shared ownership handle to the Mutex.
        - Mutex: Ensures only one worker thread can `call .recv()` at a time.
    - **My Implementation:**
        1. Main thread creates job_channel and result_channel.
        2. job_rx is wrapped: `let job_rx = Arc::new(Mutex::new(job_rx));`
        3. Spawn workers, giving each a clone() of the `Arc<Mutex<job_rx>>` and a clone() of the result_tx.
        4. **Graceful Shutdown:** Main thread sends all jobs, then calls `drop(job_tx)`. This closes the channel.
        5. Worker threads loop, calling `job_rx.lock().unwrap().recv()`. When it returns Err, it means the channel is closed, and the thread breaks its loop and terminates.
- **Advanced Insight (Performance): Minimize Lock Contention**
    - **Mistake:** Holding a Mutex lock for longer than necessary. In my first implementation, the lock was held while the worker was "processing" the job (thread::sleep).
    - **Optimization:** Hold the lock only to perform the atomic action (.recv()). Release it immediately, then do the heavy processing.
```rust
use std::thread;
use std::sync::mpsc::channel;
use std::sync::{Arc, Mutex};
use std::time::Duration;

const NUM_WORKERS: u32 = 4;
const NUM_JOBS: u32 = 20;

fn main() {
    let (job_send, job_rcv) = channel::<u32>();
    let (result_send, result_rcv) = channel::<u32>();
    let job_rcv = Arc::new(Mutex::new(job_rcv));
    let mut job_handles: Vec<thread::JoinHandle<()>> = vec![];
    
    // Workers receiving job & sending the result
    for i in 1..=NUM_WORKERS {
        let job_rcv_clone = Arc::clone(&job_rcv);
        let result_send_clone = result_send.clone();
        let join_handle = thread::spawn(move || {
            loop {
                let job_rcv = {
                    job_rcv_clone.lock().unwrap().recv()
                }; // Lock & get data & unlock immediately
                match job_rcv {
                    Ok(n) => {
                        println!("Worker {i} got job {n}");
                        thread::sleep(Duration::from_millis(50));
                        let _ = result_send_clone.send(n * 2);
                    },
                    Err(..) => break,
                }
            }
        });
        job_handles.push(join_handle);
    }
    
    // Main thread sending the job info
    for i in 1..=NUM_JOBS {
        let _ = job_send.send(i);
    }
    // Sending the Err, to notify the jobs are done.
    drop(job_send);
    
    // Main thread receving the result
    for _ in 1..=NUM_JOBS {
        println!("\t{:?}", result_rcv.recv());
    }
    
    // Wait for all threads to complete.
    for handle in job_handles {
        let _ = handle.join();
    }
}
```

### **I/O-Bound Quadrants (Async Rust)**
This section covers concurrency for tasks that are limited by waiting for external operations (I/O), such as network requests, file system access, or database queries. The goal is to handle many concurrent waiting tasks efficiently without wasting threads.
#### **Core Philosophy: Async - "Waiting Without Wasting"**
- **Concept:** Instead of blocking an entire OS thread while waiting, an async task yields control back to a scheduler. This scheduler, called a **runtime** (e.g., tokio), can then use the now-free thread to run other tasks that are ready to do work. 
- **Key Primitives:**
    - async fn: A function that returns a Future (a state machine representing a value that may not be ready yet). Futures are lazy and do nothing until polled.
    - .await: The operator used inside an async fn to pause the current task and wait for another Future to complete, allowing the runtime to execute other tasks.
    - **Runtime (tokio)**: The engine that polls Futures, manages waking them up, and schedules them on a small pool of worker threads. Your main function must be designated with an attribute like `#[tokio::main]`.
- **Critical Insight: Spawning for Concurrency**
    - Simply calling .await on multiple futures in a row results in **sequential**, not concurrent, execution.
    - To run futures concurrently, they must be handed off to the runtime's scheduler using a mechanism like tokio::spawn. This creates a new, independent task that the runtime manages in the background.
### **Quadrant 4 - Async with Message Passing**

- **Concept:** Tasks are designed to be highly independent, like miniature services or actors. They don't share state; they communicate by receiving data to begin their work and returning data when they are done.
- **"Message Passing" Styles:**
    1. **Implicit/Conceptual:** The function call itself is the "in" message (the arguments), and the return value is the "out" message. This is the most common style.
    2. **Explicit:** Using `tokio::sync::channel` to create an async-aware channel. This is used for building more formal actor systems where a long-lived task processes a stream of incoming message-enums.
- **Key Challenge & Solution (Managing Concurrent Tasks):**
    - **Problem:** How to launch many concurrent downloads and gather their results.
    - **Solution: tokio::task::JoinSet**: This is the modern, idiomatic tool for this pattern.
        1. Create a JoinSet.
        2. Use `set.spawn(my_async_fn())` to add tasks to the set.
        3. Use a while `let Some(result) = set.join_next().await` loop to process results **as they complete**. This is more responsive than waiting for all tasks to finish.
    - **My Implementation:** Created a download(url) async function. Spawned one for each URL into a JoinSet. The join_next loop then processed the content of each download as soon as it became available.
- **Critical Insight: The JoinHandle "Leash"**
    - A tokio runtime will shut down when its main task completes. If you spawn tasks but don't .await their JoinHandles (either directly or via a JoinSet), the main task will finish, and the runtime will shut down, likely aborting your spawned tasks before they can complete. Awaiting a handle creates a dependency that keeps the runtime alive.
```rust
use std::time::Duration;
use tokio::task;
use tokio::time;

async fn download(url: &str) -> String {
    println!("Starting download for {url}");
    let sleep_duration = rand::random_range(1..=5);
    time::sleep(Duration::from_secs(sleep_duration)).await;
    println!("Finished download for {url}. Took {sleep_duration} sec(s).");
    format!("{url} content")
}

#[tokio::main]
async fn main() {
    let urls = vec!["url1", "url2", "url3"];
    let mut set = task::JoinSet::<String>::new();
    
    for url in urls {
        set.spawn(async move {
            download(url).await
        });
    }
    
    // let output = set.join_all().await; // Waits for all to complete
    while let Some(result) = set.join_next().await { // Prints results as they complete 
        match result {
            // The task completed successfully.
            Ok(content) => {
                println!("Processed result: '{}'", content);
            }
            // The task failed.
            Err(e) => {
                // You can check if the task panicked
                if e.is_panic() {
                    println!("A task panicked!");
                } else {
                    println!("A task failed to complete: {}", e);
                }
            }
        }
    }
    println!("\n--- All tasks complete ---");
}
```
### **Quadrant 3 - Async with Shared State**
- **Concept:** Multiple async tasks need to access the same shared data (e.g., a connection pool, in-memory cache). This is the intersection of the async world and the `Arc<Mutex<T>>` world.
- **The Critical Rule: Use the Right Mutex**
    - **NEVER** `use std::sync::Mutex` if the lock will be held across an .await point.
    - **Why?** A standard Mutex blocks the OS thread. If an async task holds this lock and then .awaits, the thread is put to sleep by the OS and is no longer available to the async runtime. This can starve the runtime of worker threads and easily lead to a complete deadlock.
    - **Solution: tokio::sync::Mutex**: This is an async-aware mutex. When its .lock().await method is called on a contested lock, it does **not** block the thread. Instead, it cooperates with the async scheduler, suspending the task and allowing the thread to be used for other work.
- **Key Challenge & Solution (Async Shared Counter):**
    - **Problem:** Simulate many concurrent web requests incrementing a shared page-view counter.
    - **Solution:** Combine the patterns.
        1. The state is `Arc<tokio::sync::Mutex<u32>>`.
        2. Create an async fn handle_request that takes a clone of the Arc.
        3. Inside the function, the critical section is:
            ```rust
            let mut num = counter.lock().await; // Async lock acquisition
            *num += 1;
            // Async lock is released when `num` is dropped
            ```
        4. In main, use a JoinSet to spawn 100 handle_request tasks, then wait for them all to complete.
        5. The final result is correct because the tokio::sync::Mutex ensures that the *num += 1 operation is atomic with respect to other tasks.
```rust
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::Mutex;
use tokio::task;
use tokio::time;

async fn handle_request(counter: Arc<tokio::sync::Mutex<u32>>) {
    time::sleep(Duration::from_secs(1)).await;
    let mut num = counter.lock().await;
    *num += 1;
}

#[tokio::main]
async fn main() {
    let counter: Arc<Mutex<u32>> = Arc::new(Mutex::new(0));
    let mut set = task::JoinSet::<()>::new();
    
    for _ in 0..100 {
        let counter_clone = Arc::clone(&counter);
        set.spawn(async move {
            handle_request(counter_clone).await
        });
    }
    
    let _ = set.join_all().await;
    
    println!("Final result {}", counter.lock().await);
}
```

## Rayon
`rayon` is a data-parallelism library. Its philosophy is: "If your code works on an iterator, it can be made parallel with a one-line change." It's designed to be a nearly drop-in replacement for `std::iter::Iterator`.

It achieves this through a clever **work-stealing** thread pool, much like Tokio's, but optimized for CPU-bound tasks. When you convert an iterator into a parallel iterator, rayon splits the work into smaller and smaller chunks until it has enough pieces to keep all your CPU cores busy. If one core finishes its work early, it "steals" a chunk of work from another, busier core.

**The Key Trait:** `rayon::iter::ParallelIterator`.  
**The Key Method:** `.par_iter() (or .par_iter_mut(), .into_par_iter())`. You call this on a collection to turn it into a parallel iterator. From there, you can use most of the same adaptors you already know: `map, filter, fold, reduce, collect,` etc.

However, some iterator methods are **inherently sequential** and cannot be meaningfully parallelized.
- The most obvious example is **.next()**. A standard iterator produces one item at a time. A parallel iterator operates on chunks of data at once; there is no single "next" item to produce.
- Other sequential methods like **.scan()** (which relies on state from the previous iteration) or **.try_fold()** (which must process elements in order to short-circuit) don't have direct parallel equivalents because their logic is fundamentally sequential.

**The Rule of Thumb:** If an operation can be broken down into independent chunks of work that can be combined later, rayon almost certainly has a parallel version of it. If the operation relies on a specific, strict, element-by-element order, it probably doesn't.

```rust
use rayon::prelude::*;
use std::time::Instant;

const N: u32 = 2_000_000;

pub fn is_prime(n: u32) -> bool {
    if n < 2 { return false; }
    for i in 2..=n.isqrt() {
        if n % i == 0 { return false; }
    }
    true
}

fn main() {
    let now = Instant::now();

    let primes = (2..=N)
        .into_iter()
        .filter(|i| is_prime(*i))
        .count();
        
    println!("{} ms, len: {}", now.elapsed().as_millis(), primes);
    
    let now = Instant::now();

    let primes = (2..=N)
        .into_par_iter()
        .filter(|i| is_prime(*i))
        .count();
        
    println!("{} ms, len: {}", now.elapsed().as_millis(), primes);
}
```