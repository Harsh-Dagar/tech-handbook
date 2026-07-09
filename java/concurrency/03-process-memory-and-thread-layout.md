# Process Memory and Thread Layout

> **Difficulty:** 🟢 Beginner
>
> **Reading Time:** ~15 minutes
>
> **Prerequisites:** [Why Concurrency?](01-why-concurrency.md), [Programs, Processes, and Threads](02-programs-processes-and-threads.md)
>
> **In this chapter, you will learn**
>
> - How memory is organized inside a Java process.
> - Which memory regions are shared between threads.
> - Which memory regions belong to individual threads.
> - Why local variables are naturally thread-safe.
> - Why shared objects can lead to race conditions.

---

# Introduction

In the previous chapter, we learned that:

- A **program** is a file stored on disk.
- Executing a program creates a **process**.
- A process can contain multiple **threads**.

This naturally raises another question:

> **If multiple threads run inside the same process, how is memory organized?**

Understanding the answer is one of the most important steps in learning Java concurrency.

Many concurrency bugs—such as race conditions, visibility problems, and inconsistent data—occur simply because developers don't know **which memory is shared and which isn't**.

> [!IMPORTANT]
> Before learning `synchronized`, `volatile`, or `AtomicInteger`, you should understand how memory is organized inside a Java process.

---

# The Big Picture

A Java application consists of **one process** that contains both **shared memory** and **thread-specific memory**.

```mermaid
flowchart TB

P["Java Process"]

P --> H["Heap (Shared)"]
P --> M["Method Area (Shared)"]

P --> T1["Thread A"]
P --> T2["Thread B"]
P --> T3["Thread C"]
```

Notice something interesting.

The process owns the memory.

Threads **do not own the process**.

Instead, every thread executes using the resources that belong to the process.

This design allows threads to communicate efficiently by sharing the same memory.

---

# A Mental Model

Think of a process as an office building.

- The **Heap** is a shared meeting room.
- The **Method Area** is the company handbook that everyone can read.
- Each **thread** is an employee.
- Every employee has their own desk (their own stack).

```
Office Building (Java Process)

+--------------------------------------------------+
| Shared Meeting Room (Heap)                       |
|                                                  |
| Shared Handbook (Method Area)                    |
|                                                  |
|  Employee A Desk      Employee B Desk            |
|  (Stack)              (Stack)                    |
|                                                  |
|  Employee C Desk                               |
|  (Stack)                                        |
+--------------------------------------------------+
```

Employees can safely organize papers on **their own desk**.

However, if multiple employees write on the same whiteboard in the meeting room, they must coordinate.

The same principle applies to Java threads.

---

# Memory Layout of a Java Process

Although the JVM implementation may vary slightly, a Java process is conceptually divided into the following memory regions.

```text
                 Java Process

+------------------------------------------------------+
|                  Heap (Shared)                       |
+------------------------------------------------------+

+------------------------------------------------------+
|              Method Area (Shared)                    |
+------------------------------------------------------+

 Thread A                Thread B              Thread C

+-----------+          +-----------+         +-----------+
| PC        |          | PC        |         | PC        |
+-----------+          +-----------+         +-----------+
| Java Stack|          | Java Stack|         | Java Stack|
+-----------+          +-----------+         +-----------+
| Native    |          | Native    |         | Native    |
| Stack     |          | Stack     |         | Stack     |
+-----------+          +-----------+         +-----------+
```

Let's understand each region one by one.

---

# Heap (Shared Memory)

The **Heap** is the largest memory region in a Java process.

It stores objects and arrays created during program execution.

For example,

```java
Student student = new Student();
```

The `Student` object is allocated on the **Heap**.

Likewise,

```java
int[] numbers = new int[100];
```

The array is also stored on the Heap.

> [!NOTE]
> Almost every object you create using `new` is allocated on the Heap.

---

## Why is the Heap Shared?

Imagine a banking application.

```java
BankAccount account = new BankAccount();
```

Now suppose two threads are running.

- Thread A deposits money.
- Thread B checks the balance.

Both threads must access **the same bank account**.

If every thread had its own copy of the object, the balance would become inconsistent.

Therefore, the object lives in **shared memory**.

```mermaid
flowchart LR

T1["Thread A"]
T2["Thread B"]

H["BankAccount Object
(Heap)"]

T1 --> H
T2 --> H
```

Both threads reference the same object.

This makes communication fast.

Unfortunately...

It also introduces the possibility of **race conditions**, which we'll explore in later chapters.

---

# What Lives on the Heap?

Some common examples include:

| Stored on Heap | Example |
|----------------|---------|
| Objects | `new Student()` |
| Arrays | `new int[100]` |
| Instance Variables | `student.name` |
| Collections | `ArrayList`, `HashMap`, `HashSet` |
| String Objects* | `"Hello"` (with JVM optimizations such as the String Pool) |

> [!TIP]
> If multiple threads hold a reference to the same object, they are all accessing the same Heap memory.

---

# Method Area (Shared Memory)

The **Method Area** stores information about classes rather than objects.

When a class is loaded by the JVM, information such as:

- Class metadata
- Method bytecode
- Static variables
- Runtime constant pool

is stored in the Method Area.

For example,

```java
class Student {

    static int totalStudents = 0;

    String name;
}
```

The class definition itself is stored once in the Method Area.

Every `Student` object created later lives on the Heap.

```mermaid
flowchart LR

M["Method Area"]

M --> C["Student.class"]

H["Heap"]

H --> O1["Student Object"]
H --> O2["Student Object"]
H --> O3["Student Object"]
```

Notice the relationship.

One class definition.

Many object instances.

---

# Static Variables

Static variables belong to the **class**, not to individual objects.

```java
class Counter {

    static int count = 0;
}
```

There is only **one** copy of `count`.

Every object and every thread accesses the same variable.

```text
Method Area

Counter.class

count = 0
```

This makes static variables another form of **shared memory**.

> [!WARNING]
> Static variables are shared across all threads. Updating them without synchronization can lead to race conditions.

---

# Heap vs Method Area

| Heap | Method Area |
|------|-------------|
| Stores objects | Stores class metadata |
| Stores arrays | Stores bytecode |
| Shared by all threads | Shared by all threads |
| Created during object allocation | Created during class loading |

---

## Summary So Far

At this point, we've identified two important shared memory regions.

| Memory Region | Shared Between Threads? |
|---------------|-------------------------|
| Heap | ✅ Yes |
| Method Area | ✅ Yes |

This raises an interesting question.

> If everything were shared, how could one thread execute independently from another?

The answer lies in **thread-local memory**.

In the next section, we'll explore the **Java Stack**, **Program Counter**, and **Native Method Stack**, and we'll see why local variables are naturally thread-safe.


# Thread-Local Memory

In the previous section, we learned that the **Heap** and **Method Area** are shared by all threads.

If every piece of memory were shared, multiple threads would constantly interfere with each other.

Fortunately, that's not how the JVM works.

Every thread owns its own private memory, allowing it to execute independently without affecting other threads.

```mermaid
flowchart TB

P["Java Process"]

P --> H["Heap (Shared)"]
P --> M["Method Area (Shared)"]

P --> T1["Thread A"]
P --> T2["Thread B"]

T1 --> S1["Java Stack"]
T1 --> PC1["Program Counter"]

T2 --> S2["Java Stack"]
T2 --> PC2["Program Counter"]
```

Notice that while both threads access the same Heap, each thread has its own **Java Stack** and **Program Counter**.

This separation is what allows multiple threads to execute concurrently.

---

# Java Stack

Every thread has its own **Java Stack**.

Unlike the Heap, a stack is **never shared** with other threads.

Its primary responsibility is to store information about the methods currently being executed.

Whenever a method is called, the JVM creates a new **Stack Frame** and pushes it onto the thread's stack.

When the method finishes, that frame is removed.

```mermaid
flowchart TD

A["main()"] --> B["calculateSalary()"]
B --> C["calculateTax()"]
```

During execution, the stack looks like this:

```text
Top of Stack
+--------------------------+
| calculateTax()           |
+--------------------------+
| calculateSalary()        |
+--------------------------+
| main()                   |
+--------------------------+
Bottom of Stack
```

As methods return, the stack unwinds in the reverse order.

```
calculateTax() returns
        ↓
calculateSalary() returns
        ↓
main() returns
```

This behavior is known as **Last In, First Out (LIFO)**.

---

# What Does a Stack Frame Contain?

Every stack frame stores the information required to execute a method.

A typical stack frame contains:

- Local variables
- Method parameters
- Intermediate computation results
- Return address

For example,

```java
public void greet(String name) {

    int age = 20;

    System.out.println(name);
}
```

A simplified stack frame would look like:

```text
greet()

--------------------------
Parameter : name
Local Variable : age
Return Address
--------------------------
```

Once `greet()` finishes execution, the entire frame disappears.

No garbage collection is required.

The memory is automatically reclaimed.

---

# Why Are Local Variables Thread-Safe?

This is one of the most important concepts in Java concurrency.

Consider the following method.

```java
public void printSquare(int number) {

    int square = number * number;

    System.out.println(square);
}
```

Suppose two threads call this method simultaneously.

```java
printSquare(5);
printSquare(10);
```

Although both threads execute the same method, each thread creates **its own stack frame**.

```text
Thread A Stack

printSquare()

number = 5

square = 25
```

```text
Thread B Stack

printSquare()

number = 10

square = 100
```

Even though the method is the same, the variables are completely different.

Each thread owns its own copy.

> [!IMPORTANT]
> Local variables are thread-safe because they live inside a thread's private stack.

This is why methods with only local variables generally don't require synchronization.

---

# Program Counter (PC Register)

Every thread also owns a **Program Counter (PC Register)**.

The Program Counter stores the address of the **next instruction** that the thread should execute.

Think of it as a bookmark inside a book.

If you stop reading at page 150, the bookmark remembers where you left off.

Similarly, the Program Counter remembers where a thread should resume execution.

```text
Thread A

main()

↓

Instruction 24

Program Counter = 24
```

Meanwhile,

```text
Thread B

download()

↓

Instruction 87

Program Counter = 87
```

Each thread maintains its own execution position.

Without a separate Program Counter, the operating system would not know where to resume a thread after a context switch.

---

# Context Switching

Suppose a single CPU core is executing two threads.

```text
Time

Thread A
██████

Thread B
      ██████

Thread A
            ██████
```

When the CPU switches from Thread A to Thread B, it saves Thread A's current Program Counter.

Later, when Thread A is scheduled again, execution resumes from the exact instruction where it previously stopped.

This entire process is known as a **context switch**.

> [!TIP]
> The Program Counter is one of the pieces of information saved and restored during every context switch.

---

# Native Method Stack

Java applications occasionally call methods written in native languages such as C or C++.

Examples include:

- File system operations
- Network communication
- Operating system APIs

These methods execute outside the JVM.

Each thread therefore maintains a separate **Native Method Stack** for native method execution.

Although you'll rarely interact with it directly, it's part of every Java thread's memory layout.

---

# Shared vs Thread-Local Memory

We can now summarize everything we've learned.

| Memory Region | Shared? | Owned By |
|--------------|---------|----------|
| Heap | ✅ Yes | Process |
| Method Area | ✅ Yes | Process |
| Java Stack | ❌ No | Individual Thread |
| Program Counter | ❌ No | Individual Thread |
| Native Method Stack | ❌ No | Individual Thread |

This distinction is the foundation of Java concurrency.

Whenever multiple threads modify data stored in **shared memory**, synchronization may be required.

When threads work only with their own private memory, synchronization is generally unnecessary.

---

# A Complete Picture

```text
                    Java Process

+---------------------------------------------------------+
|                     Heap (Shared)                       |
|---------------------------------------------------------|
|  User Objects                                           |
|  Arrays                                                 |
|  Collections                                             |
+---------------------------------------------------------+

+---------------------------------------------------------+
|                  Method Area (Shared)                   |
|---------------------------------------------------------|
|  Class Metadata                                         |
|  Static Variables                                       |
|  Bytecode                                               |
+---------------------------------------------------------+


Thread A                           Thread B

+-------------------+         +-------------------+
| Program Counter   |         | Program Counter   |
+-------------------+         +-------------------+
| Java Stack        |         | Java Stack        |
| Frame 1           |         | Frame 1           |
| Frame 2           |         | Frame 2           |
+-------------------+         +-------------------+
| Native Stack      |         | Native Stack      |
+-------------------+         +-------------------+

         \                          /
          \                        /
           \                      /
            \                    /
             \                  /
              +----------------+
              | Shared Heap    |
              +----------------+
```

> [!IMPORTANT]
> Remember this diagram. Almost every concurrency concept in Java—from race conditions to locks, `volatile`, atomic variables, and thread pools—builds upon this memory organization.

---

## Before Moving On

Let's answer a question that often confuses beginners.

If every thread has its own stack, then why do race conditions happen at all?

The answer is simple:

- Local variables live on the stack and are private.
- Objects live on the Heap and are shared.

Two threads don't fight over their own stacks.

They fight over **shared objects on the Heap**.

In the next section, we'll use this understanding to explain race conditions with a simple `Counter` example.


---

# Putting Everything Together

Let's revisit a simple example.

```java
class Counter {

    int count = 0;

    public void increment() {
        count++;
    }
}
```

Suppose two threads execute the following code simultaneously:

```java
Counter counter = new Counter();

Thread A:
counter.increment();

Thread B:
counter.increment();
```

Where is the `Counter` object stored?

From the previous sections, we know that:

- The `Counter` object lives on the **Heap**.
- The Heap is shared by all threads.

```mermaid
flowchart LR

T1["Thread A"] --> H["Counter Object (Heap)"]
T2["Thread B"] --> H
```

Both threads access the **same object**.

This is perfectly legal—but it also introduces the possibility of both threads modifying the object at the same time.

---

# Why `count++` Isn't Atomic

Many developers assume that this statement

```java
count++;
```

is a single operation.

It isn't.

The JVM breaks it down into multiple steps.

```text
Read count
      ↓
Add 1
      ↓
Write updated value
```

Now imagine the following execution order.

```text
Initial value = 0

Thread A reads 0

Thread B reads 0

Thread A writes 1

Thread B writes 1
```

Final value:

```text
1
```

Expected value:

```text
2
```

One update has been lost.

This is known as a **Race Condition**.

> [!WARNING]
> A race condition occurs when multiple threads access and modify shared data without proper coordination, causing the result to depend on the timing of execution.

---

# Visualizing the Problem

```mermaid
sequenceDiagram

participant A as Thread A
participant H as Heap
participant B as Thread B

A->>H: Read count (0)
B->>H: Read count (0)

A->>H: Write 1
B->>H: Write 1
```

Both threads perform the correct logic.

The problem is that they perform it **at the same time**.

---

# Why Local Variables Don't Have This Problem

Now compare it with this method.

```java
public int square(int number) {

    int result = number * number;

    return result;
}
```

Suppose two threads call it simultaneously.

```java
square(5);

square(10);
```

Memory layout:

```text
Thread A Stack

number = 5

result = 25
```

```text
Thread B Stack

number = 10

result = 100
```

The variables belong to different stacks.

They never interact.

Therefore, no synchronization is required.

---

# Shared Memory vs Thread-Local Memory

This entire chapter can be summarized using one simple rule.

| Thread-Local Memory | Shared Memory |
|---------------------|---------------|
| Java Stack | Heap |
| Program Counter | Method Area |
| Native Stack | Static Variables |

When multiple threads work only with **thread-local memory**, they operate independently.

When they access **shared memory**, coordination may be required.

> [!IMPORTANT]
> Most concurrency problems occur because multiple threads access the same data stored in shared memory.

---

# Mental Checklist

Whenever you write concurrent code, ask yourself:

### Question 1

**Is this data shared between multiple threads?**

- Yes → Continue.
- No → No synchronization is needed.

---

### Question 2

**Can multiple threads modify this data?**

- Yes → There is a possibility of a race condition.
- No → Concurrent reads are generally safe.

---

### Question 3

**Do I need synchronization?**

If multiple threads can read and write the same shared data, the answer is usually **yes**.

The synchronization mechanism depends on the use case, but the underlying reason is always the same:

> Multiple threads are accessing the same memory.

---

# Common Misconceptions

> [!WARNING]
> **"Every variable is shared between threads."**
>
> Incorrect.
>
> Local variables are stored on a thread's private stack.

---

> [!WARNING]
> **"Objects are copied for every thread."**
>
> Incorrect.
>
> Objects are typically stored once on the Heap and shared by all threads that hold a reference to them.

---

> [!WARNING]
> **"`count++` is a single operation."**
>
> Incorrect.
>
> It consists of multiple steps and is therefore not thread-safe.

---

# Summary

Let's summarize everything we've learned in this chapter.

| Memory Region | Shared | Purpose |
|---------------|:------:|---------|
| Heap | ✅ | Stores objects and arrays |
| Method Area | ✅ | Stores class metadata and static members |
| Java Stack | ❌ | Stores method frames, local variables, and parameters |
| Program Counter | ❌ | Tracks the next instruction to execute |
| Native Method Stack | ❌ | Supports execution of native methods |

---

# Key Takeaways

- A Java application runs as a **process**.
- Every process contains both **shared** and **thread-local** memory.
- Objects are usually allocated on the **Heap**.
- Every thread has its own **Java Stack**.
- Local variables are naturally thread-safe because they are stored on the thread's private stack.
- Shared objects can be accessed by multiple threads simultaneously.
- Race conditions occur when shared data is modified without proper synchronization.

---

# What's Next?

Now that we understand **where threads store data** and **why shared memory can lead to race conditions**, we're ready to explore how Java creates and manages threads.

In the next chapter, we'll learn:

- The `Thread` class
- The `Runnable` interface
- The `Callable` interface
- Why `Runnable` is preferred over extending `Thread`
- `start()` vs `run()`

By the end of the next chapter, you'll be able to create and execute threads confidently while understanding the memory model behind them.