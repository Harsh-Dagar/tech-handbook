# Creating Threads in Java

> **Difficulty:** 🟢 Beginner
>
> **Reading Time:** ~15 minutes
>
> **Prerequisites:**
>
> - [Why Concurrency?](01-why-concurrency.md)
> - [Programs, Processes, and Threads](02-programs-processes-and-threads.md)
> - [Process Memory and Thread Layout](03-process-memory-and-thread-layout.md)
>
> **In this chapter, you will learn**
>
> - How Java creates new threads.
> - Different ways to execute code concurrently.
> - The `Thread` class and the `Runnable` interface.
> - Why `Runnable` is preferred over extending `Thread`.
> - The difference between `start()` and `run()`.

---

# Introduction

In the previous chapters, we answered three important questions:

- Why do we need concurrency?
- What are processes and threads?
- How is memory organized inside a Java process?

Now it's time to answer the next logical question:

> **How do we actually create a thread in Java?**

Java provides multiple ways to execute code concurrently.

At first glance, this can be confusing.

Questions such as:

- Should I extend `Thread`?
- Should I implement `Runnable`?
- Why do both exist?
- Which approach is considered best practice?

are common among developers new to concurrency.

By the end of this chapter, you'll not only know **how** to create threads, but also **why** Java provides multiple approaches.

---

# Evolution of Java Concurrency

Java's concurrency APIs evolved over time to solve different problems.

```mermaid
flowchart LR

A["Thread"] --> B["Runnable"]
B --> C["Executor Framework"]
C --> D["Callable"]
D --> E["Future"]
E --> F["CompletableFuture"]
```

Each new API addressed limitations of the previous one.

For example:

| API | Primary Goal |
|------|--------------|
| Thread | Create and manage a thread manually |
| Runnable | Separate the task from the thread |
| Executor Framework | Reuse threads efficiently |
| Callable | Return results from background tasks |
| CompletableFuture | Compose asynchronous workflows |

> [!NOTE]
> In this chapter, we'll focus only on **Thread** and **Runnable**. The remaining APIs will be covered in later chapters.

---

# The Thread Class

The `Thread` class represents an independent path of execution within a Java process.

Every Java application starts with one thread—the **main thread**.

```java
public class Main {

    public static void main(String[] args) {
        System.out.println("Running on the main thread");
    }
}
```

When the JVM starts your application, it creates the **main thread** and begins executing the `main()` method.

Additional threads can be created whenever concurrent execution is required.

---

# Creating a Thread by Extending `Thread`

The most direct way to create a thread is by extending the `Thread` class.

```java
class Worker extends Thread {

    @Override
    public void run() {
        System.out.println("Worker thread is running...");
    }
}
```

To execute the thread:

```java
public class Main {

    public static void main(String[] args) {

        Worker worker = new Worker();

        worker.start();
    }

}
```

Output

```text
Worker thread is running...
```

---

# Understanding What Happened

Let's break down the execution.

### Step 1

Create an instance of the thread.

```java
Worker worker = new Worker();
```

At this point,

- No new thread has been created.
- The object simply exists in memory.

Think of it as preparing a worker before assigning any work.

---

### Step 2

Start the thread.

```java
worker.start();
```

Calling `start()` tells the JVM:

> Create a new thread and execute this object's `run()` method.

The JVM then requests the operating system to schedule a new thread.

```mermaid
flowchart LR

A["main()"] --> B["worker.start()"]

B --> C["JVM Creates New Thread"]

C --> D["run() Executes"]
```

Notice that **`start()` does not directly execute your code.**

Instead, it asks the JVM to create a new thread, which will eventually invoke `run()`.

This distinction is extremely important and will be discussed in detail later in this chapter.

---

# What is the `run()` Method?

The `run()` method contains the code that the new thread will execute.

```java
class Worker extends Thread {

    @Override
    public void run() {

        System.out.println("Downloading file...");
    }

}
```

You can think of `run()` as the thread's entry point.

Whenever a thread starts, execution begins inside this method.

---

# A Simple Example

Consider the following program.

```java
class Worker extends Thread {

    @Override
    public void run() {

        for (int i = 1; i <= 5; i++) {
            System.out.println("Worker : " + i);
        }

    }

}

public class Main {

    public static void main(String[] args) {

        Worker worker = new Worker();

        worker.start();

        for (int i = 1; i <= 5; i++) {
            System.out.println("Main   : " + i);
        }

    }

}
```

A possible output:

```text
Main   : 1
Worker : 1
Main   : 2
Worker : 2
Worker : 3
Main   : 3
Worker : 4
Worker : 5
Main   : 4
Main   : 5
```

Your output may be different every time.

Why?

Because **thread scheduling is handled by the operating system**, and the execution order is not deterministic.

> [!IMPORTANT]
> Never assume that threads execute in a specific order unless synchronization mechanisms are used.

---

# Is Extending `Thread` a Good Idea?

Although extending `Thread` works, it has several drawbacks.

The biggest issue is that it mixes **two different responsibilities**.

```text
Thread Class

├── Represents a thread
└── Contains the task to execute
```

In other words,

the class is responsible for:

- defining the work
- representing the worker

These are two separate concepts.

Imagine hiring an employee.

The employee is **not** the work.

The employee **performs** the work.

Similarly,

- a **Thread** is the worker.
- the **task** is the work.

Combining both into a single class reduces flexibility and makes code harder to reuse.

This is exactly the problem that the `Runnable` interface solves.

---

# Limitations of Extending `Thread`

| Limitation | Explanation |
|------------|-------------|
| Single inheritance | A class can extend only one superclass. Extending `Thread` prevents it from extending any other class. |
| Tight coupling | The task is tightly coupled with the thread implementation. |
| Poor reusability | The same task cannot easily be executed by different threads. |
| Less flexible | Modern Java APIs are designed around `Runnable`, not subclasses of `Thread`. |

> [!TIP]
> Extending `Thread` is useful for learning how threads work, but in real-world applications you'll almost always implement `Runnable` or use higher-level concurrency APIs.