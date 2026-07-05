# Core Java — Multithreading Complete Guide
### Senior Java Architect Interview Prep Guide (Core Concepts → Modern Java/Loom)

> Source: DURGASOFT Core Java Study Material, Chapters 7 & 8 — merged, rewritten, corrected, expanded with
> validated/executable code, current JDK behavior (JDK 8 → JDK 21+ LTS), and interview Q&A.
>
> This single guide combines two source documents:
> - **Part I — Chapter 7: Multithreading** (core concepts: Thread/Runnable, priority, synchronization, wait/notify, deadlock, daemon threads, life cycle, race conditions, modern `java.util.concurrent`, and Virtual Threads)
> - **Part II — Chapter 8: Multi-Threading Enhancements** (ThreadGroup, ThreadLocal, InheritableThreadLocal, `java.util.concurrent.locks`, thread pools/Executor Framework, Callable & Future, and further modern Java updates)

---

## Table of Contents

### [Part I — Chapter 7: Multithreading](#part-i--chapter-7-multithreading)
- [1. Introduction to Multitasking & Multithreading](#1-introduction-to-multitasking--multithreading)
- [2. Defining, Instantiating, and Starting a Thread](#2-defining-instantiating-and-starting-a-thread)
- [3. Thread Class Constructors](#3-thread-class-constructors)
- [4. Thread Naming](#4-thread-naming)
- [5. Thread Priority](#5-thread-priority)
- [6. Methods to Pause/Control Thread Execution: `yield()`, `join()`, `sleep()`](#6-methods-to-pausecontrol-thread-execution-yield-join-sleep)
- [7. Interrupting a Thread](#7-interrupting-a-thread)
- [8. Synchronization](#8-synchronization)
- [9. Inter-Thread Communication (`wait()` / `notify()` / `notifyAll()`)](#9-inter-thread-communication-wait--notify--notifyall)
- [10. Deadlock](#10-deadlock)
- [11. Daemon Threads](#11-daemon-threads)
- [12. Deprecated APIs & Miscellaneous Concepts](#12-deprecated-apis--miscellaneous-concepts)
- [13. Life Cycle of a Thread](#13-life-cycle-of-a-thread)
- [14. Race Condition](#14-race-condition)
- [15. Modern Java Concurrency (Java 5 → Java 21+) — Must Know for Architects](#15-modern-java-concurrency-java-5--java-21--must-know-for-architects)
- [16. Virtual Threads (Project Loom, JDK 21 LTS)](#16-virtual-threads-project-loom-jdk-21-lts)
- [17. Interview Cheat Sheet](#17-interview-cheat-sheet)
- [18. Interview Questions & Answers](#18-interview-questions--answers)
- [Appendix: Full Working Reference Program (Combines Multiple Concepts)](#appendix-full-working-reference-program-combines-multiple-concepts)

### [Part II — Chapter 8: Multi-Threading Enhancements](#part-ii--chapter-8-multi-threading-enhancements)
- [1. ThreadGroup](#1-threadgroup)
- [2. ThreadLocal](#2-threadlocal)
- [3. InheritableThreadLocal](#3-inheritablethreadlocal)
- [4. java.util.concurrent.locks Package](#4-javautilconcurrentlocks-package)
- [5. Thread Pools (Executor Framework)](#5-thread-pools-executor-framework)
- [6. Callable and Future](#6-callable-and-future)
- [7. Modern Java Concurrency Updates (Java 8 → 21+)](#7-modern-java-concurrency-updates-java-8--21)
- [8. Architect-Level Cheat Sheet](#8-architect-level-cheat-sheet)
- [9. Interview Questions](#9-interview-questions)
- [Appendix: Quick Reference — Common Pitfalls](#appendix-quick-reference--common-pitfalls)

---

# Part I — Chapter 7: Multithreading

## 1. Introduction to Multitasking & Multithreading

**Multitasking**: Executing several tasks simultaneously.

| Type | Definition | Level | Example |
|---|---|---|---|
| **Process-based multitasking** | Each task is an independent, separate **process** with its own memory space | OS level | Listening to MP3, downloading a file, and typing code — all at once |
| **Thread-based multitasking (Multithreading)** | Each task is an independent part **within the same process**, sharing memory | Programmatic level | A word processor spell-checking while you type |

**Key architectural facts:**
- A **Thread** is the smallest unit of execution within a process; it is often called a *lightweight process*.
- Threads within the same process **share the heap** (instance/static data) but each thread has its **own stack**, **own PC (program counter)**, and **own set of registers**.
- Java has built-in multithreading support via a rich API: `Thread`, `Runnable`, `Callable`, `ThreadGroup`, `ThreadLocal`, `ExecutorService`, `java.util.concurrent.*`. Compared to C/C++ (pthreads, manual OS calls), Java abstracts ~90% of the low-level plumbing — the developer mostly defines *what* the thread should do (`run()`), and the JVM/JDK API does the rest (registration with the scheduler, context switching support, memory visibility rules via the *Java Memory Model*).

**Application areas of multithreading:**
1. Multimedia graphics / animations / games.
2. Web & application servers (Tomcat, Netty — one thread per request or event-loop workers).
3. Improving **throughput** and reducing **response time** — the single objective of *any* multitasking, process- or thread-based.
4. Modern: reactive systems, microservices handling concurrent I/O, background job schedulers.

---

## 2. Defining, Instantiating, and Starting a Thread

Java gives **two classic ways** (plus modern alternatives covered in §15–16):

```
                Runnable
                   ↑↑
             ┌─────┘└─────┐
          Thread      (lambda / Callable / functional impl)
             ↑
         MyThread   (extends Thread)     MyRunnable (implements Runnable)
        [1st Approach]                   [2nd Approach]
```

### 2.1 Approach 1 — Extending `Thread` class

```java
class MyThread extends Thread {
    @Override
    public void run() {                 // job of the thread
        for (int i = 0; i < 10; i++) {
            System.out.println("child Thread");
        }
    }
}

public class ThreadDemo {
    public static void main(String[] args) {
        MyThread t = new MyThread();     // Step 1: Instantiation
        t.start();                       // Step 2: Start (registers with scheduler)

        for (int i = 0; i < 5; i++) {
            System.out.println("main thread");
        }
    }
}
```

### 2.2 Approach 2 — Implementing `Runnable` interface (RECOMMENDED)

`Runnable` lives in `java.lang` and is a **functional interface** (single abstract method `run()`), so from Java 8 onward it can be expressed as a lambda.

```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            System.out.println("child Thread");
        }
    }
}

public class ThreadDemo {
    public static void main(String[] args) {
        MyRunnable r = new MyRunnable();
        Thread t = new Thread(r);        // r is the "target Runnable"
        t.start();

        for (int i = 0; i < 10; i++) {
            System.out.println("main thread");
        }
    }
}
```

**Modern (Java 8+) lambda equivalent — no separate class needed:**
```java
public class ThreadDemo {
    public static void main(String[] args) {
        Runnable r = () -> {
            for (int i = 0; i < 10; i++) System.out.println("child Thread");
        };
        new Thread(r, "worker-1").start();
    }
}
```

> ⚠️ **Non-determinism note:** Output ordering between `main thread` and `child Thread` lines is **never guaranteed**. It is decided by the **Thread Scheduler**, which is part of the JVM (mapped onto the OS scheduler). Its algorithm is **JVM-vendor dependent** — never write code (or tests) that assumes a specific interleaving. Use synchronization primitives (`join`, `CountDownLatch`, etc.) when order matters.

### 2.3 `Best approach` — Why `implements Runnable` is preferred

| Criterion | `extends Thread` | `implements Runnable` |
|---|---|---|
| Inheritance flexibility | Class already extends `Thread` → **cannot extend any other class** (Java has single inheritance) | Class remains free to extend another class + implement `Runnable` |
| Reusability / Separation of concerns | The "job" is tightly coupled to the `Thread` object | The task (`Runnable`) is decoupled from the execution mechanism (`Thread`, `ExecutorService`, etc.) — same `Runnable` can be submitted to a thread pool |
| Object-oriented design | Violates *"favor composition over inheritance"* | Follows it |
| Modern usage | Rare — used only for niche cases | Standard; `ExecutorService.submit(Runnable/Callable)` expects this abstraction |

```java
// Both are valid combos when extra flexibility is needed
class MyClass extends Thread implements Runnable { }        // valid but redundant
class MyClass extends Frame implements Runnable { }         // valid — Frame + threading capability
// class MyClass extends Frame, Thread                       // INVALID – no multiple class inheritance
```

### 2.4 Case Studies from `start()` / `run()` semantics

#### Case 1 — Thread Scheduler
If multiple threads are `Runnable` (ready), **which runs first is decided by the Thread Scheduler (JVM-internal)** — behavior is not part of the Java Language Spec and is JVM/OS dependent. Do not rely on exact interleaving or output order in multithreaded programs.

#### Case 2 — `t.start()` vs `t.run()`

| | `t.start()` | `t.run()` |
|---|---|---|
| New OS/JVM thread created? | **Yes** | **No** |
| Who executes the job? | The new thread | The **calling thread** (e.g., `main`) — behaves like a plain method call |
| Concurrency achieved? | Yes | No |

```java
MyThread t = new MyThread();
t.run();     // NO new thread; runs synchronously on main — defeats the purpose of threading
```
Output when `run()` is called directly: all "child thread" lines print **first**, fully, then all "main thread" lines — because there is no interleaving at all; it is sequential execution on the main thread.

#### Case 3 — Importance of `Thread.start()`
```
start() {
    1. Registers the thread with the Thread Scheduler (internally allocates a native OS thread).
    2. Performs other mandatory JVM-level bookkeeping.
    3. Invokes run().
}
```
`start()` is the **heart of multithreading** — there is *no other way* to start a new thread of execution in Java without invoking it.

#### Case 4 — Not overriding `run()`
```java
class MyThread extends Thread { }   // run() not overridden

MyThread t = new MyThread();
t.start();   // No compile error, no output — Thread's default run() is empty
```
It is **highly recommended to always override `run()`**; otherwise there's no point using multithreading.

#### Case 5 — Overloading `run()`
`Thread.start()` **always** invokes the **no-argument `run()`** internally (it's hardcoded in native `Thread` machinery). Any overloaded `run(int i)` etc. must be invoked **explicitly** like a normal method — it will NOT run on a separate thread.
```java
class MyThread extends Thread {
    public void run() { System.out.println("no arg method"); }
    public void run(int i) { System.out.println("int arg method"); }
}
// t.start() -> prints "no arg method" only
```

#### Case 6 — Overriding `start()` (NOT recommended)
If you override `start()`, it behaves like a plain method call from the calling thread — **no new thread is created**, and `run()` is not auto-invoked unless you call it explicitly.
```java
class MyThread extends Thread {
    public void start() { System.out.println("start method"); }
    public void run()   { System.out.println("run method"); }
}
// t.start() -> prints "start method" (executed by main); "run method" is NEVER printed
```

#### Case 7 — Overriding `start()` without calling `super.start()`
Output is entirely produced by the **main thread**; `run()` never executes because the real native thread machinery in `Thread.start()` was bypassed.

#### Case 7b — Overriding `start()` **with** `super.start()`
```java
class MyThread extends Thread {
    public void start() {
        super.start();                          // triggers real thread creation -> run()
        System.out.println("start method");
    }
    public void run() { System.out.println("run method"); }
}
```
Now a **real child thread** executes `run()`, while `start method` and `main method` are printed by the main thread — output ordering between them is non-deterministic (race), but `run method` is guaranteed to be produced by a separate thread.

> ✅ **Golden Rule:** Never override `start()`. If you must customize startup logic, do it in `run()` or wrap thread creation in a factory/`ThreadFactory`.

#### Case 8 — Life Cycle Overview (detailed in §13)
`new → start() → Runnable/Ready → (scheduler allocates CPU) → Running → run() completes → Dead`

#### Case 9 — Restarting a Thread
```java
MyThread t = new MyThread();
t.start();          // valid
t.start();          // throws java.lang.IllegalThreadStateException
```
Once a thread has been started (and moved past `NEW` state), it **can never be restarted** — this is enforced by the JVM's internal thread-state machine. If you need "restart" semantics, create a **new `Thread` instance** wrapping the same `Runnable`.

### 2.5 Case Study: Runnable target vs Thread's own `run()`

```java
class MyRunnable implements Runnable {
    public void run() {
        for (int i = 0; i < 5; i++) System.out.println("child Thread");
    }
}

MyRunnable r  = new MyRunnable();
Thread     t1 = new Thread();       // no target Runnable
Thread     t2 = new Thread(r);      // target Runnable = r
```

| Call | New Thread Created? | Method executed | Notes |
|---|---|---|---|
| `t1.start()` | ✅ | `Thread`'s own (empty) `run()` | Prints nothing extra beyond "main thread" |
| `t1.run()` | ❌ | `Thread`'s own `run()` as plain call | Sequential |
| `t2.start()` | ✅ | `MyRunnable.run()` (the *target*) | True concurrency |
| `t2.run()` | ❌ | `MyRunnable.run()` as plain call | Sequential |
| `r.start()` | — | **Compile-time error**: `cannot find symbol: method start()` | `Runnable` has no `start()`; only `Thread` does |
| `r.run()` | ❌ | `MyRunnable.run()` as plain call | Sequential |

**Interview takeaway:** Only `t2.start()` produces genuine concurrent execution of `MyRunnable.run()`.

---

## 3. Thread Class Constructors

```java
Thread t = new Thread();
Thread t = new Thread(Runnable r);
Thread t = new Thread(String name);
Thread t = new Thread(Runnable r, String name);
Thread t = new Thread(ThreadGroup g, String name);
Thread t = new Thread(ThreadGroup g, Runnable r);
Thread t = new Thread(ThreadGroup g, Runnable r, String name);
Thread t = new Thread(ThreadGroup g, Runnable r, String name, long stackSize);
```
*(JDK 9 added one more overload: `Thread(ThreadGroup, Runnable, String, long, boolean inheritThreadLocals)`.)*

### "Not-recommended" pattern — wrapping a `Thread` inside another `Thread`
```java
class MyThread extends Thread {
    public void run() { System.out.println("run method"); }
}

MyThread t  = new MyThread();
Thread   t1 = new Thread(t);   // Thread(Runnable) constructor — t acts as target Runnable
t1.start();
// Output: "run method" — because Thread.run() checks: if(target != null) target.run();
```
This works because `Thread` itself implements `Runnable`, and `Thread.run()`'s default implementation delegates to the target `Runnable` if one was supplied in the constructor. It is confusing and **not recommended** — pick one clean approach (extends OR implements).

---

## 4. Thread Naming

Every thread has a name — either explicitly set by the programmer, or auto-generated by the JVM in the form `"Thread-0"`, `"Thread-1"`, ...

```java
public final String getName()
public final void   setName(String name)
public static Thread currentThread();   // static — returns reference to the executing thread
```

```java
class MyThread extends Thread {}

public class ThreadNameDemo {
    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getName());  // "main"

        MyThread t = new MyThread();
        System.out.println(t.getName());                        // "Thread-0"

        Thread.currentThread().setName("Bhaskar Thread");
        System.out.println(Thread.currentThread().getName());   // "Bhaskar Thread"
    }
}
```

> 💡 **Architect tip:** In production systems, always give threads meaningful names (`new Thread(r, "order-processor-1")` or via a custom `ThreadFactory` for an `ExecutorService`) — critical for reading thread dumps and diagnosing deadlocks/hangs.

---

## 5. Thread Priority

- Valid range: **1 to 10** (never 0). 1 = lowest, 10 = highest.
- Constants defined in `Thread`:

```java
Thread.MIN_PRIORITY    // 1
Thread.NORM_PRIORITY   // 5
Thread.MAX_PRIORITY    // 10
```
> There are **no** constants like `Thread.LOW_PRIORITY` or `Thread.HIGH_PRIORITY`.

```java
public final int  getPriority()
public final void setPriority(int newPriority);  // throws IllegalArgumentException if not in [1,10]
```

**Default priority rules:**
- `main` thread's default priority = **5**.
- All **other** threads **inherit** priority from their **parent thread** at creation time (not always 5).

```java
class MyThread extends Thread {}

public class ThreadPriorityDemo {
    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getPriority());  // 5
        Thread.currentThread().setPriority(9);
        MyThread t = new MyThread();
        System.out.println(t.getPriority());                       // 9 (inherited from main)
    }
}
```

**Behavioral facts:**
- Higher priority threads *tend to* get scheduled first — but this is only a **hint** to the Thread Scheduler, not a guarantee.
- Between threads of equal priority, execution order is scheduler/vendor-dependent (typically round-robin/time-sliced on modern JVMs).
- Some OSes (historically Windows) provide weak or no real support for Java thread priorities — **don't design correctness-critical logic around priority**.

> ⚠️ **Architect note:** Thread priority is a **JVM scheduling hint only**. Never use it as a substitute for proper synchronization, and never assume a priority-10 thread will preempt a priority-1 thread deterministically. Modern designs favor `ExecutorService` with dedicated pools/queues over relying on OS priority.

---

## 6. Methods to Pause/Control Thread Execution: `yield()`, `join()`, `sleep()`

### 6.1 `yield()`

```java
public static native void yield();
```
- Causes the **currently executing thread** to pause and give a chance to **other waiting threads of the same priority**.
- If no equal/higher priority thread is waiting, the same thread may continue immediately.
- **Purely a hint** — the scheduler decides; behavior is not guaranteed or portable.
- Does **NOT** release any lock held by the thread.

```java
class MyThread extends Thread {
    public void run() {
        for (int i = 0; i < 5; i++) {
            Thread.yield();
            System.out.println("child thread");
        }
    }
}
```

> In modern JDKs, `yield()` is rarely used in application code — it's largely a legacy hint with vendor-dependent effect. Prefer higher-level constructs (`Thread.sleep`, `java.util.concurrent` synchronizers, or simply restructure with `ExecutorService`).

### 6.2 `join()`

Used when one thread must **wait for another thread to finish**.

```java
public final void join()                          throws InterruptedException
public final void join(long millis)                throws InterruptedException
public final void join(long millis, int nanos)      throws InterruptedException
```

```java
class MyThread extends Thread {
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println("child thread");
            try { Thread.sleep(200); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }
    }
}

public class ThreadJoinDemo {
    public static void main(String[] args) throws InterruptedException {
        MyThread t = new MyThread();
        t.start();
        t.join();                 // main waits until t finishes completely
        for (int i = 0; i < 5; i++) System.out.println("main thread");
    }
}
```
- `join()` throws the **checked** `InterruptedException` → must handle with `try/catch` or propagate via `throws`.
- If `t1` calls `t2.join()` and `t2` calls `t1.join()` → **both wait forever** (deadlock-like hang).
- If a thread calls `join()` on **itself** (`Thread.currentThread().join()`), it also hangs forever.

**Modern best practice:** In production concurrent code, prefer `CompletableFuture.join()`, `ExecutorService.awaitTermination()`, or `CountDownLatch` over raw `Thread.join()` for composability.

### 6.3 `sleep()`

```java
public static native void sleep(long millis)              throws InterruptedException
public static void        sleep(long millis, int nanos)    throws InterruptedException
```
- Pauses the **current thread** for (at least) the given time — does **not** release any locks it holds (unlike `wait()`).
- Actual sleep duration is *at least* the requested time; OS scheduling jitter may add more.

```java
public class SleepDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("M");
        Thread.sleep(1000);
        System.out.println("E");
        Thread.sleep(1000);
        System.out.println("G");
        Thread.sleep(1000);
        System.out.println("A");
    }
}
```

### 6.4 Comparison Table: `yield()` vs `join()` vs `sleep()`

| Property | `yield()` | `join()` | `sleep()` |
|---|---|---|---|
| Purpose | Give chance to same-priority waiting threads | Wait for another thread to die | Pause current thread for fixed time |
| `static`? | Yes | No (instance method) | Yes |
| `final`? | No | Yes | No |
| Overloaded? | No | Yes (3 overloads) | Yes (2 overloads) |
| Throws `InterruptedException`? | No | Yes | Yes |
| Releases lock held? | No | No | No |
| Native method? | Yes | No | `sleep(long)` → native; `sleep(long,int)` → non-native |

---

## 7. Interrupting a Thread

```java
public void interrupt()          // "signal" a thread to stop waiting/sleeping
public boolean isInterrupted()   // check flag without clearing it
public static boolean interrupted()  // static; checks AND clears the flag on current thread
```

```java
class MyThread extends Thread {
    public void run() {
        try {
            for (int i = 0; i < 5; i++) {
                System.out.println("i am lazy Thread: " + i);
                Thread.sleep(2000);
            }
        } catch (InterruptedException e) {
            System.out.println("i got interrupted");
        }
    }
}

public class ThreadInterruptDemo {
    public static void main(String[] args) {
        MyThread t = new MyThread();
        t.start();
        t.interrupt();                    // asks the sleeping child to wake up with an exception
        System.out.println("end of main thread");
    }
}
```

**Key semantics:**
- If the target thread is **sleeping or waiting** at the time `interrupt()` is called → it's interrupted **immediately**, receiving `InterruptedException`.
- If the target thread is **not** currently sleeping/waiting → the interrupt request is just a **flag set**; it takes effect the *next* time the thread calls a blocking method (`sleep`, `wait`, `join`, blocking I/O in NIO channels, etc.).
- If the thread **never** enters a blocking/waiting state during its lifetime → the interrupt call has **no effect** (the flag is simply set and ignored until thread death).

> ✅ **Architect best practice:** Always **re-interrupt** the thread if you swallow `InterruptedException` and can't propagate it, to preserve the interrupt status for higher-level code:
> ```java
> catch (InterruptedException e) {
>     Thread.currentThread().interrupt();   // restore interrupt status
> }
> ```
> Never silently swallow `InterruptedException` in production — it can cause `ExecutorService` shutdown or cancellation logic to hang.

---

## 8. Synchronization

### 8.1 Fundamentals

1. `synchronized` is a **keyword**, applicable to **methods and blocks only** — NOT to classes or variables (as a modifier).
2. If a method/block is `synchronized`, **only one thread at a time** can execute it **on a given object**.
3. **Advantage:** Prevents data-inconsistency / race conditions.
4. **Disadvantage:** Increases thread wait time; can degrade throughput — use judiciously ("synchronize only what's necessary").
5. Internally implemented via the **intrinsic lock / monitor** concept.
6. **Every object in Java has a unique intrinsic lock (monitor)**. The lock concept only becomes relevant when `synchronized` is used.
7. A thread must **acquire the object's lock** before executing any `synchronized` method/block on that object; the lock is released automatically when execution completes (including via exception).
8. While one thread holds the lock and executes a synchronized method, other threads **cannot execute any synchronized method on the *same* object** simultaneously — but they **can** freely execute **non-synchronized** methods on that same object concurrently. (Lock is per-**object**, not per-**method**.)

### 8.2 Example — Effect with & without `synchronized`

```java
class Display {
    public /*synchronized*/ void wish(String name) {
        for (int i = 0; i < 5; i++) {
            System.out.print("good morning:");
            try { Thread.sleep(200); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            System.out.println(name);
        }
    }
}

class MyThread extends Thread {
    Display d; String name;
    MyThread(Display d, String name) { this.d = d; this.name = name; }
    public void run() { d.wish(name); }
}

public class SynchronizedDemo {
    public static void main(String[] args) {
        Display d1 = new Display();
        MyThread t1 = new MyThread(d1, "dhoni");
        MyThread t2 = new MyThread(d1, "yuvaraj");
        t1.start();
        t2.start();
    }
}
```
- **Without `synchronized`:** Output is interleaved/garbled — e.g. `good morning:good morning:yuvaraj` — a classic **data inconsistency / interleaving bug**.
- **With `synchronized`:** `dhoni`'s 5 lines print completely, *then* `yuvaraj`'s 5 lines (or vice versa) — regular, non-interleaved output, because `t1` and `t2` compete for the **same lock** (`d1`'s monitor).

### 8.3 Case Study — Multiple Objects ⇒ No Synchronization Effect

```java
Display d1 = new Display();
Display d2 = new Display();
MyThread t1 = new MyThread(d1, "dhoni");
MyThread t2 = new MyThread(d2, "yuvaraj");   // DIFFERENT object!
t1.start();
t2.start();
```
Even though `wish()` is `synchronized`, output is **still irregular** — because `t1` and `t2` are locking on **different monitors** (`d1` and `d2`), so they run fully concurrently.

> **Conclusion (critical interview point):**
> - Multiple threads operating on **multiple objects** → synchronization has **no effect**.
> - Multiple threads operating on the **same object** → synchronization is **required and effective**.

### 8.4 Class-Level Lock (`static synchronized`)

1. Every **class** (its `Class` object) also has a unique lock — required to execute a `static synchronized` method.
2. Once a thread acquires the class-level lock, it can execute any `static synchronized` method of that class.
3. Other threads **cannot** execute any `static synchronized` method of the same class concurrently — but **can** execute normal instance-synchronized methods, other static (non-sync) methods, or normal instance methods freely.
4. **Class-level lock and object-level lock are completely independent** — holding one does not imply/exclude the other.

```java
class Shared {
    public static synchronized void staticSyncMethod() { /* needs class-level lock (Shared.class) */ }
    public synchronized void instanceSyncMethod()      { /* needs object-level lock (this)          */ }
}
```

### 8.5 Synchronized Block

Preferred when only a **few lines** need protection — reduces waiting time / improves throughput vs. synchronizing the entire method.

```java
// 1) Lock on current object
synchronized (this) { /* critical section */ }

// 2) Lock on a specific object 'b'
synchronized (b) { /* critical section */ }

// 3) Class-level lock via block
synchronized (Display.class) { /* critical section */ }
```

> ⚠️ The argument to `synchronized(...)` must be an **object reference** or a `.class` literal — **never a primitive**:
> ```java
> int x = 5;
> synchronized (x) { }   // COMPILE-TIME ERROR: "int cannot be converted to Object" (unexpected type)
> ```

### 8.6 Can a Thread Hold More Than One Lock at a Time?

**Yes** — as long as they are locks on **different objects**:

```java
class X {
    synchronized void methodOne() {          // acquires lock on X-instance
        Y y = new Y();
        y.methodTwo();                       // then acquires lock on Y-instance too
    }
}
class Y {
    synchronized void methodTwo() { }
}
```
A thread executing `x.methodOne()` holds **two locks simultaneously** (`x`'s and `y`'s) — this is precisely the scenario that, when done inconsistently across threads (**lock ordering inversion**), causes **deadlock** (see §10).

### 8.7 `ReentrantLock` (Modern Alternative) — *see §15* for full coverage.

---

## 9. Inter-Thread Communication (`wait()` / `notify()` / `notifyAll()`)

- Defined in **`java.lang.Object`** (NOT `Thread`) — because any object can serve as the shared monitor multiple threads coordinate on.
- To call `wait()`, `notify()`, or `notifyAll()` on an object, the **calling thread must already own that object's lock** (i.e., be inside a `synchronized` block/method on that object). Otherwise → **`IllegalMonitorStateException`** (unchecked, `RuntimeException`) at runtime.

| Method | Signature | Releases Lock? |
|---|---|---|
| `wait()` | `public final void wait() throws InterruptedException` | **Yes — immediately** |
| `wait(long ms)` | `public final native void wait(long ms) throws InterruptedException` | Yes — immediately |
| `wait(long ms, int ns)` | `public final void wait(long ms, int ns) throws InterruptedException` | Yes — immediately |
| `notify()` | `public final native void notify()` | Yes, but **may not be immediate** (releases only once the synchronized block exits) |
| `notifyAll()` | `public final void notifyAll()` | Yes, but may not be immediate |

Contrast with `yield()`, `sleep()`, `join()` — **none of those release any lock**. `wait/notify/notifyAll` are the *only* mechanism where lock release happens as part of the call.

> Calling `wait()`/`notify()`/`notifyAll()` releases the lock **only for the specific object** it was called on — not any *other* locks the thread might be holding.

### 9.1 Basic Example

```java
class ThreadB extends Thread {
    int total = 0;
    public void run() {
        synchronized (this) {
            System.out.println("child thread starts calculation");
            for (int i = 0; i <= 100; i++) total += i;
            System.out.println("child thread giving notification call");
            this.notify();
        }
    }
}

public class ThreadA {
    public static void main(String[] args) throws InterruptedException {
        ThreadB b = new ThreadB();
        b.start();
        synchronized (b) {
            System.out.println("main Thread calling wait() method");
            b.wait();                                   // releases b's lock, waits
            System.out.println("main Thread got notification call");
            System.out.println(b.total);                 // 5050
        }
    }
}
```

### 9.2 Producer–Consumer Pattern (classic use case)

```java
import java.util.LinkedList;
import java.util.Queue;

class Buffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity;

    Buffer(int capacity) { this.capacity = capacity; }

    public synchronized void produce(int value) throws InterruptedException {
        while (queue.size() == capacity) {   // NOTE: use `while`, not `if` — guards spurious wakeups
            wait();
        }
        queue.add(value);
        System.out.println("Produced: " + value);
        notifyAll();                          // wake up any waiting consumers
    }

    public synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        int value = queue.poll();
        System.out.println("Consumed: " + value);
        notifyAll();                          // wake up any waiting producers
        return value;
    }
}

public class ProducerConsumerDemo {
    public static void main(String[] args) {
        Buffer buffer = new Buffer(5);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) buffer.produce(i);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "Producer");

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) buffer.consume();
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "Consumer");

        producer.start();
        consumer.start();
    }
}
```

> ✅ **Critical interview correction to the legacy material:** Always guard `wait()` calls in a **`while` loop**, never a plain `if`. This protects against:
> 1. **Spurious wakeups** (JLS explicitly permits the JVM to wake a waiting thread without any `notify()` call).
> 2. Multiple waiting consumers/producers being notified via `notifyAll()` but the condition no longer holding true for all of them by the time they re-acquire the lock.

### 9.3 `notify()` vs `notifyAll()`

| | `notify()` | `notifyAll()` |
|---|---|---|
| Wakes | Exactly **one** arbitrarily chosen waiting thread (JVM decides — no fairness guarantee) | **All** waiting threads on that object's monitor |
| Risk | Can cause **missed signals**/starvation if the "wrong" thread is picked when multiple distinct wait-conditions exist on the same monitor | Safe default when in doubt — slight extra overhead as all threads re-contend for the lock, but avoids lost-wakeup bugs |
| Recommendation | Use only when you are certain any one waiting thread can proceed correctly | **Prefer in production code** unless profiling proves it's a bottleneck |

### 9.4 Lock Correctness Rule

You may only call `wait()`/`notify()`/`notifyAll()` on the object whose **monitor you currently hold**:

```java
Stack s1 = new Stack();
Stack s2 = new Stack();

synchronized (s1) {
    s2.wait();     // INVALID -> IllegalMonitorStateException (thread doesn't hold s2's lock)
}

synchronized (s1) {
    s1.wait();     // VALID -> thread holds s1's lock
}
```

### 9.5 True/False Recap (frequently asked)

| Statement | Answer |
|---|---|
| `wait()` enters waiting state **without** releasing the lock | **False** |
| `wait()` releases the lock but not necessarily immediately | **False** — it's immediate |
| `wait()` releases **all** locks the thread holds | **False** — only the lock of that specific object |
| `wait()` immediately releases the lock of **that particular object** and waits | **True** |
| `notify()` immediately releases the object's lock | **False** |
| `notify()` releases the lock, but maybe not immediately (only when sync block/method exits) | **True** |

---

## 10. Deadlock

**Definition:** Two (or more) threads waiting for each other **forever** (infinite mutual waiting) — no thread can proceed.

- **No resolution technique** exists at runtime for deadlock; only **prevention/avoidance** strategies (careful lock ordering, timeouts, lock-free algorithms, using `tryLock()`).
- `synchronized` is the classic root cause — improper nested-lock ordering across threads.

### 10.1 Classic Deadlock Example

```java
class A {
    public synchronized void foo(B b) {
        System.out.println("Thread1 starts execution of foo()");
        try { Thread.sleep(1000); } catch (InterruptedException e) {}
        System.out.println("Thread1 trying to call b.last()");
        b.last();
    }
    public synchronized void last() { System.out.println("inside A.last()"); }
}

class B {
    public synchronized void bar(A a) {
        System.out.println("Thread2 starts execution of bar()");
        try { Thread.sleep(1000); } catch (InterruptedException e) {}
        System.out.println("Thread2 trying to call a.last()");
        a.last();
    }
    public synchronized void last() { System.out.println("inside B.last()"); }
}

public class DeadlockDemo implements Runnable {
    A a = new A();
    B b = new B();

    DeadlockDemo() {
        Thread t = new Thread(this);
        t.start();
        a.foo(b);          // executed by main thread — needs A's lock then B's lock
    }

    public void run() {
        b.bar(a);          // executed by child thread — needs B's lock then A's lock
    }

    public static void main(String[] args) {
        new DeadlockDemo();
    }
}
```
**Output (then hangs forever):**
```
Thread1 starts execution of foo() method
Thread2 starts execution of bar() method
Thread2 trying to call a.last()
Thread1 trying to call b.last()
// <-- program hangs here indefinitely (deadlock)
```

**Root cause:** `main` thread locks `A` then wants `B`; child thread locks `B` then wants `A` — **circular wait**. Removing *at least one* `synchronized` keyword breaks the cycle and prevents deadlock, proving `synchronized` (misuse) is the root cause.

### 10.2 Deadlock Prevention Strategies (Architect-Level)

1. **Lock ordering:** Always acquire multiple locks in a globally consistent order (e.g., always lock the object with the smaller `hashCode()`/ID first).
2. **`tryLock()` with timeout** (`java.util.concurrent.locks.ReentrantLock.tryLock(timeout, unit)`) — back off and retry instead of blocking indefinitely.
3. Minimize the scope/nesting of synchronized blocks; avoid calling out to unknown/overridable code while holding a lock.
4. Use higher-level, lock-free concurrent utilities (`java.util.concurrent` collections, `Atomic*` classes) where possible.
5. Use **thread-dump analysis** (`jstack`, VisualVM, JFR) — the JVM explicitly detects and reports deadlocks ("Found one Java-level deadlock") in thread dumps for `synchronized`-based deadlocks.
6. Prefer `java.util.concurrent.locks.Lock` interface over intrinsic locks when you need interruptible or timed lock acquisition — plain `synchronized` cannot be interrupted or time-bounded.

### 10.3 Deadlock vs. Starvation vs. Livelock

| Term | Definition |
|---|---|
| **Deadlock** | Long waiting that **never ends** — threads block each other permanently |
| **Starvation** | Long waiting that **eventually ends** — e.g., a low-priority thread perpetually loses the CPU to higher-priority threads but eventually runs |
| **Livelock** | *(Modern addition, not in legacy doc but commonly asked)* Threads are **not blocked**, but keep changing state in response to each other without making real progress (e.g., two people repeatedly stepping aside for each other in a hallway) |

---

## 11. Daemon Threads

- **Daemon threads** run in the background, providing support services to **non-daemon (user) threads**.
- Classic example: **Garbage Collector** thread — runs in background reclaiming memory so the `main` thread (non-daemon) can keep running.

```java
public final boolean isDaemon();
public final void    setDaemon(boolean b);
```

**Rules:**
- Daemon status can be changed **only before `start()`** is called. Attempting to change it after starting → **`IllegalThreadStateException`**.
- `main` thread is **always non-daemon**, and its daemon status can never be changed (it's already running by the time your code executes).
- For all other threads, daemon-ness is **inherited from the parent thread** at the time of creation (if parent is daemon → child is daemon by default, and vice-versa).
- **JVM exits automatically once all non-daemon (user) threads terminate** — any remaining daemon threads are abruptly terminated (their `finally` blocks may not even run).

```java
class MyThread extends Thread {
    public void run() {
        for (int i = 0; i < 10; i++) {
            System.out.println("lazy thread");
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
        }
    }
}

public class DaemonThreadDemo {
    public static void main(String[] args) {
        MyThread t = new MyThread();
        t.setDaemon(true);      // MUST be called before start()
        t.start();
        System.out.println("end of main Thread");
        // main terminates quickly -> JVM exits -> daemon thread killed mid-loop
    }
}
```

```java
MyThread t = new MyThread();
t.start();
t.setDaemon(true);   // throws RuntimeException: IllegalThreadStateException (already started!)
```

> ⚠️ **Production caution:** Because a daemon thread can be killed abruptly (without running `finally` blocks or closing resources), **never** use daemon threads for tasks that must complete cleanly (e.g., flushing a write-ahead log, committing a DB transaction). Use them only for pure background/best-effort work (cache eviction sweepers, metrics reporters, etc.). `ExecutorService`'s default `ThreadFactory` creates **non-daemon** threads for exactly this reason — you must opt in explicitly if you want daemon worker threads.

---

## 12. Deprecated APIs & Miscellaneous Concepts

### 12.1 `stop()`, `suspend()`, `resume()` — ⚠️ Deprecated, DO NOT USE

```java
public final void stop();      // Deprecated since JDK 1.2
public final void suspend();   // Deprecated since JDK 1.2
public final void resume();    // Deprecated since JDK 1.2
```
**Why deprecated:** `stop()` releases all locks the thread was holding **at whatever point of execution it was interrupted**, potentially leaving shared objects in an **inconsistent/corrupted state** — no chance for cleanup. `suspend()` can cause deadlocks because a suspended thread continues to hold any locks it had acquired, while other threads wait on those locks. These have been **terminally deprecated** for removal in newer JDKs (marked `forRemoval = true` in JDK 9+ Javadoc) — they are effectively unusable in modern Java and are asked about **only for legacy/interview context**.

### 12.2 Safe Way to Stop a Thread (the recommended pattern)

```java
class SafeStoppableTask implements Runnable {
    private volatile boolean stopRequested = false;    // volatile: visibility across threads

    public void requestStop() { stopRequested = true; }

    @Override
    public void run() {
        while (!stopRequested) {
            // do work...
        }
        System.out.println("Thread stopping gracefully");
    }
}

public class SafeStopDemo {
    public static void main(String[] args) throws InterruptedException {
        SafeStoppableTask task = new SafeStoppableTask();
        Thread t = new Thread(task);
        t.start();
        Thread.sleep(2000);
        task.requestStop();     // cooperative cancellation flag
        t.join();
    }
}
```
The `volatile` keyword is **essential** here — without it, the JVM/JIT may cache the flag's value in a CPU register/core-local cache and the stopping thread's write may never become visible to the worker thread (a classic **Java Memory Model visibility bug**). *(Modern equivalent: use `AtomicBoolean`, or better, `ExecutorService.shutdown()`/interrupt-based cancellation — see §15.)*

### 12.3 `ThreadGroup`

Groups related threads as a single manageable unit for bulk operations.

```java
ThreadGroup g = new ThreadGroup("Printing Threads");
Thread t = new Thread(g, "Header Printing");
// ... construct t1, t2, t3 all attached to g ...
g.stop();      // stop() is deprecated -> ThreadGroup bulk-control APIs are largely legacy today
```
> **Modern note:** `ThreadGroup` is a **legacy API** predating `java.util.concurrent`. Its bulk-control methods (`stop`, `suspend`, `resume`) are deprecated for the same reasons as `Thread.stop()`. In modern code, use `ExecutorService` (a pool of threads you fully control) instead of `ThreadGroup` for grouping/managing worker threads.

### 12.4 `ThreadLocal`

Provides **thread-confined storage** — each thread accessing a `ThreadLocal` variable gets its **own, independently initialized copy**.

```java
public class ThreadLocalDemo {
    private static final ThreadLocal<Integer> counter = ThreadLocal.withInitial(() -> 0);

    public static void increment() {
        counter.set(counter.get() + 1);
    }

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            for (int i = 0; i < 5; i++) increment();
            System.out.println(Thread.currentThread().getName() + " => " + counter.get());
        };
        Thread t1 = new Thread(task, "T1");
        Thread t2 = new Thread(task, "T2");
        t1.start(); t2.start();
        t1.join(); t2.join();
        // Each thread prints "=> 5" independently — no shared/interference state
    }
}
```
**Common real-world uses:**
- Per-thread `SimpleDateFormat`/`DecimalFormat` instances (these classes are not thread-safe).
- Per-thread DB `Connection` in older frameworks.
- Servlet container **request scope** context propagation (analogous to Servlet scopes: page/request/session/application).
- Distributed tracing/logging **MDC** (Mapped Diagnostic Context, e.g., SLF4J's `MDC` uses a `ThreadLocal` internally) — carrying a correlation/request ID across a call chain on the same thread.

> ⚠️ **Memory leak warning (important for architects):** In thread-pool environments (`ExecutorService`), threads are **reused** — if you don't call `ThreadLocal.remove()` after use, stale data leaks across tasks and can also cause classloader leaks in application servers. **Always `remove()` in a `finally` block.**
>
> ⚠️ **Virtual Threads caveat (JDK 21+):** `ThreadLocal` works with virtual threads too, but since virtual threads are cheap and numerous (potentially millions), heavy `ThreadLocal` usage per-thread can balloon memory. JDK 21 introduced **`ScopedValue`** (JEP 429/446, preview) as a lighter-weight, immutable alternative designed for structured concurrency and virtual threads.

### 12.5 Green Threads vs Native OS Threads (historical/legacy)

| Model | Description | Status |
|---|---|---|
| **Green Thread Model** | Threads managed **entirely by the JVM**, without OS support (JVM does its own scheduling in user-space) | Deprecated/abandoned; used by very old JVMs (e.g., old Solaris) |
| **Native OS Model** | Threads mapped 1:1 to real OS threads; OS scheduler manages them | Standard model for all mainstream JVMs today (`java.lang.Thread` = native OS thread — pre-JDK 21) |

> 🆕 **Modern twist:** JDK 21's **Virtual Threads** (§16) effectively *reintroduce* a Green-Thread-like model (JVM-managed, extremely lightweight, many-to-few mapped onto a small pool of OS "carrier" threads) — but built on modern foundations (continuations) that avoid the pitfalls (blocking I/O starving all green threads) that killed the original Green Thread model in the late 1990s.

### 12.6 Race Condition — see §14 (dedicated section, since it's a common interview topic).

---

## 13. Life Cycle of a Thread

```
                                                    ┌─────────────────────┐
                                                    │   Waiting/Blocked    │◄───────────┐
                                                    │  (join/wait/sleep)   │            │
                                                    └──────────┬───────────┘            │
                                                               │ condition met /        │
                                                               │ time expires /         │
                                                               │ notify() / interrupt() │
                                                               ▼                        │
new MyThread() ──start()──► [NEW → RUNNABLE] ──(scheduler picks)──► [RUNNING] ──────────┘
                                    ▲                                  │
                                    └────── yield() (voluntarily) ─────┘
                                                                       │
                                                          run() completes
                                                                       ▼
                                                                  [TERMINATED/DEAD]
```

### 13.1 States per `Thread.State` enum (modern, exact JDK model)

Since Java 5, the canonical API to inspect a thread's life-cycle state is:

```java
public State getState();   // Thread.State enum
```

| `Thread.State` | Meaning |
|---|---|
| `NEW` | Thread object created, `start()` not yet called |
| `RUNNABLE` | Executing in the JVM, OR eligible to run and waiting for CPU/OS scheduler (Java conflates "ready" and "running" into one state — unlike the legacy diagram's separate "Ready" and "Running" circles) |
| `BLOCKED` | Waiting to acquire an **intrinsic lock** (`synchronized`) held by another thread |
| `WAITING` | Waiting indefinitely for another thread's action — via `Object.wait()` (no timeout), `Thread.join()` (no timeout), or `LockSupport.park()` |
| `TIMED_WAITING` | Waiting for a **bounded** time — via `Thread.sleep(ms)`, `Object.wait(ms)`, `Thread.join(ms)`, `LockSupport.parkNanos`, `LockSupport.parkUntil` |
| `TERMINATED` | `run()` has completed (normally or via uncaught exception) |

```java
public class ThreadStateDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
        });
        System.out.println(t.getState());   // NEW
        t.start();
        System.out.println(t.getState());   // RUNNABLE
        Thread.sleep(500);
        System.out.println(t.getState());   // TIMED_WAITING
        t.join();
        System.out.println(t.getState());   // TERMINATED
    }
}
```

> **Note on the legacy 7-state diagram:** The original DURGASOFT diagram (New → Ready/Runnable → Running → Sleeping/Waiting/Suspended → Dead) predates the standardized `Thread.State` enum (introduced in Java 5 / Tiger). It is conceptually correct for teaching purposes, but **for interviews, always cite the official `Thread.State` enum's 6 values** above — that's what the JLS and modern JDK actually expose via `getState()`.

---

## 14. Race Condition

**Definition:** When multiple threads execute concurrently and access/modify **shared mutable state** without proper synchronization, leading to **data inconsistency** — the final result depends on the unpredictable timing/interleaving of thread execution.

### 14.1 Classic Example — Lost Update

```java
class Counter {
    private int count = 0;
    public void increment() { count++; }   // NOT ATOMIC: read -> add 1 -> write (3 steps)
    public int getCount() { return count; }
}

public class RaceConditionDemo {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Runnable task = () -> { for (int i = 0; i < 100_000; i++) counter.increment(); };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join(); t2.join();

        System.out.println(counter.getCount());
        // Expected: 200000. Actual: usually LESS (e.g., 141327) due to lost updates —
        // classic race condition because count++ is not atomic.
    }
}
```

### 14.2 Fixes

```java
// Fix 1: synchronized method
public synchronized void increment() { count++; }

// Fix 2: synchronized block (finer-grained)
private final Object lock = new Object();
public void increment() { synchronized (lock) { count++; } }

// Fix 3 (modern, preferred, lock-free): AtomicInteger
import java.util.concurrent.atomic.AtomicInteger;
private final AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); }
```

> ✅ **Architect recommendation:** For simple counters/flags, prefer `java.util.concurrent.atomic.*` (CAS-based, lock-free, better throughput under contention) over `synchronized`. Reserve `synchronized`/`ReentrantLock` for protecting **compound invariants across multiple fields**.

---

## 15. Modern Java Concurrency (Java 5 → Java 21+) — Must Know for Architects

The legacy `Thread`/`synchronized`/`wait-notify` model (Java 1.0–1.4) is foundational, but **since Java 5 (`java.util.concurrent`, JSR-166, by Doug Lea)**, production Java code overwhelmingly uses higher-level constructs. A Senior Architect interview will almost always probe this layer.

### 15.1 `ExecutorService` — Thread Pool Management (replaces manual `new Thread()`)

```java
import java.util.concurrent.*;

public class ExecutorDemo {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        for (int i = 1; i <= 8; i++) {
            int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " running on " + Thread.currentThread().getName());
            });
        }

        executor.shutdown();                                          // graceful shutdown, no new tasks
        if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
            executor.shutdownNow();                                   // force-cancel if not finished
        }
    }
}
```

**Common factory methods (`java.util.concurrent.Executors`):**

| Factory Method | Behavior | Caveat |
|---|---|---|
| `newFixedThreadPool(n)` | Fixed pool of `n` threads, unbounded queue | Can cause `OutOfMemoryError` under sustained overload (unbounded queue) |
| `newCachedThreadPool()` | Creates threads on demand, reuses idle ones, unbounded max | Can spawn unbounded threads under heavy load → resource exhaustion |
| `newSingleThreadExecutor()` | Single worker thread, tasks run sequentially in submission order | — |
| `newScheduledThreadPool(n)` | Supports delayed/periodic task execution | — |
| `newVirtualThreadPerTaskExecutor()` | **(JDK 21+)** One *virtual thread* per submitted task | See §16 |

> ⚠️ **Modern guidance (post Java 11+ best practice, reinforced by Brian Goetz & the JDK team):** Avoid the raw `Executors.newFixedThreadPool`/`newCachedThreadPool` convenience factories in production — they hide **unbounded queues or unbounded thread counts**, a common cause of production OOMs. Prefer constructing `ThreadPoolExecutor` explicitly with **bounded queues** and an explicit `RejectedExecutionHandler`:
> ```java
> ExecutorService executor = new ThreadPoolExecutor(
>     4, 8,                                   // core, max pool size
>     60L, TimeUnit.SECONDS,                  // idle thread keep-alive
>     new ArrayBlockingQueue<>(100),          // BOUNDED work queue
>     new ThreadPoolExecutor.CallerRunsPolicy()  // backpressure policy
> );
> ```

### 15.2 `Callable<V>` and `Future<V>` — Tasks That Return Results / Throw Checked Exceptions

`Runnable.run()` returns `void` and cannot throw checked exceptions. `Callable<V>` (since Java 5) fixes both:

```java
import java.util.concurrent.*;

public class CallableDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        Callable<Integer> task = () -> {
            Thread.sleep(500);
            return 42;
        };

        Future<Integer> future = executor.submit(task);

        System.out.println("Doing other work while task runs...");
        Integer result = future.get();          // blocks until result is available (or throws)
        System.out.println("Result: " + result);

        executor.shutdown();
    }
}
```

`Future` API: `get()`, `get(timeout, unit)`, `cancel(boolean mayInterruptIfRunning)`, `isDone()`, `isCancelled()`.

### 15.3 `CompletableFuture` (Java 8+) — Asynchronous, Composable Pipelines

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureDemo {
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> fetchUser(101))              // runs on ForkJoinPool.commonPool() by default
            .thenApply(user -> "Hello, " + user)
            .thenApply(String::toUpperCase)
            .exceptionally(ex -> "Fallback: " + ex.getMessage());

        System.out.println(future.get());

        // Combining two independent async calls:
        CompletableFuture<Integer> priceFuture = CompletableFuture.supplyAsync(() -> 250);
        CompletableFuture<Integer> qtyFuture   = CompletableFuture.supplyAsync(() -> 4);
        CompletableFuture<Integer> totalFuture = priceFuture.thenCombine(qtyFuture, (p, q) -> p * q);
        System.out.println("Total: " + totalFuture.get());
    }

    static String fetchUser(int id) { return "user-" + id; }
}
```
Key methods: `supplyAsync`, `runAsync`, `thenApply`, `thenCompose` (flatMap-like chaining of futures), `thenCombine`, `allOf`, `anyOf`, `exceptionally`, `handle`, `whenComplete`.

### 15.4 `java.util.concurrent.locks` — Explicit Locking

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.*;

public class ReentrantLockDemo {
    private final Lock lock = new ReentrantLock();
    private int counter = 0;

    public void increment() {
        lock.lock();
        try {
            counter++;
        } finally {
            lock.unlock();          // MUST be in finally — unlike synchronized, lock isn't auto-released
        }
    }

    public boolean tryIncrement(long timeoutMs) throws InterruptedException {
        if (lock.tryLock(timeoutMs, TimeUnit.MILLISECONDS)) {
            try { counter++; return true; } finally { lock.unlock(); }
        }
        return false;   // couldn't acquire lock in time -> avoid indefinite blocking (deadlock mitigation)
    }
}
```

**`ReentrantLock` vs `synchronized`:**

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Lock acquisition | Implicit (JVM-managed) | Explicit — `lock()` / `unlock()` |
| Interruptible wait | No | Yes — `lockInterruptibly()` |
| Timed acquisition | No | Yes — `tryLock(timeout, unit)` |
| Fairness policy | Not configurable | Configurable — `new ReentrantLock(true)` (FIFO fairness) |
| Multiple condition variables per lock | No (one implicit monitor: `wait/notify`) | Yes — `lock.newCondition()` → multiple `Condition`s |
| Auto-release on exception | Yes | **No** — must `unlock()` in `finally` |
| Performance | JIT-optimized (biased/lightweight locking) for low contention | Comparable/better under high contention |

Also: `ReadWriteLock`/`ReentrantReadWriteLock` (multiple concurrent readers, exclusive writer) and, since Java 16, the even more scalable `StampedLock` (supports optimistic reads).

### 15.5 Atomic Variables (`java.util.concurrent.atomic`)

Lock-free, CAS (Compare-And-Swap)-based thread-safe primitives:
```java
AtomicInteger, AtomicLong, AtomicBoolean, AtomicReference<V>,
LongAdder, DoubleAdder     // (Java 8+, better than AtomicLong under very high contention)
```
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();
counter.compareAndSet(5, 10);   // atomic CAS
```

### 15.6 Concurrent Collections

| Legacy (synchronized wrappers) | Modern Concurrent Alternative |
|---|---|
| `Collections.synchronizedMap(new HashMap<>())` | `ConcurrentHashMap<>()` — segment/bucket-level locking, far higher throughput |
| `Vector`, `Collections.synchronizedList` | `CopyOnWriteArrayList` (read-heavy, rarely-mutated lists) |
| `synchronized` on a `Set` | `ConcurrentHashMap.newKeySet()` / `CopyOnWriteArraySet` |
| Blocking hand-rolled queues | `BlockingQueue` implementations: `ArrayBlockingQueue`, `LinkedBlockingQueue`, `SynchronousQueue`, `DelayQueue`, `PriorityBlockingQueue` |

```java
BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);
queue.put(1);          // blocks if full
Integer val = queue.take();   // blocks if empty
// This replaces the entire manual wait/notify Producer-Consumer pattern from §9.2!
```

### 15.7 Synchronizers

| Class | Purpose |
|---|---|
| `CountDownLatch` | One-time gate: N threads wait until a countdown reaches zero (e.g., wait for N services to start) |
| `CyclicBarrier` | Reusable barrier: N threads wait for each other at a rendezvous point, repeatedly |
| `Semaphore` | Classic counting semaphore — limits concurrent access to a resource pool |
| `Exchanger<T>` | Two threads exchange objects at a synchronization point |
| `Phaser` | Flexible, reusable barrier supporting dynamic party registration (multi-phase tasks) |

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Service starting...");
        latch.countDown();
    }).start();
}
latch.await();   // main blocks here until all 3 threads call countDown()
System.out.println("All services started!");
```

### 15.8 Fork/Join Framework (Java 7+) — Divide & Conquer Parallelism

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

class SumTask extends RecursiveTask<Long> {
    private final int[] arr; private final int start, end;
    SumTask(int[] arr, int start, int end) { this.arr = arr; this.start = start; this.end = end; }

    protected Long compute() {
        if (end - start <= 1000) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += arr[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left  = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();                              // async execute left half
        long rightResult = right.compute();       // compute right half in current thread
        long leftResult = left.join();             // wait for left half
        return leftResult + rightResult;
    }
}

public class ForkJoinDemo {
    public static void main(String[] args) {
        int[] data = new int[1_000_000];
        java.util.Arrays.fill(data, 1);
        long result = new ForkJoinPool().invoke(new SumTask(data, 0, data.length));
        System.out.println("Sum: " + result);
    }
}
```
This is also the engine behind **parallel streams** (`list.parallelStream()`), which internally use `ForkJoinPool.commonPool()`.

### 15.9 Parallel Streams (Java 8+) — Declarative Data Parallelism

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
int sum = nums.parallelStream().mapToInt(Integer::intValue).sum();
```
> ⚠️ **Caution:** Parallel streams share the **common `ForkJoinPool`** JVM-wide — a long-running/blocking task submitted via `parallelStream()` can starve unrelated parallel-stream operations elsewhere in the application (a frequent production incident cause). Prefer explicit `ExecutorService`s for I/O-bound or long-running parallel work; reserve parallel streams for **CPU-bound, short-lived, non-blocking** computations on reasonably large datasets.

### 15.10 Java Memory Model (JMM) & `volatile` — Why Visibility Matters

- The **Java Memory Model** (JLS Chapter 17, formalized by JSR-133 in Java 5) defines the rules for when writes by one thread become **visible** to reads by another thread.
- Without synchronization (`synchronized`, `volatile`, or `java.util.concurrent` classes), the JVM/JIT/CPU are free to **reorder instructions** and cache values in registers/CPU cores — a thread may **never** see another thread's update ("visibility" problem), independent of the "atomicity"/race-condition problem covered in §14.
- **`volatile`** guarantees:
  1. **Visibility** — every read of a `volatile` variable sees the most recent write from any thread (no stale caching).
  2. **Ordering** — establishes a **happens-before** relationship: all writes before a `volatile` write (in program order) are visible to any thread that reads that volatile field afterward.
  3. It does **NOT** guarantee atomicity for compound actions like `count++` (read-modify-write) — for that you still need `synchronized` or `Atomic*`.

```java
private volatile boolean running = true;   // visibility guaranteed across threads
public void stop() { running = false; }
public void run()  { while (running) { /* work */ } }   // will correctly observe the flag flip
```

---

## 16. Virtual Threads (Project Loom, JDK 21 LTS)

> **This is one of the most significant Java platform changes in 20+ years — expect deep questions on this in any 2024+ Senior Architect interview.**

### 16.1 Background & Motivation

- **JEP 444** — *Virtual Threads* — finalized (out of preview) in **JDK 21 (September 2023, LTS)**. Previewed earlier in JDK 19 (JEP 425) and JDK 20 (JEP 436).
- Traditional `Thread` objects (pre-Loom) are **1:1 mapped to OS/kernel threads** — each is relatively "heavyweight" (~1MB+ default stack, OS-level context-switch cost). This caps practical concurrency to roughly thousands of threads per JVM, forcing architectures toward async/reactive/callback-based, non-blocking I/O styles (e.g., Netty, reactive frameworks) purely to scale — at the cost of code readability ("callback hell"/complex reactive chains).
- **Virtual threads** are JVM-managed, extremely lightweight threads (a few hundred bytes to a few KB), enabling **millions** of concurrent virtual threads in a single JVM, while still writing simple, sequential, blocking-style code.

### 16.2 How It Works

- Virtual threads are scheduled by the JVM (via `ForkJoinPool`-based schedulers), running on top of a small, fixed pool of **platform (carrier) threads** — analogous to (but far more robust than) the old Green Thread model.
- When a virtual thread performs a **blocking operation** (e.g., blocking I/O, `Thread.sleep()`, blocking on a `java.util.concurrent` lock/queue), the JVM **unmounts** it from its carrier thread and mounts a *different* waiting virtual thread — the underlying OS thread is **not** blocked/wasted.
- Existing blocking-style APIs (`InputStream.read()`, JDBC calls, `synchronized`, `ReentrantLock`, etc.) work "for free" with virtual threads in most cases — **no code rewrite needed** to benefit, unlike migrating to reactive/async APIs.

### 16.3 Creating Virtual Threads

```java
// 1. Directly:
Thread vt = Thread.ofVirtual().name("vt-1").start(() -> {
    System.out.println("Running in: " + Thread.currentThread());
});
vt.join();

// 2. Via an ExecutorService (RECOMMENDED for task-oriented workloads):
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        int taskId = i;
        executor.submit(() -> {
            Thread.sleep(Duration.ofMillis(100));   // blocking call — cheap with virtual threads!
            return taskId;
        });
    }
} // executor auto-closes (AutoCloseable since JDK 19+) and awaits completion of submitted tasks

// 3. Unstarted builder form (useful with ThreadFactory-based frameworks):
ThreadFactory factory = Thread.ofVirtual().factory();
Thread t = factory.newThread(() -> System.out.println("hi"));
t.start();
```

You can spin up **100,000+ virtual threads**, each blocking on `Thread.sleep(100)`, and the whole program completes in roughly ~100ms wall-clock — something that would exhaust memory/OS resources instantly with 100,000 platform threads.

### 16.4 Platform Thread vs Virtual Thread — Comparison

| Aspect | Platform Thread (`Thread`, pre-Loom style) | Virtual Thread (JDK 21+) |
|---|---|---|
| Mapping | 1:1 with OS kernel thread | M:N — many virtual threads share few carrier (platform) threads |
| Creation cost | Expensive (~1MB stack, OS call) | Extremely cheap (grows/shrinks dynamically, starts at a few hundred bytes) |
| Max practical count | Thousands | Millions |
| Blocking I/O impact | Blocks (wastes) the OS thread | "Unmounts" from carrier — OS thread freed to run other virtual threads |
| Thread pooling needed? | Yes, essential (reuse is expensive to create) | **No — create a new one per task; never pool virtual threads** |
| `synchronized` blocking behavior | Fine | ⚠️ **Pinning**: a virtual thread blocked inside a `synchronized` block/method (or during native method calls) **cannot unmount** — it pins its carrier thread. Prefer `ReentrantLock` over `synchronized` in virtual-thread-heavy hot paths (improved further in JDK 24, which largely removed this pinning limitation for `synchronized`) |
| `ThreadLocal` | Fine, standard usage | Works, but discouraged at scale — prefer `ScopedValue` (JEP 446, finalized-track) |
| Daemon by default? | No | **Yes — virtual threads are always daemon threads**; `setDaemon(false)` throws/ignored |
| Priority | Configurable, honored (loosely) by OS scheduler | `setPriority()` is a no-op (fixed at `NORM_PRIORITY`) |

### 16.5 Structured Concurrency (Preview, JEP 480 in JDK 21/24 track)

Complements virtual threads — treats a group of related concurrent subtasks as a single unit of work with well-defined lifetimes, propagating cancellation/errors cleanly:

```java
// Preview API — package: java.util.concurrent (StructuredTaskScope), enable with --enable-preview
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user  = scope.fork(() -> fetchUser());
    Future<Integer> order = scope.fork(() -> fetchOrderCount());

    scope.join();           // wait for both, or fail fast if either throws
    scope.throwIfFailed();  // propagate first exception, if any

    System.out.println(user.resultNow() + " has " + order.resultNow() + " orders");
}
```
This directly solves the ergonomic and error-handling gaps of manually juggling multiple `Future`s / `CompletableFuture`s.

### 16.6 When to Use Virtual Threads

✅ **Great fit:** High-throughput I/O-bound server workloads (e.g., handling many concurrent HTTP requests/DB calls each doing blocking I/O), replacing complex reactive pipelines with simple sequential blocking code, thread-per-request server architectures (Tomcat, Jetty, Spring Boot 3.2+ support "virtual threads" mode via `spring.threads.virtual.enabled=true`).

❌ **Not a fit:** CPU-bound parallel computation (virtual threads don't add computational parallelism — you're still bounded by CPU cores; use Fork/Join or parallel streams for that). Also not useful if you rely heavily on thread-pool-size-based throttling/backpressure semantics — that model doesn't map directly onto "unlimited" virtual threads (use `Semaphore` for concurrency limiting instead).

---

## 17. Interview Cheat Sheet

### 17.1 One-Liners

| Concept | One-Line Definition |
|---|---|
| Thread | Smallest, independently schedulable unit of execution within a process |
| Process | Independent OS-level program instance with its own memory space |
| `start()` | Registers thread with scheduler + spawns new call stack → eventually invokes `run()` |
| `run()` | Contains the actual job; if called directly, executes synchronously on the caller's thread |
| Race Condition | Unsynchronized concurrent access to shared mutable state causing unpredictable results |
| Deadlock | Circular, permanent mutual waiting between ≥2 threads for locks held by each other |
| Starvation | A thread is perpetually denied CPU/resources (but not forever — eventually resolves) |
| Livelock | Threads keep responding to each other's state changes without making forward progress |
| Synchronized | Keyword ensuring mutual exclusion on a monitor (object or class lock) |
| `wait()`/`notify()` | Object-level methods for inter-thread signaling; require holding the object's monitor |
| Daemon Thread | Background/service thread; JVM exits once all non-daemon threads finish, killing daemons |
| `volatile` | Guarantees visibility + ordering (happens-before), NOT atomicity of compound operations |
| Virtual Thread | JVM-managed, ultra-lightweight thread (JDK 21+) enabling millions of concurrent blocking-style tasks |

### 17.2 Method Origin Quick Reference

| Method | Declared In | Notes |
|---|---|---|
| `start()`, `run()`, `sleep()`, `yield()`, `join()`, `interrupt()`, `setPriority()`, `setDaemon()`, `getState()` | `java.lang.Thread` | — |
| `wait()`, `notify()`, `notifyAll()` | `java.lang.Object` | Callable on ANY object, must hold its monitor |
| `lock()`, `unlock()`, `tryLock()`, `newCondition()` | `java.util.concurrent.locks.Lock` (e.g. `ReentrantLock`) | Explicit locking |

### 17.3 Locks / Concurrency Toolbox Map

```
synchronized (intrinsic lock)  ─┬─► method-level  (instance / static)
                                 └─► block-level   (this / object / Class literal)

java.util.concurrent
 ├─ locks:       ReentrantLock, ReentrantReadWriteLock, StampedLock
 ├─ atomic:      AtomicInteger/Long/Boolean/Reference, LongAdder
 ├─ executors:   ExecutorService, ScheduledExecutorService, ThreadPoolExecutor, ForkJoinPool
 ├─ futures:     Future, CompletableFuture
 ├─ sync aids:   CountDownLatch, CyclicBarrier, Semaphore, Exchanger, Phaser
 ├─ collections: ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue impls
 └─ (JDK 21+)    Virtual Threads, StructuredTaskScope, ScopedValue
```

### 17.4 Do's and Don'ts

| ✅ Do | ❌ Don't |
|---|---|
| Guard `wait()` in a `while` loop | Use `if` around `wait()` (misses spurious wakeups) |
| Always `unlock()` a `ReentrantLock` in `finally` | Forget `unlock()` — causes permanent deadlock |
| Re-interrupt (`Thread.currentThread().interrupt()`) after swallowing `InterruptedException` | Silently swallow `InterruptedException` |
| Prefer `java.util.concurrent` utilities over raw `wait/notify`/`Thread` | Hand-roll producer-consumer with raw `wait/notify` in new code |
| Name your threads / configure custom `ThreadFactory` | Leave threads as "Thread-0", "Thread-1"... in production |
| Use bounded queues + explicit `ThreadPoolExecutor` | Use `Executors.newFixedThreadPool`/`newCachedThreadPool` blindly in production |
| Use `ReentrantLock`/`Atomic*` in virtual-thread hot paths to avoid pinning | Overuse `synchronized` around blocking calls inside virtual threads (pre-JDK24 pinning) |

---

## 18. Interview Questions & Answers

### Basics & Fundamentals

**Q1. What is a Thread? How is it different from a Process?**
A Thread is the smallest unit of execution scheduled by the OS/JVM, existing within a Process. Threads within the same process **share heap memory** (instance/static variables) but have their **own stack, PC register, and call frames**. A Process is an independent, isolated execution environment with its own memory space; inter-process communication is expensive (IPC), while inter-thread communication is cheap (shared memory) but requires careful synchronization.

**Q2. Which thread runs by default in every Java program?**
The `main` thread, created automatically by the JVM to invoke `public static void main(String[] args)`. It has default priority `5` and is non-daemon.

**Q3. What is the default priority of a thread?**
`main` thread defaults to `5` (`Thread.NORM_PRIORITY`). Any other newly-created thread **inherits its parent's priority** at creation time (not necessarily 5).

**Q4. Which method does a Thread execute internally?**
Only `public void run()`. `start()` sets up the native thread machinery and then internally invokes `run()`.

**Q5. What is the difference between `extends Thread` and `implements Runnable`? Which is preferred and why?**
`extends Thread` overrides `Thread`'s `run()` directly, but locks you out of extending any other class (Java single inheritance). `implements Runnable` decouples the task from the execution mechanism, keeps your class free to extend another class, and integrates naturally with `ExecutorService.submit()`. **`implements Runnable` (or better, a lambda / `Callable`) is the recommended, idiomatic approach.**

**Q6. What happens if you call `run()` directly instead of `start()`?**
No new thread is created. `run()` executes as an ordinary synchronous method call on the **calling thread** — you lose all concurrency benefits.

**Q7. Can you restart a thread once it has completed?**
No. Calling `start()` on an already-started (or terminated) `Thread` instance throws `IllegalThreadStateException`. You must create a **new** `Thread` object (it can wrap the same `Runnable`/task) to run it again.

**Q8. What is the Thread Scheduler?**
A JVM-internal (backed by the OS scheduler) component responsible for deciding which `RUNNABLE` thread gets CPU time next. Its exact algorithm (priority-based, round-robin, time-sliced, etc.) is **implementation/vendor-dependent** and unspecified by the JLS — Java programs must never rely on a specific execution order.

**Q9. What are the Thread class constructors?**
`Thread()`, `Thread(Runnable r)`, `Thread(String name)`, `Thread(Runnable r, String name)`, plus overloads taking a `ThreadGroup` and optionally a `stackSize` (and, since JDK 9, an `inheritThreadLocals` boolean).

**Q10. How do you name a thread, and how do you get the currently executing thread?**
`thread.setName("...")` / `thread.getName()`; the currently executing thread is obtained via the **static** method `Thread.currentThread()`.

### Priority, `yield`, `join`, `sleep`

**Q11. What is the valid range of thread priority, and what are the standard constants?**
1 (lowest) to 10 (highest) — `Thread.MIN_PRIORITY = 1`, `Thread.NORM_PRIORITY = 5`, `Thread.MAX_PRIORITY = 10`. Setting any other value throws `IllegalArgumentException`. There is no `LOW_PRIORITY`/`HIGH_PRIORITY` constant.

**Q12. Does higher priority guarantee earlier/faster execution?**
No — priority is only a **scheduling hint**. Execution order among equal, or even different, priority threads is not deterministically guaranteed by the JLS; it's OS/JVM dependent.

**Q13. Explain `yield()`. Does it release the lock?**
`yield()` is a static, native hint that tells the scheduler the current thread is willing to pause for other **same-priority** threads. It does **not** release any lock the thread holds and doesn't guarantee anything will actually change—purely advisory.

**Q14. What does `join()` do? What exception does it throw and why?**
`join()` makes the calling thread wait until the target thread terminates (optionally bounded by a timeout). It throws the **checked** `InterruptedException` because the waiting thread can be interrupted by another thread while blocked in `join()`.

**Q15. What happens if two threads call `join()` on each other?**
Both threads wait for each other indefinitely — a hang analogous to deadlock (mutual permanent waiting), though it's not "deadlock" in the classic *lock-acquisition* sense; it's a mutual-join hang.

**Q16. Difference between `sleep()` and `wait()`?**
`sleep(ms)` is a **static** `Thread` method that pauses the current thread for a fixed duration **without releasing any lock** it holds, and does not require synchronization context. `wait()` is an **instance method of `Object`**, requires the caller to hold the object's monitor (else `IllegalMonitorStateException`), and **immediately releases that object's lock** while waiting — resuming only via `notify()`/`notifyAll()`/timeout/interruption.

**Q17. Compare `yield()`, `join()`, `sleep()` across: static, final, overloaded, throws checked exception, releases lock.**
See the comparison table in §6.4. Key exam answer: none of the three release any lock; only `wait/notify/notifyAll` do.

### Synchronization & Locks

**Q18. Explain the `synchronized` keyword, its advantages, and disadvantages.**
`synchronized` (method or block) enforces mutual exclusion — only one thread can execute a given synchronized method/block on a given object/class at a time, preventing race conditions/data corruption. Disadvantage: it increases thread waiting time and can hurt throughput/scalability if overused; also risks deadlock if lock ordering isn't managed carefully.

**Q19. What is an object lock (monitor) and when is it required?**
Every object has an implicit intrinsic lock. It's required/acquired when a thread enters a `synchronized` **instance** method or a `synchronized(objectRef){}` block.

**Q20. What is a class-level lock and when is it required?**
Every `Class` object also has a unique lock, acquired when a thread executes a `static synchronized` method (or a `synchronized(SomeClass.class){}` block). It's independent of any instance-level object lock.

**Q21. Is object lock the same as class-level lock?**
No — completely independent. A thread can hold both simultaneously without conflict; holding one does not block or grant the other.

**Q22. If Thread-A is executing a synchronized method on object `obj`, can Thread-B execute a *different* synchronized method on the *same* `obj` simultaneously?**
**No.** The lock is per-object (monitor), not per-method — any synchronized method/block on the same object is mutually exclusive across threads. Thread-B *can*, however, execute any **non-synchronized** method on `obj` concurrently.

**Q23. What is a synchronized block, and why prefer it over a synchronized method?**
A `synchronized(lockObj){ ... }` block wraps only the specific lines needing protection, rather than the whole method — reducing lock hold time and improving throughput/concurrency. Use it when only a small critical section (not the entire method) touches shared mutable state.

**Q24. Can a thread hold more than one lock simultaneously?**
Yes, provided the locks are on different objects (e.g., calling a synchronized method on object `y` from within a synchronized method on object `x`). This scenario, done with inconsistent ordering across threads, is the classic root cause of **deadlock**.

**Q25. What object types can be passed as the argument to `synchronized(...)`?**
Any object reference or a `.class` literal (for class-level locking). **Primitives cannot** be used — `synchronized(x)` where `x` is `int` is a **compile-time error** ("unexpected type — found: int, required: reference").

**Q26. Difference between `synchronized` and `ReentrantLock`?**
See §15.4 comparison table. Key differentiators: explicit lock/unlock (must `unlock()` in `finally`), interruptible (`lockInterruptibly()`) and timed (`tryLock(timeout)`) acquisition, configurable fairness, and support for multiple `Condition` objects per lock — none of which plain `synchronized` supports.

### wait/notify & Deadlock

**Q27. Why are `wait()`/`notify()`/`notifyAll()` declared in `Object`, not `Thread`?**
Because any object can act as a shared monitor/lock that multiple, unrelated threads coordinate on — the coordination target is the *shared resource object*, not a specific `Thread` instance.

**Q28. What happens if you call `wait()` outside a synchronized context?**
Throws `IllegalMonitorStateException` (unchecked) — the calling thread must already own the object's monitor.

**Q29. Does `wait()` release all locks held by the thread?**
No — only the lock of the **specific object** on which `wait()` was invoked. Any other locks the thread holds remain held.

**Q30. Difference between `notify()` and `notifyAll()`?**
`notify()` wakes exactly one arbitrarily-chosen waiting thread (no fairness guarantee, risk of missed signals when multiple distinct conditions share a monitor). `notifyAll()` wakes all waiting threads, which then re-compete for the lock — safer default to avoid lost-wakeup bugs, at a small performance cost.

**Q31. Why must `wait()` be called in a `while` loop, not an `if`?**
To protect against **spurious wakeups** (JLS-permitted) and against the possibility that, by the time a woken thread re-acquires the lock, the condition it was waiting for is no longer true (e.g., another thread already consumed the item in a producer-consumer scenario).

**Q32. What is deadlock? How do you prevent/detect it?**
Deadlock = circular, permanent mutual waiting for locks among ≥2 threads (e.g., T1 holds Lock-A wants Lock-B; T2 holds Lock-B wants Lock-A). No runtime *resolution* exists — prevention techniques: consistent global lock ordering, `tryLock()` with timeout, minimizing nested locking, and lock-free data structures. Detection: thread dumps (`jstack`) explicitly report "Found one Java-level deadlock", JConsole/VisualVM/JFR deadlock detectors.

**Q33. Difference between deadlock, starvation, and livelock?**
Deadlock never resolves; starvation eventually resolves (a thread is delayed, not permanently blocked); livelock means threads are active (not blocked) but making no real progress because they keep reacting to each other's state changes.

### Daemon Threads & Life Cycle

**Q34. What is a daemon thread? Give an example.**
A background/service thread supporting non-daemon (user) threads; e.g., the JVM's Garbage Collector thread. The JVM terminates once all non-daemon threads finish, abruptly killing any remaining daemon threads (their `finally` blocks may not run).

**Q35. Can you change a thread's daemon status after calling `start()`?**
No — throws `IllegalThreadStateException`. Daemon status must be set **before** `start()`.

**Q36. Is the `main` thread daemon or non-daemon? Can this be changed?**
Always **non-daemon**, and this cannot be changed — it's already running when your code gets control.

**Q37. Explain the `Thread.State` enum values.**
`NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, `TERMINATED` — see §13.1 for exact transition triggers for each.

**Q38. What's the difference between `BLOCKED` and `WAITING` states?**
`BLOCKED` = waiting to **acquire an intrinsic (`synchronized`) lock** held by another thread. `WAITING`/`TIMED_WAITING` = a thread has voluntarily suspended itself pending another thread's action (`wait()`, `join()`, `LockSupport.park()`), independent of lock contention.

### Race Conditions, Deprecated APIs, Misc

**Q39. What is a race condition? How do you fix it?**
Unsynchronized concurrent read-modify-write access to shared mutable state causing unpredictable/incorrect results (e.g., `count++` not being atomic). Fix via `synchronized`, `ReentrantLock`, or (preferably, for simple counters) `java.util.concurrent.atomic` classes.

**Q40. Why are `Thread.stop()`, `suspend()`, `resume()` deprecated?**
`stop()` can leave shared objects in a corrupted/inconsistent state because it releases all locks the thread held at whatever arbitrary point it was forcibly terminated, with no chance for cleanup. `suspend()` can cause deadlocks because a suspended thread keeps holding any locks it acquired while other threads wait on them. Both were **deprecated since JDK 1.2**, and marked `forRemoval=true` in modern Javadocs.

**Q41. How do you safely stop a running thread today?**
Use a cooperative cancellation flag (`volatile boolean` or `AtomicBoolean`) checked periodically inside `run()`, or (preferred with `ExecutorService`/thread pools) call `Thread.interrupt()` and have the task check `Thread.currentThread().isInterrupted()` / handle `InterruptedException` to exit gracefully.

**Q42. What is `ThreadLocal` used for? What's a common pitfall?**
Provides a per-thread, independently-initialized copy of a variable (e.g., non-thread-safe `SimpleDateFormat`, MDC/correlation IDs in logging). Pitfall: in thread-pool environments, forgetting to call `remove()` causes stale-data leaks (and potential memory/classloader leaks) because pooled threads are reused across tasks.

**Q43. What is `ThreadGroup`? Is it still relevant?**
A legacy mechanism to group related threads for bulk operations (`stop`, `interrupt`, etc.). Its destructive bulk methods are deprecated for the same reasons as `Thread.stop()`. Modern code uses `ExecutorService` for managing groups of worker threads instead.

**Q44. What's the difference between the Green Thread model and the Native OS Thread model?**
Green threads are scheduled entirely by the JVM without OS involvement (historical, deprecated, e.g., old Solaris JVMs). Native OS threads map 1:1 to real OS/kernel threads and are scheduled by the OS — the standard model for all modern JVMs pre-Loom.

**Q45. Explain the `volatile` keyword. Does it make compound operations atomic?**
`volatile` guarantees **visibility** (every thread reads the latest write) and **ordering** (happens-before relationship for surrounding code) for that specific field, but does **NOT** make compound operations (like `count++`) atomic — you still need `synchronized` or `Atomic*` for that.

### Modern Concurrency / java.util.concurrent

**Q46. Why prefer `ExecutorService` over manually creating `Thread` objects?**
Thread creation/destruction is relatively expensive; unbounded manual thread creation can exhaust OS resources. `ExecutorService` provides pooling/reuse, queuing, backpressure, lifecycle management (`shutdown`/`awaitTermination`), and integrates with `Future`/`CompletableFuture` for result handling — far more scalable and maintainable.

**Q47. Difference between `Runnable` and `Callable`?**
`Runnable.run()` returns `void` and cannot throw checked exceptions. `Callable<V>.call()` returns a typed result `V` and can throw checked exceptions — submitted via `ExecutorService.submit(Callable)`, returning a `Future<V>`.

**Q48. What's the danger of `Executors.newFixedThreadPool()`/`newCachedThreadPool()` in production?**
`newFixedThreadPool` uses an **unbounded** `LinkedBlockingQueue` internally — under sustained overload, the queue can grow unbounded and cause `OutOfMemoryError`. `newCachedThreadPool` can create **unbounded numbers of threads** under load, exhausting OS/thread resources. Prefer explicitly configuring a `ThreadPoolExecutor` with a **bounded** queue and a defined rejection policy.

**Q49. What is `CompletableFuture` and how does it improve on `Future`?**
A `Future` implementation (Java 8+) supporting **non-blocking composition/chaining** of asynchronous computations (`thenApply`, `thenCompose`, `thenCombine`, `exceptionally`, `allOf`/`anyOf`), unlike plain `Future` which only supports blocking `get()` with no built-in chaining or combinator API.

**Q50. What is `ReentrantLock`'s "reentrant" property, and does `synchronized` have it too?**
"Reentrant" means the **same thread** can acquire the same lock multiple times (nested calls) without deadlocking itself — each acquisition is matched with a corresponding release, tracked via a hold count. Both intrinsic `synchronized` locks and `ReentrantLock` are reentrant in Java.

**Q51. What is `ConcurrentHashMap` and how does it outperform a `synchronized` `HashMap`?**
A thread-safe `Map` implementation using fine-grained internal locking (historically per-segment; since Java 8, per-bin/node with CAS operations for many operations) instead of locking the **entire map** on every access (as `Collections.synchronizedMap` does) — allowing much higher concurrent read/write throughput. It also guarantees weakly-consistent iterators that don't throw `ConcurrentModificationException`.

**Q52. Explain `CountDownLatch` vs `CyclicBarrier`.**
Both let threads wait for each other, but `CountDownLatch` is **single-use** (once the count hits zero, it can't be reset) and doesn't require the counting-down threads to be the same as those waiting. `CyclicBarrier` is **reusable/cyclic** — it's specifically for a fixed set of N threads to repeatedly rendezvous at a barrier point (each phase), and can run an optional barrier action when the last thread arrives.

**Q53. What is the Fork/Join framework, and what powers parallel streams under the hood?**
A framework (Java 7+) for divide-and-conquer parallel algorithms using `RecursiveTask`/`RecursiveAction` and **work-stealing** scheduling (`ForkJoinPool`) to balance load across CPU cores. Java 8 parallel streams (`.parallelStream()`) run on the shared `ForkJoinPool.commonPool()` by default.

### Virtual Threads / Latest Java (JDK 21+)

**Q54. What are Virtual Threads? What problem do they solve?**
Introduced in **JDK 21 (JEP 444, finalized LTS feature)**, virtual threads are extremely lightweight, JVM-managed threads that are **not** 1:1 mapped to OS threads. They let you write simple, sequential, blocking-style code (easy to read/debug) while achieving the throughput/scalability previously only possible with complex async/reactive/non-blocking architectures — because blocking a virtual thread doesn't block/waste its underlying OS carrier thread.

**Q55. How do virtual threads achieve high scalability without wasting OS threads on blocking calls?**
When a virtual thread blocks (I/O, `sleep`, lock acquisition via `java.util.concurrent`), the JVM **unmounts** it from its carrier (platform) thread, freeing that carrier to run a different virtual thread. When the blocking operation completes, the virtual thread is **remounted** on any available carrier thread to resume execution.

**Q56. What is "carrier thread pinning" with virtual threads, and when does it happen?**
Pinning occurs when a virtual thread **cannot** be unmounted from its carrier during a blocking operation — historically this happened when blocking occurs **inside a `synchronized` block/method**, or during certain native method calls. A pinned virtual thread ties up its carrier thread for the blocking duration, reducing scalability. Mitigation (pre-JDK 24): replace `synchronized` with `java.util.concurrent.locks.ReentrantLock` in virtual-thread-heavy hot paths. (JDK 24 substantially reduced/removed `synchronized`-related pinning.)

**Q57. Should you pool virtual threads like you pool platform threads?**
**No.** Virtual threads are meant to be created **cheaply and abundantly, one per task**, and discarded after use (`Executors.newVirtualThreadPerTaskExecutor()`). Pooling them provides no benefit and defeats their design purpose — unlike platform threads, where pooling exists specifically because creation is expensive.

**Q58. Are virtual threads good for CPU-bound work?**
No. Virtual threads add scalability for **I/O-bound/blocking-heavy concurrency**, not computational parallelism — CPU-bound work is still bounded by the number of physical cores. Use Fork/Join or parallel streams for CPU-bound parallel computation instead.

**Q59. What is `ScopedValue` and how does it relate to `ThreadLocal` in the virtual threads era?**
`ScopedValue` (JEP 429/446, incubating/preview track around JDK 21+) is an immutable, structured alternative to `ThreadLocal`, designed to safely and efficiently share data within a bounded call scope — better suited to virtual threads (avoiding the per-thread memory overhead / lifecycle-management issues `ThreadLocal` can have at millions-of-threads scale) and integrating with Structured Concurrency.

**Q60. What is Structured Concurrency (JEP 480 / related JEPs)?**
A programming model (built atop virtual threads) that treats a group of related concurrent subtasks launched together as a **single unit of work** — if one subtask fails, siblings are automatically cancelled; the parent task doesn't complete/return until all children complete; error handling and cancellation propagate predictably, unlike loosely-tracked ad hoc `Future`/thread-pool usage.

**Q61. If your team migrates a Tomcat/Spring Boot app to virtual threads, what do you still need to check?**
- Audit `synchronized` blocks around blocking calls (DB, HTTP calls) in hot paths for pinning — replace with `ReentrantLock` if on JDK < 24.
- Check any code relying on `ThreadLocal`-heavy patterns at scale, or `Thread` pool size-based backpressure/rate-limiting logic (won't work the same way with "unlimited" virtual threads — use `Semaphore` instead).
- Ensure downstream connection pools (DB, HTTP clients) are sized to handle the potentially much higher concurrency virtual threads enable — the bottleneck often shifts to the pool, not the app-thread layer.
- Confirm framework/version support (Spring Boot 3.2+, Tomcat 10.1+ support enabling virtual threads via configuration).

---

## Appendix: Full Working Reference Program (Combines Multiple Concepts)

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class MultithreadingReferenceDemo {

    public static void main(String[] args) throws Exception {
        // 1) Classic Thread + Runnable
        Runnable job = () -> System.out.println("[Classic] Running on " + Thread.currentThread().getName());
        new Thread(job, "classic-thread").start();

        // 2) Synchronized counter (race-condition-safe)
        SafeCounter counter = new SafeCounter();
        ExecutorService pool = Executors.newFixedThreadPool(4);
        for (int i = 0; i < 1000; i++) pool.submit(counter::increment);
        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
        System.out.println("[Synchronized] Final count: " + counter.get());   // always 1000

        // 3) Atomic counter (lock-free alternative)
        AtomicInteger atomicCounter = new AtomicInteger();
        ExecutorService pool2 = Executors.newFixedThreadPool(4);
        for (int i = 0; i < 1000; i++) pool2.submit(atomicCounter::incrementAndGet);
        pool2.shutdown();
        pool2.awaitTermination(5, TimeUnit.SECONDS);
        System.out.println("[Atomic] Final count: " + atomicCounter.get());   // always 1000

        // 4) CompletableFuture pipeline
        CompletableFuture<String> future = CompletableFuture
                .supplyAsync(() -> "Hello")
                .thenApply(s -> s + ", World!");
        System.out.println("[CompletableFuture] " + future.get());

        // 5) Virtual Threads (JDK 21+) — comment out if running on JDK < 21
        try (ExecutorService vExecutor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 5; i++) {
                int id = i;
                vExecutor.submit(() -> System.out.println("[Virtual] Task " + id + " on " + Thread.currentThread()));
            }
        }
    }

    static class SafeCounter {
        private int count = 0;
        public synchronized void increment() { count++; }
        public synchronized int get() { return count; }
    }
}
```

**Expected behavior:** All counters converge to exact expected totals (no lost updates); virtual thread tasks print with thread names like `VirtualThread[#23]/runnable@ForkJoinPool-1-worker-1`, confirming they're mounted on carrier pool workers rather than dedicated OS threads.

---

*End of Chapter 7 — Multithreading Study Material.*


---

# Part II — Chapter 8: Multi-Threading Enhancements

## 1. ThreadGroup

### 1.1 What is a ThreadGroup?
- Based on functionality, threads can be grouped into a **single unit**, called a `ThreadGroup`. A `ThreadGroup` represents a **set of threads**.
- A `ThreadGroup` can also contain other **sub-ThreadGroups**, forming a tree structure.

```
t1 t2 t3 ------------- tn
        |
   t5 t6 t7   (SubThreadGroup)
        |
    ThreadGroup
```

- `ThreadGroup` is present in the `java.lang` package and is the **direct child class of `Object`** (it does **not** extend `Thread`).
- It provides a convenient way to perform a **common operation on all threads belonging to a particular group** at once.
  - Example: Stop all Consumer threads.
  - Example: Suspend all Producer threads.

> **Architect's note:** `ThreadGroup` is largely a **legacy API**. Most of its bulk-control methods (`stop()`, `suspend()`, `resume()`) are **deprecated** because they are inherently unsafe (see Section 7). In modern code, prefer `ExecutorService`, `Thread` naming/factories, and structured concurrency for grouping/managing related threads.

### 1.2 Constructors

| Constructor | Description |
|---|---|
| `ThreadGroup g = new ThreadGroup(String gname);` | Creates a new ThreadGroup. The parent of this new group is the ThreadGroup of the **currently running thread**. |
| `ThreadGroup g = new ThreadGroup(ThreadGroup pg, String gname);` | Creates a new ThreadGroup whose parent is the **specified** ThreadGroup. |

### 1.3 Notes
- In Java, **every thread belongs to some ThreadGroup**.
- Every `ThreadGroup` is (directly or indirectly) a child group of the **`system`** ThreadGroup. Hence `system` acts as the **root** for all ThreadGroups in Java.
- The `system` ThreadGroup represents JVM-internal/system-level threads such as `Reference Handler`, `Signal Dispatcher`, `Finalizer`, `Attach Listener`, etc.

```
                 System
        ┌──────────┼───────────┬─────────────────┐
       Main   ReferenceHandler  SignalDispatcher  ...
    ┌───┼────────────┬───────────────┐
MainThread   ChildThread1   ChildThread2   SubThreadGroup
   Class                                        │
                                     Thread1  Thread2  Thread3
```

### 1.4 Demo: Parent-Child Relationship

```java
class ThreadGroupDemo {
    public static void main(String[] args) {
        System.out.println(Thread.currentThread().getThreadGroup().getName());
        System.out.println(Thread.currentThread().getThreadGroup().getParent().getName());

        ThreadGroup pg = new ThreadGroup("Parent Group");
        System.out.println(pg.getParent().getName());

        ThreadGroup cg = new ThreadGroup(pg, "Child Group");
        System.out.println(cg.getParent().getName());
    }
}
```
**Output:**
```
main
system
main
Parent Group
```

### 1.5 Important Methods of ThreadGroup

| # | Method | Description |
|---|---|---|
| 1 | `String getName()` | Returns the name of the ThreadGroup. |
| 2 | `int getMaxPriority()` | Returns the maximum priority of the ThreadGroup. |
| 3 | `void setMaxPriority(int pri)` | Sets maximum priority. **Default max priority is 10.** Threads *already* in the group with higher priority are **not affected**; the new max priority applies only to **newly added** threads. |
| 4 | `ThreadGroup getParent()` | Returns the parent group of the current ThreadGroup. |
| 5 | `void list()` | Prints information about the ThreadGroup to the console (for debugging). |
| 6 | `int activeCount()` | Returns the number of active threads present in the ThreadGroup. |
| 7 | `int activeGroupCount()` | Returns the number of active sub-ThreadGroups present in the current ThreadGroup. |
| 8 | `int enumerate(Thread[] t)` | Copies all active threads of this group (including sub-group threads) into the provided array. |
| 9 | `int enumerate(ThreadGroup[] g)` | Copies all active sub-ThreadGroups into the provided array. |
| 10 | `boolean isDaemon()` | Checks daemon status of the group. |
| 11 | `void setDaemon(boolean b)` | Marks group as daemon (does **not** make its threads daemon; only affects auto-destroy behavior when empty). |
| 12 | `void interrupt()` | Interrupts **all** threads present in the ThreadGroup. |
| 13 | `void destroy()` | Destroys the ThreadGroup and its sub-ThreadGroups (must be empty of threads). |

> **Deprecated (avoid in modern code):** `stop()`, `suspend()`, `resume()` — deprecated since Java 1.2/9 because they can leave shared objects in an inconsistent (corrupted) state or cause deadlocks.

### 1.6 Demo: `setMaxPriority()` behavior

```java
class ThreadGroupDemo {
    public static void main(String[] args) {
        ThreadGroup g1 = new ThreadGroup("tg");
        Thread t1 = new Thread(g1, "Thread 1");
        Thread t2 = new Thread(g1, "Thread 2");

        g1.setMaxPriority(3);

        Thread t3 = new Thread(g1, "Thread 3");

        System.out.println(t1.getPriority()); // 5 (created before setMaxPriority, unaffected)
        System.out.println(t2.getPriority()); // 5
        System.out.println(t3.getPriority()); // 3 (created after, capped)
    }
}
```

### 1.7 Demo: `activeCount()`, `activeGroupCount()`, `list()`

```java
class MyThread extends Thread {
    MyThread(ThreadGroup g, String name) { super(g, name); }
    public void run() {
        System.out.println("Child Thread");
        try { Thread.sleep(2000); } catch (InterruptedException e) {}
    }
}

class ThreadGroupDemo {
    public static void main(String[] args) throws InterruptedException {
        ThreadGroup pg = new ThreadGroup("Parent Group");
        ThreadGroup cg = new ThreadGroup(pg, "Child Group");

        MyThread t1 = new MyThread(pg, "Child Thread 1");
        MyThread t2 = new MyThread(pg, "Child Thread 2");
        t1.start();
        t2.start();

        System.out.println(pg.activeCount());      // 2
        System.out.println(pg.activeGroupCount());  // 1
        pg.list();

        Thread.sleep(5000);
        System.out.println(pg.activeCount());       // 0 (threads finished)
        pg.list();
    }
}
```
**Output:**
```
2
1
java.lang.ThreadGroup[name=Parent Group,maxpri=10]
    Thread[Child Thread 1,5,Parent Group]
    Thread[Child Thread 2,5,Parent Group]
    java.lang.ThreadGroup[name=Child Group,maxpri=10]
Child Thread
Child Thread
0
java.lang.ThreadGroup[name=Parent Group,maxpri=10]
    java.lang.ThreadGroup[name=Child Group,maxpri=10]
```

### 1.8 Program: List All Threads Belonging to `system` Group

```java
class ThreadGroupDemo {
    public static void main(String[] args) {
        ThreadGroup system = Thread.currentThread().getThreadGroup().getParent();
        Thread[] t = new Thread[system.activeCount()];
        system.enumerate(t);
        for (Thread t1 : t) {
            System.out.println(t1.getName() + "-------" + t1.isDaemon());
        }
    }
}
```
**Typical Output (JVM/version dependent):**
```
Reference Handler-------true
Finalizer-------true
Signal Dispatcher-------true
Attach Listener-------true
main-------false
```
> **Note:** On modern JDKs (9+), the exact set of system threads may differ slightly (e.g., `Finalizer` thread may be absent if finalization is not triggered, `Common-Cleaner` may appear due to `java.lang.ref.Cleaner`). Don't hardcode this list in production assertions.

---

## 2. ThreadLocal

### 2.1 What is ThreadLocal?
- `ThreadLocal` provides **thread-local variables** — each thread has its **own, independently initialized copy** of the variable.
- The `ThreadLocal` class maintains values **on a per-thread basis**. Each `ThreadLocal` object maintains a separate value (e.g., `userID`, `transactionID`) **for each thread** that accesses it.
- A thread can access its own local value, mutate it, and even remove it.
- Any code executed by that thread (anywhere in the call stack) can access its local variables through the same `ThreadLocal` reference — this makes it ideal for **implicit context propagation** without passing parameters through every method signature.

### 2.2 Classic Use Case
> Consider a Servlet that calls several business methods. You need to generate a **unique `transactionID`** per request and make it available to all business methods for logging — **without** passing it as a parameter everywhere. `ThreadLocal` solves this by maintaining a separate `transactionID` per thread (per request, assuming thread-per-request model).

This is exactly the mechanism behind **MDC (Mapped Diagnostic Context)** in logging frameworks like SLF4J/Logback, and **Spring's `RequestContextHolder`**.

### 2.3 Key Notes
- `ThreadLocal` class was introduced in **JDK 1.2**.
- `ThreadLocal` is associated with **Thread scope** — bound to a specific thread's lifetime.
- All code executed by a thread has access to its corresponding `ThreadLocal` variables.
- A thread can access **only its own** local variables — **not** other threads' values.
- Once a thread enters the **dead** state, its ThreadLocal values become **eligible for garbage collection** by default (technically, this happens because the `ThreadLocalMap` is a field of the `Thread` object itself — see Architect Deep Dive below).

### 2.4 Constructor & Methods

**Constructor:**
```java
ThreadLocal<T> tl = new ThreadLocal<>();
```

| Method | Description |
|---|---|
| `T get()` | Returns the value of the ThreadLocal variable associated with the **current thread**. |
| `T initialValue()` | Returns the initial value for the current thread. Default implementation returns `null`. Override to customize. |
| `void set(T value)` | Sets a new value for the current thread. |
| `void remove()` | Removes the current thread's local value. After removal, a subsequent `get()` re-initializes via `initialValue()`. **Added in JDK 1.5.** |

### 2.5 Basic Demo

```java
class ThreadLocalDemo {
    public static void main(String[] args) {
        ThreadLocal<String> tl = new ThreadLocal<>();
        System.out.println(tl.get());  // null
        tl.set("Durga");
        System.out.println(tl.get());  // Durga
        tl.remove();
        System.out.println(tl.get());  // null
    }
}
```

### 2.6 Overriding `initialValue()`

```java
class ThreadLocalDemo {
    public static void main(String[] args) {
        ThreadLocal<String> tl = new ThreadLocal<String>() {
            @Override
            protected String initialValue() {
                return "abc";
            }
        };
        System.out.println(tl.get());  // abc
        tl.set("Durga");
        System.out.println(tl.get());  // Durga
        tl.remove();
        System.out.println(tl.get());  // abc  (re-initialized)
    }
}
```

> **Modern API tip:** Since **Java 8**, prefer the functional factory:
> ```java
> ThreadLocal<String> tl = ThreadLocal.withInitial(() -> "abc");
> ```
> This is equivalent to overriding `initialValue()` but avoids anonymous inner class boilerplate.

### 2.7 Real-World Demo: Per-Thread Customer ID Generator

```java
class CustomerThread extends Thread {
    static Integer custID = 0;

    private static ThreadLocal<Integer> tl = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return ++custID;
        }
    };

    CustomerThread(String name) {
        super(name);
    }

    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(Thread.currentThread().getName()
                    + " Executing with Customer ID:" + tl.get());
        }
    }
}

class CustomerThreadDemo {
    public static void main(String[] args) {
        for (int i = 1; i <= 4; i++) {
            new CustomerThread("CustomerThread - " + i).start();
        }
    }
}
```
**Output (order across threads may interleave, but each thread's ID is stable):**
```
CustomerThread - 1 Executing with Customer ID:1
CustomerThread - 1 Executing with Customer ID:1
...
CustomerThread - 2 Executing with Customer ID:2
...
CustomerThread - 3 Executing with Customer ID:3
...
CustomerThread - 4 Executing with Customer ID:4
```
> Each `CustomerThread` gets its **own** `custID` value stamped once (via `initialValue()`), and every `tl.get()` call within that thread returns the same value — demonstrating per-thread isolation.

### 2.8 Architect Deep Dive: How ThreadLocal Actually Works Internally
- Every `Thread` object internally holds a field: `ThreadLocal.ThreadLocalMap threadLocals`.
- `ThreadLocalMap` is a specialized hash map where **keys are `ThreadLocal` instances (held as `WeakReference`)** and values are the stored objects.
- When you call `tl.set(value)`, it actually does: `Thread.currentThread().threadLocals.set(this, value)`.
- Because the map lives **inside the Thread object**, when the thread dies, the whole map (and its values) becomes eligible for GC — **provided no other strong reference to the ThreadLocal or its value exists**.
- **Common production bug — Memory Leaks in Thread Pools:** Since pooled threads (`ExecutorService`) are **reused** and never die, `ThreadLocal` values set on them are **not automatically cleared** between tasks. If you don't call `remove()` after use, stale/large objects can accumulate and leak, and **data can leak across unrelated tasks/requests** run on the same pooled thread. This is one of the most common real-world `ThreadLocal` production bugs (e.g., leaking authenticated user context between HTTP requests).

  **Golden Rule:** Always pair `set()` with `remove()` in a `finally` block when using ThreadLocal with pooled threads:
  ```java
  ThreadLocal<String> ctx = new ThreadLocal<>();
  try {
      ctx.set("txn-123");
      doWork();
  } finally {
      ctx.remove(); // prevents leak / cross-task contamination
  }
  ```

---

## 3. InheritableThreadLocal

### 3.1 ThreadLocal vs Inheritance
- By default, a **parent thread's `ThreadLocal` variables are NOT available** to child threads it spawns.
- To make a parent thread's local variable available to (and inheritable by) child threads, use **`InheritableThreadLocal`**.
- `InheritableThreadLocal` is the **child class of `ThreadLocal`**.
- By default, a child thread's value is the **same** as the parent thread's value at the time the child thread is created, but you can supply **customized values** for child threads by overriding `childValue()`.

**Constructor:**
```java
InheritableThreadLocal<T> itl = new InheritableThreadLocal<>();
```

- `InheritableThreadLocal` inherits **all** methods of `ThreadLocal` (`get()`, `set()`, `remove()`, etc.).
- Additional method: `protected T childValue(T parentValue)` — override to customize the value propagated to child threads (default implementation just returns `parentValue`).

### 3.2 Demo

```java
class ParentThread extends Thread {
    public static InheritableThreadLocal<String> itl = new InheritableThreadLocal<String>() {
        @Override
        protected String childValue(String p) {
            return "cc";
        }
    };

    public void run() {
        itl.set("pp");
        System.out.println("Parent Thread --" + itl.get());
        ChildThread ct = new ChildThread();
        ct.start();
    }

    static class ChildThread extends Thread {
        public void run() {
            System.out.println("Child Thread --" + ParentThread.itl.get());
        }
    }
}

class ThreadLocalDemo {
    public static void main(String[] args) {
        ParentThread pt = new ParentThread();
        pt.start();
    }
}
```
**Output:**
```
Parent Thread --pp
Child Thread --cc
```
> Without overriding `childValue()`, the child thread would have printed `pp` (inherited as-is at the time the child `Thread` object was constructed).

> **Architect's note (Modern Java 21+):** `InheritableThreadLocal` does **not** work well with **Virtual Threads** and **Structured Concurrency** because it snapshots/copies value at thread-creation time and can be expensive/leaky at scale (millions of virtual threads). Java 20/21 introduced **`ScopedValue`** (JEP 429 / JEP 446, finalized as JEP 481 in Java 24) as a modern, safer, immutable alternative for sharing context across threads — see Section 7.

---

## 4. java.util.concurrent.locks Package

### 4.1 Problems with Traditional `synchronized` Keyword
The `java.util.concurrent.locks` package (introduced in **JDK 1.5**) was created to overcome these limitations of the intrinsic `synchronized` mechanism:

1. **No control over lock hand-off** — If a thread releases a lock, you have **no control** over which waiting thread gets it next.
2. **No timeout** — You **can't specify a maximum wait time** for a thread trying to acquire a lock; it waits indefinitely, which can hurt performance or cause deadlock.
3. **No non-blocking "try"** — There's no flexibility to **try** for a lock without blocking/waiting.
4. **No introspection API** — There is no API to list all threads waiting for a lock.
5. **Structural rigidity** — `synchronized` **must** be defined within a single method/block; it's **not possible** to acquire a lock in one method and release it in another (span across multiple methods).

`java.util.concurrent.locks` addresses all these gaps and gives programmers **fine-grained control over concurrency**.

### 4.2 `Lock` Interface
- A `Lock` object is conceptually similar to the **implicit lock** acquired by a thread executing a `synchronized` method/block.
- `Lock` implementations provide **more extensive operations** than traditional implicit locking.

#### Important Methods of `Lock`

| # | Method | Description |
|---|---|---|
| 1 | `void lock()` | Locks the object. If already locked by another thread, the calling thread **waits (blocks)** until it's unlocked. |
| 2 | `boolean tryLock()` | Attempts to acquire the lock **without waiting**. Returns `true` if acquired, `false` otherwise (thread is **never blocked**). |
| 3 | `boolean tryLock(long time, TimeUnit unit)` | Attempts to acquire the lock, waiting up to the given time if unavailable. Returns `false` if still unavailable after timeout. |
| 4 | `void lockInterruptibly()` | Acquires the lock unless the current thread is interrupted. If unavailable, the thread waits, but if **interrupted while waiting**, it will **not** acquire the lock (throws `InterruptedException`). |
| 5 | `void unlock()` | Releases the lock. |

```java
// Typical tryLock usage pattern:
if (lock.tryLock()) {
    try {
        // perform safe operations
    } finally {
        lock.unlock();
    }
} else {
    // perform alternative operations
}
```

```java
// tryLock(time, unit) example:
if (lock.tryLock(1000, TimeUnit.SECONDS)) {
    // ...
}
```

`TimeUnit` is an `enum` in `java.util.concurrent`:
```java
enum TimeUnit {
    NANOSECONDS, MICROSECONDS, MILLISECONDS, SECONDS,
    MINUTES, HOURS, DAYS;
}
```
> *(Note: the original source lists an informal ordering `DAYS, HOURS, MINUTES, SECONDS, MILLI SECONDS, MICRO SECONDS, NANO SECONDS`; the actual JDK enum constants — validated against `java.util.concurrent.TimeUnit` — are `NANOSECONDS, MICROSECONDS, MILLISECONDS, SECONDS, MINUTES, HOURS, DAYS`.)*

### 4.3 `ReentrantLock` Class
- `ReentrantLock` **implements `Lock`** and is a **direct subclass of `Object`**.
- **Reentrant** means a thread that already holds the lock can **acquire it again** (multiple times) without blocking itself.
- Internally, `ReentrantLock` maintains a **per-thread hold count**: it **increments** on each `lock()` call and **decrements** on each `unlock()` call by the owning thread. The lock is only truly **released** when the count reaches **0**.

#### Constructors

| Constructor | Description |
|---|---|
| `new ReentrantLock()` | Creates an instance with **default (non-fair)** policy. |
| `new ReentrantLock(boolean fairness)` | Creates an instance with the given fairness policy. |

- **Fair (`true`)**: The **longest-waiting thread** gets the lock once available — follows **First-In-First-Out (FIFO)** ordering.
- **Non-fair (`false`)**: **No guarantee** of which waiting thread gets the lock next (can lead to better throughput but risk of "barging"/starvation).
- **Default:** If fairness is not specified, it is **non-fair** by default.

> Quick check: `new ReentrantLock()` is **equivalent** to `new ReentrantLock(false)` — **not** `new ReentrantLock(true)`.

#### Important Methods of `ReentrantLock`

| # | Method | Description |
|---|---|---|
| 1 | `void lock()` | Acquire lock (inherited behavior from `Lock`). |
| 2 | `boolean tryLock()` | Try to acquire without waiting. |
| 3 | `boolean tryLock(long l, TimeUnit t)` | Try to acquire with timeout. |
| 4 | `void lockInterruptibly()` | Acquire unless interrupted. |
| 5 | `void unlock()` | Releases the lock. If the current thread is **not the owner**, throws `IllegalMonitorStateException` (a `RuntimeException`). |
| 6 | `int getHoldCount()` | Returns the number of holds on this lock by the **current thread**. |
| 7 | `boolean isHeldByCurrentThread()` | Returns `true` **iff** the lock is held by the current thread. |
| 8 | `int getQueueLength()` | Returns the (estimated) number of threads **waiting** to acquire this lock. |
| 9 | `Collection<Thread> getQueuedThreads()` *(protected)* | Returns threads waiting to acquire the lock (protected — used by subclasses). |
| 10 | `boolean hasQueuedThreads()` | Returns `true` if any thread is waiting for this lock. |
| 11 | `boolean isLocked()` | Returns `true` if the lock is currently acquired by **any** thread. |
| 12 | `boolean isFair()` | Returns `true` if the lock's fairness policy is `true`. |
| 13 | `Thread getOwner()` *(protected)* | Returns the thread currently holding the lock. |

#### Demo: Hold Count & Reentrancy

```java
import java.util.concurrent.locks.ReentrantLock;

class Test {
    public static void main(String[] args) {
        ReentrantLock l = new ReentrantLock();
        l.lock();
        l.lock();  // reentrant acquisition, holdCount = 2

        System.out.println(l.isLocked());             // true
        System.out.println(l.isHeldByCurrentThread()); // true
        System.out.println(l.getQueueLength());        // 0

        l.unlock();  // holdCount = 1
        System.out.println(l.getHoldCount());  // 1
        System.out.println(l.isLocked());      // true (still held)

        l.unlock();  // holdCount = 0, released
        System.out.println(l.isLocked());  // false
        System.out.println(l.isFair());    // false
    }
}
```

#### Demo: Serialized Access with `lock()`/`unlock()`

```java
import java.util.concurrent.locks.ReentrantLock;

class Display {
    ReentrantLock l = new ReentrantLock(true); // fair lock

    public void wish(String name) {
        l.lock();                                  // -> acquire (line 1)
        for (int i = 0; i < 5; i++) {
            System.out.println("Good Morning");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {}
            System.out.println(name);
        }
        l.unlock();                                // -> release (line 2)
    }
}

class MyThread extends Thread {
    Display d;
    String name;
    MyThread(Display d, String name) {
        this.d = d;
        this.name = name;
    }
    public void run() {
        d.wish(name);
    }
}

class ReentrantLockDemo {
    public static void main(String[] args) {
        Display d = new Display();
        new MyThread(d, "Dhoni").start();
        new MyThread(d, "Yuva Raj").start();
        new MyThread(d, "ViratKohli").start();
    }
}
```
**Behavior:**
- **With `lock()`/`unlock()` present:** Threads execute **one by one** (serialized) → output is **regular/predictable** (Dhoni's 5 iterations complete, then Yuva Raj's, then Kohli's).
- **If both `lock()` and `unlock()` calls are removed/commented out:** All three threads execute the `wish()` method **simultaneously** → output becomes **interleaved/irregular**.

#### Demo: `tryLock()` — Non-blocking attempt

```java
import java.util.concurrent.locks.ReentrantLock;

class MyThread extends Thread {
    static ReentrantLock l = new ReentrantLock();
    MyThread(String name) { super(name); }

    public void run() {
        if (l.tryLock()) {
            System.out.println(Thread.currentThread().getName()
                    + " Got Lock and Performing Safe Operations");
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
            l.unlock();
        } else {
            System.out.println(Thread.currentThread().getName()
                    + " Unable To Get Lock and Hence Performing Alternative Operations");
        }
    }
}

class ReentrantLockDemo {
    public static void main(String args[]) {
        new MyThread("First Thread").start();
        new MyThread("Second Thread").start();
    }
}
```
**Typical Output:**
```
First Thread Got Lock and Performing Safe Operations
Second Thread Unable To Get Lock and Hence Performing Alternative Operations
```

#### Demo: `tryLock(timeout, unit)` — Retry Loop

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

class MyThread extends Thread {
    static ReentrantLock l = new ReentrantLock();
    MyThread(String name) { super(name); }

    public void run() {
        do {
            try {
                if (l.tryLock(1000, TimeUnit.MILLISECONDS)) {
                    System.out.println(Thread.currentThread().getName() + "------- Got Lock");
                    Thread.sleep(5000);
                    l.unlock();
                    System.out.println(Thread.currentThread().getName() + "------- Releases Lock");
                    break;
                } else {
                    System.out.println(Thread.currentThread().getName()
                            + "------- Unable To Get Lock And Will Try Again");
                }
            } catch (InterruptedException e) {}
        } while (true);
    }
}

class ReentrantLockDemo {
    public static void main(String args[]) {
        new MyThread("First Thread").start();
        new MyThread("Second Thread").start();
    }
}
```
**Sample Output:**
```
First Thread------- Got Lock
Second Thread------- Unable To Get Lock And Will Try Again
Second Thread------- Unable To Get Lock And Will Try Again
Second Thread------- Unable To Get Lock And Will Try Again
Second Thread------- Unable To Get Lock And Will Try Again
Second Thread------- Got Lock
First Thread------- Releases Lock
Second Thread------- Releases Lock
```

### 4.4 Architect Deep Dive: `synchronized` vs `ReentrantLock`

| Aspect | `synchronized` | `ReentrantLock` |
|---|---|---|
| Lock acquisition | Implicit | Explicit (`lock()`/`unlock()`) |
| Try without blocking | Not possible | `tryLock()` |
| Timed wait | Not possible | `tryLock(time, unit)` |
| Interruptible wait | Not possible | `lockInterruptibly()` |
| Fairness policy | Not configurable | Configurable (`true`/`false`) |
| Span across methods | No (must be single block/method) | Yes (`lock()` in one method, `unlock()` in another — **risky**, use with care) |
| Condition variables | Single implicit monitor (`wait`/`notify`/`notifyAll`) | Multiple `Condition` objects via `newCondition()` — supports selective signaling |
| Auto-release on exception | Yes (JVM releases automatically) | **No** — must manually `unlock()` in a `finally` block, else lock leaks forever |
| Performance (modern JVMs) | Highly optimized via biased/lightweight locking, adaptive spinning | Comparable/better under high contention; more overhead for uncontended, short critical sections in older JVMs |

> **Best Practice:** Always release a `ReentrantLock` in a `finally` block:
> ```java
> lock.lock();
> try {
>     // critical section
> } finally {
>     lock.unlock();
> }
> ```

### 4.5 Other Members of `java.util.concurrent.locks` (Architect Awareness — not in source but commonly asked)
- **`ReadWriteLock` (interface) / `ReentrantReadWriteLock` (class):** Allows multiple concurrent **readers** OR one exclusive **writer** — improves throughput for read-heavy workloads.
- **`StampedLock`** (Java 8+): An optimized read-write lock supporting **optimistic reads** (non-blocking, validated after the fact) in addition to normal read/write locking — generally faster than `ReentrantReadWriteLock` but **not reentrant** and more complex to use correctly.
- **`Condition`**: Obtained via `lock.newCondition()`; replaces `Object.wait()/notify()/notifyAll()` with `await()/signal()/signalAll()`, and supports **multiple wait-sets per lock**.

---

## 5. Thread Pools (Executor Framework)

### 5.1 Motivation
- Creating a **new `Thread` object for every job** can cause serious **performance and memory problems** at scale (thread creation/teardown overhead, excessive context switching, OS resource exhaustion).
- **Solution:** the **Thread Pool** concept — a pool of **already-created threads** ready to execute submitted jobs, reusing threads across multiple tasks.

### 5.2 Executor Framework
- **Java 1.5** introduced the **Thread Pool Framework**, also known as the **Executor Framework** (`java.util.concurrent`).

**Creating a thread pool:**
```java
ExecutorService service = Executors.newFixedThreadPool(3); // pool size is our choice
```

**Submitting a job:**
```java
service.submit(job); // job implements Runnable (or Callable — see Section 6)
```

**Shutting down the pool:**
```java
service.shutdown();
```

### 5.3 Demo: Fixed Thread Pool Executing Multiple Jobs

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class PrintJob implements Runnable {
    String name;
    PrintJob(String name) { this.name = name; }

    public void run() {
        System.out.println(name + "....Job Started By Thread:" + Thread.currentThread().getName());
        try { Thread.sleep(10000); } catch (InterruptedException e) {}
        System.out.println(name + "....Job Completed By Thread:" + Thread.currentThread().getName());
    }
}

class ExecutorDemo {
    public static void main(String[] args) {
        PrintJob[] jobs = {
            new PrintJob("Durga"), new PrintJob("Ravi"), new PrintJob("Nagendra"),
            new PrintJob("Pavan"), new PrintJob("Bhaskar"), new PrintJob("Varma")
        };

        ExecutorService service = Executors.newFixedThreadPool(3);
        for (PrintJob job : jobs) {
            service.submit(job);
        }
        service.shutdown();
    }
}
```
**Sample Output:**
```
Durga....Job Started By Thread:pool-1-thread-1
Ravi....Job Started By Thread:pool-1-thread-2
Nagendra....Job Started By Thread:pool-1-thread-3
Ravi....Job Completed By Thread:pool-1-thread-2
Pavan....Job Started By Thread:pool-1-thread-2
Durga....Job Completed By Thread:pool-1-thread-1
Bhaskar....Job Started By Thread:pool-1-thread-1
Nagendra....Job Completed By Thread:pool-1-thread-3
Varma....Job Started By Thread:pool-1-thread-3
Pavan....Job Completed By Thread:pool-1-thread-2
Bhaskar....Job Completed By Thread:pool-1-thread-1
Varma....Job Completed By Thread:pool-1-thread-3
```
> **3 threads execute 6 jobs** — demonstrating that a single thread is **reused** for multiple jobs, avoiding the cost of creating 6 separate threads.

> **Note:** Thread pools are the standard mechanism used to implement **servers** (Web Servers and Application Servers) — e.g., Tomcat's request-handling thread pool.

### 5.4 Types of Executors (`Executors` factory methods — Architect Must-Know)

| Factory Method | Behavior |
|---|---|
| `Executors.newFixedThreadPool(n)` | Fixed number of threads; unbounded queue for pending tasks. |
| `Executors.newCachedThreadPool()` | Creates new threads as needed, reuses idle threads (60s keep-alive); unbounded thread growth — **risk of resource exhaustion** under heavy load. |
| `Executors.newSingleThreadExecutor()` | Single worker thread; tasks execute sequentially in submission order. |
| `Executors.newScheduledThreadPool(n)` | Supports delayed/periodic task execution (`schedule`, `scheduleAtFixedRate`, `scheduleWithFixedDelay`). |
| `Executors.newWorkStealingPool()` *(Java 8+)* | Uses `ForkJoinPool` internally with work-stealing; good for divide-and-conquer/parallel workloads. |
| `Executors.newVirtualThreadPerTaskExecutor()` *(Java 21+)* | Creates a **new virtual thread per task** — see Section 7. |

> **Production caution:** `Executors.newFixedThreadPool`/`newCachedThreadPool` use **unbounded queues/thread growth**, which can hide backpressure problems and cause `OutOfMemoryError` under sustained load. Many style guides (and Java's own docs post-JDK 8) now recommend constructing `ThreadPoolExecutor` **directly** with explicit `corePoolSize`, `maximumPoolSize`, a **bounded** `BlockingQueue`, and a defined `RejectedExecutionHandler`, rather than relying on the `Executors` convenience factories.

### 5.5 `ExecutorService` Lifecycle Methods (Architect Must-Know)

| Method | Purpose |
|---|---|
| `shutdown()` | Initiates orderly shutdown; previously submitted tasks execute, but no new tasks accepted. |
| `shutdownNow()` | Attempts to stop all actively executing tasks, halts processing of waiting tasks, returns list of tasks awaiting execution. |
| `awaitTermination(timeout, unit)` | Blocks until all tasks complete after shutdown, or timeout, or interrupt. |
| `isShutdown()` / `isTerminated()` | Query shutdown/termination state. |

---

## 6. Callable and Future

### 6.1 Why Callable?
- With a `Runnable` job, the executing thread **cannot return any result** (`run()` returns `void` and cannot throw checked exceptions).
- If a thread is required to **return a result** after execution, use **`Callable`** instead.
- `Callable` interface contains exactly **one method**:
  ```java
  public Object call() throws Exception;
  ```
  (Generic form: `V call() throws Exception;`)
- When you submit a `Callable` object to an `ExecutorService`, the framework returns an object of type **`java.util.concurrent.Future`**.
- The `Future` object is used to **retrieve the result** of the `Callable` job (once it's done), or to check status / cancel it.

### 6.2 Demo: Callable + Future

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.List;
import java.util.ArrayList;

class MyCallable implements Callable<Integer> {
    int num;
    MyCallable(int num) { this.num = num; }

    public Integer call() throws Exception {
        int sum = 0;
        for (int i = 0; i < num; i++) {
            sum = sum + i;
        }
        return sum;
    }
}

class CallableFutureDemo {
    public static void main(String args[]) throws Exception {
        MyCallable[] jobs = {
            new MyCallable(10), new MyCallable(20), new MyCallable(30),
            new MyCallable(40), new MyCallable(50), new MyCallable(60)
        };

        ExecutorService service = Executors.newFixedThreadPool(3);
        List<Future<Integer>> results = new ArrayList<>();

        for (MyCallable job : jobs) {
            results.add(service.submit(job));   // submit() returns a Future
        }

        for (Future<Integer> f : results) {
            System.out.println("Result: " + f.get());  // get() blocks until result available
        }

        service.shutdown();
    }
}
```
**Output (results deterministic; execution order across threads may vary):**
```
Result: 45
Result: 190
Result: 435
Result: 780
Result: 1225
Result: 1770
```
> *(Note: The original source snippet was truncated mid-listing; the completed, runnable version above demonstrates correct end-to-end usage of `Callable`, `submit()`, and `Future.get()`.)*

### 6.3 Important `Future<V>` Methods (Architect Must-Know)

| Method | Description |
|---|---|
| `V get()` | Waits (blocks) if necessary for the computation to complete, then retrieves its result. Throws `InterruptedException`, `ExecutionException`. |
| `V get(long timeout, TimeUnit unit)` | Same as above, but waits **at most** the given time; throws `TimeoutException` if not done in time. |
| `boolean cancel(boolean mayInterruptIfRunning)` | Attempts to cancel execution of the task. |
| `boolean isCancelled()` | Returns `true` if the task was cancelled before completion. |
| `boolean isDone()` | Returns `true` if the task completed (normally, exceptionally, or via cancellation). |

### 6.4 `ExecutorService` Convenience Methods for Multiple Callables

```java
List<Callable<Integer>> tasks = List.of(new MyCallable(10), new MyCallable(20));
List<Future<Integer>> futures = service.invokeAll(tasks);       // waits for all to complete
Integer firstResult = service.invokeAny(tasks);                 // returns result of first to complete
```

---

## 7. Modern Java Concurrency Updates (Java 8 → 21+)

The source material reflects Java 1.5–era concurrency APIs. As a Senior Java Architect, you're expected to know how the concurrency landscape has evolved:

### 7.1 Java 8
- **`CompletableFuture`**: A far more powerful alternative to `Future` — supports **non-blocking**, chainable, composable async pipelines (`thenApply`, `thenCompose`, `thenCombine`, `exceptionally`, `allOf`, `anyOf`).
  ```java
  CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> compute())
          .thenApply(r -> r * 2)
          .exceptionally(ex -> -1);
  ```
- **`ThreadLocal.withInitial(Supplier<S>)`**: Functional-style factory replacing anonymous `initialValue()` overrides.
- **Parallel Streams** (`collection.parallelStream()`): Built on the common `ForkJoinPool`.
- **`LongAdder` / `LongAccumulator` / `DoubleAdder`**: Higher-throughput alternatives to `AtomicLong` under high contention.

### 7.2 Java 9
- **`ThreadGroup.stop/suspend/resume`** further deprecated/discouraged; `Flow` API (Reactive Streams: `Flow.Publisher`, `Flow.Subscriber`, `Flow.Processor`) introduced for reactive programming interoperability.
- **`CompletableFuture`** enhancements: `orTimeout()`, `completeOnTimeout()`, delay/timeout executors.

### 7.3 Java 9–16: Deprecation for Removal
- `Thread.stop()`, `Thread.suspend()`, `Thread.resume()`, `Thread.destroy()`, and `ThreadGroup` equivalents were marked **deprecated for removal** (`@Deprecated(forRemoval=true)`) because they can corrupt shared state or cause permanent deadlocks. **Never use these in production code.**
- Prefer cooperative cancellation via `interrupt()`/`isInterrupted()`/checking a `volatile boolean` flag, or via `Future.cancel()`.

### 7.4 Java 19 & 21 — Project Loom (Landmark Changes)
This is the **single biggest concurrency shift** since `java.util.concurrent` itself, and a common senior-level interview topic:

- **Virtual Threads (JEP 444, finalized in Java 21):** Lightweight threads managed by the JVM (not 1:1 with OS threads). You can create **millions** of virtual threads cheaply.
  ```java
  Thread vt = Thread.ofVirtual().start(() -> System.out.println("Running in virtual thread"));

  // Or via Executor:
  try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
      executor.submit(() -> doWork());
  }
  ```
  - Virtual threads are ideal for **I/O-bound, high-concurrency** workloads (e.g., thread-per-request servers) because blocking operations (I/O, `Thread.sleep`) **unmount** the virtual thread from its carrier OS thread instead of blocking it.
  - **Traditional (`Thread.ofPlatform()`) threads** are now called **"platform threads"** to distinguish from virtual threads.
  - **Caveat:** `synchronized` blocks used to **pin** the virtual thread to its carrier thread in early versions; this was substantially improved in **Java 24** (JEP 491) where `synchronized` no longer pins virtual threads in most cases.
  - Thread pools (`newFixedThreadPool`, etc.) are generally **discouraged** for virtual threads — since creation is cheap, the recommended pattern is "one virtual thread per task," not pooling them.

- **Structured Concurrency (JEP 428 → JEP 453/499, still evolving/preview through Java 21–24):** Treats a group of related tasks running in different threads as a **single unit of work**, simplifying error handling and cancellation.
  ```java
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
      Future<String> user = scope.fork(() -> fetchUser());
      Future<Integer> order = scope.fork(() -> fetchOrder());

      scope.join();           // wait for both
      scope.throwIfFailed();  // propagate errors

      String result = combine(user.resultNow(), order.resultNow());
  }
  ```

- **`ScopedValue` (JEP 429 → finalized as JEP 481 in Java 24, formerly a preview since Java 20/21):** An **immutable**, safer, more scalable alternative to `InheritableThreadLocal` for sharing context (e.g., request IDs, security principals) with child tasks — designed specifically to work efficiently with virtual threads at scale (avoids the memory/performance overhead of `ThreadLocal` inheritance across millions of threads).
  ```java
  static final ScopedValue<String> USER_ID = ScopedValue.newInstance();

  ScopedValue.where(USER_ID, "user-123").run(() -> {
      // any nested call can read USER_ID.get() within this scope
      processRequest();
  });
  ```

### 7.5 Summary Table — Legacy vs Modern

| Legacy (this chapter) | Modern Replacement / Complement |
|---|---|
| `Thread`/`ThreadGroup` for bulk control (`stop`, `suspend`) | Cooperative interruption; `ExecutorService`; Structured Concurrency |
| `InheritableThreadLocal` | `ScopedValue` (Java 20+/21+, finalized Java 24) for virtual-thread-friendly context propagation |
| `Future` (blocking `get()`) | `CompletableFuture` (non-blocking, composable) |
| `Executors.newFixedThreadPool()` (platform threads) | `Executors.newVirtualThreadPerTaskExecutor()` for I/O-bound workloads (Java 21+) |
| Manual thread lifecycle management | Structured Concurrency (`StructuredTaskScope`) |
| `synchronized` only | `ReentrantLock`, `ReentrantReadWriteLock`, `StampedLock`, `Condition` |

---

## 8. Architect-Level Cheat Sheet

| Topic | One-liner |
|---|---|
| `ThreadGroup` | Legacy grouping construct; tree-structured; root is `system`; avoid `stop/suspend/resume`. |
| `ThreadLocal` | Per-thread variable storage; always `remove()` in pooled-thread scenarios to avoid leaks. |
| `InheritableThreadLocal` | Propagates parent's value to child threads; override `childValue()` to customize; consider `ScopedValue` in Loom-based code. |
| `Lock` (interface) | `lock()`, `tryLock()`, `tryLock(timeout)`, `lockInterruptibly()`, `unlock()`. |
| `ReentrantLock` | Reentrant, explicit lock with fairness option, hold-count tracking, condition support; always `unlock()` in `finally`. |
| Fair vs Non-fair | Fair = FIFO ordering (more predictable, less throughput); Non-fair = default, better throughput, no ordering guarantee. |
| Thread Pool / Executor Framework | Reuses threads to avoid per-task creation overhead; introduced Java 1.5. |
| `Callable` vs `Runnable` | `Callable<V>` returns a result and can throw checked exceptions; `Runnable` cannot. |
| `Future` | Handle to retrieve async result (`get`), check status (`isDone`), or cancel (`cancel`). |
| Modern upgrade path | `CompletableFuture` > `Future`; Virtual Threads > pooled platform threads for I/O-bound work; `ScopedValue` > `InheritableThreadLocal`. |

---

## 9. Interview Questions

### Foundational (from source FAQ list — curated & consolidated)
1. What is multitasking, and how does it differ from multithreading?
2. What is multithreading? Explain its key application areas (servers, UI responsiveness, parallel computation, background tasks).
3. What are the advantages of multithreading over multi-processing?
4. When compared with C++, what advantage does Java provide with respect to multithreading? *(Answer hint: built-in `Thread`/`Runnable` API, JVM-managed thread scheduling, memory model guarantees via JLS/JMM, garbage-collected memory reduces certain classes of bugs.)*
5. In how many ways can you define a thread in Java? *(Extending `Thread`, implementing `Runnable`, implementing `Callable` + Executor, and via `Thread.ofVirtual()`/`Thread.ofPlatform()` builders in modern Java.)*
6. Between extending `Thread` and implementing `Runnable`, which approach is recommended, and why? *(Implementing `Runnable`/`Callable` is preferred — avoids single-inheritance limitation, promotes composition, decouples task from execution mechanism, works with Executor Framework.)*
7. What is the difference between `t.start()` and `t.run()`? *(`start()` creates a new call stack and invokes `run()` on a new thread; calling `run()` directly executes it on the current thread like a normal method call — no new thread is created.)*
8. Explain the Thread Scheduler and its role.
9. What happens if we don't override `run()`?
10. Is overloading of `run()` possible? *(Yes, syntactically legal, but `start()` will only invoke the no-arg `run()` defined by `Runnable`/`Thread`.)*
11. Is it possible to override `start()`? What happens if you do? *(Legal, but if you don't call `super.start()`, no new thread of execution is actually created — it behaves like a normal method call.)*
12. Explain the complete life cycle of a Thread (`NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, `TERMINATED` — per `Thread.State` enum).
13. What is the importance of `Thread.start()`?
14. What happens if you try to restart an already-started thread? *(Throws `IllegalThreadStateException`.)*
15. Explain the `Thread` class constructors.
16. How do you get and set the name of a thread? (`getName()`/`setName()`)
17. Who uses thread priorities, and how? *(Thread scheduler uses priority as a *hint*, not a guarantee, for scheduling order — behavior is JVM/OS dependent.)*
18. What is the default priority of the main thread? *(5 — `NORM_PRIORITY`.)*
19. What is the default priority of a newly created thread? *(Inherits the priority of the creating/parent thread.)*
20. How do you get/set a thread's priority? (`getPriority()`/`setPriority(int)`, valid range 1–10: `MIN_PRIORITY`, `NORM_PRIORITY`, `MAX_PRIORITY`.)
21. What happens if you try to set a thread's priority to 100? *(Throws `IllegalArgumentException` — valid range is 1 to 10.)*
22. If two threads have different priorities, which one executes first? *(No absolute guarantee in the JLS; higher priority threads are generally *more likely* to be scheduled first — but ultimately platform-dependent.)*
23. If two threads have the same priority, which one executes first? *(No guarantee; depends on the thread scheduler — often round-robin/time-sliced.)*
24. How can you prevent a thread from execution temporarily? *(`sleep()`, `wait()`, `join()`, blocking on a lock.)*
25. What is `yield()`, and what is its purpose? *(A hint to the scheduler that the current thread is willing to yield its current use of a processor; the scheduler is free to ignore it.)*
26. Is `join()` overloaded? *(Yes — `join()`, `join(long millis)`, `join(long millis, int nanos)`.)*
27. What is the purpose of `sleep()`? *(Pause execution for a specified time; does **not** release any locks held.)*
28. What is the `synchronized` keyword? Explain its advantages and disadvantages. *(Advantage: prevents race conditions via mutual exclusion. Disadvantage: can reduce performance/throughput due to serialized access; risk of deadlock if locks are nested incorrectly.)*
29. What is an object-level lock, and when is it required? *(Used to synchronize instance methods/blocks — one lock per object instance; needed when protecting instance state shared across threads.)*
30. What is a class-level lock, and when is it required? *(Used for `static synchronized` methods — one lock per `Class` object; needed to protect shared static state.)*
31. While a thread is executing a synchronized method on a given object, can another thread execute a **different** synchronized method on the **same** object simultaneously? *(No — only one thread can hold the object's intrinsic lock at a time, regardless of which synchronized method it's executing.)*
32. Difference between a synchronized instance method and a `static synchronized` method? *(Instance method locks on `this`; static method locks on the `Class` object — these are independent locks.)*
33. Advantages of a synchronized **block** over a synchronized **method**? *(Finer-grained locking — reduces the scope of code under lock, improving concurrency; can lock on an arbitrary object, not just `this`.)*
34. What is a synchronized statement (block)? Give the syntax. `synchronized(lockObject) { ... }`
35. How do two threads communicate with each other? *(Via `wait()`/`notify()`/`notifyAll()` on a shared monitor object, or via higher-level constructs like `BlockingQueue`, `CountDownLatch`, `CyclicBarrier`, `Condition`.)*
36. In which class are `wait()`, `notify()`, `notifyAll()` defined? *(`java.lang.Object` — not `Thread`.)*
37. Why are `wait()`, `notify()`, `notifyAll()` defined in `Object` instead of `Thread`? *(Because locks are associated with **every object** (any object can be a monitor), not just `Thread` instances — inter-thread communication is tied to the object's monitor, which any object can have.)*
38. Can you call `wait()` without holding the lock? *(No — throws `IllegalMonitorStateException` if the calling thread is not the owner of the object's monitor.)*
39. If a waiting thread receives a notification, which state does it enter? *(`BLOCKED`/`RUNNABLE`-eligible — it moves out of `WAITING` and must **re-acquire the lock** before resuming; it doesn't run immediately.)*
40. In which methods can a thread release its lock while still alive? *(`wait()` — releases the lock while waiting; `sleep()` and `yield()` do **NOT** release any locks held.)*
41. Explain `wait()`, `notify()`, and `notifyAll()` in detail with a producer-consumer style example.
42. Difference between `notify()` and `notifyAll()`? *(`notify()` wakes up a **single** arbitrary waiting thread; `notifyAll()` wakes up **all** waiting threads, which then compete for the lock.)*
43. Once a thread gives a notification via `notify()`, which waiting thread gets the chance? *(No guarantee — JVM/OS decides arbitrarily among threads waiting on that monitor.)*
44. How can one thread interrupt another? (`thread.interrupt()`; the target thread checks `isInterrupted()` or catches `InterruptedException` if blocked in `sleep()`/`wait()`/`join()`.)
45. What is a deadlock? Is it possible to resolve/recover from a deadlock situation programmatically? *(Deadlock = circular wait for locks. Java provides **no automatic** deadlock recovery; must be prevented via careful lock-ordering design, timeouts (`tryLock`), or detected via tools like `jstack`/`ThreadMXBean.findDeadlockedThreads()`.)*
46. Which keyword is most associated with causing deadlock situations? *(`synchronized`, especially with nested/multiple locks acquired in inconsistent order.)*
47. How can you stop a thread explicitly (safely)? *(There's no safe forced-stop API; use a cooperative `volatile boolean` flag or `interrupt()` and have `run()` check it periodically, or use `Future.cancel()`.)*
48. Explain `suspend()` and `resume()` — why are they deprecated? *(They can suspend a thread while it holds a lock, causing deadlock, since it never gets a chance to release it. Deprecated since JDK 1.2.)*
49. What is starvation? How does it differ from deadlock? *(Starvation: a thread is perpetually denied access to a resource because other threads keep getting priority — no circular wait, just unfairness. Deadlock: threads are mutually blocked forever waiting on each other.)*
50. What is a race condition? *(Two or more threads access shared mutable state concurrently without proper synchronization, and the outcome depends on timing/interleaving.)*
51. What is a daemon thread? Give an example/purpose. *(Background/service thread that doesn't prevent JVM exit, e.g., GC thread, JIT compiler thread.)*
52. How do you check if a thread is a daemon? Can you change its daemon nature? Is the main thread daemon or non-daemon? *(`isDaemon()`; `setDaemon(boolean)` — **must be called before `start()`**, else throws `IllegalThreadStateException`; the `main` thread is **non-daemon**.)*
53. Explain `ThreadGroup` — its purpose, hierarchy, and key methods.
54. What is `ThreadLocal`? Explain with a real-world use case.

### Advanced / Architect-Level (Extended Beyond Source)
55. How does `ThreadLocal` cause memory leaks in a thread-pool-based application, and how do you prevent it?
56. Explain the internal data structure (`ThreadLocalMap`) used by `ThreadLocal` and why its keys are `WeakReference`s.
57. Compare `ThreadLocal` vs `InheritableThreadLocal` vs `ScopedValue` (Java 21+). When would you choose each?
58. Why is `ReentrantLock` "reentrant"? Walk through a scenario where non-reentrant locking would deadlock a recursive method.
59. Explain the difference between fair and non-fair `ReentrantLock`. What's the performance trade-off?
60. When would you choose `ReentrantLock` over `synchronized`, and vice versa?
61. What is `ReentrantReadWriteLock`, and when would it outperform a plain `ReentrantLock`?
62. What is `StampedLock`, and how does its optimistic-read mode differ from a normal read lock?
63. Explain the risk of forgetting `unlock()` when using explicit `Lock` objects, and how to structure code to avoid it (`try/finally`).
64. What is the difference between `Runnable`, `Callable`, and `Future` — and how do they interact with `ExecutorService`?
65. What happens if an exception is thrown inside a `Callable`'s `call()` method? How do you retrieve it? *(Wrapped in `ExecutionException`, thrown when calling `Future.get()`.)*
66. Compare `Future.get()` (blocking) with `CompletableFuture` (non-blocking, composable). Why was `CompletableFuture` introduced?
67. What is the danger of using `Executors.newCachedThreadPool()` or `newFixedThreadPool()` in production without bounding queues? How would you configure a custom `ThreadPoolExecutor` safely?
68. Explain `corePoolSize`, `maximumPoolSize`, `keepAliveTime`, `workQueue`, and `RejectedExecutionHandler` in `ThreadPoolExecutor`.
69. What are Virtual Threads (Project Loom, Java 21)? How do they differ from platform threads, and what workloads benefit most?
70. Why is thread-pooling generally discouraged for virtual threads?
71. What is "pinning" in the context of virtual threads, and how was it addressed in later JDK releases (Java 24 / JEP 491)?
72. What is Structured Concurrency, and what problem does it solve compared to manually managing multiple `Future`s?
73. Explain `ScopedValue` and why it's considered safer/more scalable than `InheritableThreadLocal` for virtual threads.
74. How would you diagnose a deadlock in a running production JVM? *(Thread dump via `jstack`, `kill -3`, or `ThreadMXBean.findDeadlockedThreads()`; look for `Found one Java-level deadlock` in the dump.)*
75. What's the Java Memory Model's (JMM) role in visibility/ordering guarantees for multithreaded code, and how does `volatile` relate to `synchronized`/locks?

---

## Appendix: Quick Reference — Common Pitfalls

| Pitfall | Why It's Wrong | Fix |
|---|---|---|
| Using `ThreadLocal` in a pooled executor without `remove()` | Stale data leaks across tasks/requests; memory leak | Always `remove()` in `finally` |
| Forgetting `unlock()` on `ReentrantLock` | Lock never released → permanent block for other threads | Wrap critical section in `try { } finally { lock.unlock(); }` |
| Calling `run()` instead of `start()` | Executes synchronously on the caller's thread — no concurrency achieved | Always call `start()` to spawn a new thread |
| Using `Thread.stop()/suspend()/resume()` | Deprecated, unsafe — can corrupt shared state, cause deadlock | Use cooperative flags/`interrupt()`, or `Future.cancel()` |
| Unbounded thread pools in production | Resource exhaustion, `OutOfMemoryError` under load | Use bounded queues + explicit `ThreadPoolExecutor` configuration |
| Ignoring `InterruptedException` (swallowing silently) | Breaks cooperative cancellation, hides shutdown signals | Restore interrupt status (`Thread.currentThread().interrupt();`) or propagate |
| Assuming `notify()` wakes a *specific* thread | No such guarantee exists | Use `notifyAll()` unless you can prove single-waiter safety, or prefer `java.util.concurrent` constructs |

---
*End of Chapter 8 Study Material.*
