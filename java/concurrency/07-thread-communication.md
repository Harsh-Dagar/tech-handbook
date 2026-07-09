# Thread Communication

> **Difficulty:** 🟡 Intermediate
>
> **Reading Time:** ~18 minutes
>
> **Prerequisites:**
>
> - [Thread Lifecycle](05-thread-lifecycle.md)
> - [Thread Control](06-thread-control.md)
>
> **In this chapter, you will learn**
>
> - Why thread communication is necessary.
> - What busy waiting is and why it is inefficient.
> - The purpose of object monitors.
> - How `wait()` works.
> - Why `wait()` must be called inside a synchronized block.
> - Why `while` should be preferred over `if`.

---

# Introduction

So far we've learned how to:

- Create threads.
- Pause threads.
- Wait for another thread using `join()`.

Now let's consider a different problem.

Suppose two threads are working together.

One thread **produces** data.

Another thread **consumes** that data.

How does the consumer know when new data becomes available?

Should it continuously check?

Should it sleep every few milliseconds?

Neither approach is ideal.

Java provides a much better solution through **thread communication**.

---

# Real-World Motivation

Imagine a restaurant.

```
Chef

↓

Prepares Food

↓

Delivery Partner

↓

Delivers Food
```

The delivery partner cannot leave until the chef finishes preparing an order.

One possible solution is:

```
while(true){

    Is food ready?

}
```

This works...

but it wastes CPU.

The delivery partner keeps asking the same question thousands of times every second.

A much better approach is:

```
Chef

↓

Food Ready

↓

Notify Delivery Partner

↓

Delivery Starts
```

Instead of constantly checking,

the delivery partner simply waits until the chef signals that food is ready.

This is exactly how thread communication works.

---

# The Problem with Busy Waiting

Suppose we write:

```java
while (!dataAvailable) {

    // do nothing

}
```

This loop continuously checks the same condition.

```
Check

↓

Check

↓

Check

↓

Check

↓

Check

↓

...
```

Even though the thread isn't doing useful work,

it still consumes CPU cycles.

This pattern is called **Busy Waiting** (or **Busy Spinning**).

> [!WARNING]
> Busy waiting wastes CPU time because the thread repeatedly checks a condition instead of sleeping efficiently.

---

# A Better Solution

Instead of repeatedly checking,

the thread should sleep until another thread tells it that work is available.

Conceptually:

```
Consumer

↓

wait()

↓

Sleeping

↓

Producer creates data

↓

notify()

↓

Consumer wakes up
```

Notice the difference.

The CPU isn't wasted.

The consumer remains inactive until it actually has work to do.

---

# Enter the Object Monitor

To understand `wait()` and `notify()`, we first need to understand **monitors**.

Every object in Java has an associated **monitor**.

You can think of a monitor as a room where threads coordinate access to that object.

```
Java Object

┌─────────────────┐
│                 │
│    Monitor      │
│                 │
└─────────────────┘
```

Whenever a thread enters a synchronized block,

it acquires the object's monitor.

```java
synchronized(lock) {

    // Thread owns the monitor

}
```

Only one thread can own a monitor at a time.

This makes the monitor the perfect place for threads to communicate.

---

# What Does `wait()` Do?

The `wait()` method tells the current thread:

> "Release the monitor and go to sleep until another thread wakes me up."

This is one of the most important differences between `wait()` and `sleep()`.

When a thread calls:

```java
lock.wait();
```

it performs **three actions**:

1. Releases the monitor.
2. Enters the `WAITING` state.
3. Waits until another thread calls `notify()` or `notifyAll()`.

```mermaid
flowchart TD

A["Owns Monitor"]

↓

B["wait()"]

↓

C["Releases Monitor"]

↓

D["WAITING"]

↓

E["notify()"]

↓

F["Attempts to Reacquire Monitor"]

↓

G["Continues Execution"]
```

Notice something important.

After receiving a notification,

the thread **does not continue immediately**.

It must first **reacquire the monitor**.

This subtle detail is responsible for many concurrency bugs.

---

# Why Must `wait()` Be Called Inside `synchronized`?

Consider this code.

```java
lock.wait();
```

Who owns the monitor?

Nobody.

How can Java release a monitor that the thread never acquired?

It can't.

That's why the following code throws:

```text
IllegalMonitorStateException
```

The correct approach is:

```java
synchronized(lock) {

    lock.wait();

}
```

Here,

the thread already owns the monitor.

Therefore,

Java knows exactly which monitor should be released.

> [!IMPORTANT]
> A thread must own an object's monitor before calling `wait()`, `notify()`, or `notifyAll()`.

---

# Visualizing `wait()`

Suppose Thread A owns the monitor.

```
Thread A

Acquire Lock

↓

wait()

↓

Release Lock

↓

WAITING
```

Now another thread can acquire the monitor.

```
Thread B

Acquire Lock

↓

Produce Data

↓

notify()

↓

Release Lock
```

Only after Thread B releases the monitor

can Thread A continue.

```
WAITING

↓

Notification Received

↓

Waiting for Monitor

↓

Monitor Acquired

↓

Execution Continues
```

This is why a notification does **not** guarantee immediate execution.

---

# Why We Use `while` Instead of `if`

Suppose we write:

```java
if (!dataAvailable) {

    lock.wait();

}
```

Looks reasonable.

But it has a subtle problem.

When the thread wakes up,

how do we know that `dataAvailable` is still true?

Another thread may have already consumed the data.

Or the thread may wake up **without any notification at all**.

This rare situation is called a **spurious wakeup**.

Because of this,

Java recommends:

```java
while (!dataAvailable) {

    lock.wait();

}
```

Now every time the thread wakes up,

it checks the condition again before proceeding.

> [!IMPORTANT]
> Always call `wait()` inside a `while` loop that checks the waiting condition.

This is considered the standard pattern in Java concurrency.

---

# Summary So Far

We've learned several important ideas:

- Busy waiting wastes CPU.
- Every Java object has an associated monitor.
- `wait()` releases the monitor and suspends the current thread.
- A thread must own the monitor before calling `wait()`.
- Waking up does **not** guarantee that the condition is satisfied.
- Always use `while`, not `if`, when waiting for a condition.

In the next section, we'll complete the picture by learning:

- `notify()`
- `notifyAll()`
- Producer–Consumer
- Common communication mistakes
- Best practices