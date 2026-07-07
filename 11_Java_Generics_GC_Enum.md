# Core Java Deep Dive: Generics, Garbage Collection & Enum
### Senior Java Architect — Interview Preparation Guide
*(Consolidated from DurgaSoft Core Java study material + modernized with Java 9–21 updates)*

---

# PART 1: GENERICS

## 1.1 Introduction — Why Generics?

Generics were introduced in **Java 1.5** to solve two core problems of the pre-generics collection framework:

| Problem | Without Generics | With Generics |
|---|---|---|
| **Type Safety** | Collections accept any `Object`; wrong type added → **Runtime error** | Wrong type addition → **Compile-time error** |
| **Type Casting** | Mandatory explicit cast on retrieval | No cast needed; compiler auto-inserts safe cast |

### Case 1: Type Safety Problem (pre-1.5)

```java
ArrayList l = new ArrayList();
l.add("vijaya");
l.add("bhaskara");
l.add(new Integer(10));      // No compile error — silently allowed

String name3 = (String) l.get(2);   // Compiles fine...
// Runtime Exception:
// Exception in thread "main" java.lang.ClassCastException:
// java.lang.Integer cannot be cast to java.lang.String
```

Arrays, by contrast, **are** type-safe — `String[]` will throw a **compile-time** error (`incompatible types`) if you try to insert an `Integer`.

### Case 2: Type Casting Problem (pre-1.5)

```java
ArrayList l = new ArrayList();
l.add("vijaya");
String name1 = l.get(0);        // Compile Error: incompatible types
                                 // found: Object, required: String
String name1 = (String) l.get(0); // Mandatory cast
```

### The Generics Solution

```java
ArrayList<String> l = new ArrayList<String>();
l.add("vijaya");
l.add(10);          // Compile-time error: no suitable method add(int)
String name1 = l.get(0);   // No cast required
```

> **Architect Insight:** Generics achieve this through **compile-time type checking + type erasure**, not through runtime reification (unlike C#/Java arrays). This has deep implications discussed in §1.8.

### Modern Java Update — `var` + Diamond Operator
Since **Java 7** (diamond operator `<>`) and **Java 10** (`var`), verbosity is reduced:
```java
// Java 7+
List<String> list = new ArrayList<>();

// Java 10+ (local variable type inference — use judiciously)
var list = new ArrayList<String>();
```

---

## 1.2 Generic Classes

Until 1.4, a class handled only `Object`:

```java
class ArrayList {
    void add(Object o) { }
    Object get(int index) { return null; }
}
```

From 1.5, a **type parameter** `<T>` is introduced:

```java
class ArrayList<T> {
    void add(T t) { }
    T get(int index) { return null; }
}
```

At usage site, `T` is substituted by the actual type argument:
```java
ArrayList<String> l = new ArrayList<String>();
// Conceptually becomes:
// class ArrayList<String> {
//     void add(String s) { }
//     String get(int index) { return null; }
// }
```

### Custom Generic Class — Fully Validated Example

```java
class Gen<T> {
    private T obj;

    Gen(T obj) {
        this.obj = obj;
    }

    public void show() {
        System.out.println("The type of object is: " + obj.getClass().getName());
    }

    public T getObject() {
        return obj;
    }
}

public class GenericsDemo {
    public static void main(String[] args) {
        Gen<Integer> g1 = new Gen<>(10);
        g1.show();
        System.out.println(g1.getObject());

        Gen<String> g2 = new Gen<>("Akshay");
        g2.show();
        System.out.println(g2.getObject());

        Gen<Double> g3 = new Gen<>(10.5);
        g3.show();
        System.out.println(g3.getObject());
    }
}
/* Output:
The type of object is: java.lang.Integer
10
The type of object is: java.lang.String
Akshay
The type of object is: java.lang.Double
10.5
*/
```

Multiple type parameters are also allowed (e.g., `HashMap<K, V>`):
```java
class Pair<K, V> {
    K key; V value;
    Pair(K key, V value) { this.key = key; this.value = value; }
}
HashMap<Integer, String> h = new HashMap<>();
```

### ⚠️ Key Restriction — No Primitives as Type Arguments
```java
ArrayList<int> l = new ArrayList<int>();
// Compile Error: unexpected type, found: int, required: reference
```
**Reason:** Generics work only with reference types (via erasure to `Object`); autoboxing (`Integer`, `Double`, etc.) is required.

### Polymorphism Applies Only to the *Base Type*, Not the *Parameter Type*
```java
ArrayList<String> l1 = new ArrayList<String>();      // valid
List<String>      l2 = new ArrayList<String>();      // valid (base-type polymorphism)
Collection<String> l3 = new ArrayList<String>();     // valid

ArrayList<Object> l4 = new ArrayList<String>();
// Compile Error: incompatible types
// found: ArrayList<String>, required: ArrayList<Object>
```
> **Architect Insight:** This is precisely why Java Generics are **invariant** — `List<String>` is NOT a subtype of `List<Object>`, even though `String` IS a subtype of `Object`. This is the root cause behind the need for wildcards (`? extends`, `? super`) — see §1.5.

---

## 1.3 Bounded Types

Restrict the range of acceptable type arguments using `extends`.

```java
class Test<T> { }                      // Unbounded — any type accepted
Test<Integer> t1 = new Test<>();
Test<String>  t2 = new Test<>();
```

### Class Bound
```java
class Test<T extends Number> { }
Test<Integer> t1 = new Test<>();   // valid (Integer extends Number)
Test<String>  t2 = new Test<>();
// Compile Error: type parameter java.lang.String is not within its bound
```

### Interface Bound
```java
class Test<T extends Runnable> { }
Test<Thread> t1 = new Test<>();    // valid — Thread implements Runnable
Test<String> t2 = new Test<>();    // Compile Error
```

> **Rule:** `implements` and `super` keywords **cannot** be used in a bounded type parameter declaration — always use `extends`, regardless of whether the bound is a class or an interface.
```java
class Test<T implements Runnable> { }  // INVALID syntax
class Test<T super String> { }         // INVALID syntax
class Test<T extends Runnable> { }     // Correct — replaces "implements"
```

### Combining Multiple Bounds (Intersection Types)
```java
class Test<T extends Number & Runnable> { }                    // valid
class Test<T extends Number & Runnable & Comparable<T>> { }    // valid
class Test<T extends Number & String> { }     // INVALID — cannot extend 2 classes
class Test<T extends Runnable & Comparable<T>> { }              // valid
class Test<T extends Runnable & Number> { }   // INVALID — class must come FIRST
```
**Rule:** At most **one class**, and it must be listed **before** any interfaces.

---

## 1.4 Generic Methods & Wildcards (`?`)

### Declaring Type Parameters at Method Level
The type parameter is declared **just before the return type**:
```java
public <T> void methodOne1(T t) { }                                   // valid
public <T extends Number> void methodOne2(T t) { }                    // valid
public <T extends Number & Comparable<T>> void methodOne3(T t) { }    // valid
public <T extends Runnable & Number> void methodOne4(T t) { }
// Compile Error: interface expected here — class must precede interfaces
```

### Wildcard Semantics — The 4 Flavors

| Declaration | Accepts | Can `add()`? | Use case |
|---|---|---|---|
| `ArrayList<String> l` | Only `ArrayList<String>` | `String`, `null` | Exact type known |
| `ArrayList<?> l` | Any `ArrayList<T>` | Only `null` | **Read-only**, unknown type |
| `ArrayList<? extends X> l` | `X` or any subclass/implementer of `X` | Only `null` | **Producer** (you only read/consume `X`-typed elements) |
| `ArrayList<? super X> l` | `X` or any superclass of `X` | `X` and its subtypes, `null` | **Consumer** (you only write `X`-typed elements) |

```java
void methodOne(ArrayList<String> l) {
    l.add("A"); l.add(null);
    l.add(10);          // INVALID
}

void methodOne(ArrayList<?> l) {
    l.add(null);        // valid
    l.add("A");         // INVALID
    l.add(10);          // INVALID
}

void methodOne(ArrayList<? extends Number> l) {
    l.add(null);         // valid only
    // l.add(10);        // INVALID — compiler can't guarantee actual type
}

void methodOne(ArrayList<? super Integer> l) {
    l.add(10);           // valid — Integer or subtype allowed
    l.add(null);          // valid
}
```

> **Architect-Level Mnemonic — PECS (Producer Extends, Consumer Super)**
> Coined by Joshua Bloch (*Effective Java*):
> - If a structure only **produces** (you `get()` from it) → use `? extends T`
> - If a structure only **consumes** (you `add()`/`put()` into it) → use `? super T`
> - If it does both → use exact type `T`, no wildcard

Real-world JDK example (`Collections.copy`):
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src)
```

### Valid / Invalid Wildcard Declarations (Common Interview Trap)
```java
ArrayList<String> l1 = new ArrayList<String>();                 // valid
ArrayList<?> l2 = new ArrayList<String>();                       // valid
ArrayList<?> l3 = new ArrayList<Integer>();                      // valid
ArrayList<? extends Number> l4 = new ArrayList<Integer>();       // valid
ArrayList<? extends Number> l5 = new ArrayList<String>();
// Compile Error: incompatible types
// found: ArrayList<String>, required: ArrayList<? extends Number>

ArrayList<?> l6 = new ArrayList<? extends Number>();
// Compile Error: unexpected type — required: class or interface WITHOUT bounds
// (You cannot use a wildcard with bound on the RIGHT side of `new`)

ArrayList<?> l7 = new ArrayList<?>();
// Compile Error: unexpected type — same reason as above
```
**Rule:** A wildcard (bounded or unbounded) can appear on the **left** (declared type), but the object created via `new` must always specify a **concrete, unbounded type**.

---

## 1.5 Type Erasure — Internals Every Architect Must Know

Generics in Java are a **compile-time-only** construct implemented via **type erasure**:

- The compiler uses generic type info only for **compile-time type checking**.
- After compilation, all generic type information is **erased**; type parameters are replaced by their bound (`Object` if unbounded, or the first bound if bounded).
- The JVM has **zero knowledge of generics at runtime**.

```java
ArrayList<String> l1 = new ArrayList<String>();
ArrayList<Integer> l2 = new ArrayList<Integer>();
System.out.println(l1.getClass() == l2.getClass());   // true — both are ArrayList.class!
```

### Runtime Equivalence
```java
ArrayList l  = new ArrayList<String>();
ArrayList l2 = new ArrayList<Integer>();
ArrayList l3 = new ArrayList();
// All three are 100% identical at runtime (erased to raw ArrayList)
```

```java
import java.util.*;
class Test {
    public static void main(String[] args) {
        ArrayList l = new ArrayList<String>();
        l.add(10);
        l.add(10.5);
        l.add(true);
        System.out.println(l);   // [10, 10.5, true]
    }
}
```

### Consequence: Overloading by Erased Generic Type Is Illegal
```java
class Test {
    public void methodOne(ArrayList<String> l) { }
    public void methodOne(ArrayList<Integer> l) { }
}
/* Compile Error:
Test.java:4: name clash:
methodOne(ArrayList<String>) and methodOne(ArrayList<Integer>)
have the same erasure
*/
```
> **Architect Insight:** This is one of the most frequently asked *"gotcha"* interview questions — both signatures erase to `methodOne(ArrayList)`.

### Communication With Legacy (Non-Generic) Code
Raw types allow calling generic code from legacy code without compile errors, but risk `ClassCastException` at runtime:
```java
import java.util.*;
class Test {
    public static void main(String[] args) {
        ArrayList<String> l = new ArrayList<>();
        l.add("A");
        methodOne(l);                 // raw-type param accepts it silently
        l.add(10.5);                  // now compiles fine too! (raw type bypassed checks inside methodOne)
        System.out.println(l);        // [A, 10, 10.5, true]
    }
    public static void methodOne(ArrayList l) {   // raw type — legacy signature
        l.add(10);
        l.add(10.5);
        l.add(true);
    }
}
```
Modern compilers emit an **"unchecked call"** warning here — always treat these warnings seriously in code review; they are a common source of latent production `ClassCastException`s.

### Bridge Methods (Architect-Level Detail)
When a generic method is overridden, the compiler generates a synthetic **bridge method** to preserve polymorphism after erasure:
```java
class Node<T> {
    public void setData(T data) { }
}
class MyNode extends Node<Integer> {
    @Override
    public void setData(Integer data) { }
    // Compiler generates: public void setData(Object data) { setData((Integer) data); }
}
```
This is why `getClass().getMethods()` on such a class shows **two** `setData` methods — a very common trick question in senior interviews.

---

## 1.6 Generics + Arrays — Why You Can't Create `new T[]` or `new List<String>[]`

Arrays are **covariant and reified** (know their type at runtime); Generics are **invariant and erased**. Mixing them breaks type safety:
```java
List<String>[] arr = new List<String>[10];   // Compile Error: generic array creation
```
**Workaround:** Use `List<String>[] arr = (List<String>[]) new List[10];` with an `@SuppressWarnings("unchecked")`, or prefer `List<List<String>>`.

---

## 1.7 Generics Conclusions (from source material)

1. Generics apply **only at compile time**; the following are runtime-equivalent:
   ```java
   ArrayList l = new ArrayList<String>();
   ArrayList l = new ArrayList<Integer>();
   ArrayList l = new ArrayList();
   ```
2. `ArrayList<String> l1 = new ArrayList();` and `ArrayList<String> l2 = new ArrayList<String>();` are **equivalent** — the raw-type constructor call is allowed (with an unchecked warning) and the compile-time type still governs `l1`'s usage:
   ```java
   l1.add("A");    // valid
   l1.add(10);     // invalid (compile-time check still applies to l1 itself)
   ```

---

## 1.8 Generics — Interview Q&A (Architect Level)

**Q1. What are the two main objectives of Generics?**
A: Type-safety (compile-time checking of collection content type) and elimination of explicit type casting.

**Q2. Why is Java Generics implemented via type erasure instead of reification (like C#)?**
A: Backward compatibility — Generics were retrofitted onto an existing collections framework and JVM in Java 5 without breaking binary compatibility with pre-5 bytecode/class files.

**Q3. Is `List<String>` a subtype of `List<Object>`? Why/why not?**
A: No. Generics are **invariant**. If it were allowed, you could add an `Integer` to a `List<Object>` reference that's actually pointing to a `List<String>`, breaking type safety at runtime with no compiler warning.

**Q4. What is PECS?**
A: "Producer Extends, Consumer Super" — a guideline for choosing between `? extends T` (read-only sources) and `? super T` (write-only sinks) when designing generic APIs.

**Q5. Can you overload two methods that differ only in generic type parameter, e.g. `foo(List<String>)` vs `foo(List<Integer>)`?**
A: No — both erase to `foo(List)`, causing a "name clash" compile error.

**Q6. Can a static method/field use the class's type parameter?**
A: No. Static members belong to the class, not to an instance, but type parameters are resolved per-instance. However, a **static generic method** can declare its **own** independent type parameter: `static <T> void foo(T t)`.

**Q7. What is a bridge method?**
A: A synthetic method the compiler generates when a generic method is overridden with a more specific type, to preserve polymorphism after erasure.

**Q8. Why can't we create `new T[]` inside a generic class?**
A: Because arrays are reified (track element type at runtime) while `T` is erased to `Object` (or its bound) at runtime — the JVM cannot verify array-store type-safety.

**Q9. What does `Test<T extends Number & Comparable<T>>` mean, and what are the ordering rules?**
A: `T` must be a subtype of `Number` **and** implement `Comparable<T>`. Rule: exactly one class bound is allowed and it must appear first; interfaces follow, separated by `&`.

**Q10. What happens when you mix raw types and generic types (heap pollution)?**
A: You bypass compile-time checks, risking a `ClassCastException` at a point far removed from the actual insertion — this is called **heap pollution**, and is exactly why varargs + generics (`@SafeVarargs`) warnings exist.

---

# PART 2: GARBAGE COLLECTION (GC)

## 2.1 Introduction

- In languages like **C++**, the programmer is responsible for both **creating** and **destroying** objects — negligence in `free()`/`delete` leads to memory leaks or crashes.
- In **Java**, the programmer is responsible only for **object creation**; destruction is handled automatically by a background daemon thread: the **Garbage Collector (GC)**.
- **Primary objective of GC:** identify and destroy objects that are no longer reachable ("useless objects"), reclaiming heap memory.

> **Modern Update:** GC in Java is not a single fixed algorithm — since Java 9, **G1 (Garbage First)** is the **default** collector. Java offers pluggable collectors: **Serial, Parallel, CMS (deprecated in 9, removed in 14), G1, ZGC, Shenandoah, Epsilon** (a "no-op" GC for benchmarking, since Java 11). See §2.6.

---

## 2.2 Ways to Make an Object Eligible for GC

An object becomes eligible for GC **if and only if it has zero live references** (barring resurrection edge cases in `finalize()`).

### 1. Nullifying the Reference Variable
```java
Student s1 = new Student();
Student s2 = new Student();
s1 = null;   // 1 object eligible for GC
s2 = null;   // 2 objects eligible for GC
```

### 2. Reassigning the Reference Variable
```java
Student s1 = new Student();
Student s2 = new Student();
s2 = s1;     // the object originally referenced by s2 is now eligible for GC
```

### 3. Objects Created Inside a Method
Local objects become eligible once the method completes (stack frame is popped), **unless** the reference escapes (is returned, or assigned to a wider-scoped variable).

```java
class Test {
    public static void main(String[] args) {
        methodOne();
        // Both Student objects created inside methodOne() are now eligible for GC
    }
    public static void methodOne() {
        Student s1 = new Student();
        Student s2 = new Student();
    }
}
```

```java
class Test {
    public static void main(String[] args) {
        Student s = methodOne();
        // Only 1 object eligible for GC (s2); s1 escaped via return and is still referenced by 's'
    }
    public static Student methodOne() {
        Student s1 = new Student();
        Student s2 = new Student();
        return s1;
    }
}
```

```java
class Test {
    static Student s1;    // static field — outlives the method call
    public static void main(String[] args) {
        methodOne();
        // Only s2's object is eligible; s1's object is retained via the static reference
    }
    public static void methodOne() {
        s1 = new Student();
        Student s2 = new Student();
    }
}
```

### 4. Island of Isolation
A group of objects referencing **only each other** (circular references) with **no external reference** are all simultaneously eligible for GC — modern **tracing/mark-and-sweep** collectors handle this correctly (unlike naive reference-counting GCs, e.g., old CPython/COM).

```java
class Test {
    Test i;
    public static void main(String[] args) {
        Test t1 = new Test();
        Test t2 = new Test();
        Test t3 = new Test();

        t1.i = t2;
        t2.i = t3;
        t3.i = t1;     // circular chain: t1 -> t2 -> t3 -> t1

        t1 = null;
        t2 = null;
        t3 = null;
        // All 3 objects form an "Island of Isolation" — all 3 eligible for GC
        // even though each still has an internal reference from within the island.
    }
}
```

> **Note:** Even with a reference, an object CAN be eligible for GC if that reference itself is unreachable from any GC root (thread stacks, static fields, JNI references, etc.). Conversely, an object with **zero** references is **always** eligible.

---

## 2.3 Requesting JVM to Run GC

GC eligibility ≠ immediate destruction. The JVM decides **when** to actually run GC (vendor/implementation-dependent). We can only *request*, never *force*, a GC run.

### Via `System` class
```java
System.gc();   // static method — recommended way
```

### Via `Runtime` class
`Runtime` is a **Singleton** (private constructor); obtain the instance via the factory method `getRuntime()`.

```java
import java.util.Date;
class RuntimeDemo {
    public static void main(String[] args) {
        Runtime r = Runtime.getRuntime();
        System.out.println("Total memory: " + r.totalMemory());
        System.out.println("Free memory before: " + r.freeMemory());

        for (int i = 0; i < 10000; i++) {
            Date d = new Date();
            d = null;
        }

        System.out.println("Free memory after loop (before gc): " + r.freeMemory());
        r.gc();
        System.out.println("Free memory after gc: " + r.freeMemory());
    }
}
```

### Which of the Following Are Valid?
```java
System.gc();                  // valid (static method)
Runtime.gc();                 // INVALID — gc() is an instance method in Runtime
(new Runtime()).gc();         // INVALID — constructor is private (Singleton)
Runtime.getRuntime().gc();    // valid
```

> **Notes:**
> - `System.gc()` is preferred over `Runtime.getRuntime().gc()` for readability, but internally `System.gc()` delegates to `Runtime.getRuntime().gc()`.
> - Java provides **no API** to determine an object's exact size or memory address (unlike C/C++'s `sizeof`/pointers) — by design, for platform independence and safety.

---

## 2.4 Finalization — `finalize()`

- Just before destroying an object, GC (historically) invoked `finalize()` on it to allow cleanup.
- If the class overrides `finalize()`, that version executes; otherwise `Object`'s no-op version runs.
```java
protected void finalize() throws Throwable
```

### ⚠️ CRITICAL MODERN UPDATE — `finalize()` is DEPRECATED

| Java Version | Status of `Object.finalize()` |
|---|---|
| Java 8 and earlier | Standard mechanism, widely used (though always discouraged by experts) |
| **Java 9** | `Object.finalize()` marked **`@Deprecated`** |
| **Java 18** | `Object.finalize()` marked **`@Deprecated(since="9", forRemoval=true)`** |
| Future release | Slated for **removal entirely** |

**Why deprecated?**
- Unpredictable timing, no guarantee of execution at all (JVM shutdown can skip it).
- Performance overhead — finalizable objects require an extra GC cycle (they're queued, not collected immediately).
- Can **resurrect** objects (`this` escape in `finalize()`), causing memory leaks and confusing lifecycle bugs.
- Exceptions thrown from GC-invoked `finalize()` are **silently swallowed**, hiding bugs.

**Recommended Modern Replacements:**
1. **`try-with-resources`** + `AutoCloseable`/`Closeable` (deterministic, since Java 7) — the primary replacement.
2. **`java.lang.ref.Cleaner`** (Java 9+) — a safer, non-`finalize()`-based hook for post-mortem cleanup, using `PhantomReference` internally:
```java
import java.lang.ref.Cleaner;

class Resource implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    private static class State implements Runnable {
        public void run() {
            System.out.println("Cleaning up native resource...");
        }
    }

    Resource() {
        this.cleanable = cleaner.register(this, new State());
    }

    @Override
    public void close() {
        cleanable.clean();
    }
}
// Usage:
try (Resource r = new Resource()) {
    // use r
} // close() called deterministically here
```

The legacy `finalize()` examples below remain **valid for interview/legacy-code discussion** but should NEVER be used in new production code.

### Case 1: GC Calls `finalize()` Just Before Destroying the Object
```java
class Test {
    public static void main(String[] args) {
        String s = new String("bhaskar");
        Test t = new Test();
        t = null;
        System.gc();
        System.out.println("End of main.");
    }
    public void finalize() {
        System.out.println("finalize() method is executed");
    }
}
/* Output:
finalize() method is executed
End of main.
*/
```
(If `s = null` instead of `t = null`, `String`'s own empty `finalize()` runs — not `Test`'s.)

### Case 2: `finalize()` Can Be Called Explicitly (Just a Normal Method Call)
```java
class Test {
    public static void main(String[] args) {
        Test t = new Test();
        t.finalize();
        t.finalize();
        t = null;
        System.gc();
        System.out.println("End of main.");
    }
    public void finalize() {
        System.out.println("finalize() method called");
    }
}
/* Output:
finalize() method called.   (explicit call #1)
finalize() method called.   (explicit call #2)
finalize() method called.   (GC's own call)
End of main.
*/
```
Calling `finalize()` explicitly does **NOT** destroy the object — it behaves exactly like any other method invocation.

> **Servlet analogy:** Similarly, calling `destroy()` explicitly from `init()`/`service()` executes it as a normal method call — the Servlet container's lifecycle is unaffected.

### Case 3: Exception Handling Inside `finalize()`
- If the **programmer** calls `finalize()` explicitly and an **uncaught exception** occurs → program terminates **abnormally**.
- If **GC** calls `finalize()` and an uncaught exception occurs → JVM **silently ignores** it, and the program terminates **normally**.

```java
class Test {
    public static void main(String[] args) {
        Test t = new Test();
        // t.finalize();   // if uncommented: ArithmeticException propagates -> abnormal termination
        t = null;
        System.gc();
        System.out.println("End of main.");
    }
    public void finalize() {
        System.out.println("finalize() method called");
        System.out.println(10 / 0);   // ArithmeticException
    }
}
```

### Case 4: GC Calls `finalize()` on Any Given Object Only Once
```java
class FinalizeDemo {
    static FinalizeDemo s;
    public static void main(String[] args) throws Exception {
        FinalizeDemo f = new FinalizeDemo();
        System.out.println(f.hashCode());
        f = null;
        System.gc();
        Thread.sleep(5000);
        System.out.println(s.hashCode());   // 'this' was resurrected inside finalize()
        s = null;
        System.gc();
        Thread.sleep(5000);
        System.out.println("end of main method");
        // Even though the object is eligible AGAIN, finalize() will NOT run a 2nd time —
        // GC destroys it directly without invoking finalize() again.
    }
    public void finalize() {
        System.out.println("finalize method called");
        s = this;   // "resurrecting" the object — an anti-pattern, shown for interview awareness only
    }
}
/* Output:
<hashcode>
finalize method called
<same hashcode>
end of main method
*/
```

---

## 2.5 Memory Leaks in Java

- An object that is **no longer used** by the application but **still has a reachable reference** (hence NOT eligible for GC) is called a **memory leak**.
- GC **cannot** help here — it only reclaims *unreachable* objects. Memory leaks in Java are almost always caused by **unintentionally retained references** (e.g., static collections, unclosed listeners, ThreadLocal misuse, inner class holding outer reference).
- Left unresolved, memory leaks eventually cause `OutOfMemoryError`.

**Common Modern Culprits (Architect Awareness):**
- Unbounded `static` caches/collections.
- Listener/callback registrations without deregistration.
- `ThreadLocal` variables not removed in thread-pool environments.
- Inner (non-static) classes holding implicit references to the outer instance.
- Improperly closed resources (streams, connections) — mitigated by `try-with-resources`.

**Monitoring / Profiling Tools:**
- Legacy: HPJmeter, HP OVO, IBM Tivoli, JProbe, Patrol.
- **Modern (recommended today):** `VisualVM`, `JConsole`, **Java Flight Recorder (JFR)** + **JDK Mission Control (JMC)** (production-grade, low-overhead, built into OpenJDK since 8u/11+), Eclipse MAT (Memory Analyzer Tool), `jcmd`, `jmap`, async-profiler.

---

## 2.6 GC Algorithms — Modern JVM Landscape (Architect-Level Update)

The source material notes (correctly) that GC behavior is **vendor/JVM dependent** and we cannot rely on:
1. The exact algorithm used.
2. Exact timing of a GC run.
3. Order of eligible-object identification.
4. Order of destruction.
5. Whether ALL eligible objects get collected in one pass.

Historically most GCs used **Mark-and-Sweep** (and generational variants: Mark-Sweep-Compact). Here's how this has evolved:

| Collector | JDK Introduced | Status | Best For |
|---|---|---|---|
| Serial GC | Legacy | Available | Small heaps, single-threaded apps, client-mode |
| Parallel GC ("Throughput collector") | Legacy | Available | Batch jobs prioritizing throughput over pause time |
| **CMS** (Concurrent Mark Sweep) | Legacy | **Deprecated (Java 9), Removed (Java 14)** | (Superseded by G1) |
| **G1 (Garbage First)** | Java 7 (experimental), | **Default since Java 9** | General-purpose, balances throughput & low pause time, heaps up to ~large multi-GB |
| **ZGC** | Java 11 (experimental) | **Production-ready since Java 15**; generational ZGC since Java 21 | Ultra-low latency (<1ms pauses) even on multi-TB heaps |
| **Shenandoah** | Java 12 | Production-ready | Low-pause concurrent collector (Red Hat-driven), similar goals to ZGC |
| **Epsilon** | Java 11 | Experimental/no-op | Performance testing / memory-pressure testing — allocates but NEVER collects |

**Interview-relevant generational heap model (still foundational, used by G1/Parallel/CMS/Serial):**
```
Heap
 ├── Young Generation
 │     ├── Eden Space
 │     └── Survivor Spaces (S0, S1)
 └── Old (Tenured) Generation
Metaspace (off-heap, replaced PermGen since Java 8)
```
- New objects → Eden. Survive a minor GC → moved to Survivor space, age incremented.
- After surviving enough minor GCs (tenuring threshold) → promoted to Old Gen.
- Minor GC = collects Young Gen only (fast, frequent). Major/Full GC = collects Old Gen (or entire heap) — slower, less frequent.
- **PermGen removed in Java 8**, replaced by **Metaspace**, which grows dynamically in native (off-heap) memory, eliminating the classic `java.lang.OutOfMemoryError: PermGen space`.

**Choosing/Tuning example (JVM flags):**
```
-XX:+UseG1GC          # Default in modern JDKs — usually no flag needed
-XX:+UseZGC           # Ultra-low-latency
-XX:+UseShenandoahGC  # Low-pause alternative (OpenJDK builds with Shenandoah support)
-Xms4g -Xmx4g         # Heap sizing
```

---

## 2.7 GC & Finalization — Interview Q&A (Architect Level)

**Q1. When exactly does an object become eligible for GC?**
A: When it has zero reachable references from any GC root (local variables on live thread stacks, active static fields, JNI references, etc.).

**Q2. Can a `System.gc()` call guarantee garbage collection?**
A: No — it's only a *request/hint* to the JVM; the JVM may ignore it entirely. In fact, `-XX:+DisableExplicitGC` can make the JVM ignore such calls altogether.

**Q3. Why is `finalize()` deprecated, and what should replace it?**
A: It's unreliable (no execution guarantee), slow (extra GC cycle for finalizable objects), can resurrect objects, and silently swallows exceptions. Replace with `try-with-resources`/`AutoCloseable` for deterministic cleanup, or `java.lang.ref.Cleaner` for last-resort native-resource cleanup.

**Q4. What is "Island of Isolation"? Why does reference-counting GC fail to detect it, while mark-and-sweep succeeds?**
A: A set of mutually-referencing objects unreachable from any GC root. Reference counting only tracks incoming-reference counts per object, which never drop to zero within an island (cyclical references keep the count ≥1), so it never collects them — a classic reference-counting weakness. Mark-and-sweep (tracing) instead starts from GC roots and marks all reachable objects; anything unmarked (including full islands) is swept — thus it correctly identifies isolated islands.

**Q5. What replaced PermGen, and why?**
A: **Metaspace** (Java 8+). PermGen had a fixed max size (`-XX:MaxPermSize`) causing frequent `OutOfMemoryError: PermGen space`, especially with heavy classloading (app servers, OSGi). Metaspace uses native memory and grows dynamically (bounded optionally via `-XX:MaxMetaspaceSize`).

**Q6. What's the difference between Minor GC, Major GC, and Full GC?**
A: Minor GC collects only the Young Generation (fast, frequent). Major/Full GC collects the Old Generation (or the entire heap including Young + Old + Metaspace), which is slower and causes longer pauses — this is a key latency-tuning target.

**Q7. Name low-pause collectors suited for large heaps in real-time-sensitive systems.**
A: **ZGC** and **Shenandoah** — both aim for sub-millisecond/very-low pause times regardless of heap size by doing most work concurrently with application threads.

**Q8. What is a memory leak in a garbage-collected language like Java?**
A: An object that is functionally "dead" (no longer needed by the application) but still holds a live, reachable reference — so GC (correctly) does NOT collect it, leading to unbounded memory growth and eventual `OutOfMemoryError`.

**Q9. What is the "Stop-The-World" (STW) pause?**
A: A phase during GC where all application threads are suspended so the collector can safely traverse/modify the heap (e.g., during the "mark" or "compact" phases). Modern collectors (G1, ZGC, Shenandoah) aim to minimize STW duration by doing most work concurrently.

**Q10. Difference between `WeakReference`, `SoftReference`, and `PhantomReference`?**
A (Architect bonus topic tightly related to GC):
- `SoftReference` — cleared only when JVM is low on memory (good for memory-sensitive caches).
- `WeakReference` — cleared at the **next** GC cycle regardless of memory pressure (used in `WeakHashMap`, avoiding classloader leaks).
- `PhantomReference` — object already finalized; `get()` always returns `null`; used with `ReferenceQueue` for pre-reclamation cleanup notification — the basis of `java.lang.ref.Cleaner`.

---

# PART 3: ENUM

## 3.1 Introduction

`enum` (introduced in **Java 1.5**) defines a **group of named constants**.

```java
enum Month {
    JAN, FEB, MAR /* ... */, DEC;   // trailing semicolon optional if no other members
}

enum Beer {
    KF, KO, RC, FO;
}
```

Java's `enum` is significantly more powerful than C/C++ enums (which are just named integers) — Java enums are **full-fledged classes**.

---

## 3.2 Internal Implementation of Enum

- Internally, every `enum` is implemented as a **class** that implicitly **extends `java.lang.Enum`**.
- Every enum constant is a `public static final` reference variable pointing to an instance of that enum type.

```java
enum Beer {
    KF, KO;
}

// Conceptually compiles to:
final class Beer extends java.lang.Enum<Beer> {
    public static final Beer KF = new Beer();
    public static final Beer KO = new Beer();
}
```

---

## 3.3 Declaration and Usage

```java
enum Beer {
    KF, KO, RC, FO;   // semicolon optional here
}

class Test {
    public static void main(String[] args) {
        Beer b1 = Beer.KF;
        System.out.println(b1);   // KF
    }
}
```

**Notes:**
- Every enum constant is implicitly `static` — accessed via `EnumName.CONSTANT`.
- `Enum` implicitly overrides `toString()` to return the constant's declared name.

---

## 3.4 Enum vs `switch` Statement

| Java Version | Allowed `switch` argument types |
|---|---|
| Up to 1.4 | `byte`, `short`, `char`, `int` |
| **1.5+** | + corresponding wrapper classes + **`enum`** |
| **1.7+** | + `String` |
| **14 (preview)/16 (standard)+** | **Switch expressions** (`->` syntax, `yield`) |
| **21+** | **Pattern matching for switch** (record patterns, `case null`, guarded patterns) |

```java
enum Beer { KF, KO, RC, FO; }

class Test {
    public static void main(String[] args) {
        Beer b1 = Beer.RC;
        switch (b1) {
            case KF: System.out.println("it is childrens brand"); break;
            case KO: System.out.println("it is too lite"); break;
            case RC: System.out.println("it is too hot"); break;
            case FO: System.out.println("buy one get one"); break;
            default: System.out.println("other brands are not good");
        }
    }
}
// Output: It is too hot
```

**Rule:** Case labels for an enum-typed `switch` must be the **unqualified constant name** (NOT `Beer.KF`) and must be valid constants of that enum, else compile error:
```java
switch (b1) {
    case KF:
    case RC:
    case KALYANI:   // Compile Error: unqualified enumeration constant name required
}
```

### Modern Java Update — Switch Expressions & Pattern Matching (Java 14/17/21)
```java
// Java 14+ switch expression (arrow syntax, no fall-through, exhaustive)
enum Beer { KF, KO, RC, FO }

String describe(Beer b) {
    return switch (b) {
        case KF -> "it is childrens brand";
        case KO -> "it is too lite";
        case RC -> "it is too hot";
        case FO -> "buy one get one";
    };   // no 'default' needed if all enum constants are covered — compiler verifies exhaustiveness!
}
```
> **Architect Insight:** Exhaustiveness checking on enum switch expressions is a huge win — adding a new enum constant later will cause a **compile error** at every switch expression that doesn't handle it (when no `default` is present), catching bugs early. This pairs powerfully with **sealed classes** (Java 17) for exhaustive pattern matching across a closed type hierarchy.

---

## 3.5 Allowed Modifiers for `enum`

### Declared Outside a Class
```
public, default (package-private), strictfp
```

### Declared Inside a Class
```
public, default, protected, private, static, strictfp
```

```java
enum X { }
class X { enum Y { } }         // valid — enum nested inside a class
class X {
    public void methodOne() {
        enum X { }              // Compile Error: enum types must not be local
    }
}
```
**Rule:** `enum` can be declared **top-level** or as a **member of a class**, but **never locally inside a method**.
(Note: **Java 16** introduced local `record`/enum/interface support for some constructs, but a **local `enum` inside a method body is still NOT allowed** as of current Java versions — this restriction from the source material remains accurate today.)

---

## 3.6 Enum vs Inheritance

- Every `enum` implicitly extends `java.lang.Enum` — so it **cannot extend any other class** (single inheritance already consumed).
- Every `enum` is implicitly `final` — so **no enum can be sub-classed** (except the special "constant-specific class body" mechanism, §3.9).
- **Conclusion:** `extends` keyword can **never** be used explicitly with `enum` — neither `enum X extends SomeClass` nor `enum Y extends AnotherEnum`.
- However, an `enum` **CAN implement any number of interfaces**.

```java
enum X { }
enum X extends Enum { }        // INVALID — cannot explicitly extend Enum (implicit already)
class X { }
enum Y extends X { }           // INVALID — enum types are not extensible / cannot inherit from final X

interface X { }
enum Y implements X { }        // VALID
```

---

## 3.7 `java.lang.Enum` Class

- Every enum is a **direct child class of `java.lang.Enum<E>`**, which is itself an **abstract class** and a direct child of `Object`.
- `Enum<E>` implements `Serializable` and `Comparable<E>`.

### `values()` Method
Implicitly generated static method returning an array of all constants (in declaration order):
```java
Beer[] b = Beer.values();
```

### `ordinal()` Method
Returns the **zero-based index** (declaration order) of the constant:
```java
public final int ordinal();
```

```java
enum Beer { KF, KO, RC, FO; }

class Test {
    public static void main(String[] args) {
        Beer[] b = Beer.values();
        for (Beer b1 : b) {                       // enhanced for-each loop
            System.out.println(b1 + "......." + b1.ordinal());
        }
    }
}
/* Output:
KF.......0
KO.......1
RC.......2
FO.......3
*/
```

### Other Key `Enum`/Related Methods (Architect Additions)
```java
public final String name();                      // exact declared constant name (final, cannot be overridden)
public static <T extends Enum<T>> T valueOf(Class<T> enumType, String name);
public final int compareTo(E o);                  // compares by ordinal — natural ordering
```
```java
Beer b = Enum.valueOf(Beer.class, "KF");   // == Beer.KF
Beer b2 = Beer.valueOf("KF");              // compiler-generated per-enum shortcut
```
> **Interview tip:** `name()` is `final` (always returns the literal declared identifier); `toString()` is **overridable** — a common pattern is overriding `toString()` for display purposes while `name()` remains stable for lookups/persistence/serialization keys.

---

## 3.8 Speciality of Java Enum

Unlike C/C++ enums (plain integers), Java enums can contain: **fields, constructors, instance/static methods, and even a `main()` method** — an enum can be run directly from the command line!

```java
enum Fish {
    GOLD, APOLO, STAR;
    public static void main(String[] args) {
        System.out.println("enum main() method called");
    }
}
// D:\>java Fish
// enum main() method called
```

### Structural Rule
If an enum declares **any extra member** (fields, methods, constructors) beyond constants, then:
1. The constant list **must appear first**, and
2. The constant list **must end with a semicolon** (mandatory in this case — unlike the simple case where `;` is optional).
3. The enum must declare **at least one constant** (an empty-bodied enum with zero constants is still valid on its own, but can't mix "zero constants" with "extra members" cleanly per source rules).

```java
enum X {
    A, B, C;                 // semicolon MANDATORY here
    public void methodOne() { }
}                             // valid

enum X {
    public void methodOne() { }
    A, B, C;                 // INVALID — constants must be declared FIRST
}
```

---

## 3.9 Enum vs Constructor

- `enum` **can** declare constructors — but they are implicitly `private` (or package-private) — **never `public`/`protected`** — because you cannot instantiate an enum externally.
- Every enum constant represents a **`static` object** of the enum class, so **all constants (and hence their constructors) are created automatically at class-loading time**, once, in declaration order.

```java
enum Beer {
    KF, KO, RC, FO;
    Beer() {
        System.out.println("Constructor called.");
    }
}
class Test {
    public static void main(String[] args) {
        Beer b = Beer.KF;              // <-- triggers class loading
        System.out.println("hello.");
    }
}
/* Output:
Constructor called.
Constructor called.
Constructor called.
Constructor called.
hello.
*/
```
(If the reference to `Beer.KF` is removed entirely, the `Beer` class may never load, and only `"hello."` prints — constructors run at **class-loading time**, triggered by first active use.)

### ⚠️ You Cannot Instantiate an Enum Explicitly with `new`
```java
enum Beer {
    KF, KO, RC, FO;
    Beer() { System.out.println("constructor called"); }
}
class Test {
    public static void main(String[] args) {
        Beer b = new Beer();
        // Compile Error: enum types may not be instantiated
    }
}
```

### Constant-Specific Constructor Arguments
```java
enum Beer {
    KF(100), KO(70), RC(65), FO(90), KALYANI;   // KALYANI uses the no-arg constructor

    int price;

    Beer(int price) {
        this.price = price;
    }
    Beer() {
        this.price = 125;      // default for constants without explicit args
    }
    public int getPrice() {
        return price;
    }
}
class Test {
    public static void main(String[] args) {
        for (Beer b1 : Beer.values()) {
            System.out.println(b1 + "......." + b1.getPrice());
        }
    }
}
/* Output:
KF.......100
KO.......70
RC.......65
FO.......90
KALYANI.......125
*/
```

---

## 3.10 Instance / Static Methods & Constant-Specific Class Bodies

- Enums can have **both static and instance methods**, but **NOT abstract methods declared directly at the enum level unless every constant provides a body** (via constant-specific class bodies).

### Case: Uniform `info()` Method Shared by All Constants
```java
enum Color {
    BLUE, RED, GREEN;
    public void info() {
        System.out.println("Universal color");
    }
}
```

### Case: Constant-Specific Class Body (Overriding Per-Constant Behavior)
This is the enum equivalent of an **anonymous subclass**, letting individual constants override behavior:
```java
enum Color {
    BLUE,
    RED {
        @Override
        public void info() {
            System.out.println("Dangerous color");
        }
    },
    GREEN;

    public void info() {
        System.out.println("Universal color");
    }
}
class Test {
    public static void main(String[] args) {
        for (Color c1 : Color.values()) {
            c1.info();
        }
    }
}
/* Output:
Universal color
Dangerous color
Universal color
*/
```
> **Architect Insight:** This is precisely how **true polymorphic enums** are built (the classic "Strategy pattern via enum" — e.g., JDK's own `java.util.concurrent.TimeUnit` uses this pattern extensively for per-unit conversion logic). This is also the ONLY legitimate way enums achieve "subclassing" — each constant with a body is compiled to an anonymous subclass of the enum, but the enum itself remains sealed/final to the outside world.

### Valid Expressions on Enum Constants
```java
Beer.KF == Beer.RC;                          // false — reference equality (safe; enums are singletons)
Beer.KF.equals(Beer.RC);                     // false
Beer.KF < Beer.RC;                            // INVALID — no relational operators on enum
Beer.KF.ordinal() < Beer.RC.ordinal();        // valid — true
Beer.KF.compareTo(Beer.RC);                   // valid, modern equivalent — negative if KF before RC
```
> **Best Practice:** Always use `==` (not `.equals()`) to compare enum constants — since each constant is a unique singleton instance, `==` is both safe and more efficient. Static analyzers/IDEs typically flag `.equals()` on enums as unnecessary.

### Static Import With Enums
```java
package pack1;
public enum Fish { STAR, GUPPY; }
```
```java
package pack2;
import static pack1.Fish.STAR;     // valid — static import for direct unqualified access
// import pack1.Fish;               // valid if you want to refer to it as Fish.STAR
// import pack1.*;                  // INVALID for accessing STAR directly (only imports the type name)
class A {
    public static void main(String[] args) {
        System.out.println(STAR);
    }
}
```
**Rule:** Normal `import` brings in the **class/enum name**; `import static` is required to access **static members** (including enum constants) **without qualification**.

---

## 3.11 EnumSet & EnumMap (Modern Additions Every Architect Should Know)

Since Java 5, the Collections Framework provides **highly optimized** Set/Map implementations specifically for enum keys — internally backed by **bit-vectors**, making them extremely fast and memory-efficient compared to `HashSet`/`HashMap`.

```java
import java.util.EnumSet;
import java.util.EnumMap;

enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

EnumSet<Day> weekend = EnumSet.of(Day.SAT, Day.SUN);
EnumSet<Day> weekdays = EnumSet.complementOf(weekend);
EnumSet<Day> range = EnumSet.range(Day.MON, Day.FRI);

EnumMap<Day, String> schedule = new EnumMap<>(Day.class);
schedule.put(Day.MON, "Sprint Planning");
schedule.put(Day.FRI, "Retro");
```
> **Architect Insight:** Always prefer `EnumSet`/`EnumMap` over `HashSet<Enum>`/`HashMap<Enum,...>` in performance-sensitive code — O(1) bitwise operations, guaranteed iteration order (declaration order), and no boxing/hashing overhead.

---

## 3.12 Enum as a Singleton (Effective Java — Joshua Bloch's Recommended Pattern)

Because enum constants are guaranteed by the JVM to be instantiated exactly **once**, are inherently **serialization-safe** (no risk of creating duplicate instances via deserialization, unlike classic Singleton), and are **thread-safe** by construction (class loading is synchronized by the JVM), the **best way to implement a Singleton in modern Java** is via a single-element enum:

```java
public enum ConfigurationManager {
    INSTANCE;

    private final Map<String, String> settings = new ConcurrentHashMap<>();

    public void set(String key, String value) { settings.put(key, value); }
    public String get(String key) { return settings.get(key); }
}

// Usage:
ConfigurationManager.INSTANCE.set("env", "production");
```

---

## 3.13 `enum` vs `Enum` vs `Enumeration` (Classic Trick Question)

| Term | What it is | Package |
|---|---|---|
| `enum` | A **keyword** used to define a group of named constants | Language keyword |
| `Enum` | An **abstract class**, the implicit base class of every user-defined enum | `java.lang` |
| `Enumeration` | A legacy **interface** used to iterate over elements of legacy Collections (`Vector`, `Hashtable`) one at a time | `java.util` |

> **Modern Note:** `Enumeration` is a legacy pre-Collections-Framework interface (predates even `Iterator`). In modern code, prefer `Iterator`/`Iterable` (enhanced for-loop) — `Enumeration` survives today mainly for backward compatibility with `Vector`, `Hashtable`, and legacy JDK plumbing (e.g., `ZipFile.entries()`).

---

## 3.14 Enum — Interview Q&A (Architect Level)

**Q1. What class does every enum implicitly extend, and what does it implement?**
A: `java.lang.Enum<E>` (abstract class) which implements `Serializable` and `Comparable<E>`.

**Q2. Why can't an enum extend another class?**
A: Java has single inheritance for classes, and the enum already implicitly extends `java.lang.Enum` — there's no inheritance slot left. It CAN, however, implement any number of interfaces.

**Q3. Why is enum considered the best way to implement Singleton in Java (Effective Java, Item 3)?**
A: The JVM guarantees enum constants are instantiated exactly once, in a thread-safe manner (guarded by class-loading semantics), and enums are inherently protected against breaking Singleton via reflection (a private-constructor Singleton can still be broken via `setAccessible(true)`, but enum constructors cannot be invoked reflectively) and against breaking it via **deserialization** (the default `readResolve()` behavior for enums returns the existing singleton instance rather than creating a new one).

**Q4. Can an enum have abstract methods?**
A: Yes, indirectly — declare the method without a body at the enum level, and every single constant MUST supply its own class-body implementation (constant-specific class body), similar to abstract method + anonymous subclasses.

**Q5. What's the difference between `name()` and `toString()` on an enum constant?**
A: `name()` is `final` and always returns the exact declared identifier — safe for persistence/lookup. `toString()` is overridable and defaults to calling `name()`, but developers commonly override it for display purposes (e.g., `"Monday"` instead of `MON`). **Never use `toString()` output for `valueOf()` lookups if it's been overridden** — use `name()` instead.

**Q6. Why should you use `EnumMap`/`EnumSet` instead of `HashMap`/`HashSet` for enum keys?**
A: They're backed by bit-vector arrays indexed by `ordinal()`, giving O(1) performance without hashing overhead, guaranteed enum-declaration-order iteration, and lower memory footprint.

**Q7. Can enum constructors be public?**
A: No — they are implicitly `private` (or package-private in older bytecode representations); you cannot mark them `public`/`protected`, since external instantiation via `new` is always forbidden by the compiler ("enum types may not be instantiated").

**Q8. How does `switch` on enum achieve exhaustiveness checking in modern Java (14+/17+)?**
A: With **switch expressions**, if every enum constant is covered by a `case` and no `default` is present, the compiler verifies exhaustiveness at compile time — if a new constant is later added without updating the switch, compilation FAILS, surfacing the gap immediately rather than silently falling through to a missing `default` at runtime.

**Q9. Is `ordinal()` safe to use for business logic (e.g., persistence, serialization order)?**
A: No — strongly discouraged. `ordinal()` reflects **declaration order**, which is fragile: reordering, inserting, or removing constants silently shifts ordinals for all subsequent constants, silently corrupting persisted data or comparisons. Prefer an explicit field (e.g., constant-specific constructor argument) for any externally-persisted or business-meaningful value.

**Q10. Can you have a local enum declared inside a method body?**
A: No — `enum` types must not be local; they can only be top-level or nested inside a class (as a static member).

**Q11. What happens if GC-eligible enum constants exist — can enum constants ever be garbage collected?**
A: No — enum constants are `public static final` fields of a loaded class; they live for the lifetime of the classloader that loaded the enum class (effectively forever in most application lifecycles), similar to any other static field.

---

# APPENDIX: QUICK-REFERENCE CHEAT SHEET

## Generics Cheat Sheet
| Concept | Rule |
|---|---|
| Objective | Type-safety + eliminate explicit casting |
| Type parameter | `<T>` on class or `<T> returnType method(...)` on method (declared before return type) |
| Bound | `<T extends X>` — for interfaces too, always use `extends`, never `implements`/`super` |
| Multiple bounds | Class first, then interfaces: `<T extends A & I1 & I2>` |
| Wildcard `?` | Unknown type, read-only (`add()` only accepts `null`) |
| `? extends T` | Producer — safe to read as `T`, cannot safely add (except `null`) |
| `? super T` | Consumer — safe to add `T` or subtype, reading gives `Object` |
| Polymorphism | Applies to base type (`List<String> l = new ArrayList<>()`), NOT to parameter type (`List<Object> ≠ List<String>`) |
| Primitives | Not allowed as type arguments — use wrapper classes |
| Runtime | Erased — all generic type info gone after compilation (type erasure) |
| Overloading | Cannot overload methods differing only by generic parameter (same erasure → name clash) |

## Garbage Collection Cheat Sheet
| Concept | Rule |
|---|---|
| GC responsibility | Destruction of unreachable objects only; creation is always programmer's job |
| Eligibility | Nullify ref / reassign ref / method-local scope end / island of isolation |
| Request GC | `System.gc()` or `Runtime.getRuntime().gc()` — no guarantee it runs |
| `finalize()` | **Deprecated since Java 9, forRemoval since Java 18** — use `try-with-resources`/`Cleaner` instead |
| finalize() called once | GC invokes finalize() on an object at most once, even if resurrected |
| Uncaught exception in finalize() | GC-invoked → silently ignored; explicitly-invoked → propagates normally |
| Memory leak | Object no longer needed but still referenced — GC can't help; monitor via JFR/VisualVM/MAT |
| Default modern collector | G1 (since Java 9); ZGC/Shenandoah for ultra-low latency |
| PermGen | Removed in Java 8, replaced by Metaspace (native memory) |

## Enum Cheat Sheet
| Concept | Rule |
|---|---|
| Base class | Implicitly extends `java.lang.Enum<E>` (abstract, implements Serializable & Comparable) |
| Inheritance | Cannot extend any class/enum (implicitly final); CAN implement interfaces |
| Constructors | Implicitly private; cannot be `new`'d externally |
| Members order | If extra members exist, constants must come first & be semicolon-terminated |
| `values()` | Static, implicit, returns array of constants in declaration order |
| `ordinal()` | Zero-based declaration-order index — avoid using for persisted/business logic |
| `name()` vs `toString()` | `name()` is final & stable; `toString()` overridable for display |
| Switch support | Since 1.5; unqualified constant names as case labels; exhaustive switch expressions since Java 14+ |
| Singleton pattern | Single-constant enum is the JVM-safest Singleton implementation |
| Collections | Prefer `EnumSet`/`EnumMap` over `HashSet`/`HashMap` for enum keys |
| Local declaration | `enum` cannot be declared inside a method body |

---

*End of Study Material — Generics, Garbage Collection, and Enum (Architect Edition)*
