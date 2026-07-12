# Visibility & Atomic Operations

> **Difficulty:** 🟡 Intermediate
>
> **Reading Time:** ~20 minutes
>
> **Prerequisites**
>
> - Race Conditions & Synchronization
>
> **In this chapter, you'll learn**
>
> - Why threads sometimes cannot see each other's changes.
> - What memory visibility means.
> - How the `volatile` keyword works.
> - What guarantees `volatile` provides.
> - What `volatile` does **not** guarantee.
> - When to use `volatile`.

---

# Introduction

In the previous chapter, we learned how to protect shared data using `synchronized`.

Now let's look at a different problem.

Imagine two threads sharing a variable.

```java
boolean running = true;
```

Thread A repeatedly checks the value.

```java
while (running) {

    // Do some work

}
```

Thread B eventually stops the thread.

```java
running = false;
```

Intuitively, Thread A should stop.

But surprisingly, it may continue running forever.

How is that possible?

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