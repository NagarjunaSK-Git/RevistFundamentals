# Core Java — Exception Handling
### Study Material for Senior Java Architect Interview Preparation

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Runtime Stack Mechanism](#2-runtime-stack-mechanism)
3. [Default Exception Handling in Java](#3-default-exception-handling-in-java)
4. [Exception Hierarchy](#4-exception-hierarchy)
5. [Checked vs Unchecked Exceptions](#5-checked-vs-unchecked-exceptions)
6. [Customized Exception Handling using try-catch](#6-customized-exception-handling-using-try-catch)
7. [Control Flow in try-catch](#7-control-flow-in-try-catch)
8. [Methods to Print Exception Information](#8-methods-to-print-exception-information)
9. [Try with Multiple Catch Blocks](#9-try-with-multiple-catch-blocks)
10. [The finally Block](#10-the-finally-block)
11. [final vs finally vs finalize](#11-final-vs-finally-vs-finalize)
12. [Control Flow in try-catch-finally](#12-control-flow-in-try-catch-finally)
13. [Control Flow in Nested try-catch-finally](#13-control-flow-in-nested-try-catch-finally)
14. [Valid/Invalid Combinations of try-catch-finally](#14-validinvalid-combinations-of-try-catch-finally)
15. [throw Keyword](#15-throw-keyword)
16. [throws Keyword](#16-throws-keyword)
17. [Exception Handling Keywords Summary](#17-exception-handling-keywords-summary)
18. [Compile Time Errors in Exception Handling](#18-compile-time-errors-in-exception-handling)
19. [Customized (User-Defined) Exceptions](#19-customized-user-defined-exceptions)
20. [Top-10 Exceptions](#20-top-10-exceptions)
21. [Java 7 Enhancements](#21-java-7-enhancements)
22. [Exception Propagation](#22-exception-propagation)
23. [Rethrowing an Exception](#23-rethrowing-an-exception)
24. [Modern Java Updates (Java 8 → Java 21+)](#24-modern-java-updates-java-8--java-21)
25. [Architect-Level Best Practices](#25-architect-level-best-practices)
26. [Interview Questions & Cheat Sheet](#26-interview-questions--cheat-sheet)

---

## 1. Introduction

**Exception**: An unwanted, unexpected event that disturbs the normal flow of a program is called an **exception**.

Examples (conceptual): `SleepingException`, `TyrePuncturedException`, `FileNotFoundException`, etc.

> It is **highly recommended** to handle exceptions. The main objective of exception handling is the **graceful (normal) termination** of the program.

**What does "handling" actually mean?**
Exception handling does **not** mean repairing the exception. It means defining an **alternative path** so the rest of the program can continue normally.

**Example scenario:** Suppose the requirement is to read data from a remote file located in London. If, at runtime, the London file is unavailable, the program should not terminate abnormally — instead, it should fall back to a local file and continue.

```java
try {
    // read data from London file
} catch (FileNotFoundException e) {
    // use local file and continue rest of the program normally
}
```

**Architect's Insight:** Exception handling is a *control-flow* and *resilience* design decision, not merely a syntax feature. In distributed/microservices architecture, this same principle scales up to patterns like **Circuit Breaker**, **Fallback**, and **Retry** (e.g., Resilience4j) — the London-file example is literally a mini fallback pattern.

---

## 2. Runtime Stack Mechanism

- For every **thread**, the JVM creates a **separate stack** at the time of thread creation.
- Every method call made by that thread is stored in the stack.
- Each entry in the stack is called an **"Activation Record"** (or **"Stack Frame"**).
- After a method call completes, JVM **removes** the corresponding entry from the stack.
- After all method calls complete, JVM **destroys the (empty) stack** and the program terminates **normally**.

```java
class Test {
    public static void main(String[] args) {
        doStuff();
    }
    public static void doStuff() {
        doMoreStuff();
    }
    public static void doMoreStuff() {
        System.out.println("Hello");
    }
}
```

**Output:**
```
Hello
```

**Stack progression:**
```
[]  →  [main]  →  [doStuff, main]  →  [doMoreStuff, doStuff, main]
      →  [doStuff, main]  →  [main]  →  [] (destroyed by JVM)
```

---

## 3. Default Exception Handling in Java

When no explicit handling code exists, the JVM follows this sequence:

1. If an exception is raised inside a method, **that method is responsible** for creating an **Exception object** containing:
   1. Name of the exception
   2. Description of the exception
   3. Location of the exception (**stack trace**)
2. The method hands this Exception object over to the **JVM**.
3. JVM checks if the method has handling code. If not, JVM **terminates that method abnormally** and removes its entry from the stack.
4. JVM checks the **caller method**. If it also has no handling code, it too is terminated abnormally, and its stack entry removed.
5. This continues up to `main()`. If `main()` also has no handling code, it's terminated too.
6. JVM then hands control to the **Default Exception Handler**.
7. The Default Exception Handler prints the exception information to the console in this format, then terminates the program **abnormally**:

```
Exception in thread "xxx(main)" Name of exception: description
Location of exception (stack trace)
```

**Example:**
```java
class Test {
    public static void main(String[] args) {
        doStuff();
    }
    public static void doStuff() {
        doMoreStuff();
    }
    public static void doMoreStuff() {
        System.out.println(10 / 0);
    }
}
```

**Output:**
```
Exception in thread "main" java.lang.ArithmeticException: / by zero
    at Test.doMoreStuff(Test.java:10)
    at Test.doStuff(Test.java:7)
    at Test.main(Test.java:4)
```

> **Note:** The Default Exception Handler internally uses `printStackTrace()` to print this information.

---

## 4. Exception Hierarchy

```
                    Object
                       ↑
                   Throwable
                   /        \
             Exception       Error
             /    |    \         \
   RuntimeException  IOException  ...   VirtualMachineError
        |               |                    |    |
  ArithmeticException  FileNotFoundException  OutOfMemoryError  StackOverflowError
  NullPointerException
  IndexOutOfBoundsException
     ├─ ArrayIndexOutOfBoundsException
     └─ StringIndexOutOfBoundsException
  IllegalArgumentException
     └─ NumberFormatException
  ClassCastException
  IllegalStateException
```

- **`Throwable`** acts as the **root** of the exception hierarchy. It has two direct child classes: **`Exception`** and **`Error`**.

### Exception
- Most exceptions are **caused by the program** and are **recoverable**.
- Example: `FileNotFoundException` → fall back to a local file and continue normally.

### Error
- Most errors are **not caused by the program**; they stem from **lack of system resources** and are **non-recoverable**.
- Example: `OutOfMemoryError` → as a programmer we can't fix this; only the System/Server admin can increase heap memory.

---

## 5. Checked vs Unchecked Exceptions

| Aspect | Checked Exception | Unchecked Exception |
|---|---|---|
| Compiler check | Compiler forces the programmer to handle it (`try-catch` or `throws`) | Compiler does **not** force handling |
| Examples | `FileNotFoundException`, `IOException`, `SQLException` | `ArithmeticException`, `NullPointerException`, `ArrayIndexOutOfBoundsException` |
| Parent classes | `Exception` and its children (excluding `RuntimeException` tree) | `RuntimeException` and its children, `Error` and its children |

> **Note:** Whether checked or unchecked, an exception can **only occur at runtime**. There is **no possibility of an exception occurring at compile time** — compile time errors are separate from exceptions.

### Fully Checked vs Partially Checked

- A checked exception is **fully checked** if and only if **all its child classes are also checked**.
  - Examples: `IOException`, `InterruptedException`
- A checked exception is **partially checked** if and only if **some of its child classes are unchecked**.
  - Examples: `Exception`, `Throwable`

> **Note:** The **only two** partially-checked exceptions in Java are `Throwable` and `Exception`.

### Behavior Cheat Table

| Exception/Error | Category |
|---|---|
| `RuntimeException` | Unchecked |
| `Error` | Unchecked |
| `IOException` | Fully checked |
| `Exception` | Partially checked |
| `InterruptedException` | Fully checked |
| `Throwable` | Partially checked |
| `ArithmeticException` | Unchecked |
| `NullPointerException` | Unchecked |
| `FileNotFoundException` | Fully checked |

---

## 6. Customized Exception Handling using try-catch

- The code that **may** raise an exception is called **risky code**.
- Risky code goes inside the `try` block; the corresponding handling code goes inside the `catch` block.

```java
try {
    // Risky code
} catch (Exception e) {
    // Handling code
}
```

### Without try-catch vs With try-catch

**Without try-catch (Abnormal termination):**
```java
class Test {
    public static void main(String[] args) {
        System.out.println("statement1");
        System.out.println(10 / 0);
        System.out.println("statement3");
    }
}
```
```
statement1
Exception in thread "main" java.lang.ArithmeticException: / by zero
    at Test.main(Test.java:...)
```

**With try-catch (Normal termination):**
```java
class Test {
    public static void main(String[] args) {
        System.out.println("statement1");
        try {
            System.out.println(10 / 0);
        } catch (ArithmeticException e) {
            System.out.println(10 / 2);
        }
        System.out.println("statement3");
    }
}
```
```
statement1
5
statement3
```

---

## 7. Control Flow in try-catch

```java
try {
    statement1;
    statement2;
    statement3;
}
catch (X e) {
    statement4;
}
statement5;
```

| Case | Scenario | Result |
|---|---|---|
| 1 | No exception | 1, 2, 3, 5 → **Normal termination** |
| 2 | Exception at statement2, catch matched | 1, 4, 5 → **Normal termination** |
| 3 | Exception at statement2, catch **not** matched | 1 executes, then **abnormal termination** |
| 4 | Exception raised at statement4 or statement5 | Always **abnormal termination** |

**Important Notes:**
1. Within the `try` block, if an exception is raised at any point, **the rest of the try block is skipped**, even though the exception is later handled. Hence, keep the try block as **short as possible**, containing only risky code.
2. If a statement that can raise an exception is **not** part of any `try` block, it always causes abnormal termination.
3. Exceptions can also be raised **inside catch and finally blocks**.

---

## 8. Methods to Print Exception Information

`Throwable` defines the following methods to print exception details:

| Method | Output Format |
|---|---|
| `printStackTrace()` | `Name of exception: description` + full **stack trace** |
| `toString()` | `Name of exception: description` (no stack trace) |
| `getMessage()` | Only the **description** |

```java
class Test {
    public static void main(String[] args) {
        try {
            System.out.println(10 / 0);
        } catch (ArithmeticException e) {
            e.printStackTrace();               // java.lang.ArithmeticException: / by zero  \n  at Test.main(Test.java:6)
            System.out.println(e);              // java.lang.ArithmeticException: / by zero
            System.out.println(e.getMessage()); // / by zero
        }
    }
}
```

---

## 9. Try with Multiple Catch Blocks

The way of handling an exception varies from exception to exception. Hence, for every exception type, it's recommended to take a **separate catch block**. `try` with multiple `catch` blocks is possible and **recommended**.

```java
try {
    // risky code
}
catch (FileNotFoundException e) {
    // use local file
}
catch (ArithmeticException e) {
    // perform arithmetic-specific handling
}
catch (SQLException e) {
    // switch to a fallback DB
}
catch (Exception e) {
    // generic/default handler — placed LAST
}
```

### ⚠️ Ordering Rule — Child to Parent

If multiple catch blocks are used, their **order matters** — they must go from **child (more specific) to parent (more generic)**. Reversing the order causes a **compile-time error**.

```java
// ❌ INVALID — CE: exception java.lang.ArithmeticException has already been caught
try {
    System.out.println(10 / 0);
} catch (Exception e) {
    e.printStackTrace();
} catch (ArithmeticException e) {   // unreachable — Exception already covers it
    e.printStackTrace();
}
```

```java
// ✅ VALID
try {
    System.out.println(10 / 0);
} catch (ArithmeticException e) {
    e.printStackTrace();
} catch (Exception e) {
    e.printStackTrace();
}
```

---

## 10. The finally Block

- It is **not recommended** to place cleanup code inside `try` (no guarantee every statement executes).
- It is **not recommended** to place cleanup code inside `catch` (won't execute if no exception occurs).
- **`finally`** is the ideal place for cleanup code — it executes **always**, irrespective of whether an exception was raised, and irrespective of whether it was handled.

```java
try {
    // risky code
}
catch (X e) {
    // handling code
}
finally {
    // cleanup code — ALWAYS executes
}
```

### Case 1 — No Exception
```java
try {
    System.out.println("try block executed");
} catch (ArithmeticException e) {
    System.out.println("catch block executed");
} finally {
    System.out.println("finally block executed");
}
```
```
try block executed
finally block executed
```

### Case 2 — Exception Raised, Matching Catch
```java
try {
    System.out.println("try block executed");
    System.out.println(10 / 0);
} catch (ArithmeticException e) {
    System.out.println("catch block executed");
} finally {
    System.out.println("finally block executed");
}
```
```
try block executed
catch block executed
finally block executed
```

### Case 3 — Exception Raised, Catch NOT Matching
```java
try {
    System.out.println("try block executed");
    System.out.println(10 / 0);
} catch (NullPointerException e) {
    System.out.println("catch block executed");
} finally {
    System.out.println("finally block executed");
}
```
```
try block executed
finally block executed
Exception in thread "main" java.lang.ArithmeticException: / by zero
    at Test.main(Test.java:8)
```
(`finally` runs **before** the JVM propagates the unmatched exception upward — abnormal termination.)

### return vs finally

Even if a `return` statement is present in `try`/`catch`, `finally` executes **first**, and only then the `return` is honored. **`finally` dominates `return`.**

```java
class Test {
    public static void main(String[] args) {
        try {
            System.out.println("try block executed");
            return;
        } catch (ArithmeticException e) {
            System.out.println("catch block executed");
        } finally {
            System.out.println("finally block executed");
        }
    }
}
```
```
try block executed
finally block executed
```

If `return` is present in **try, catch, AND finally**, the **finally's return value wins**:

```java
class Test {
    public static void main(String[] args) {
        System.out.println(m1());
    }
    public static int m1() {
        try {
            System.out.println(10 / 0);
            return 777;
        } catch (ArithmeticException e) {
            return 888;
        } finally {
            return 999;
        }
    }
}
```
```
999
```

> **Architect's caution:** Returning from a `finally` block **silently swallows exceptions** and overrides values from `try`/`catch` — this is flagged by static analyzers (SonarQube, SpotBugs) as an anti-pattern. **Never `return` from `finally` in production code.**

### finally vs System.exit(0)

The **only** situation where `finally` does **not** execute is when `System.exit(0)` (or any status code) is called — this shuts down the JVM itself.

```java
try {
    System.out.println("try");
    System.exit(0);
} catch (ArithmeticException e) {
    System.out.println("catch block executed");
} finally {
    System.out.println("finally block executed"); // NEVER PRINTED
}
```
```
try
```

**Notes:**
1. The argument to `System.exit()` acts as a **status code**. Any integer can be used.
2. `0` = normal termination; **non-zero** = abnormal termination.
3. This status code is used **internally by the OS/JVM**; the program's outward behavior/result is the same regardless of the code value.

---

## 11. final vs finally vs finalize

| Keyword | Type | Purpose |
|---|---|---|
| **`final`** | Modifier | Applicable to classes, methods, variables. <br>• Final class → cannot be extended (no child class). <br>• Final method → cannot be overridden. <br>• Final variable → cannot be reassigned. |
| **`finally`** | Block | Always associated with `try-catch`; used to maintain **cleanup code** that must run regardless of exception outcome. |
| **`finalize()`** | Method | Invoked by the **Garbage Collector** just before destroying an object, to perform cleanup. |

**Notes:**
1. `finally` handles cleanup related to **try-block resources**; `finalize()` handles cleanup related to an **object's lifecycle**.
2. `finally` is **recommended over `finalize()`** because GC behavior is unpredictable (no guaranteed timing/invocation).

> **⚠️ Modern Java Update:** `Object.finalize()` was **deprecated since Java 9** and **marked for removal**. Use **`try-with-resources`** (Java 7+) with `AutoCloseable`, or the **`java.lang.ref.Cleaner`** API (Java 9+) as the modern replacement for object cleanup logic.

---

## 12. Control Flow in try-catch-finally

```java
try {
    stmt1;
    stmt2;
    stmt3;
}
catch (Exception e) {
    stmt4;
}
finally {
    stmt5;
}
stmt6;
```

| Case | Scenario | Result |
|---|---|---|
| 1 | No exception | 1,2,3,5,6 → normal termination |
| 2 | Exception at stmt2, catch matched | 1,4,5,6 → normal termination |
| 3 | Exception at stmt2, catch not matched | 1,5 → abnormal termination |
| 4 | Exception at stmt4 | Always abnormal termination — but `finally` (stmt5) executes **before** propagation |
| 5 | Exception at stmt5 or stmt6 | Always abnormal termination |

---

## 13. Control Flow in Nested try-catch-finally

```java
try {
    stmt1; stmt2; stmt3;
    try {
        stmt4; stmt5; stmt6;
    }
    catch (X e) {
        stmt7;
    }
    finally {
        stmt8;
    }
    stmt9;
}
catch (Y e) {
    stmt10;
}
finally {
    stmt11;
}
stmt12;
```

| Case | Scenario | Result |
|---|---|---|
| 1 | No exception | 1,2,3,4,5,6,8,9,11,12 → normal |
| 2 | Exception at stmt2, outer catch matched | 1,10,11,12 → normal |
| 3 | Exception at stmt2, outer catch not matched | 1,11 → abnormal |
| 4 | Exception at stmt5, inner catch matched | 1,2,3,4,7,8,9,11,12 → normal |
| 5 | Exception at stmt5, inner not matched, outer matched | 1,2,3,4,8,10,11,12 → normal |
| 6 | Exception at stmt5, neither matched | 1,2,3,4,8,11 → abnormal |
| 7 | Exception at stmt7, outer catch matched | 1,2,3,4,5,6,8,10,11,12 → normal |
| 8 | Exception at stmt7, outer not matched | 1,2,3,4,5,6,8,11 → abnormal |
| 9 | Exception at stmt8, outer matched | 1,2,3,4,5,6,7,10,11,12 → normal |
| 10 | Exception at stmt8, outer not matched | 1,2,3,4,5,6,7,11 → abnormal |
| 11 | Exception at stmt9, outer matched | 1,2,3,4,5,6,7,8,10,11,12 → normal |
| 12 | Exception at stmt9, outer not matched | 1,2,3,4,5,6,7,8,11 → abnormal |
| 13 | Exception at stmt10 | Always abnormal, but stmt11 (finally) executes first |
| 14 | Exception at stmt11 or stmt12 | Always abnormal |

**Notes:**
1. If we never enter a `try` block, its `finally` won't execute. Once we enter `try`, we **cannot exit** without `finally` executing (except `System.exit()`).
2. **Nested try-catch is legal.**
3. The **most specific** exceptions should be handled by the **inner** try-catch; **generalized** exceptions by the **outer** try-catch.

### Multiple Exceptions — Only the Most Recent is Reported

```java
class Test {
    public static void main(String[] args) {
        try {
            System.out.println(10 / 0);
        }
        catch (ArithmeticException e) {
            System.out.println(10 / 0);  // second exception raised inside catch
        }
        finally {
            String s = null;
            System.out.println(s.length()); // third exception raised inside finally
        }
    }
}
```
```
Exception in thread "main" java.lang.NullPointerException
```

> **Note:** The Default Exception Handler can handle only **one** exception at a time — the **most recently raised** one. Earlier exceptions are effectively **lost/suppressed** in classic exception handling.
>
> **Modern Java Update (Java 7+):** This "lost exception" problem is solved by **suppressed exceptions**. When using `try-with-resources`, if the `close()` method throws an exception while the try block is already propagating one, the close-time exception is **not lost** — it's attached via `Throwable.addSuppressed()` and retrievable via `getSuppressed()`.

---

## 14. Valid/Invalid Combinations of try-catch-finally

### Rules
1. `try` **must** be followed by either `catch` or `finally` (or both). `try` alone is invalid *(pre-Java 7; Java 7+ also allows try-with-resources alone)*.
2. `catch` **must** be preceded by `try`. `catch` without `try` is invalid.
3. `finally` **must** be preceded by `try`. `finally` without `try` is invalid.
4. Order matters: `try` → `catch` → `finally`.
5. Nesting of `try-catch-finally` inside `try`, `catch`, or `finally` is **allowed**.
6. Curly braces `{}` are **mandatory** for `try`, `catch`, and `finally` blocks.

### Valid Examples ✔️
```java
try {}
catch (X e) {}

try {}
finally {}

try {}
catch (X e) {}
finally {}

try {}
catch (X e) {}
try {}
finally {}

try {}
catch (X e) {
    try {}
    catch (Y e1) {}
}

try {}
catch (X e) {}
finally {
    try {}
    catch (Y e1) {}
    finally {}
}
```

### Invalid Examples ✘ (with Compile Errors)
```java
finally {}                    // CE: 'finally' without 'try'

try {}                        // CE: 'try' without 'catch', 'finally' or resource declarations

catch (X e) {}                // CE: 'catch' without 'try'

try {}
catch (X e) {}
catch (X e) {}                // CE: exception X has already been caught

try {}                        // CE: '{' expected (syntax) if braces misused
catch (X e)
System.out.println("Hello"); // missing braces
```

---

## 15. throw Keyword

Sometimes we can explicitly create an Exception object and hand it over to the JVM **manually** using `throw`.

```java
throw new ArithmeticException("/ by zero");
```
- `new ArithmeticException(...)` → creates the exception object explicitly.
- `throw` → hands that object over to the JVM manually.

**These two programs behave identically:**

```java
// Implicit creation by JVM
class Test {
    public static void main(String[] args) {
        System.out.println(10 / 0);
    }
}
```
```java
// Explicit creation & throw
class Test {
    public static void main(String[] args) {
        throw new ArithmeticException("/ by zero");
    }
}
```

> **Note:** In general, `throw` is used for **customized exceptions**, though it can technically be used with predefined ones too.

### Case 1 — `throw e;` where `e` refers to `null`

```java
class Test3 {
    static ArithmeticException e; // null by default
    public static void main(String[] args) {
        throw e;
    }
}
```
```
Exception in thread "main" java.lang.NullPointerException
    at Test3.main(Test3.java:5)
```

### Case 2 — Unreachable Statement After throw

```java
class Test3 {
    public static void main(String[] args) {
        throw new ArithmeticException("/ by zero");
        System.out.println("hello");   // CE: unreachable statement
    }
}
```

### Case 3 — Only Throwable Types Allowed

```java
class Test3 {
    public static void main(String[] args) {
        throw new Test3();   // CE: incompatible types
                              // found: Test3, required: java.lang.Throwable
    }
}
```
```java
// ✅ Fix: extend Throwable/Exception/RuntimeException
class Test3 extends RuntimeException {
    public static void main(String[] args) {
        throw new Test3();
    }
}
```
```
Exception in thread "main" Test3
    at Test3.main(Test3.java:4)
```

---

## 16. throws Keyword

If there's a chance of raising a **checked exception**, we **must** handle it via `try-catch` **or** declare it with `throws` — otherwise the code **won't compile**.

```java
import java.io.*;
class Test3 {
    public static void main(String[] args) {
        PrintWriter out = new PrintWriter("abc.txt");
        out.println("hello");
    }
}
```
```
CE: Unreported exception java.io.FileNotFoundException; must be caught or declared to be thrown.
```

**Fix Option 1 — try-catch:**
```java
class Test3 {
    public static void main(String[] args) {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {}
    }
}
```

**Fix Option 2 — throws (delegates responsibility to the caller):**
```java
class Test3 {
    public static void main(String[] args) throws InterruptedException {
        Thread.sleep(5000);
    }
}
```

### Propagating throws Across the Call Chain

```java
class Test {
    public static void main(String[] args) throws InterruptedException {
        doStuff();
    }
    public static void doStuff() throws InterruptedException {
        doMoreStuff();
    }
    public static void doMoreStuff() throws InterruptedException {
        Thread.sleep(5000);
    }
}
```
> If **any one** `throws` in this chain is removed, the code **won't compile**.

### Key Notes on throws
- Main objective: **delegate** exception-handling responsibility to the caller method.
- `throws` is only meaningful for **checked exceptions**. Using it for unchecked exceptions has no real effect (though syntactically legal).
- `throws` is only required to **satisfy the compiler**. It does **not** prevent abnormal termination at runtime if the exception is never actually caught anywhere in the chain.
- ⚠️ **Recommended:** Prefer `try-catch` over `throws` wherever real recovery is possible; use `throws` when delegation to a higher layer makes architectural sense (e.g., service layer delegates to a global exception handler).

### Case 1 — throws Must Use Throwable Types
```java
class Test3 {
    public static void main(String[] args) throws Test3 {}   // CE: incompatible types
}
```
```java
class Test3 extends RuntimeException {
    public static void main(String[] args) throws Test3 {}   // ✅ Compiles fine
}
```

### Case 2 — Declaring throws Doesn't Prevent Abnormal Termination
```java
class Test3 {
    public static void main(String[] args) {
        throw new Exception();   // CE: unreported exception; must be caught or declared to be thrown
    }
}
```
```java
class Test3 {
    public static void main(String[] args) {
        throw new Error();       // Compiles fine (Error is unchecked) — Runtime error at execution
    }
}
```

### Case 3 — "Exception Never Thrown" Rule (Applies Only to Fully Checked Exceptions)

If, within a `try` block, there's **no chance** of a particular checked exception being raised, we **cannot** write a `catch` block for it — this causes: `exception XXX is never thrown in body of corresponding try statement`. **This rule applies only to fully checked exceptions.**

| Catch Type | Result |
|---|---|
| `catch(Exception e)` | ✅ Compiles (partially checked) |
| `catch(ArithmeticException e)` | ✅ Compiles (unchecked) |
| `catch(java.io.IOException e)` | ❌ CE (fully checked, never thrown in try) |
| `catch(InterruptedException e)` | ❌ CE (fully checked, never thrown in try) |
| `catch(Error e)` | ✅ Compiles (unchecked) |

### Case 4 — throws is Only for Constructors and Methods, NOT Classes
```java
class Test throws Exception {   // ❌ INVALID
    Test() throws Exception {}          // ✅ VALID
    methodOne() throws Exception {}     // ✅ VALID
}
```

---

## 17. Exception Handling Keywords Summary

| Keyword | Purpose |
|---|---|
| `try` | To maintain risky code |
| `catch` | To maintain handling code |
| `finally` | To maintain cleanup code |
| `throw` | To hand over our created exception object to the JVM **manually** |
| `throws` | To **delegate** the responsibility of exception handling to the caller method |

---

## 18. Compile Time Errors in Exception Handling

1. Exception `XXX` has already been caught.
2. Unreported exception `XXX`; must be caught or declared to be thrown.
3. Exception `XXX` is never thrown in body of corresponding try statement.
4. `try` without `catch` or `finally`.
5. `catch` without `try`.
6. `finally` without `try`.
7. Incompatible types — `found: Test, required: java.lang.Throwable`.
8. Unreachable statement.

---

## 19. Customized (User-Defined) Exceptions

Sometimes we need our **own exceptions** to model business/domain rules.

Examples: `InSufficientFundsException`, `TooYoungException`, `TooOldException`

```java
class TooYoungException extends RuntimeException {
    TooYoungException(String s) { super(s); }
}

class TooOldException extends RuntimeException {
    TooOldException(String s) { super(s); }
}

class CustomizedExceptionDemo {
    public static void main(String[] args) {
        int age = Integer.parseInt(args[0]);
        if (age > 60) {
            throw new TooYoungException("please wait some more time.... u will get best match");
        } else if (age < 18) {
            throw new TooOldException("u r age already crossed....no chance of getting married");
        } else {
            System.out.println("you will get match details soon by e-mail");
        }
    }
}
```

**Sample Runs:**
```
$ java CustomizedExceptionDemo 61
Exception in thread "main" TooYoungException: please wait some more time.... u will get best match
    at CustomizedExceptionDemo.main(CustomizedExceptionDemo.java:21)

$ java CustomizedExceptionDemo 27
you will get match details soon by e-mail

$ java CustomizedExceptionDemo 9
Exception in thread "main" TooOldException: u r age already crossed....no chance of getting married
    at CustomizedExceptionDemo.main(CustomizedExceptionDemo.java:25)
```

> **Note:** It's highly recommended to make customized exceptions **unchecked** by extending `RuntimeException` — this avoids polluting method signatures with `throws` clauses everywhere in the call chain (a common pain point in large enterprise codebases).

> We can also catch **any `Throwable` type**, including `Error`:
> ```java
> try { }
> catch (Error e) { }   // valid
> ```

**Architect's Insight — Checked vs Unchecked Custom Exceptions:**
- Modern frameworks (Spring, Hibernate) largely favor **unchecked exceptions** for custom exception hierarchies (`DataAccessException` in Spring is unchecked) to avoid "checked exception hell" through layered architectures.
- Reserve **checked exceptions** for truly recoverable conditions the *caller is expected to handle differently* (e.g., `InsufficientFundsException` in a banking API where the caller must show a specific UI message).

---

## 20. Top-10 Exceptions

Exceptions are classified based on **who raises them**:
1. **JVM Exceptions** — raised automatically by the JVM when a particular event occurs.
2. **Programmatic Exceptions** — raised explicitly by the programmer or API developer.

| # | Exception/Error | Parent | Checked? | Raised By |
|---|---|---|---|---|
| 1 | `ArrayIndexOutOfBoundsException` | `RuntimeException` | Unchecked | JVM |
| 2 | `NullPointerException` | `RuntimeException` | Unchecked | JVM |
| 3 | `StackOverflowError` | `Error` | Unchecked | JVM |
| 4 | `NoClassDefFoundError` | `Error` | Unchecked | JVM |
| 5 | `ClassCastException` | `RuntimeException` | Unchecked | JVM |
| 6 | `ExceptionInInitializerError` | `Error` | Unchecked | JVM |
| 7 | `IllegalArgumentException` | `RuntimeException` | Unchecked | Programmer/API |
| 8 | `NumberFormatException` | `IllegalArgumentException` | Unchecked | Programmer/API |
| 9 | `IllegalStateException` | `RuntimeException` | Unchecked | Programmer/API |
| 10 | `AssertionError` | `Error` | Unchecked | Programmer/API |

### 1. ArrayIndexOutOfBoundsException (AIOOBE)
Child of `RuntimeException`. Raised automatically when accessing an array with an out-of-range index.
```java
int[] x = new int[10];
System.out.println(x[0]);    // valid
System.out.println(x[100]);  // AIOOBE
System.out.println(x[-100]); // AIOOBE
```

### 2. NullPointerException (NPE)
Child of `RuntimeException`. Raised when invoking a method (or accessing a field) on `null`.
```java
String s = null;
System.out.println(s.length()); // NullPointerException
```

### 3. StackOverflowError
Child of `Error`. Raised when recursive calls exceed stack capacity.
```java
class Test {
    public static void methodOne() { methodTwo(); }
    public static void methodTwo() { methodOne(); }
    public static void main(String[] args) { methodOne(); }
}
```
```
Exception in thread "main" java.lang.StackOverflowError
```

### 4. NoClassDefFoundError
Child of `Error`. Raised when the JVM can't find a required `.class` file at runtime (was present at compile time but missing at runtime).

### 5. ClassCastException (CCE)
Child of `RuntimeException`. Raised when illegally casting a parent-type reference to an unrelated child type.
```java
Object o = new String("bhaskar");
Integer i = (Integer) o;  // Runtime exception: ClassCastException
```

### 6. ExceptionInInitializerError
Child of `Error`. Raised when an exception occurs during **static variable initialization** or **static block execution**.
```java
class Test {
    static int i = 10 / 0;
}
```
```
Exception in thread "main" java.lang.ExceptionInInitializerError
Caused by: java.lang.ArithmeticException: / by zero
```

### 7. IllegalArgumentException (IAE)
Child of `RuntimeException`. Raised explicitly to indicate a method was invoked with an **inappropriate argument**.
```java
Thread t = new Thread();
t.setPriority(10);   // valid   (1–10)
t.setPriority(100);  // IllegalArgumentException
```

### 8. NumberFormatException (NFE)
Child of `IllegalArgumentException`. Raised when converting a **malformed** String to a number.
```java
int i = Integer.parseInt("10");   // valid
int j = Integer.parseInt("ten");  // NumberFormatException
```

### 9. IllegalStateException (ISE)
Child of `RuntimeException`. Raised when a method is invoked at an **inappropriate time**.
```java
HttpSession session = req.getSession();
System.out.println(session.getId());
session.invalidate();
System.out.println(session.getId()); // IllegalStateException
```

### 10. AssertionError (AE)
Child of `Error`. Raised when an `assert` statement fails.
```java
assert(false);  // AssertionError (only if assertions enabled via -ea flag)
```

---

## 21. Java 7 Enhancements

### 21.1 try-with-resources

**Legacy approach (pre-Java 7):**
```java
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("abc.txt"));
    // use br
} catch (IOException e) {
    // handling code
} finally {
    if (br != null) br.close();
}
```

**Problems with the legacy approach:**
- The programmer must **explicitly** close every opened resource — increases complexity.
- A `finally` block **must be written explicitly**, increasing code length and reducing readability.

**Java 7 solution: try-with-resources**
Resources declared in the `try(...)` header are **automatically closed** once control leaves the try block — normally or abnormally.

```java
try (BufferedReader br = new BufferedReader(new FileReader("abc.txt"))) {
    // use br — closed automatically, no explicit close() or finally needed
} catch (IOException e) {
    // handling code
}
```

**Rules:**
1. Multiple resources can be declared, separated by `;`:
   ```java
   try (R1 r1 = ...; R2 r2 = ...; R3 r3 = ...) { ... }
   ```
2. All resources **must implement `java.lang.AutoCloseable`** (directly or indirectly). Most DB/network/File-IO classes already implement it.
3. Resource reference variables are **implicitly final** — reassignment inside the try block is a compile error.
   ```java
   try (BufferedReader br = new BufferedReader(new FileReader("abc.txt"))) {
       br = new BufferedReader(new FileReader("abc.txt")); // CE: cannot assign a value to final variable br
   }
   ```
4. Unlike pre-1.7, `try` **can stand alone with resources**, without a mandatory `catch` or `finally`:
   ```java
   try (R r = ...) {
       // valid — no catch/finally required
   }
   ```
5. Main advantage: the `finally` block essentially becomes **dummy/unnecessary** since resources are closed automatically.

### 21.2 Multi-Catch Block

**Legacy problem:** Multiple exceptions sharing the same handling logic still required **separate catch blocks**, bloating the code.

```java
// verbose — pre 1.7 style
try {
    // risky code
}
catch (ArithmeticException e) { e.printStackTrace(); }
catch (NullPointerException e) { e.printStackTrace(); }
catch (ClassCastException e) { System.out.println(e.getMessage()); }
catch (IOException e) { System.out.println(e.getMessage()); }
```

**Java 7 solution: multi-catch (`|`)**
```java
try {
    // risky code
}
catch (ArithmeticException | NullPointerException e) {
    e.printStackTrace();
}
catch (ClassCastException | IOException e) {
    System.out.println(e.getMessage());
}
```

> **Rule:** In a multi-catch block, the listed exception types must have **no relationship** (no child-to-parent, parent-to-child, or duplicate types) — otherwise it's a **compile-time error**.

```java
// ❌ INVALID — CE (Exception is parent of ArithmeticException)
try { }
catch (ArithmeticException | Exception e) { }
```

> **Bytecode note:** The catch parameter in a multi-catch clause is **implicitly final**; you cannot reassign `e` inside the block.

---

## 22. Exception Propagation

Within a method, if an exception is raised and the method does **not** handle it, the Exception object is **propagated to the caller**. The caller method then becomes responsible for handling it. This mechanism is called **Exception Propagation**.

```java
class Test {
    public static void main(String[] args) {
        try {
            m1();
        } catch (ArithmeticException e) {
            System.out.println("Handled in main: " + e);
        }
    }
    static void m1() {
        m2(); // no try-catch here — propagates upward
    }
    static void m2() {
        System.out.println(10 / 0); // exception originates here
    }
}
```
```
Handled in main: java.lang.ArithmeticException: / by zero
```

> **Note:** For **checked exceptions**, propagation must be **explicitly declared** via `throws` at every level of the call chain (compiler-enforced). For **unchecked exceptions**, propagation happens **automatically** without any `throws` declaration.

---

## 23. Rethrowing an Exception

To **convert one exception type into another**, we use the **rethrowing exception** technique — catch one exception and `throw` a different, more meaningful one.

```java
class Test {
    public static void main(String[] args) {
        try {
            System.out.println(10 / 0);
        }
        catch (ArithmeticException e) {
            throw new NullPointerException();
        }
    }
}
```
```
Exception in thread "main" java.lang.NullPointerException
    at Test.main(Test.java:...)
```

> **Architect's Insight — Exception Chaining:** Blindly rethrowing a *different* exception type **destroys the original stack trace and root cause**, making production debugging difficult. Prefer **exception chaining** using the cause-constructor or `initCause()`:
> ```java
> try {
>     System.out.println(10 / 0);
> } catch (ArithmeticException e) {
>     throw new ServiceException("Calculation failed", e); // preserves root cause
> }
> ```
> This preserves the **full causal chain**, visible via `printStackTrace()` as `Caused by: ...` — essential for RCA (Root Cause Analysis) in production incidents.

---

## 24. Modern Java Updates (Java 8 → Java 21+)

The source material predates Java 8. Below are the **critical, interview-relevant** updates every Senior Architect should know:

### 24.1 Helpful NullPointerExceptions — JEP 358 (Java 14+, default from Java 15)
Traditional NPEs told you *which line* failed, but not *which variable* was `null` in a chained call. Modern JVMs (with `-XX:+ShowCodeDetailsInExceptionMessages`, **default ON since Java 15**) give **precise, actionable messages**.

```java
class Test {
    public static void main(String[] args) {
        String s = null;
        System.out.println(s.length());
    }
}
```
**Java 8 output:**
```
Exception in thread "main" java.lang.NullPointerException
    at Test.main(Test.java:4)
```
**Java 15+ output:**
```
Exception in thread "main" java.lang.NullPointerException:
    Cannot invoke "String.length()" because "s" is null
    at Test.main(Test.java:4)
```

### 24.2 try-with-resources with "Effectively Final" Variables (Java 9+)
Prior to Java 9, you had to declare a **new** variable inside the `try(...)` header. Java 9 allows reusing an **effectively final** variable declared outside:

```java
// Java 9+ — no need to re-declare inside try()
BufferedReader br = new BufferedReader(new FileReader("abc.txt"));
try (br) {   // just reference it — must be effectively final
    System.out.println(br.readLine());
}
```

### 24.3 Suppressed Exceptions (Java 7+, often missed in interviews)
When `try-with-resources` auto-closes a resource and `close()` itself throws, that exception is **not lost** — it's added as a **suppressed exception** to the primary exception.

```java
class MyResource implements AutoCloseable {
    public void use() { throw new RuntimeException("Failure during use"); }
    public void close() { throw new RuntimeException("Failure during close"); }
}

public class Demo {
    public static void main(String[] args) {
        try (MyResource r = new MyResource()) {
            r.use();
        } catch (RuntimeException e) {
            System.out.println("Primary: " + e.getMessage());
            for (Throwable t : e.getSuppressed()) {
                System.out.println("Suppressed: " + t.getMessage());
            }
        }
    }
}
```
```
Primary: Failure during use
Suppressed: Failure during close
```

### 24.4 Pattern Matching for `instanceof` (Java 16+) — Useful in Exception Handling
```java
catch (Exception e) {
    if (e.getCause() instanceof SQLException sqlEx) {   // no explicit cast needed
        System.out.println("SQL error code: " + sqlEx.getErrorCode());
    }
}
```

### 24.5 Records as Lightweight Exception Payloads (Java 16+)
While exceptions themselves can't be records (they must extend `Throwable`), **Records** are commonly used to carry **structured error details** inside custom exceptions:
```java
record ErrorDetail(String code, String message, Instant timestamp) {}

class BusinessException extends RuntimeException {
    private final ErrorDetail detail;
    BusinessException(ErrorDetail detail) {
        super(detail.message());
        this.detail = detail;
    }
    public ErrorDetail getDetail() { return detail; }
}
```

### 24.6 Pattern Matching for switch + Exception Type Dispatch (Java 21 — Standardized)
```java
static String handle(Exception e) {
    return switch (e) {
        case NullPointerException npe -> "Null reference issue: " + npe.getMessage();
        case ArithmeticException ae   -> "Math error: " + ae.getMessage();
        case IllegalStateException ise -> "Invalid state: " + ise.getMessage();
        default -> "Unhandled: " + e.getClass().getSimpleName();
    };
}
```
This is now widely used in modern **global exception handlers** (e.g., Spring `@ExceptionHandler` combined with sealed exception hierarchies).

### 24.7 Sealed Classes for Exception Hierarchies (Java 17+)
Architects increasingly use **sealed classes** to create **closed, exhaustive exception hierarchies** for domain errors — enabling compiler-checked exhaustiveness in `switch` pattern matching.
```java
public sealed class OrderException extends RuntimeException
        permits OutOfStockException, PaymentFailedException, InvalidOrderException {
    protected OrderException(String msg) { super(msg); }
}
public final class OutOfStockException extends OrderException {
    public OutOfStockException(String msg) { super(msg); }
}
public final class PaymentFailedException extends OrderException {
    public PaymentFailedException(String msg) { super(msg); }
}
public final class InvalidOrderException extends OrderException {
    public InvalidOrderException(String msg) { super(msg); }
}
```

### 24.8 Structured Concurrency & Exception Propagation (Java 21 Preview / 24 finalize track)
With `StructuredTaskScope` (JEP 428/453), exceptions from child virtual-thread tasks are **captured and propagated** to the parent scope in a structured, deterministic way — replacing ad-hoc `Future.get()` exception unwrapping (`ExecutionException`).
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Supplier<String> task1 = scope.fork(() -> fetchUser());
    Supplier<String> task2 = scope.fork(() -> fetchOrders());
    scope.join();
    scope.throwIfFailed(e -> new RuntimeException("Fetch failed", e));
    // safe to use task1.get(), task2.get()
}
```

### 24.9 `finalize()` Deprecation (Java 9) & Removal Track
- `Object.finalize()` deprecated in Java 9 (`@Deprecated(since="9")`), marked `forRemoval = true` since Java 18.
- **Replacement:** `java.lang.ref.Cleaner` (Java 9+) or `AutoCloseable` + `try-with-resources`.
```java
public class ResourceHolder implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    public ResourceHolder() {
        this.cleanable = cleaner.register(this, () -> System.out.println("Cleaned up!"));
    }
    @Override
    public void close() { cleanable.clean(); }
}
```

---

## 25. Architect-Level Best Practices

1. **Never swallow exceptions silently** (`catch (Exception e) {}`) — always log with context or rethrow.
2. **Don't use exceptions for normal control flow** — they're expensive (stack trace capture) and hurt readability.
3. **Prefer specific exceptions over generic `Exception`/`Throwable`** in catch blocks — avoid masking bugs.
4. **Keep the `try` block minimal** — only the statement(s) that can actually fail.
5. **Preserve the causal chain** when wrapping/rethrowing (`new MyException(msg, originalException)`).
6. **Favor unchecked custom exceptions** in service/business layers to avoid checked-exception propagation noise across layers (especially with Streams/Lambdas, which don't support checked exceptions natively).
7. **Centralize exception handling** at architecture boundaries — e.g., `@ControllerAdvice`/`@ExceptionHandler` in Spring, or a global filter in Servlet-based apps.
8. **Use `try-with-resources` for every `AutoCloseable`** (streams, connections, sockets) — never manually manage `close()` in `finally`.
9. **Avoid overriding `finalize()`** — use `Cleaner` or `AutoCloseable`.
10. **Log stack traces once**, at the boundary where the exception is finally handled — avoid duplicate logging at every layer it passes through.
11. **Use `Optional` where the "exceptional" case is really just "absence of a value"** — not every missing value warrants an exception.
12. In **reactive/async** code (`CompletableFuture`, Reactor, RxJava), remember exceptions propagate via `onError`/`exceptionally` callbacks — traditional `try-catch` around async submission does **not** catch exceptions thrown inside the async task.

---

## 26. Interview Questions & Cheat Sheet

### Conceptual / Fundamentals
1. **What is an exception, and what is the real goal of exception handling?**
   → An unwanted event disrupting normal flow; the goal is **graceful termination** by defining an alternative execution path, not "fixing" the exception.

2. **Differentiate `Exception` and `Error`.**
   → `Exception` = mostly caused by the program, recoverable. `Error` = usually resource/JVM-level, non-recoverable.

3. **What is the difference between checked and unchecked exceptions?**
   → Checked exceptions are verified by the compiler at compile time (must be caught or declared); unchecked (RuntimeException/Error subtypes) are not compiler-checked.

4. **Can an exception occur at compile time?**
   → No. Whether checked or unchecked, exceptions **only** occur at **runtime**. What look like "compile time exceptions" are actually **compile time errors** (a different concept).

5. **What are "fully checked" and "partially checked" exceptions? Give examples.**
   → Fully checked: all child classes are checked (`IOException`, `InterruptedException`). Partially checked: some children are unchecked (`Exception`, `Throwable` — the *only two*).

6. **What is the Default Exception Handler, and what does it print?**
   → JVM's built-in handler used when no explicit handling code exists anywhere in the call chain; it prints `Exception in thread "xxx" Name: description` plus the stack trace, using `printStackTrace()` internally.

### try-catch-finally Mechanics
7. **Why should the `try` block be as small as possible?**
   → Because execution stops at the first exception within `try`; remaining statements in `try` are skipped even if handled — larger try blocks risk skipping important logic silently.

8. **What is the order rule for multiple catch blocks, and what happens if violated?**
   → Must go from **child to parent** (most specific first). Reversing causes: `exception XXX has already been caught` — a compile-time error.

9. **Does `finally` always execute? What's the one exception?**
   → Yes, always — except when `System.exit()` is called, which shuts down the JVM before `finally` can run.

10. **What happens if `return` appears in `try`, `catch`, and `finally`?**
    → The `finally` block's `return` **wins** — it overrides values from `try`/`catch`. This is considered a **code smell**.

11. **Difference between `final`, `finally`, and `finalize()`.**
    → `final`: modifier (non-inheritable/non-overridable/non-reassignable). `finally`: cleanup block tied to try-catch. `finalize()`: deprecated GC hook method (replaced by `Cleaner`/`AutoCloseable`).

12. **If an exception is raised in `try`, then again in `catch`, then again in `finally` — what does the Default Handler report?**
    → Only the **most recently raised** exception; earlier ones are lost (unless using try-with-resources suppressed-exception mechanism).

### throw / throws
13. **Difference between `throw` and `throws`.**
    → `throw` is a statement used to explicitly hand an exception instance to the JVM. `throws` is a method-signature clause used to delegate exception-handling responsibility to the caller (compile-time contract, mainly for checked exceptions).

14. **Can you write a statement right after `throw`?**
    → No — it results in "unreachable statement" compile error since `throw` unconditionally terminates that code path.

15. **Is `throws` useful for unchecked exceptions?**
    → Syntactically legal but functionally useless — the compiler doesn't require it, and it changes nothing about actual runtime behavior.

16. **Does declaring `throws Exception` guarantee the program won't crash?**
    → No. It only satisfies the *compiler*. If the exception is never caught anywhere up the call chain, the program still terminates abnormally at runtime.

### Custom Exceptions & Design
17. **Why should custom exceptions extend `RuntimeException` rather than `Exception`?**
    → To avoid forcing `throws` declarations through every layer of the call stack ("checked exception hell"), especially problematic with Streams/Lambdas which don't support checked exceptions.

18. **How do you preserve the root cause when converting one exception type to another?**
    → Use exception chaining: `throw new MyException("msg", originalException);` instead of blind rethrow, which preserves the `Caused by:` trace.

19. **What's wrong with `catch (Exception e) { }` (empty catch block)?**
    → Silently swallows errors, hides bugs, and makes production debugging nearly impossible. Always at least log with context.

### try-with-resources / Java 7+
20. **What are the requirements for a class to be used in try-with-resources?**
    → It must implement `java.lang.AutoCloseable` (or `java.io.Closeable`), exposing a `close()` method.

21. **Can you reassign a resource variable inside a try-with-resources header?**
    → No — resource variables are implicitly `final`.

22. **What is a suppressed exception?**
    → When `try-with-resources` auto-closes and `close()` throws while a primary exception is already propagating, the close-time exception is attached to the primary one via `addSuppressed()`, retrievable via `getSuppressed()`, instead of being silently lost.

23. **What are the rules for a valid multi-catch block?**
    → The exception types must share **no inheritance relationship** (not parent-child, not duplicates); the caught variable is implicitly `final`.

24. **Since Java 9, do you need to redeclare a resource inside the `try()` header?**
    → No — an existing **effectively final** variable can be referenced directly: `try (br) { ... }`.

### Exception Propagation
25. **What is exception propagation?**
    → When a method doesn't handle a raised exception, it's automatically passed up to the caller, which becomes responsible for handling it — continues up the call stack until handled or reaches `main()`/JVM.

26. **For checked exceptions, is propagation automatic like unchecked ones?**
    → No — checked exception propagation must be **explicitly declared** via `throws` at every level; unchecked exceptions propagate automatically without declaration.

### Modern Java (Architect-Level)
27. **What is JEP 358 and why does it matter?**
    → "Helpful NullPointerExceptions" (Java 14+, default since 15) — enriches NPE messages with the exact variable/method causing the null dereference, drastically improving production debuggability.

28. **Why was `Object.finalize()` deprecated, and what replaces it?**
    → Deprecated in Java 9 (removal-flagged since 18) due to unpredictable GC timing/performance issues; replaced by `java.lang.ref.Cleaner` and the `AutoCloseable`/try-with-resources pattern.

29. **How do sealed classes improve exception hierarchy design?**
    → They create a **closed, exhaustive** set of subclasses, enabling compiler-verified exhaustive handling via pattern-matching `switch` — reducing the risk of unhandled exception subtypes.

30. **How does exception handling differ in Structured Concurrency (`StructuredTaskScope`)?**
    → Child task exceptions are captured and propagated deterministically to the parent scope (`ShutdownOnFailure` + `throwIfFailed`), replacing manual `ExecutionException` unwrapping from `Future.get()`.

31. **Why doesn't a `try-catch` around `CompletableFuture.supplyAsync(...)` catch exceptions thrown inside the async task?**
    → Because the task runs on a separate thread; the exception surfaces asynchronously via the `CompletableFuture`'s internal state (`exceptionally`, `handle`, or when calling `.get()`/`.join()`), not synchronously at the submission call site.

### Rapid-Fire Cheat Sheet

| Question | Quick Answer |
|---|---|
| Root of exception hierarchy | `Throwable` |
| Two children of `Throwable` | `Exception`, `Error` |
| Checked or Unchecked: `RuntimeException` | Unchecked |
| Checked or Unchecked: `Error` | Unchecked |
| Fully checked examples | `IOException`, `InterruptedException` |
| Partially checked (only 2 in Java) | `Throwable`, `Exception` |
| Catch block order rule | Child → Parent |
| Does `finally` run if `System.exit()` is called | No |
| Does `finally` override `return` in try/catch | Yes |
| Multi-catch operator | `\|` |
| Resource interface for try-with-resources | `AutoCloseable` |
| Are try-with-resources variables final | Yes (implicitly) |
| Replacement for `finalize()` | `Cleaner` API / `AutoCloseable` |
| JEP for Helpful NPEs | JEP 358 (Java 14, default 15) |
| Keyword to manually throw | `throw` |
| Keyword to delegate to caller | `throws` |

---

*End of Study Material — Exception Handling*
