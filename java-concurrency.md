# Java Concurrency: Threads, Executors, and Synchronization

This document covers core Java concurrency topics from basic threads to advanced executors and synchronization. It follows key themes (as per the provided timeline) in a structured, interview-prep style. Each section includes concise explanations, examples, and diagrams to aid understanding.

## Processes vs Threads

 **Figure:** A *Process* (left) is an independent program with its own memory space, while a *Thread* (right) is a lightweight execution unit within a process sharing memory and resources.

- **Process:** Has its own **address space**, memory (heap, stacks), and resources. Context switches between processes are slow (heavyweight).
- **Thread:** Has its own stack and Thread ID but **shares heap and resources** with other threads in the same process. Context switching is fast (lightweight).
- **Communication:** Processes communicate via IPC mechanisms. Threads communicate faster via shared memory.
- **Overhead:** Creating/terminating a process is expensive; a thread is cheaper.
- **Isolation vs Sharing:** Blocking one process does not directly affect others; blocking a thread may block its process if not carefully managed.

In short, use **threads** for finer-grained parallelism within an application, and **processes** for isolation/fault-tolerance. Java supports OS-level threads (one Java thread per native thread).

## Thread Basics

- **What is a Thread?** A thread is a single sequential flow of control within a program. Multiple threads within a process can run concurrently on multi-core CPUs. Java was built with concurrency in mind: “The Java platform is designed from the ground up to support concurrent programming”.
- **Multi-core & Concurrency:** Modern CPUs have multiple cores; threads let Java programs utilize multiple cores for parallelism. Without threads, a program can only run on one core at a time.
- **Shared Memory:** Threads in the same process share the same heap and static data. This makes communication between threads fast (no IPC), but also requires careful synchronization.
- **When to Use Threads vs Processes:** Threads are great for lightweight tasks (e.g., handling user requests, background tasks). Processes (separate JVMs) are used for strong isolation or different security contexts. For high fault tolerance, isolate critical tasks in separate processes (so a thread crash doesn’t bring down everything).

## Creating and Running Threads

Java provides two main ways to define tasks for threads:

- **Extending `Thread`:** Subclass `Thread` and override its `run()` method.
- **Implementing `Runnable`:** Implement the `Runnable` interface and define the `run()` method.

**Example – Extending Thread:**  
```java
class WorkerThread extends Thread {
    public void run() {
        System.out.println("WorkerThread running");
    }
}
new WorkerThread().start();
```
This creates a new thread by extending `Thread`. However, **limitation:** you cannot extend any other class (Java has single inheritance). Also, this approach often involves more boilerplate.

**Example – Implementing Runnable:**  
```java
class WorkerTask implements Runnable {
    public void run() {
        System.out.println("WorkerTask running");
    }
}
Thread t = new Thread(new WorkerTask());
t.start();
```
This uses the `Runnable` interface. As Baeldung notes, implementing `Runnable` is usually preferred. Key reasons:
- **Flexibility:** A class can implement `Runnable` and still extend another class, unlike extending `Thread`.
- **Composition Over Inheritance:** By passing a `Runnable` to a `Thread` (or executor), you decouple task from execution mechanism.
- **Lambda-friendly:** Since `Runnable` is a functional interface, you can use a lambda for brevity:
  ```java
  ExecutorService exec = Executors.newSingleThreadExecutor();
  exec.submit(() -> System.out.println("Lambda runnable executed!"));
  ```
- **Code Simplicity:** Extending `Thread` can add unnecessary overhead. For instance, a simple log operation required a lot of code when using a `Thread` subclass. With `Runnable`, the code is cleaner and can easily leverage lambdas.

In practice, **favor `Runnable` (or `Callable`) over extending `Thread`**.

## Runnable vs Callable vs Future

- **`Runnable`:** The original Java task interface. Its `run()` method returns no result and cannot throw checked exceptions. Use `Runnable` for tasks that do work but don’t return a value.
- **`Callable<V>`:** Introduced in Java 5 (`java.util.concurrent`). Its `call()` method returns a value of type `V` and may throw checked exceptions.
  - Callable tasks **must** be run by an executor (you cannot directly pass a `Callable` to `new Thread()`).
  - Useful when you need a task to produce a result or throw an exception.
- **`Future<V>`:** Represents the result of an asynchronous computation.
  - When you submit a `Callable` (or `Runnable`) to an `ExecutorService.submit()`, you get back a `Future<V>`.
  - You can call `future.get()` to wait for and retrieve the result; `future.isDone()` to check completion; or `future.cancel()` to attempt cancellation.
  - The combination allows tasks to be executed asynchronously and results fetched later, possibly blocking until available.

**Example – Callable and Future:**  
```java
ExecutorService exec = Executors.newSingleThreadExecutor();
Callable<Integer> task = () -> {
    int sum = 0;
    for(int i = 1; i <= 5; i++) sum += i;
    return sum;
};
Future<Integer> future = exec.submit(task);
int result = future.get();  // waits for result
System.out.println("Result: " + result);  // e.g., 15
exec.shutdown();
```
Here, `Callable` computes a sum and returns it. `Future.get()` blocks until done.

**Key differences (Runnable vs Callable):**  
| Feature                    | Runnable                      | Callable                  |
|----------------------------|-----------------------------------------------|------------------------------------------|
| Return value               | *None* (void)                                | Returns a value (`V`)                   |
| Exceptions                 | Cannot throw checked exceptions             | Can throw checked exceptions            |
| Package                    | `java.lang`                                  | `java.util.concurrent`                  |

Java 5+ allows you to submit either to an executor:
```java
executorService.submit((Callable<String>) () -> "Hello World");  // returns Future<String>
executorService.submit((Runnable) () -> System.out.println("Hi"));  // returns Future<?>
```
Use `Future` to manage the task and get the outcome.

## Thread Pools and Executors

Creating a new thread for every task is expensive and can exhaust system resources. The **Thread Pool** pattern addresses this by reusing a fixed (or bounded) set of threads to execute many tasks.

- **Benefits of Thread Pools:** Reuse threads to avoid constant creation/destruction overhead. Control the maximum number of concurrent threads. Queue tasks to avoid overwhelming the system. (This prevents thread leaks and resource exhaustion.)

- **Executors and ExecutorService:**  
  The `java.util.concurrent.Executors` class provides factory methods to create common thread pools:
  - `newFixedThreadPool(n)`: Creates a pool with *n* threads. If all are busy, tasks wait in a queue.
  - `newCachedThreadPool()`: Creates a pool that can grow as needed (up to `Integer.MAX_VALUE` threads) and reclaims idle threads after 60 seconds. Good for many short-lived tasks.  
  - `newSingleThreadExecutor()`: Single worker thread (tasks run sequentially).
  - `newScheduledThreadPool(n)`: For scheduling tasks to run after a delay or periodically.

  Example – Fixed Thread Pool:
  ```java
  ExecutorService executor = Executors.newFixedThreadPool(3);
  for (int i = 1; i <= 5; i++) {
      final int taskId = i;
      executor.submit(() -> {
          System.out.println("Task " + taskId + " running by " + Thread.currentThread().getName());
      });
  }
  executor.shutdown();
  ```
  Here 5 tasks are submitted to a pool of 3 threads. Two tasks will wait in queue. This **reuses** threads for all tasks.

- **ThreadPoolExecutor (core vs max threads):**  
  Under the hood, `ExecutorService` implementations use `ThreadPoolExecutor`. Its key parameters (set via constructor or factory) are:
  - **corePoolSize:** Number of threads always kept alive, even if idle.
  - **maximumPoolSize:** Maximum allowed threads (beyond core).  
  - **keepAliveTime:** Time to keep extra (beyond-core) threads alive when idle.
  - **workQueue:** Queue holding tasks when all threads are busy.
  
  A fixed thread pool sets `corePoolSize = maxPoolSize`, so thread count is constant. A cached thread pool sets `corePoolSize=0`, `maxPoolSize=∞`, and 60s keepalive, meaning it can grow without bound but prunes idle threads.

- **Cached Thread Pool:** As Baeldung explains, `newCachedThreadPool()` “may grow without bounds to accommodate any number of tasks, but threads are disposed after 60s of inactivity”. Internally it uses a `SynchronousQueue`, so tasks are handed directly to threads with no queueing.

- **Scheduled Thread Pool:** Extends thread pool with scheduling: `schedule()`, `scheduleAtFixedRate()`, `scheduleWithFixedDelay()`. E.g., `executor.scheduleAtFixedRate(task, initialDelay, period, TimeUnit)`.

- **Shutdown (thread pool termination):**  
  - `executor.shutdown()`: Initiates an *orderly* shutdown. No new tasks accepted, but previously submitted tasks execute.
  - `executor.shutdownNow()`: Attempts to **stop all executing tasks immediately**, halts waiting tasks, and returns a list of tasks that were submitted but not yet started. It also interrupts running threads.
  
  Use shutdown to clean up pools. Failing to shutdown can lead to thread leaks (threads preventing JVM exit).

- **Starvation and Fairness:** Thread starvation occurs if certain threads never get CPU or lock access because others monopolize it (e.g. high-priority threads hog CPU or an unfair lock never grants others access). Java’s `ReentrantLock` and other locks allow a “fair” mode (FIFO granting) to mitigate starvation at the cost of throughput.

- **Best Practice:** Always limit thread pool size. For I/O-bound tasks, more threads may help; for CPU-bound, typically use ~`# of cores` threads. Avoid `Executors.newCachedThreadPool()` if task submission could be unbounded—this can create too many threads. Use fixed or bounded pools and handle rejections if overloaded.

## Executors: `execute()` vs `submit()`

- **`execute(Runnable)`:** Provided by `Executor` interface. Fire-and-forget; submits a `Runnable` task.
- **`submit()`:** Provided by `ExecutorService`. Can submit `Runnable` or `Callable` and returns a `Future`. Even for `Runnable`, `submit()` returns a `Future<?>` (you can call `get()` to wait for completion, though it returns `null`). Key differences:
  - `execute()` does not give feedback or exceptions (just runs task).
  - `submit()` allows for retrieving result or checking for exceptions via the returned `Future`.
  
Interview tip: Use `submit()` if you need to know when the task finishes or get a result. Use `execute()` for fire-and-forget.

## Synchronization and Concurrency Control

Concurrent code must handle shared data carefully to avoid **race conditions**. 

- **Race Condition:** Occurs when multiple threads access and modify shared data without proper coordination, leading to inconsistent or unpredictable results. For example, incrementing a shared counter without synchronization can miss updates.

### `synchronized` Keyword

The simplest Java mechanism for mutual exclusion is the `synchronized` keyword. It ensures that only one thread at a time executes a critical section (monitored block).

- **Block Synchronization:** 
  ```java
  synchronized(lockObject) {
      // Only one thread can execute here at a time per lockObject
      sharedCounter++;
  }
  ```
- **Method Synchronization:** Mark a method `synchronized`, causing the entire method to hold the object’s lock.

Baeldung points out: *“synchronized… allows only one thread to execute [the block] at any given time”*. Using `synchronized` prevents the race seen in the example where 1000 increments by 3 threads resulted in fewer than 1000. When properly synchronized, the result would be correct.

### `wait()` and `notify()/notifyAll()`

To coordinate threads (e.g., producer-consumer problems), Java provides `Object.wait()`, `notify()`, and `notifyAll()`, which must be used inside synchronized blocks.

- Calling **`wait()`** (on an object’s monitor) causes the current thread to release the lock and **sleep** until another thread calls `notify()` or `notifyAll()` on the same object.
- **`notify()`** wakes up one waiting thread; **`notifyAll()`** wakes all waiting threads.
- Always use `wait()` in a loop checking a condition, to avoid spurious wakeups.
  
For example, the Oracle tutorial shows a `guardedJoy()` method that waits in a loop:
```java
synchronized void guardedJoy() {
    while (!joy) {
        wait();
    }
    // proceed when joy is true
}
```
The producer would then set `joy = true;` and call `notifyAll()`. This is the classic guarded block pattern.

- **`wait()` vs `sleep()`:** Although both pause a thread, they differ:
  - `Thread.sleep(ms)`: Static method. Pauses current thread for a fixed time but **does not release locks**. Can be called anywhere.
  - `Object.wait()`: Instance method. Must be called within a synchronized block. The thread **releases the lock** and waits until `notify`.  
  The GFG article illustrates this difference (see diagram below).

 **Figure:** Thread state when using `sleep()` vs `wait()`. With `sleep()`, the thread stays blocked without releasing its lock. With `wait()`, the thread releases the object’s lock and waits to be notified.

- **`join()`:** A convenience over `wait()`. When Thread A calls `t.join()` on Thread B, A waits (releases CPU) until B finishes.

### `volatile` Keyword

The `volatile` modifier ensures visibility of changes to a variable across threads (avoiding cached copies). Writes to a `volatile` variable are immediately visible to other threads. However, `volatile` **does not ensure atomicity** for compound actions. For example, `volatile int counter; counter++;` is still not thread-safe because increment is not atomic.

Use `volatile` for flags or simple writes where you don’t need atomic increment (e.g., `volatile boolean done;`). Do **not** use it as a substitute for synchronization when performing multi-step updates.

### Atomic Variables

Java provides classes like `AtomicInteger`, `AtomicLong`, etc., in `java.util.concurrent.atomic` for lock-free thread-safe operations. These classes use low-level CPU instructions (compare-and-swap) to ensure atomic updates without blocking.

Example – thread-safe counter with `AtomicInteger`:
```java
public class SafeCounter {
    private final AtomicInteger counter = new AtomicInteger(0);
    void increment() {
        counter.incrementAndGet();  // atomic increment
    }
    int get() {
        return counter.get();
    }
}
```
This avoids `synchronized` entirely. Methods like `incrementAndGet()` and `compareAndSet()` are atomic and thread-safe.

**Note:** Atomic classes guarantee atomicity of individual operations (like increment), but if you need compound actions (read-modify-write sequences), you may still need explicit locking or use `AtomicReference` with CAS loops.

### Locks (`ReentrantLock` and others)

Besides `synchronized`, Java offers explicit Lock implementations (`java.util.concurrent.locks.Lock` interface). For example, `ReentrantLock` (with optional fairness) can replace synchronized blocks:
```java
Lock lock = new ReentrantLock(true);  // fair lock
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```
Fair locks grant access in queue order, reducing starvation. Unfair locks (default) can give threads quicker access (higher throughput) but risk starvation under contention.

### Concurrency Collections

Java’s `java.util.concurrent` package includes thread-safe collections:
- **`ConcurrentHashMap`:** Thread-safe, high-performance alternative to synchronized `HashMap`.
- **`CopyOnWriteArrayList/Set`:** Optimized for many reads, few writes (writes make a new copy).
- **Blocking Queues (`LinkedBlockingQueue`, `ArrayBlockingQueue`):** Useful for producer-consumer patterns; they handle waiting when empty/full.
- **`ConcurrentLinkedQueue`:** Lock-free queue for many producers/consumers.
  
These are beyond Java’s synchronized collections and handle their own internal synchronization, avoiding the need for external locks in many scenarios.

## Thread Communication Patterns

- **Producer-Consumer:** A classic use of wait/notify. A producer thread puts data into a shared buffer; if full, it waits. A consumer takes data; if empty, it waits. Upon put or take, threads `notify()` or `notifyAll()`. (Oracle’s example of `Drop` object with `put()` and `take()` shows this pattern.)
  
- **Other examples:** Many interview problems involve coordinating threads:
  - *FizzBuzz multithreaded:* Three threads alternate printing numbers, "Fizz", "Buzz", etc.
  - *Print Zero Even Odd:* Two threads print even/odd numbers, one prints zeros in between, coordinating via locks or semaphores.
  - *Dining Philosophers:* A classic concurrency puzzle about avoiding deadlock using semaphores or ordered locking.
  - *Bounded Blocking Queue (Design):* Implement a thread-safe queue with fixed size, using wait/notify.
  - *Multithreaded Web Crawler:* Using a thread pool to fetch pages, concurrency controls for shared visited-set.
  
These problems test understanding of locks, conditions, and thread coordination. Solutions typically use synchronized blocks, wait/notify, semaphores, or high-level constructs (e.g., `BlockingQueue`).

## Advanced Executors and Future Composability

- **ExecutorService Methods:** Besides `submit`, useful methods include:
  - `invokeAll()`: Submit a collection of tasks and wait for all.
  - `invokeAny()`: Submit multiple tasks and get the first result.
  - `shutdown()`, `shutdownNow()`, `awaitTermination()`.
- **Scheduled Executors:** As shown, allow delayed or periodic tasks.
- **Fork/Join Pool:** For divide-and-conquer parallelism (`ForkJoinPool`, `RecursiveTask`) with work-stealing (Java 7+). Use for parallelizing recursive algorithms, not covered here in depth.
- **Future vs CompletableFuture:** The original `Future` (Java 5) is limited (no chaining, no callback on completion). Java 8 introduced `CompletableFuture` (implements `Future` plus `CompletionStage`), allowing composition of asynchronous tasks. With `CompletableFuture`, you can attach callbacks, combine futures, and handle exceptions in a fluent way. For example:
  ```java
  CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> {
      // compute something
      return 42;
  });
  cf.thenApply(x -> x * 2)
    .thenAccept(result -> System.out.println("Result: " + result));
  ```
  This executes asynchronously, doubles the result, and prints it when done. See Baeldung for a deep guide.

## Best Practices

- Always shutdown executors to avoid thread leaks.
- Minimize synchronized sections and lock contention for performance.
- Prefer higher-level concurrency constructs (executors, concurrent collections, atomic variables) over manual thread management.
- Document thread safety (use thread-safe classes for shared data).
- When in doubt, keep it simple: **immutable objects** and local variables avoid a lot of concurrency headaches.

## References

- Oracle Java Tutorials – Concurrency (Threads, Synchronization, Executors)  
- GeeksforGeeks – Process vs Thread, Thread Lifecycle, Callable vs Runnable, Callable & Future, wait() vs sleep().  
- Baeldung – Runnable vs Thread, Thread Pools & Executors, Synchronized Guide, Volatile vs Atomic, CompletableFuture.  

