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