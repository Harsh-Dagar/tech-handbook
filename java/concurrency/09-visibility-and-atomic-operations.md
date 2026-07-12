# Visibility & Atomic Operations

> **Difficulty:** 🟡 Intermediate
>
> **Reading Time:** ~20 minutes
>
> **Prerequisites**
>
> - Race Conditions & Synchronization
>
> **Core Question**
>
> > **Why doesn't one thread always see the changes made by another thread, and how can Java safely update shared data without locking?**
>
> **In this chapter, you'll learn**
>
> - Why threads may observe stale data.
> - What memory visibility means.
> - How the `volatile` keyword ensures visibility.
> - Why `volatile` cannot make `counter++` thread-safe.
> - How atomic classes provide lock-free updates.
> - The idea behind Compare-And-Set (CAS).

---

> [!TIP]
> **Mental Model**
>
> Concurrency problems usually fall into one of three categories:
>
> - **Visibility** → Can other threads see my changes?
> - **Atomicity** → Can an operation be interrupted halfway?
> - **Ordering** → Can instructions execute in an unexpected order?
>
> This chapter focuses on the first two.
> We'll study instruction ordering and the Java Memory Model later.

---

# Introduction

In the previous chapter, we learned how `synchronized` prevents multiple threads from modifying shared data at the same time.

But synchronization isn't the only challenge in concurrent programming.

Sometimes, threads don't interfere with each other at all...

yet the program still behaves incorrectly.

Imagine one thread updates a variable:

```java
running = false;
```

Another thread keeps checking:

```java
while (running) {
    // Work...
}
```

Should the loop eventually stop?

Most developers would say **yes**.

Surprisingly, the answer isn't always yes.

A thread may continue reading an **old value**, even though another thread has already updated it.

This isn't a race condition.

It's a **visibility problem**.

In this chapter, we'll learn why visibility problems occur, how the `volatile` keyword solves them, and why atomic classes such as `AtomicInteger` are needed for safe updates without traditional locks.

---

# The Visibility Problem

Modern CPUs are incredibly fast.

Accessing main memory, however, is relatively slow.

To improve performance, processors use **CPU caches**.

Instead of reading from main memory every time, a thread may read a cached copy.

```text
             Main Memory
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
   CPU Core 1          CPU Core 2
      Cache               Cache
        │                   │
    Thread A            Thread B
```

Now imagine the following sequence:

1. `running` is initially `true`.
2. Thread A caches the value.
3. Thread B updates `running` to `false`.
4. Thread A continues reading its cached value.

Even though the variable changed in main memory, Thread A keeps seeing the old value.

This is called a **visibility problem**.

---

# A Real Example

```java
class Worker extends Thread {

    boolean running = true;

    @Override
    public void run() {

        while (running) {

            // Working...

        }

        System.out.println("Stopped");
    }

    public void stopWorker() {

        running = false;

    }

}
```

At first glance, this looks perfectly correct.

However, there is no guarantee that the worker thread will observe the updated value of `running`.

As a result, the loop may never terminate.

---

# Why Doesn't Java Always Read from Main Memory?

If every variable access required reading from main memory:

- Every loop iteration would become slower.
- CPUs would spend more time waiting for memory.
- Overall performance would suffer significantly.

Instead, the JVM and the CPU are allowed to optimize memory accesses.

This is why visibility problems exist.

These optimizations are intentional—they improve performance.

The trade-off is that programmers sometimes need a way to ensure visibility.

---

# Enter `volatile`

The `volatile` keyword tells the JVM:

> **Always make writes to this variable visible to other threads.**

Example:

```java
private volatile boolean running = true;
```

Now:

```java
while (running) {

    // Work

}
```

will eventually observe:

```java
running = false;
```

when another thread updates the variable.

---

# What Guarantees Does `volatile` Provide?

A `volatile` variable provides two important guarantees.

## 1. Visibility

When one thread writes:

```java
running = false;
```

other threads will eventually observe that updated value.

There is no risk of indefinitely reading a stale cached value.

---

## 2. Ordering

The JVM and CPU are allowed to reorder certain instructions for optimization.

For `volatile` variables, Java places restrictions on these reorderings.

This helps ensure that operations occur in a predictable order around volatile reads and writes.

> **Note:** We'll explore instruction reordering in detail when we study the Java Memory Model.

---

# What `volatile` Does NOT Guarantee

A very common misconception is:

> "Using `volatile` makes my code thread-safe."

It does not.

Consider this example.

```java
private volatile int counter = 0;

public void increment() {

    counter++;

}
```

Is this thread-safe?

No.

Even though every thread sees the latest value of `counter`, the operation:

```java
counter++;
```

still consists of three separate steps:

```text
Read
  ↓
Increment
  ↓
Write
```

Two threads can still interfere with each other.

`volatile` solves **visibility**, not **atomicity**.

---

# When Should You Use `volatile`?

`volatile` is a good choice when:

- Multiple threads read a variable.
- One or a few threads update the variable.
- The update is independent of the current value.

Typical examples include:

- Shutdown flags
- Feature toggles
- Configuration values
- Status indicators

Example:

```java
private volatile boolean shutdownRequested;
```

One thread updates the flag.

Other threads periodically check it.

No locking is required.

---

# When Should You NOT Use `volatile`?

Avoid `volatile` when multiple operations must happen atomically.

For example:

```java
balance += amount;
```

or

```java
counter++;
```

These involve:

- Reading the current value.
- Computing a new value.
- Writing it back.

Multiple threads can still overwrite each other's changes.

In such cases, use:

- `synchronized`
- `AtomicInteger`
- Other atomic classes

We'll explore atomic variables in the next section.

---

# Summary So Far

We've introduced a new concurrency problem:

**Visibility.**

Unlike race conditions, visibility issues occur even when threads are not modifying a variable simultaneously.

To solve this, Java provides the `volatile` keyword.

`volatile` ensures that updates made by one thread become visible to others, but it does **not** make compound operations atomic.

In the next section, we'll learn how Java performs atomic updates without explicit locking using classes such as `AtomicInteger`.


---

# Atomic Variables

In the previous section, we learned that `volatile` solves the **visibility** problem.

However, it does **not** make compound operations like:

```java
counter++;
```

thread-safe.

To understand why, let's revisit the increment operation.

```text
Read counter
      │
      ▼
Add 1
      │
      ▼
Write counter
```

Even if `counter` is declared as `volatile`, multiple threads can still execute these steps simultaneously, causing updates to be lost.

We need a way to perform the entire operation as **one indivisible step**.

This property is called **atomicity**.

---

# What Does Atomic Mean?

An **atomic operation** is an operation that appears to happen **completely or not at all**.

No other thread can observe the operation halfway through.

For example:

```java
counter.incrementAndGet();
```

behaves as a single logical operation.

Another thread cannot interrupt it between the read and the write.

---

# Meet `AtomicInteger`

Java provides a set of atomic classes in the `java.util.concurrent.atomic` package.

One of the most commonly used is:

```java
AtomicInteger
```

Instead of writing:

```java
private int counter = 0;
```

we write:

```java
private final AtomicInteger counter = new AtomicInteger(0);
```

Updating the counter becomes:

```java
counter.incrementAndGet();
```

Reading the value:

```java
int value = counter.get();
```

---

# Common AtomicInteger Operations

| Method | Description |
|---------|-------------|
| `get()` | Returns the current value |
| `set(value)` | Sets a new value |
| `incrementAndGet()` | Increment, then return the new value |
| `getAndIncrement()` | Return the current value, then increment |
| `decrementAndGet()` | Decrement, then return the new value |
| `addAndGet(n)` | Add `n` and return the updated value |
| `compareAndSet(expected, update)` | Update only if the current value matches the expected value |

We'll study `compareAndSet()` shortly.

---

# Example

Without atomic variables:

```java
class Counter {

    private int count = 0;

    public void increment() {

        count++;

    }

}
```

Thread-safe version:

```java
class Counter {

    private final AtomicInteger count =
            new AtomicInteger();

    public void increment() {

        count.incrementAndGet();

    }

    public int getCount() {

        return count.get();

    }

}
```

Now multiple threads can safely increment the counter without using `synchronized`.

---

# Why Not Just Use `volatile`?

Consider the following code.

```java
private volatile int count = 0;
```

Increment:

```java
count++;
```

Even though `count` is visible to every thread, the increment still consists of:

```text
Read
  ↓
Add 1
  ↓
Write
```

Two threads can still read the same value before either writes the updated result.

Now compare that with:

```java
AtomicInteger count =
        new AtomicInteger();
```

Increment:

```java
count.incrementAndGet();
```

The entire update is performed atomically.

No update is lost.

---

# `volatile` vs `AtomicInteger`

| `volatile` | `AtomicInteger` |
|------------|-----------------|
| Solves visibility | Solves visibility **and** atomic updates |
| Does not protect `counter++` | Safely increments counters |
| Very lightweight | Slightly more overhead than `volatile`, but avoids explicit locking |
| Cannot perform compound atomic operations | Provides many built-in atomic operations |

---

# When Should You Use Atomic Variables?

Atomic variables work well for:

- Request counters
- Page view counters
- Statistics
- Sequence numbers
- Retry counts
- ID generators
- Flags that require atomic updates

Example:

```java
private final AtomicInteger activeUsers =
        new AtomicInteger();
```

When a user connects:

```java
activeUsers.incrementAndGet();
```

When a user disconnects:

```java
activeUsers.decrementAndGet();
```

No explicit synchronization is required.

---

# When Should You Avoid Atomic Variables?

Atomic variables are designed for **single-variable state**.

Suppose we need to update two related variables together.

```java
balance

transactionCount
```

Updating them independently with two `AtomicInteger` objects does **not** make the overall operation atomic.

Sometimes multiple values must change together as a single unit.

In such cases, use:

- `synchronized`
- `ReentrantLock`
- Other higher-level synchronization mechanisms

---

# Best Practices

✅ Use `volatile` for simple visibility flags.

✅ Use `AtomicInteger` for frequently updated counters.

✅ Prefer atomic classes over `synchronized` for simple numeric state.

✅ Remember that atomic classes solve only single-variable atomicity.

❌ Don't replace every integer with an `AtomicInteger`.

❌ Don't assume multiple atomic variables form one atomic transaction.

---

# Summary

We've now seen two different concurrency tools.

| Problem | Solution |
|----------|----------|
| Visibility | `volatile` |
| Atomic updates | `AtomicInteger` |

Although both help with thread safety, they solve different problems.

Choosing the correct tool depends on the type of shared state you're protecting.

In the next section, we'll answer an important question:

> **How can `AtomicInteger` update a value atomically without using `synchronized`?**

The answer is a powerful hardware-supported technique called **Compare-And-Set (CAS)**.


---

# How Does `AtomicInteger` Work?

In the previous section, we learned that:

```java
counter.incrementAndGet();
```

is thread-safe.

A natural question follows:

> **How can multiple threads update the same variable safely without using `synchronized`?**

The answer is a technique called **Compare-And-Set (CAS)**.

---

# The Problem with Traditional Updates

Suppose the current value is:

```text
counter = 5
```

Two threads execute:

```java
counter++;
```

At nearly the same time.

```text
Thread A           Thread B

Read 5            Read 5

Add 1             Add 1

Write 6           Write 6
```

Final value:

```text
6
```

Expected:

```text
7
```

This is the classic **lost update** problem.

---

# A Better Approach

Instead of saying:

> "Set the value to 6."

CAS says:

> **"Set the value to 6 only if it is still 5."**

Notice the difference.

The update succeeds only if the value hasn't changed since we last observed it.

---

# Compare-And-Set (CAS)

Conceptually, CAS performs three steps **atomically**:

1. Read the current value.
2. Compare it with the expected value.
3. Update it only if they match.

```
Current Value = 5

Expected Value = 5

↓

Match?

├── Yes → Update to 6

└── No → Fail
```

All three steps happen as **one atomic operation**.

No thread can observe the value halfway through.

---

# Visual Example

Suppose two threads attempt to increment the same counter.

### Thread A

```
Expected = 5

New Value = 6
```

### Thread B

```
Expected = 5

New Value = 6
```

Only one thread succeeds.

```text
Counter = 5

↓

Thread A CAS

Success

↓

Counter = 6

↓

Thread B CAS

Expected = 5

Actual = 6

↓

Fail
```

Notice that no incorrect value is written.

The second thread simply detects that another thread already updated the counter.

---

# What Happens After CAS Fails?

A failed CAS does **not** mean the operation is over.

Instead, the thread simply tries again.

Conceptually:

```text
Read Current Value

↓

Compute New Value

↓

CAS

├── Success → Done

└── Failed

        │
        ▼

Retry
```

This retry mechanism is called a **CAS loop**.

---

# A Simplified CAS Loop

This is roughly how an atomic increment works internally.

```java
while (true) {

    int current = counter.get();

    int next = current + 1;

    if (counter.compareAndSet(current, next)) {

        break;

    }

}
```

Let's understand what happens.

1. Read the current value.
2. Compute the new value.
3. Try to update it.
4. If another thread changed the value first, try again.

Eventually, one attempt succeeds.

> **Note:** The real implementation inside the JDK is highly optimized and uses low-level JVM and CPU instructions. The code above is only a conceptual model.

---

# Why Is This Faster Than `synchronized`?

With `synchronized`:

```text
Acquire Lock

↓

Critical Section

↓

Release Lock
```

If another thread already holds the lock:

```
WAIT
```

The thread is blocked until the lock becomes available.

With CAS:

```text
Read

↓

Try Update

↓

Success?

├── Yes → Done

└── No → Retry
```

No lock is acquired.

No thread is suspended.

Instead, unsuccessful threads retry the operation.

Because no thread blocks, CAS is often called a **lock-free** technique.

> [!NOTE]
> Lock-free does **not** mean "no synchronization." It means threads coordinate without using traditional locks like `synchronized` or `ReentrantLock`.

---

# When Is CAS a Good Choice?

CAS performs well when:

- Many threads read shared data.
- Updates are short and independent.
- Contention is relatively low.

Typical examples:

- Counters
- Metrics
- Request statistics
- ID generation
- Rate limiting

---

# When Is CAS a Poor Choice?

Imagine hundreds of threads repeatedly updating the same counter.

```
Thread A

CAS Failed

↓

Retry

↓

CAS Failed

↓

Retry
```

At high contention, many retries may occur.

Instead of blocking,

threads repeatedly compete for the same variable.

This wastes CPU cycles.

In these situations, traditional locking or more scalable data structures may perform better.

---

# A Brief Note on the ABA Problem

CAS compares only the current value.

Suppose a variable changes like this:

```text
A

↓

B

↓

A
```

Another thread observes:

```text
Expected = A

Actual = A
```

CAS succeeds,

even though the value changed twice in between.

This is known as the **ABA Problem**.

Fortunately, for simple counters such as `AtomicInteger`, the ABA problem is usually not an issue.

For more advanced concurrent algorithms, Java provides classes such as:

- `AtomicStampedReference`
- `AtomicMarkableReference`

We'll leave those for advanced topics.

---

# Choosing the Right Tool

| Problem | Recommended Tool |
|----------|------------------|
| Visibility only | `volatile` |
| Atomic update of one variable | `AtomicInteger` |
| Multiple related variables | `synchronized` or `ReentrantLock` |
| Complex coordination | Higher-level concurrency utilities |

---

# Chapter Summary

In this chapter, we explored two fundamental concepts of concurrent programming:

### Visibility

We learned that one thread may not immediately observe changes made by another thread due to CPU caches and compiler optimizations.

The `volatile` keyword solves this problem by ensuring that updates become visible to other threads.

### Atomicity

We then discovered that visibility alone is not enough.

Operations like:

```java
counter++;
```

are not atomic.

To perform safe lock-free updates, Java provides atomic classes such as `AtomicInteger`.

Internally, these classes rely on **Compare-And-Set (CAS)**, an atomic operation that updates a value only if it hasn't changed since it was last observed.

This allows many common operations to be implemented efficiently without explicit locks.

---

# Key Takeaways

✅ `volatile` guarantees **visibility**, not atomicity.

✅ `AtomicInteger` guarantees atomic updates for a single variable.

✅ CAS updates a value only if it still matches the expected value.

✅ Failed CAS operations simply retry.

✅ Lock-free does not mean synchronization-free.

---

# Quick Quiz

### 1. What problem does `volatile` solve?

- [x] Visibility
- [ ] Atomicity

---

### 2. What problem does `AtomicInteger` solve?

- [x] Atomic updates to a single variable
- [ ] Communication between threads

---

### 3. What happens if `compareAndSet()` fails?

- [ ] The thread terminates
- [x] The operation is typically retried
- [ ] The JVM throws an exception

---

### 4. Is CAS completely free of synchronization?

<details>
<summary>Answer</summary>

No. CAS is a synchronization mechanism, but it is **lock-free**. Threads coordinate using an atomic hardware-supported operation instead of acquiring traditional locks.

</details>

---

# What's Next?

So far, we've relied on Java's built-in monitor (`synchronized`) and atomic variables.

In the next chapter, we'll explore a more flexible locking mechanism:

- `Lock` interface
- `ReentrantLock`
- Fair vs non-fair locks
- `tryLock()`
- `lockInterruptibly()`
- `ReadWriteLock`

These utilities provide greater control than `synchronized` and are widely used in high-performance concurrent applications.