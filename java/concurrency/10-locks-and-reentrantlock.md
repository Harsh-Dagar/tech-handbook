# Locks & ReentrantLock

> **Difficulty:** 🟠 Intermediate
>
> **Reading Time:** ~25 minutes
>
> **Prerequisites**
>
> - Race Conditions & Synchronization
> - Visibility & Atomic Operations
>
> **Core Question**
>
> > **If `synchronized` already provides thread safety, why does Java also provide `ReentrantLock`?**
>
> **Mental Model**
>
> `synchronized` and `ReentrantLock` solve the same core problem—ensuring mutual exclusion.
>
> The difference is **control**.
>
> - `synchronized` is simple and automatic.
> - `ReentrantLock` is explicit and flexible.
>
> As concurrency requirements become more complex, explicit locks provide features that `synchronized` cannot.

---

# Introduction

In the previous chapters, we've relied on the `synchronized` keyword to protect shared state.

For many applications, this is the right choice.

```java
public synchronized void increment() {
    count++;
}
```

It's simple.

It's easy to read.

The JVM automatically acquires and releases the lock.

So why introduce another locking mechanism?

The answer is flexibility.

Real-world applications often need capabilities beyond simple mutual exclusion.

For example:

- Try to acquire a lock without waiting forever.
- Wait only for a limited amount of time.
- Interrupt a thread while it's waiting for a lock.
- Choose whether waiting threads should acquire the lock fairly.

These features are not available with `synchronized`.

To solve these problems, Java introduced the **Lock API**.

---

# The Lock Interface

The `Lock` interface lives in:

```java
java.util.concurrent.locks
```

It provides an abstraction for explicit locking.

The most commonly used implementation is:

```java
ReentrantLock
```

Creating a lock is straightforward.

```java
Lock lock = new ReentrantLock();
```

Unlike `synchronized`, acquiring and releasing the lock are explicit operations.

```java
lock.lock();

try {

    // Critical section

} finally {

    lock.unlock();

}
```

Notice that we are responsible for releasing the lock.

The JVM will not do it automatically.

---

# Why Use `try...finally`?

Suppose we write:

```java
lock.lock();

updateDatabase();

lock.unlock();
```

Looks fine.

But what happens if:

```java
updateDatabase();
```

throws an exception?

Execution immediately leaves the method.

The `unlock()` call is skipped.

The lock remains held forever.

Every other thread attempting to acquire the same lock will block indefinitely.

The correct approach is:

```java
lock.lock();

try {

    updateDatabase();

} finally {

    lock.unlock();

}
```

The `finally` block executes whether the protected code succeeds or throws an exception.

> [!IMPORTANT]
> Every successful `lock()` should have a corresponding `unlock()` inside a `finally` block.

---

# `synchronized` vs `ReentrantLock`

Although both provide mutual exclusion, they differ in several important ways.

| Feature | `synchronized` | `ReentrantLock` |
|----------|----------------|-----------------|
| Part of the language | ✅ | ❌ (Java API) |
| Automatic lock release | ✅ | ❌ |
| Manual unlock required | ❌ | ✅ |
| Timeout support | ❌ | ✅ |
| Interruptible lock acquisition | ❌ | ✅ |
| Fair scheduling option | ❌ | ✅ |

For many applications, `synchronized` is still the simplest and safest choice.

Use `ReentrantLock` when you need additional control over how locks are acquired and released.

---

# Why Is It Called "Reentrant"?

Suppose a thread already owns a lock.

Later, while executing the protected code, it calls another method that attempts to acquire the **same lock**.

Would this cause a deadlock?

Fortunately, no.

A `ReentrantLock` allows the thread that already owns the lock to acquire it again.

Example:

```java
class Counter {

    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {

        lock.lock();

        try {

            log();

        } finally {

            lock.unlock();

        }

    }

    private void log() {

        lock.lock();

        try {

            System.out.println("Logging...");

        } finally {

            lock.unlock();

        }

    }

}
```

Even though `increment()` and `log()` both acquire the same lock, the thread does not deadlock.

Instead, the lock maintains a **hold count**.

```text
First lock()
Hold Count = 1

↓

Second lock()
Hold Count = 2

↓

First unlock()
Hold Count = 1

↓

Second unlock()
Hold Count = 0

↓

Lock Released
```

The lock is released only when the hold count returns to zero.

---

# ReentrantLock vs Reentrancy in `synchronized`

This behavior isn't unique to `ReentrantLock`.

Java's intrinsic locks (`synchronized`) are also **reentrant**.

```java
public synchronized void methodA() {

    methodB();

}

public synchronized void methodB() {

}
```

If `methodA()` already owns the object's monitor, calling `methodB()` does **not** deadlock.

The same thread is allowed to re-enter the monitor.

> [!NOTE]
> The term **reentrant** means that a thread already holding a lock can safely acquire the same lock again.

---

# Summary So Far

We've introduced Java's explicit locking API and answered the question of why it exists.

So far, we've learned:

- `ReentrantLock` provides explicit locking.
- Locks must always be released manually.
- `unlock()` belongs inside a `finally` block.
- Both `ReentrantLock` and `synchronized` are reentrant.
- `ReentrantLock` offers greater flexibility than `synchronized`.

In the next section, we'll explore the features that make `ReentrantLock` so powerful:

- `tryLock()`
- Timed lock acquisition
- `lockInterruptibly()`
- Fair vs non-fair locks

---

# `tryLock()`

So far, we've used:

```java
lock.lock();
```

This method blocks the current thread until the lock becomes available.

```text
Thread A
    │
    ▼
Acquires Lock

Thread B
    │
    ▼
Calls lock()
    │
    ▼
WAITING...
```

Sometimes, waiting indefinitely isn't desirable.

Imagine a web server handling thousands of requests.

If a request cannot acquire a lock immediately, it may be better to:

- Return an error.
- Retry later.
- Process another request.

Instead of blocking forever.

This is where `tryLock()` becomes useful.

---

# Acquiring a Lock Without Waiting

`tryLock()` attempts to acquire the lock immediately.

```java
if (lock.tryLock()) {

    try {

        // Critical section

    } finally {

        lock.unlock();

    }

} else {

    System.out.println("Lock is busy.");

}
```

If the lock is available:

```text
Acquire Lock

↓

Execute Critical Section
```

Otherwise:

```text
Lock Busy

↓

Continue Without Blocking
```

The thread never enters the **BLOCKED** state.

---

# Why Is This Useful?

Consider a payment service.

Only one thread should process a transaction for a given account at a time.

However, if another request is already processing that account, waiting for several seconds might provide a poor user experience.

Instead:

```text
Try Lock

├── Success
│       │
│       ▼
│   Process Payment
│
└── Failed
        │
        ▼
Return "Please try again."
```

The application remains responsive instead of making users wait.

---

# Timed `tryLock()`

Sometimes we don't want to fail immediately.

Instead, we're willing to wait for a short period.

`ReentrantLock` provides an overloaded version:

```java
if (lock.tryLock(2, TimeUnit.SECONDS)) {

    try {

        // Critical section

    } finally {

        lock.unlock();

    }

} else {

    System.out.println("Couldn't acquire the lock.");

}
```

The thread waits for **up to 2 seconds**.

If the lock becomes available during that time, it proceeds.

Otherwise, it gives up.

```text
Request Lock

↓

Wait (Max 2 Seconds)

├── Lock Available
│       │
│       ▼
│   Continue
│
└── Timeout
        │
        ▼
Continue Without Lock
```

This gives developers much more control than `synchronized`, which always waits indefinitely.

---

# `lockInterruptibly()`

Imagine a thread waiting for a lock.

While it's waiting, the application is shutting down.

Should the thread continue waiting?

Usually, no.

With `lock()`, the thread remains blocked until the lock becomes available.

`ReentrantLock` offers a better alternative:

```java
lock.lockInterruptibly();

try {

    // Critical section

} finally {

    lock.unlock();

}
```

If another thread interrupts it while it's waiting, an `InterruptedException` is thrown.

The waiting thread can then clean up and exit gracefully.

```text
Waiting for Lock
        │
        ▼
Interrupted?
    ├── Yes → Exit
    └── No  → Acquire Lock
```

This is particularly useful for:

- Background worker threads
- Task cancellation
- Graceful application shutdown

---

# Fair vs Non-Fair Locks

Suppose three threads are waiting for the same lock.

```text
Thread A

↓

Thread B

↓

Thread C
```

Who should acquire the lock next?

There are two strategies.

---

## Non-Fair Lock (Default)

By default:

```java
new ReentrantLock();
```

creates a **non-fair** lock.

When the lock becomes available,

**any waiting thread** may acquire it.

Even a newly arrived thread might "cut in line."

```text
Waiting Queue

A

B

C

↓

New Thread D Arrives

↓

D Acquires Lock
```

This may sound unfair,

but it usually provides **higher throughput** because it minimizes context switching.

---

## Fair Lock

We can request fair scheduling.

```java
Lock lock = new ReentrantLock(true);
```

Now threads acquire the lock roughly in the order they requested it.

```text
Waiting Queue

A

↓

B

↓

C
```

When the lock is released:

```text
A

↓

B

↓

C
```

The oldest waiting thread gets the next opportunity.

---

# Fairness vs Performance

Fair locks reduce the chance that a thread waits for a very long time.

However, fairness comes at a cost.

The JVM has less flexibility when choosing which thread should run next.

This often results in lower throughput.

| Non-Fair Lock | Fair Lock |
|---------------|-----------|
| Higher throughput | More predictable scheduling |
| Better overall performance | Reduces starvation |
| Default choice | Used only when fairness is important |

> [!TIP]
> Unless your application has a specific fairness requirement, prefer the default non-fair lock.

---

# Choosing the Right Lock Method

| Method | Behavior | Typical Use Case |
|--------|----------|------------------|
| `lock()` | Wait indefinitely | Most common choice |
| `tryLock()` | Return immediately if unavailable | Avoid blocking |
| `tryLock(timeout)` | Wait for a limited time | Prevent indefinite waits |
| `lockInterruptibly()` | Can be interrupted while waiting | Task cancellation and graceful shutdown |

---

# Best Practices

✅ Always release the lock in a `finally` block.

✅ Prefer `lock()` unless you specifically need timeout or interruption support.

✅ Use `tryLock()` to avoid unnecessary blocking.

✅ Use fair locks only when starvation is a real concern.

❌ Never forget to call `unlock()`.

❌ Don't assume fair locks are always better—they often reduce performance.

---

# Summary So Far

In this section, we've explored the features that make `ReentrantLock` more flexible than `synchronized`.

Unlike intrinsic locks, `ReentrantLock` allows us to:

- Attempt lock acquisition without blocking (`tryLock()`).
- Wait for a limited amount of time (`tryLock(timeout)`).
- Interrupt a thread while it's waiting (`lockInterruptibly()`).
- Choose between fair and non-fair scheduling.

These capabilities make `ReentrantLock` a powerful tool for building responsive, production-grade concurrent applications where waiting forever is not always acceptable.

---

# Choosing Between `synchronized` and `ReentrantLock`

Throughout this chapter, we've seen that both `synchronized` and `ReentrantLock` provide **mutual exclusion**.

Both ensure that only one thread can execute a critical section at a time.

So which one should you choose?

The answer depends on the requirements of your application.

---

# Decision Guide

| Requirement | Recommended Choice | Why |
|-------------|--------------------|-----|
| Simple mutual exclusion | `synchronized` | Built into the language, easy to read, automatically releases the lock |
| Need to try acquiring a lock without waiting | `ReentrantLock` | Supports `tryLock()` |
| Need timeout while waiting | `ReentrantLock` | Supports timed `tryLock()` |
| Need interruptible lock acquisition | `ReentrantLock` | Supports `lockInterruptibly()` |
| Need fair scheduling | `ReentrantLock` | Supports fair locks |
| Protect a small critical section | Either | Choose the simpler solution unless additional features are required |

> [!TIP]
> Prefer the simplest tool that satisfies your requirements.
>
> If `synchronized` is sufficient, there is usually no benefit in replacing it with `ReentrantLock`.

---

# Common Mistakes

## 1. Forgetting to Release the Lock

This is probably the most common mistake.

❌ Incorrect

```java
lock.lock();

updateDatabase();

lock.unlock();
```

If `updateDatabase()` throws an exception, the lock is never released.

Other threads waiting for the lock may block forever.

---

✅ Correct

```java
lock.lock();

try {

    updateDatabase();

} finally {

    lock.unlock();

}
```

The `finally` block guarantees that the lock is always released.

---

## 2. Unlocking Without Owning the Lock

Calling:

```java
lock.unlock();
```

without first acquiring the lock results in:

```text
IllegalMonitorStateException
```

A thread may only release a lock that it currently owns.

---

## 3. Holding Locks Longer Than Necessary

Consider:

```java
lock.lock();

try {

    readLargeFile();

    processData();

    updateCounter();

} finally {

    lock.unlock();

}
```

Only `updateCounter()` modifies shared state.

The file reading and processing don't require synchronization.

A better approach is:

```java
readLargeFile();

processData();

lock.lock();

try {

    updateCounter();

} finally {

    lock.unlock();

}
```

Keeping critical sections small improves concurrency and reduces contention.

---

## 4. Using Multiple Locks Without a Consistent Order

Imagine two threads.

### Thread A

```java
lockA.lock();
lockB.lock();
```

### Thread B

```java
lockB.lock();
lockA.lock();
```

Now suppose:

- Thread A acquires `lockA`.
- Thread B acquires `lockB`.

Both threads now wait forever for the other lock.

```text
Thread A
Owns Lock A
    │
    ▼
Waiting for Lock B

Thread B
Owns Lock B
    │
    ▼
Waiting for Lock A
```

This is a **deadlock**.

We'll study deadlocks in detail in a later chapter.

For now, remember this rule:

> Always acquire multiple locks in a consistent order.

---

# Best Practices

✅ Prefer `synchronized` for simple locking.

✅ Use `ReentrantLock` only when you need additional features.

✅ Always pair `lock()` with `unlock()` in a `finally` block.

✅ Keep critical sections as short as possible.

✅ Acquire multiple locks in a consistent order.

✅ Document which lock protects which shared resource.

---

# Summary

In this chapter, we explored Java's explicit locking API.

We learned that:

- `ReentrantLock` provides the same mutual exclusion as `synchronized`.
- Unlike `synchronized`, it gives developers explicit control over lock acquisition and release.
- Features such as `tryLock()`, timed waits, interruptible locking, and fairness make it suitable for more advanced synchronization scenarios.
- Manual lock management requires extra care, especially ensuring that every `lock()` has a matching `unlock()`.

Although `ReentrantLock` is more powerful, it is not always the better choice.

For many applications, `synchronized` remains the simplest and most maintainable solution.

The key is choosing the right tool for the problem you're solving.

---

# Key Takeaways

✅ `synchronized` and `ReentrantLock` both provide mutual exclusion.

✅ `ReentrantLock` offers greater flexibility, but also greater responsibility.

✅ Always release locks in a `finally` block.

✅ Smaller critical sections improve concurrency.

✅ Consistent lock ordering helps prevent deadlocks.

---

# Quick Quiz

### 1. Which locking mechanism automatically releases the lock?

- [x] `synchronized`
- [ ] `ReentrantLock`

---

### 2. Which method allows you to attempt lock acquisition without blocking?

- [ ] `lock()`
- [x] `tryLock()`

---

### 3. Which lock type supports fairness?

- [ ] `synchronized`
- [x] `ReentrantLock`

---

### 4. Why should locks be held for the shortest possible time?

<details>
<summary>Answer</summary>

Holding a lock longer than necessary increases contention, causing other threads to wait longer and reducing overall concurrency.

</details>

---

# What's Next?

So far, we've learned how to coordinate threads using:

- `synchronized`
- `volatile`
- Atomic variables
- `ReentrantLock`

The next step is to explore how Java manages threads efficiently at scale.

Creating a new thread for every task is expensive.

Instead, modern applications reuse threads through **thread pools**.

In the next chapter, we'll learn:

- Why thread pools exist.
- How they improve performance.
- Different types of thread pools.
- How to choose the right one for your workload.