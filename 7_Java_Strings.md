# Core Java Language Fundamentals — `java.lang` Package
### Senior Java Architect Study Guide & Interview Preparation

> Scope: `Object`, `String`, `StringBuffer`, `StringBuilder`, Wrapper Classes, Autoboxing/Autounboxing, Overload Resolution, `java.lang` hierarchy — with modern Java (9 → 21+) updates, validated runnable code, and an interview cheat sheet.

---

## Table of Contents

1. [Introduction to `java.lang`](#1-introduction-to-javalang)
2. [The `Object` Class](#2-the-object-class)
3. [The `String` Class](#3-the-string-class)
4. [`StringBuffer`](#4-stringbuffer)
5. [`StringBuilder`](#5-stringbuilder)
6. [Wrapper Classes](#6-wrapper-classes)
7. [Autoboxing & Autounboxing](#7-autoboxing--autounboxing)
8. [Overload Resolution: Widening vs Autoboxing vs Var-args](#8-overload-resolution-widening-vs-autoboxing-vs-var-args)
9. [Partial Hierarchy of `java.lang` & `Void`](#9-partial-hierarchy-of-javalang--void)
10. [Modern Java Updates (Java 8 → 21+) Relevant to `java.lang`](#10-modern-java-updates-java-8--21-relevant-to-javalang)
11. [Interview Questions Cheat Sheet](#11-interview-questions-cheat-sheet)

---

## 1. Introduction to `java.lang`

- `java.lang` bundles the classes/interfaces **most commonly required** by every Java program: `Object`, `String`, `StringBuffer`, `StringBuilder`, wrapper classes, `Math`, `System`, `Thread`, `Runnable`, `Iterable`, `Comparable`, `Record`, etc.
- **It is the only package that is imported implicitly** by the compiler into every `.java` source file — you never need `import java.lang.*;`.
- **Interview Q: "What is your favorite package? Why `java.lang`?"** — A classic HR-technical hybrid question. Good answer: *"`java.lang` is implicitly imported into every class, unlike every other package which requires an explicit `import`, and it hosts the foundational types (`Object`, `String`, wrapper classes) that every Java program depends on."*

Key classes covered in this document:

| # | Class/Concept | Introduced |
|---|---|---|
| 1 | `Object` | JDK 1.0 |
| 2 | `String` | JDK 1.0 |
| 3 | `StringBuffer` | JDK 1.0 |
| 4 | `StringBuilder` | JDK 1.5 |
| 5 | Wrapper classes (`Byte`,`Short`,`Integer`,`Long`,`Float`,`Double`,`Character`,`Boolean`,`Void`) | JDK 1.0 |
| 6 | Autoboxing / Autounboxing | JDK 1.5 |
| 7 | `Record` (data carriers) | JDK 16 (preview 14) |

---

## 2. The `Object` Class

`Object` is the **root of the class hierarchy**. Every class is a direct or indirect child of `Object`.

> **Rule:** If a class does not explicitly `extend` another class, it is a **direct child** of `Object`. If it extends any other class, it is an **indirect child** of `Object`.

### 2.1 Complete method list of `java.lang.Object`

```java
public String toString();
public native int hashCode();
public boolean equals(Object o);
protected native Object clone() throws CloneNotSupportedException;
public final native Class<?> getClass();
protected void finalize() throws Throwable;                 // Deprecated since Java 9, for removal
public final void wait() throws InterruptedException;
public final native void wait(long timeoutMillis) throws InterruptedException;
public final native void wait(long timeoutMillis, int nanos) throws InterruptedException;
public final native void notify();
public final native void notifyAll();
```

> **Interview Nuance:** The book text lists two overloaded `wait()` — this reflects the *effective* overload set: `wait()`, `wait(long)`, `wait(long, int)`. `wait()` internally delegates to `wait(0)`.

---

### 2.2 `toString()` Method

- Used to get the **String representation** of an object.
- Printing any object reference **implicitly invokes `toString()`**:
  `System.out.println(s1);` internally becomes `System.out.println(s1.toString());`
- Default implementation (from `Object`):

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```
Format: `classname@hexadecimal_hashcode`.

#### Example 1 — Default `toString()`

```java
class Student {
    String name;
    int rollno;

    Student(String name, int rollno) {
        this.name = name;
        this.rollno = rollno;
    }

    public static void main(String[] args) {
        Student s1 = new Student("saicharan", 101);
        Student s2 = new Student("ashok", 102);
        System.out.println(s1);            // Student@<hash1>
        System.out.println(s1.toString()); // Student@<hash1>  (same)
        System.out.println(s2);            // Student@<hash2>  (different object -> different hash)
    }
}
```
> Note: Actual hash values are JVM/run dependent — do NOT hardcode expected hash values in production tests.

#### Example 2 — Overriding `toString()`

```java
class Test {
    public String toString() { return "Test"; }

    public static void main(String[] args) {
        Integer i = 10;                 // Autoboxing (Java 5+)
        String  s = new String("ashok");
        Test t = new Test();

        System.out.println(i); // 10   (Integer.toString())
        System.out.println(s); // ashok (String.toString())
        System.out.println(t); // Test  (our overridden toString())
    }
}
```

- **Highly recommended** to override `toString()` in your domain classes — `String`, `StringBuffer`, `StringBuilder`, wrapper classes, and all Collection classes already override it for meaningful output.

---

### 2.3 `hashCode()` Method

- JVM generates a (nearly) unique `int` per object — used when storing objects into hash-based structures (`HashSet`, `HashMap`, `Hashtable`).
- Default `Object.hashCode()` is derived from **internal JVM state (often address-based, but not guaranteed and NOT the literal memory address)** — it's a native method, implementation-defined.
- If not overridden → `Object`'s version executes.
- **Correctness rule when overriding**: every *unequal* object should ideally produce a different hash for efficient hashing distribution (not mandatory for correctness, only for performance).

```java
// BAD override — same hash for all instances → degrades hashing to O(n) lookup
class Student {
    public int hashCode() { return 100; }
}

// GOOD override — hash varies with distinguishing state
class Student {
    int rollno;
    public int hashCode() { return rollno; }
}
```

#### `toString()` vs `hashCode()` interaction

```java
class Test {
    int i;
    Test(int i) { this.i = i; }
    public static void main(String[] args) {
        Test t1 = new Test(10);
        Test t2 = new Test(100);
        System.out.println(t1);  // Object.toString() -> internally calls Object.hashCode()
        System.out.println(t2);
    }
}
```
- If **only `hashCode()`** is overridden (not `toString()`), `Object.toString()` still executes but internally calls **your overridden** `hashCode()`.
- If **`toString()`** is overridden, it need **not** call `hashCode()` at all — the behavior is entirely up to your implementation.

> **Key takeaways:**
> 1. `toString()` → used while **printing** object references.
> 2. `hashCode()` → used while **storing objects into hash-based collections**.

---

### 2.4 `equals()` Method

- Used to check **equivalence** of two objects.
- Default `Object.equals()` implementation:

```java
public boolean equals(Object obj) {
    return (this == obj);   // reference comparison
}
```
- i.e., only returns `true` when both references point to the **exact same object**.

#### Example — default reference-based equality

```java
class Student {
    String name; int rollno;
    Student(String name, int rollno) { this.name = name; this.rollno = rollno; }

    public static void main(String[] args) {
        Student s1 = new Student("vijayabhaskar", 101);
        Student s2 = new Student("bhaskar", 102);
        Student s3 = new Student("vijayabhaskar", 101);
        Student s4 = s1;

        System.out.println(s1.equals(s2)); // false (different objects)
        System.out.println(s1.equals(s3)); // false (different objects, same content!)
        System.out.println(s1.equals(s4)); // true  (s4 refers to same object as s1)
    }
}
```

#### Overriding `equals()` for content comparison — three checklist items

When overriding `equals()`, you MUST handle:
1. **Meaning of equality** — which fields define equivalence.
2. **Heterogeneous objects** — passing a different type must return `false`, never throw `ClassCastException`.
3. **`null` argument** — must return `false`, never throw `NullPointerException`.

#### Naive version (handles exceptions defensively)

```java
public boolean equals(Object obj) {
    try {
        String name1 = this.name;
        int rollno1 = this.rollno;
        Student s2 = (Student) obj;
        if (name1.equals(s2.name) && rollno1 == s2.rollno) return true;
        else return false;
    } catch (ClassCastException e) {
        return false;
    } catch (NullPointerException e) {
        return false;
    }
}
```

#### Simplified (idiomatic, industry-standard) version

```java
public boolean equals(Object o) {
    if (this == o) return true;                 // fast-path / performance optimization
    if (!(o instanceof Student)) return false;   // instanceof safely returns false for null too
    Student s2 = (Student) o;
    return name.equals(s2.name) && rollno == s2.rollno;
}
```
> `o instanceof Student` returns `false` if `o` is `null` — so this single check **subsumes both** the null-check and the type-check requirements. This is the standard modern idiom.

#### Full working example (Example 6/7 combined)

```java
class Student {
    String name;
    int rollno;

    Student(String name, int rollno) {
        this.name = name;
        this.rollno = rollno;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Student)) return false;
        Student s2 = (Student) o;
        return name.equals(s2.name) && rollno == s2.rollno;
    }

    public static void main(String[] args) {
        Student s1 = new Student("vijayabhaskar", 101);
        Student s2 = new Student("bhaskar", 102);
        Student s3 = new Student("vijayabhaskar", 101);
        Student s4 = s1;

        System.out.println(s1.equals(s2));          // false
        System.out.println(s1.equals(s3));          // true  (content matches)
        System.out.println(s1.equals(s4));          // true
        System.out.println(s1.equals("vijayabhaskar")); // false (heterogeneous, no CCE)
        System.out.println(s1.equals(null));            // false (no NPE)
    }
}
```

### 2.5 `==` vs `.equals()`

| `==` (double equal operator) | `.equals()` method |
|---|---|
| An **operator**, applicable to both primitives and object references. | A **method**, applicable only to object references (not primitives). |
| For primitives → content comparison. For object references → reference comparison. | By default (`Object`), reference comparison; can be **overridden** for content comparison. |
| Cannot be overridden. | Can be overridden. |
| If argument types are unrelated → **compile-time error**: "incomparable types". | If argument types are unrelated → simply returns `false` (no compile/runtime error). |
| `r == null` → always `false`. | `r.equals(null)` → always `false` (per contract). |

```java
String s = new String("ashok");
StringBuffer sb = new StringBuffer("ashok");
// System.out.println(s == sb);     // COMPILE ERROR: incomparable types: String and StringBuffer
System.out.println(s.equals(sb));   // false (no compile error, StringBuffer.equals() is Object's reference-based)
```

#### Logical relationship between `==` and `.equals()`

1. If `r1 == r2` is `true` → `r1.equals(r2)` is **always** `true`.
2. If `r1 == r2` is `false` → `r1.equals(r2)` may be `true` or `false` (no conclusion).
3. If `r1.equals(r2)` is `true` → `r1 == r2` may be `true` or `false` (no conclusion).
4. If `r1.equals(r2)` is `false` → `r1 == r2` is **always** `false`.

Case comparisons (String content-overridden `equals` vs StringBuffer's non-overridden `equals`):

```java
String s1 = new String("ashok");
String s2 = new String("ashok");
System.out.println(s1 == s2);        // false (different heap objects)
System.out.println(s1.equals(s2));   // true  (String overrides equals() for content)

StringBuffer sb1 = new StringBuffer("ashok");
StringBuffer sb2 = new StringBuffer("ashok");
System.out.println(sb1 == sb2);      // false
System.out.println(sb1.equals(sb2)); // false (StringBuffer does NOT override equals() for content!)
```

> **Interview trap:** `StringBuffer`/`StringBuilder` **do NOT override `equals()`** for content comparison — they use `Object`'s reference-based `equals()`. Only `String`, wrapper classes, and Collection classes override `equals()`.

---

### 2.6 Contract between `equals()` and `hashCode()`

The formal contract (from `Object` Javadoc, essential for correct `HashMap`/`HashSet` behavior):

1. If two objects are equal by `.equals()` → their `hashCode()` **must** be equal.
   `r1.equals(r2) == true  ⟹  r1.hashCode() == r2.hashCode()`  (**mandatory**)
2. If two objects are **not** equal by `.equals()` → hash codes **may or may not** be equal (no restriction).
3. If hash codes are equal → `.equals()` **may** return `true` or `false` (hash collisions are legal — no conclusion).
4. If hash codes are **not** equal → objects are **guaranteed** to be unequal by `.equals()`.
   `r1.hashCode() != r2.hashCode() ⟹ r1.equals(r2) == false`

> **Rule of thumb:** Whenever you override `equals()`, you **must** override `hashCode()` using the **same fields** — otherwise the contract is violated. Violating it causes **no compile-time or runtime error**, but breaks `HashMap`/`HashSet`/`Hashtable` semantics silently (e.g., duplicate "equal" keys stored, or a key that should be found via `get()` is "lost").

```java
class Person {
    String name; int age;
    Person(String name, int age) { this.name = name; this.age = age; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person p = (Person) o;
        return name.equals(p.name) && age == p.age;
    }

    @Override
    public int hashCode() {
        return name.hashCode() + age;    // MUST use same fields as equals()
    }
}
```

> **Modern shortcut** — use `java.util.Objects` (Java 7+) instead of hand-rolling:
```java
import java.util.Objects;

@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Person p)) return false;   // Pattern Matching for instanceof (Java 16+)
    return age == p.age && Objects.equals(name, p.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);   // handles null-safety and combines hashes correctly
}
```

#### Valid/Invalid combinations (frequently asked as MCQ in OCJP/SCJP)

| Statement | Valid? |
|---|---|
| If hashcodes are **not** equal, `.equals()` always returns `false`. | ✅ Valid |
| If two objects are equal by `==`, their hashcodes must be same. | ✅ Valid |
| If `==` returns `false`, hashcodes **must** be different. | ❌ Invalid (may be same — hash collision is legal) |
| If hashcodes are equal, objects are always equal by `==`. | ❌ Invalid (hash collision ≠ same reference) |

---

### 2.7 `clone()` Method

- **Cloning** = creating an exact duplicate object — primarily used for **backup/recovery** purposes.

```java
protected native Object clone() throws CloneNotSupportedException;
```

#### Preconditions for cloning

- Class **must implement `Cloneable`** (a **marker interface** — has no methods; JVM grants the cloning ability automatically when detected).
- `Cloneable` lives in `java.lang`.
- Attempting to clone a non-`Cloneable` object → `CloneNotSupportedException` (checked, runtime-thrown).

```java
class Test implements Cloneable {
    int i = 10, j = 20;

    public static void main(String[] args) throws CloneNotSupportedException {
        Test t1 = new Test();
        Test t2 = (Test) t1.clone();
        t2.i = 888;
        t2.j = 999;
        System.out.println(t1.i + "---------------" + t1.j); // 10---------------20
        System.out.println(t2.i + "---------------" + t2.j); // 888---------------999
    }
}
```

### 2.8 Shallow Cloning vs Deep Cloning

**Shallow Cloning:**
- Bitwise copy: primitive fields get exact duplicate values.
- Reference fields are **copied as references only** — the *contained* object is **not duplicated**; both original and clone point to the **same** nested object.
- Changing the contained object via the original reference **is visible** through the clone too.
- `Object.clone()` performs **Shallow Cloning by default**.

```java
class Cat {
    int j;
    Cat(int j) { this.j = j; }
}

class Dog implements Cloneable {
    Cat c; int i;
    Dog(Cat c, int i) { this.c = c; this.i = i; }

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();     // shallow clone
    }
}

public class ShallowCloneDemo {
    public static void main(String[] args) throws CloneNotSupportedException {
        Cat c = new Cat(20);
        Dog d1 = new Dog(c, 10);
        System.out.println(d1.i + "......" + d1.c.j);   // 10......20

        Dog d2 = (Dog) d1.clone();
        d1.i = 888;
        d1.c.j = 999;                                     // mutating shared Cat object
        System.out.println(d2.i + "......" + d2.c.j);     // 10......999  <-- d2 impacted!
    }
}
```
- Best choice **only when the object contains purely primitive fields**.

**Deep Cloning:**
- Creates an **independent** duplicate, including all nested/contained objects.
- Requires **manual implementation** — the programmer overrides `clone()` to recursively clone member objects.

```java
class Cat {
    int j;
    Cat(int j) { this.j = j; }
}

class Dog implements Cloneable {
    Cat c; int i;
    Dog(Cat c, int i) { this.c = c; this.i = i; }

    @Override
    public Object clone() {
        Cat c1 = new Cat(c.j);          // manually clone nested object
        return new Dog(c1, i);
    }
}

public class DeepCloneDemo {
    public static void main(String[] args) {
        Cat c = new Cat(20);
        Dog d1 = new Dog(c, 10);
        System.out.println(d1.i + "......" + d1.c.j);  // 10......20

        Dog d2 = (Dog) d1.clone();
        d1.i = 888;
        d1.c.j = 999;
        System.out.println(d2.i + "......" + d2.c.j);  // 10......20  <-- d2 unaffected
    }
}
```

**Which to choose?**
- Object has only primitive fields → **Shallow Cloning** is sufficient and cheaper.
- Object has reference-type fields → **Deep Cloning** is recommended.

> **Modern Architect Note:** `Object.clone()`/`Cloneable` is widely considered a **design flaw** in Java (Josh Bloch, *Effective Java* Item 13 — "Override clone judiciously"). In modern code, prefer:
> - Copy constructors: `new Dog(originalDog)`
> - Static factory "copy-of" methods
> - Serialization-based deep copy (or libraries like Jackson/Gson for deep JSON copy) — but avoid this for performance-critical code.
> - Immutable objects / **`record`** types (Java 16+) that eliminate the need for cloning altogether, since they can't be mutated.

---

### 2.9 `getClass()` Method

```java
public final native Class<?> getClass();
```
- Returns the **runtime class** representation of an object (used heavily in **reflection**).

```java
Object o = new String("ashok");
System.out.println("Runtime object type of o is: " + o.getClass().getName());
// Output: Runtime object type of o is: java.lang.String
```

- Useful, e.g., to print the vendor-specific implementation class of an interface reference:
```java
System.out.println(con.getClass().getName());  // e.g., prints driver-specific Connection impl class
```

### 2.10 `finalize()` Method

```java
protected void finalize() throws Throwable;
```
- Called by the **Garbage Collector**, just before destroying an object, to perform cleanup activities.

> ⚠️ **Modern Java Update:** `Object.finalize()` was **deprecated in Java 9** (`@Deprecated(since="9")`) and is marked "for removal" as of Java 18 (JEP 421 finalized removal path continues; actual method removal is being phased in progressively). **Never rely on `finalize()` in new code.**
>
> **Recommended modern alternatives:**
> - **`try-with-resources`** + `java.lang.AutoCloseable` for deterministic resource cleanup.
> - **`java.lang.ref.Cleaner`** (Java 9+) — a safer, non-deprecated replacement for post-mortem cleanup:
```java
import java.lang.ref.Cleaner;

class Resource implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    private static class State implements Runnable {
        public void run() { System.out.println("Cleaning native resource..."); }
    }

    Resource() {
        this.cleanable = cleaner.register(this, new State());
    }

    @Override
    public void close() { cleanable.clean(); }
}
```

### 2.11 `wait()`, `notify()`, `notifyAll()`

- Defined in `Object` (not `Thread`) because the **monitor/lock** belongs to every object, not to threads.
- Used for **inter-thread communication** (classic Producer-Consumer pattern).
- Must be called from within a `synchronized` block/method on the object whose monitor is held, else → `IllegalMonitorStateException`.

```java
class SharedResource {
    synchronized void produce() throws InterruptedException {
        System.out.println("Producing...");
        wait();                 // releases lock, waits for notify()
        System.out.println("Resumed after notify");
    }
    synchronized void consume() {
        notify();                // wakes up a single waiting thread
    }
}
```
> **Modern preference:** For new concurrent code, prefer **`java.util.concurrent`** primitives — `Lock`/`Condition`, `BlockingQueue`, `CompletableFuture`, `ExecutorService`, **Virtual Threads (Java 21, JEP 444)** — over raw `wait/notify`.

---

## 3. The `String` Class

### 3.1 Immutability

> **Once a `String` object is created, its content can never be changed.** Any "modifying" operation creates a **new** object; if there's no change in content, the JVM may reuse the existing object.

```java
String s = new String("bhaskar");
s.concat("software");        // return value discarded!
System.out.println(s);       // "bhaskar" — original untouched
```

Contrast with mutable `StringBuffer`:

```java
StringBuffer sb = new StringBuffer("bhaskar");
sb.append("software");
System.out.println(sb);      // "bhaskarsoftware" — modified in place
```

### 3.2 String Constant Pool (SCP)

- A **specially designed memory area** (part of the heap since Java 7, previously PermGen pre-Java 7) reserved for `String` literals.
- **`new` operator always creates a new object on the heap** — never reuses SCP.
- String **literals** are placed in SCP; JVM checks SCP first — if content already exists, it **reuses** it; duplicate objects with identical content are **never** allowed in SCP.
- `String s3 = "bhaskar";` and `String s4 = "bhaskar";` → both point to the **same** SCP object.

```java
String s1 = new String("bhaskar");   // heap object #1
String s2 = new String("bhaskar");   // heap object #2 (different from s1)
String s3 = "bhaskar";               // SCP object
String s4 = "bhaskar";               // reuses SAME SCP object as s3

System.out.println(s1 == s2); // false
System.out.println(s3 == s4); // true
```

**Why SCP exists (Interview: Advantage of SCP):**
Instead of creating a separate object for every repeated literal, we reuse one object → improves **performance & memory utilization**.

**Disadvantage of SCP (Interview classic):**
Since multiple references point to the *same* SCP object, mutating via one reference would corrupt all others. **To prevent this, Sun/Oracle made `String` immutable** — immutability is essentially **the price paid for having SCP**.

**Why is SCP-like pooling not needed for `StringBuffer`?**
- `String` is the *most commonly used* object → needs a shared pool for efficiency.
- `StringBuffer` objects are created per-requirement (not reused across the app) → no shared-pool benefit, hence no such pool exists, and hence no immutability requirement either.

#### Facts about SCP

1. Object creation in SCP is **optional** — JVM reuses if content already present.
2. **GC cannot access SCP** — even a "referenceless" String literal in SCP is **never eligible for GC** (JDK 6 and earlier, when SCP lived in PermGen). ⚠️ *Since Java 7, the String pool moved to the regular heap — see [Modern Updates](#10-modern-java-updates-java-8--21-relevant-to-javalang) — so pooled Strings CAN be garbage collected if unreferenced.*
3. All SCP objects are destroyed automatically at JVM shutdown.
4. Duplicate objects are possible in the **heap**, but **never** in the SCP.

#### Detailed heap/SCP tracing example

```java
String s = new String("bhaskar");
s.concat("software");
s = s.concat("solutions");
s = "bhaskarsoft";
```
- `new String("bhaskar")` → 1 heap object `"bhaskar"` + 1 SCP object `"bhaskar"` (for the literal).
- `s.concat("software")` → creates a NEW heap object `"bhaskarsoftware"` + SCP object `"software"`, but result is discarded (`s` still points to original heap `"bhaskar"`).
- `s = s.concat("solutions")` → new heap object `"bhaskarsolutions"` + SCP `"solutions"`; `s` reassigned to it.
- `s = "bhaskarsoft"` → new SCP entry `"bhaskarsoft"`; `s` reassigned.

Total distinct objects across both areas, and which remain reachable, is a classic OCJP diagram exercise (see original source diagrams for full picture).

#### Compile-time constant folding (`final` + literal concatenation)

```java
class StringDemo {
    public static void main(String[] args) {
        String s1 = new String("you cannot change me!");
        String s2 = new String("you cannot change me!");
        System.out.println(s1 == s2);                 // false

        String s3 = "you cannot change me!";
        System.out.println(s1 == s3);                 // false

        String s4 = "you cannot change me!";
        System.out.println(s3 == s4);                 // true (same SCP object)

        String s5 = "you cannot " + "change me!";      // compile-time constant expr -> folded into ONE literal
        System.out.println(s3 == s5);                  // true

        String s6 = "you cannot ";
        String s7 = s6 + "change me!";                 // s6 is a VARIABLE -> runtime concatenation -> new heap object
        System.out.println(s3 == s7);                  // false

        final String s8 = "you cannot ";                // s8 is a COMPILE-TIME CONSTANT (final + literal init)
        String s9 = s8 + "change me!";                   // compiler treats as constant expression -> SCP reuse
        System.out.println(s3 == s9);                    // true
        System.out.println(s6 == s8);                    // true (both point to same SCP literal)
    }
}
```
> **Interview gold:** `final` local/field variables initialized with a **compile-time constant** literal participate in **constant folding**, so `s8 + "change me!"` is resolved by the **compiler** (not JVM at runtime) into a single literal placed in SCP — hence `s3 == s9` is `true`. Remove `final` from `s8`, and it becomes a runtime concatenation (`StringBuilder.append()` under the hood) producing a **new heap object**, so the comparison becomes `false`.

### 3.3 Interning Strings — `intern()`

```java
String s1 = new String("bhaskar");
String s2 = s1.intern();      // fetches (or creates) the SCP equivalent
System.out.println(s1 == s2); // false
String s3 = "bhaskar";
System.out.println(s2 == s3); // true
```
- If the equivalent object is **not already present** in SCP, `intern()` **creates it** and returns it.
```java
String s1 = new String("bhaskar");
String s2 = s1.concat("software");     // heap-only, no SCP entry for "bhaskarsoftware" yet
String s3 = s2.intern();                // now creates/fetches SCP "bhaskarsoftware"
String s4 = "bhaskarsoftware";
System.out.println(s3 == s4);           // true
```

### 3.4 `String` Class Constructors

| Constructor | Description |
|---|---|
| `String()` | Empty string object. |
| `String(String literal)` | Equivalent heap object for the given literal. |
| `String(StringBuffer sb)` | Equivalent String for a `StringBuffer`. |
| `String(char[] ch)` | Equivalent String from a char array. |
| `String(byte[] b)` | Equivalent String from a byte array (platform default charset — prefer explicit `Charset` overload in modern code). |

```java
char[] ch = {'a', 'b', 'c'};
String s = new String(ch);
System.out.println(s);   // abc

byte[] b = {100, 101, 102};
String s2 = new String(b);
System.out.println(s2);  // def   (ASCII 100='d', 101='e', 102='f')
```

> **Modern best practice:** Prefer `new String(bytes, StandardCharsets.UTF_8)` (explicit charset) over the platform-default-charset constructor to avoid environment-dependent bugs. Also, since Java 9, `String.valueOf(char[])` and constructors are unaffected functionally but internal storage changed (Compact Strings — see Modern Updates).

### 3.5 Important Methods of `String`

| # | Method | Purpose |
|---|---|---|
| 1 | `char charAt(int index)` | Char at index (0-based); OOB → `StringIndexOutOfBoundsException`. |
| 2 | `String concat(String str)` | Concatenation (returns NEW string; original unaffected). |
| 3 | `boolean equals(Object o)` | Content comparison, case-sensitive. |
| 4 | `boolean equalsIgnoreCase(String s)` | Content comparison, case-insensitive. |
| 5 | `String substring(int begin)` | From `begin` to end. |
| 6 | `String substring(int begin, int end)` | `[begin, end)` — end exclusive. |
| 7 | `int length()` | Character count (method, NOT a field — unlike arrays' `.length`). |
| 8 | `String replace(char old, char new)` | Replace every occurrence. |
| 9 | `String toLowerCase()` / `toUpperCase()` | Case conversion. |
| 10 | `String trim()` | Removes **leading/trailing** whitespace only (not middle). |
| 11 | `int indexOf(char ch)` | First occurrence index, or `-1`. |
| 12 | `int lastIndexOf(char ch)` | Last occurrence index, or `-1`. |

```java
class StringDemo {
    public static void main(String[] args) {
        String s = "ashok";
        System.out.println(s.charAt(3));                 // o
        // System.out.println(s.charAt(100));            // RE: StringIndexOutOfBoundsException

        s = s.concat("software");
        System.out.println(s);                            // ashoksoftware

        System.out.println("java".equals("JAVA"));            // false
        System.out.println("java".equalsIgnoreCase("JAVA"));  // true

        String x = "ashoksoft";
        System.out.println(x.substring(5));      // soft
        System.out.println(x.substring(3, 7));   // okso

        System.out.println("jobs4times".length());  // 10  (method call, not .length field)

        System.out.println("ababab".replace('a', 'b'));  // bbbbbb

        System.out.println("ASHOK".toLowerCase());   // ashok
        System.out.println("ashok".toUpperCase());   // ASHOK

        System.out.println(" sai charan ".trim());   // "sai charan" (no leading/trailing spaces)

        System.out.println("saicharan".indexOf('c'));     // 3
        System.out.println("saicharan".indexOf('z'));     // -1
        System.out.println("arunkumar".lastIndexOf('a')); // 7
    }
}
```

> **Validation practice tip:** Username validation → `equalsIgnoreCase()` (case doesn't matter). Password validation → `equals()` (case matters — security requirement).

#### `length` field vs `length()` method (classic trap)

```java
int[] arr = {1,2,3};
System.out.println(arr.length);     // FIELD (arrays)

String s = "abc";
System.out.println(s.length());     // METHOD (String)
// System.out.println(s.length);    // COMPILE ERROR: cannot find symbol
```

#### Reuse rule for runtime String operations

> If there is **no change in content**, the **same object is reused** — this holds whether the object lives on the Heap or in SCP. If there **is** a change, a **new object is created only on the Heap**, never directly in SCP (SCP entries are only created for **literals**, via compiler-time placement, or explicitly via `intern()`).

```java
String s1 = "bhaskar";
String s2 = s1.toUpperCase();   // content changes -> new heap object
String s3 = s1.toLowerCase();   // content SAME -> reuses s1's SCP object
System.out.println(s1 == s2);   // false
System.out.println(s1 == s3);   // true

String s4 = s1.toString();      // String.toString() returns "this" -> same object
System.out.println(s1 == s4);   // true
```

### 3.6 Creating Our Own Immutable Class

> **Once created, an immutable object's state can never change.** Any "modification" produces a new object; if no actual change occurs, the existing object is reused.

```java
final class CreateImmutable {
    private final int i;

    CreateImmutable(int i) { this.i = i; }

    public CreateImmutable modify(int i) {
        if (this.i == i) return this;                 // no change -> reuse existing object
        else return new CreateImmutable(i);            // change -> new object
    }

    public static void main(String[] args) {
        CreateImmutable c1 = new CreateImmutable(10);
        CreateImmutable c2 = c1.modify(100);
        CreateImmutable c3 = c1.modify(10);
        System.out.println(c1 == c2);   // false
        System.out.println(c1 == c3);   // true (10==10, no change -> reused)

        CreateImmutable c4 = c1.modify(100);
        System.out.println(c2 == c4);   // false (new object each time content actually differs)
    }
}
```

**Recipe for a proper immutable class (extended, architect-level checklist):**
1. Declare the class `final` (prevents subclassing from breaking immutability).
2. Make all fields `private final`.
3. No setters; only a constructor to initialize state.
4. If a field is a **mutable object reference** (e.g., `Date`, `List`, array), **defensively copy** it in the constructor AND in any getter that exposes it.
5. Do not let `this` escape during construction (no listener registration, no leaking `this` to another thread before construction is complete).

```java
import java.util.*;

final class ImmutablePerson {
    private final String name;
    private final List<String> hobbies;

    public ImmutablePerson(String name, List<String> hobbies) {
        this.name = name;
        this.hobbies = new ArrayList<>(hobbies);   // defensive copy IN
    }

    public String getName() { return name; }

    public List<String> getHobbies() {
        return Collections.unmodifiableList(hobbies);   // defensive copy OUT (immutable view)
    }
}
```

> **Modern Java Update:** For simple immutable data carriers, prefer **`record`** (Java 16+, see [§10](#10-modern-java-updates-java-8--21-relevant-to-javalang)) — the compiler auto-generates the constructor, accessors, `equals()`, `hashCode()`, and `toString()`.

### 3.7 `final` vs Immutability

- `final` applies to **variables** (reference reassignment is blocked). Immutability applies to **objects/state**.
- Declaring a reference `final` does **NOT** make the referenced object immutable — you simply cannot reassign the reference to a different object.

```java
class Test {
    public static void main(String[] args) {
        final StringBuffer sb = new StringBuffer("ashok");
        sb.append("software");                       // ALLOWED — object state changes
        System.out.println(sb);                       // ashoksoftware
        // sb = new StringBuffer("solutions");        // COMPILE ERROR: cannot assign a value to final variable sb
    }
}
```

| Combination | Meaningful? |
|---|---|
| `final` variable | ✅ Valid |
| "final object" | ❌ Invalid concept (finality applies to variables, not objects) |
| "immutable variable" | ❌ Invalid concept (immutability applies to objects, not variables) |
| immutable object | ✅ Valid |

---

## 4. `StringBuffer`

- Use when content **changes frequently** — every change happens **in the existing object** (no new object created), unlike `String`.

### 4.1 Constructors

| Constructor | Initial Capacity |
|---|---|
| `StringBuffer()` | 16 (default) |
| `StringBuffer(int initialCapacity)` | as specified |
| `StringBuffer(String s)` | `s.length() + 16` |

**Growth formula:** Once capacity is exceeded, `newCapacity = (currentCapacity + 1) * 2`.

```java
class StringBufferDemo {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer();
        System.out.println(sb.capacity());        // 16
        sb.append("abcdefghijklmnop");             // exactly 16 chars, fits
        System.out.println(sb.capacity());         // 16
        sb.append("q");                             // 17th char -> exceeds capacity
        System.out.println(sb.capacity());          // (16+1)*2 = 34
    }
}
```

```java
StringBuffer sb2 = new StringBuffer(19);
System.out.println(sb2.capacity()); // 19

StringBuffer sb3 = new StringBuffer("ashok");
System.out.println(sb3.capacity()); // 5 + 16 = 21
```

### 4.2 Important Methods

```java
class StringBufferDemo {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer("saiashokkumarreddy");
        System.out.println(sb.length());     // 18
        System.out.println(sb.capacity());   // 34  (18+16)
        System.out.println(sb.charAt(14));   // e
        // sb.charAt(30);                    // RE: StringIndexOutOfBoundsException

        sb.setCharAt(0, 'S');
        System.out.println(sb);

        StringBuffer app = new StringBuffer();
        app.append("PI value is :").append(3.14).append(" this is exactly ").append(true);
        System.out.println(app); // PI value is :3.14 this is exactly true

        StringBuffer ins = new StringBuffer("abcdefgh");
        ins.insert(2, "xyz");
        ins.insert(11, "9");
        System.out.println(ins); // abxyzcdefgh9

        StringBuffer del = new StringBuffer("saicharankumar");
        del.delete(6, 13);
        System.out.println(del); // saichar
        del.deleteCharAt(5);
        System.out.println(del); // saichr

        StringBuffer rev = new StringBuffer("ashokkumar");
        System.out.println(rev.reverse()); // ramukkohsa

        StringBuffer setLen = new StringBuffer("ashokkumar");
        setLen.setLength(6);
        System.out.println(setLen); // ashokk

        StringBuffer trimSb = new StringBuffer(1000);
        System.out.println(trimSb.capacity());   // 1000
        trimSb.append("ashok");
        trimSb.trimToSize();
        System.out.println(trimSb.capacity());   // 5

        StringBuffer ensureSb = new StringBuffer();
        System.out.println(ensureSb.capacity());  // 16
        ensureSb.ensureCapacity(1000);
        System.out.println(ensureSb.capacity());  // 1000
    }
}
```

**Method summary table:**

| Method | Purpose |
|---|---|
| `length()` | # of characters currently stored. |
| `capacity()` | Total characters it can currently hold before resizing. |
| `charAt(int)` | Char at index. |
| `setCharAt(int, char)` | Replace char at index. |
| `append(...)` (overloaded for `String`,`int`,`long`,`boolean`,`double`,`float`, etc.) | Add to the end. |
| `insert(int index, ...)` | Insert at a given position (overloaded like `append`). |
| `delete(int begin, int end)` | Remove `[begin, end)`. |
| `deleteCharAt(int index)` | Remove single char. |
| `reverse()` | Reverse contents. |
| `setLength(int)` | Truncate/pad to exact length. |
| `trimToSize()` | Shrink capacity to match current size. |
| `ensureCapacity(int)` | Grow capacity proactively. |

> **Every method in `StringBuffer` is `synchronized`** → thread-safe, but with a **performance penalty** (only one thread can operate at a time, others wait). This is the exact motivation behind `StringBuilder`.

---

## 5. `StringBuilder`

- Introduced in **Java 1.5**.
- **API-identical** to `StringBuffer` (same constructors & methods) EXCEPT:

| `StringBuffer` | `StringBuilder` |
|---|---|
| Every method is `synchronized`. | No method is `synchronized`. |
| Thread-safe — only one thread can operate at a time. | NOT thread-safe — multiple threads can operate simultaneously. |
| Higher waiting time → relatively lower performance. | No waiting → relatively higher performance. |
| Introduced in JDK 1.0. | Introduced in JDK 1.5. |

### `String` vs `StringBuffer` vs `StringBuilder` — decision matrix

| Requirement | Choose |
|---|---|
| Content fixed, won't change frequently | `String` |
| Content changes frequently + thread-safety required | `StringBuffer` |
| Content changes frequently + thread-safety NOT required | `StringBuilder` |

### Method Chaining

- Since most `String`/`StringBuffer`/`StringBuilder` methods **return the same type**, calls can be chained: `sb.m1().m2().m3()...`, evaluated strictly **left to right**.

```java
class StringBufferDemo {
    public static void main(String[] args) {
        StringBuffer sb = new StringBuffer();
        sb.append("ashok").insert(5, "arunkumar").delete(11, 13)
          .reverse().append("solutions").insert(18, "abcdf").reverse();
        System.out.println(sb);
    }
}
```
> Trace each chained call step-by-step during interviews — this is a favorite "predict the output" style question.

---

## 6. Wrapper Classes

**Purpose:**
1. Wrap primitives into object form so they can be handled polymorphically (e.g., stored in Collections pre-generics-era, used with `Object` APIs).
2. Provide utility methods for the primitives (parsing, converting, formatting).

### 6.1 Constructors — Summary Table

| Wrapper | Constructor Arg Types |
|---|---|
| `Byte` | `byte`, `String` |
| `Short` | `short`, `String` |
| `Integer` | `int`, `String` |
| `Long` | `long`, `String` |
| `Float` | `float`, `String`, `double` (3 constructors) |
| `Double` | `double`, `String` |
| `Character` | `char` **only** (no String constructor) |
| `Boolean` | `boolean`, `String` |

```java
Integer i = new Integer(10);         // int arg
Integer i2 = new Integer("10");      // String arg -> parsed
// Integer bad = new Integer("ten"); // RE: NumberFormatException

Float f1 = new Float(10.5f);
Float f2 = new Float("10.5f");
Float f3 = new Float(10.5);          // double arg
Float f4 = new Float("10.5");

Character ch = new Character('a');       // valid
// Character ch2 = new Character("a");   // COMPILE ERROR: no such constructor

Boolean b1 = new Boolean(true);
Boolean b2 = new Boolean(false);
// Boolean b3 = new Boolean(True);       // COMPILE ERROR (case-sensitive keyword)
```

**`Boolean(String)` parsing rule:** Case-**insensitive** match against `"true"` → `true`; **anything else** (including `null`, `"yes"`, garbage) → `false`.

```java
System.out.println(new Boolean("true"));   // true
System.out.println(new Boolean("True"));   // true
System.out.println(new Boolean("TRUE"));   // true
System.out.println(new Boolean("false"));  // false
System.out.println(new Boolean("ashok"));  // false
System.out.println(new Boolean("yes"));    // false  <-- trap! "yes" is NOT true
```

> ⚠️ **Modern Java Update — CONSTRUCTORS ARE DEPRECATED:**
> As of **Java 9**, all wrapper class constructors (`new Integer(10)`, `new Boolean(true)`, `new Long(5)`, etc.) are **`@Deprecated(since="9")`** ("for removal"). Compiling with `new Integer(10)` on Java 9+ produces a deprecation warning.
> **Use factory methods instead:**
> ```java
> Integer i = Integer.valueOf(10);      // preferred (may return cached instance)
> Integer i2 = 10;                       // Autoboxing — internally calls Integer.valueOf(10)
> ```
> **Why deprecated?** Constructors *always* allocate a new object; `valueOf()` can leverage the **internal cache** (see §7) for small values, improving performance and reducing memory churn. `Long`, `Integer`, `Short`, `Byte`, `Character`, `Boolean`, `Float`, `Double` constructors are ALL deprecated for this reason.

- **Note:** in all wrapper classes, `toString()` is overridden (returns content), and `.equals()` is overridden for **content comparison** (unlike `StringBuffer`).

```java
Integer i1 = new Integer(10);   // (deprecated form, shown for legacy understanding)
Integer i2 = new Integer(10);
System.out.println(i1);              // 10
System.out.println(i1.equals(i2));   // true (content compared)
```

### 6.2 Utility Methods

#### a) `valueOf()` — three overloaded forms

**Form 1** — String → Wrapper (every wrapper except `Character`):
```java
Integer i = Integer.valueOf("10");
Double  d = Double.valueOf("10.5");
Boolean b = Boolean.valueOf("ashok");   // false
```

**Form 2** — radix-based String → Wrapper (`Byte`, `Short`, `Integer`, `Long` only):
```java
Integer i = Integer.valueOf("100", 2);   // parse "100" as BASE-2 -> decimal 4
System.out.println(i);                    // 4
```
Radix range: **2 to 36**. (base2: 0-1, base8: 0-7, base10: 0-9, base16: 0-9,a-f, base36: 0-9,a-z)

**Form 3** — primitive → Wrapper (all wrapper classes, including `Character`):
```java
Integer i = Integer.valueOf(10);
Double  d = Double.valueOf(10.5);
Boolean b = Boolean.valueOf(true);
Character ch = Character.valueOf('a');
```

#### b) `xxxValue()` — Wrapper → primitive

Every numeric wrapper (`Byte`,`Short`,`Integer`,`Long`,`Float`,`Double`) defines **all six**:
```java
public byte byteValue();
public short shortValue();
public int intValue();
public long longValue();
public float floatValue();
public double doubleValue();
```

```java
Integer i = new Integer(130);
System.out.println(i.byteValue());   // -126 (overflow: 130 doesn't fit in signed byte range -128..127)
System.out.println(i.shortValue());  // 130
System.out.println(i.intValue());    // 130
System.out.println(i.longValue());   // 130
System.out.println(i.floatValue());  // 130.0
System.out.println(i.doubleValue()); // 130.0
```

`Character.charValue()` and `Boolean.booleanValue()` are the type-specific equivalents:
```java
char c = new Character('a').charValue();       // a
boolean bv = new Boolean("ashok").booleanValue(); // false
```
> Total possible `xxxValue()` methods across the wrapper family = **38** (6 numeric wrappers × 6 methods = 36, + `charValue()` + `booleanValue()`).

#### c) `parseXxx()` — String → primitive (NOT an object!)

**Form 1** (every wrapper except `Character`):
```java
int i = Integer.parseInt("10");
boolean b = Boolean.parseBoolean("ashok");  // false
double d = Double.parseDouble("10.5");
```

**Form 2** (radix — `Byte`,`Short`,`Integer`,`Long` only):
```java
int i = Integer.parseInt("100", 2);   // 4
```

> **`valueOf()` vs `parseXxx()`:** `valueOf()` returns a **wrapper object**; `parseXxx()` returns a **primitive value**. Choose based on whether you need object semantics (e.g., for Collections) or raw primitive performance.

#### d) `toString()` — Wrapper/primitive → String

**Form 1** — instance method (overridden `Object.toString()`):
```java
Integer i = Integer.valueOf("10");
System.out.println(i.toString());  // "10"
```

**Form 2** — static, primitive → String:
```java
String s1 = Integer.toString(10);
String s2 = Boolean.toString(true);
String s3 = Character.toString('a');
```

**Form 3** — `Integer`/`Long` only, primitive → radix-specific String:
```java
String s1 = Integer.toString(7, 2);    // "111"
String s2 = Integer.toString(17, 2);   // "10001"
```

**Form 4** — `Integer`/`Long` convenience radix methods:
```java
System.out.println(Integer.toBinaryString(7));   // 111
System.out.println(Integer.toOctalString(10));   // 12
System.out.println(Integer.toHexString(20));     // 14
System.out.println(Integer.toHexString(10));     // a
```

### 6.3 The "Dance" Between `String`, Wrapper Object, and Primitive

```
                    toString()
   String  <───────────────────────  wrapper object
     │  ▲                                  │  ▲
     │  │           toString()             │  │
parseXxx()│                            xxxValue()│ valueOf()
     │  │                                  │  │
     ▼  │           valueOf()               ▼  │
   primitive ───────────────────────► (via valueOf, autoboxing)
```
- `String → primitive`: `parseXxx()`
- `primitive → String`: `toString()` (static form)
- `String/primitive → wrapper object`: `valueOf()`
- `wrapper object → String`: `toString()` (instance form)
- `wrapper object → primitive`: `xxxValue()`

---

## 7. Autoboxing & Autounboxing

- **Autoboxing** = automatic **primitive → wrapper object** conversion, performed by the **compiler** (introduced JDK 1.5).
- **Autounboxing** = automatic **wrapper object → primitive** conversion, performed by the **compiler**.
- Before 1.5, ALL such conversions had to be done **explicitly** by the programmer.

### Pre-1.5 pain point (why Autoboxing was introduced)

```java
// Pre-1.5 style (still valid, just verbose)
Boolean b = new Boolean(true);
if (b.booleanValue()) {                // manual unboxing required before 1.5
    System.out.println("hello");
}
```
```java
// 1.5+ Autoboxing/Autounboxing — compiler inserts conversions automatically
Boolean b = true;      // Autoboxing:   Boolean.valueOf(true)
if (b) {                // Autounboxing: b.booleanValue()
    System.out.println("hello");
}
```

### Internal compiler translation

```java
Integer i = 10;
// compiles to:
Integer i = Integer.valueOf(10);     // Autoboxing implemented via valueOf(), NOT the constructor!

Integer I = new Integer(10);
int i = I;
// compiles to:
int i = I.intValue();                // Autounboxing implemented via xxxValue()
```

```java
import java.util.*;
class AutoBoxingAndUnboxingDemo {
    static Integer I = 10;                  // ① Autoboxing
    public static void main(String[] args) {
        int i = I;                          // ② Autounboxing
        methodOne(i);                       // ③ Autoboxing (int -> Integer, matching method param)
    }
    public static void methodOne(Integer I) {
        int k = I;                          // ④ Autounboxing
        System.out.println(k);              // 10
    }
}
```

### ⚠️ `NullPointerException` Trap

```java
class AutoBoxingAndUnboxingDemo {
    static Integer I;              // null by default
    public static void main(String[] args) {
        int i = I;                 // Autounboxing null -> I.intValue() -> NPE!
        System.out.println(i);
    }
}
// Runtime Exception: java.lang.NullPointerException
```
> **This is one of the MOST common production bugs** — a `null` `Integer`/`Long`/etc. field silently blows up on unboxing in an `if` condition, arithmetic expression, or ternary operator. Extremely popular in interviews and code reviews.

### `==` Semantics With Autoboxing — The Integer Cache

> **All wrapper objects are immutable** — once created, state can't change; any "modification" produces a new object.

```java
Integer x = 10;
Integer y = x;
++x;                         // x is reassigned to a NEW Integer(11) via autobox/unbox round-trip
System.out.println(x);       // 11
System.out.println(y);       // 10  (y still points to the original object)
System.out.println(x == y);  // false
```

#### The Integer/wrapper caching mechanism (a top interview topic)

> To implement Autoboxing efficiently, the JVM pre-creates and **caches** a buffer of wrapper objects at class-loading time. When Autoboxing needs an object, it **first checks the cache**; only if absent does it create a new object. This improves performance & memory usage — **but only within a specific value range**:

| Wrapper | Cache Range |
|---|---|
| `Byte` | **Always** (all 256 possible byte values) |
| `Short` | **-128 to 127** |
| `Integer` | **-128 to 127** |
| `Long` | **-128 to 127** |
| `Character` | **0 to 127** |
| `Boolean` | **Always** (`TRUE`/`FALSE` constants) |
| `Float`, `Double` | **Never cached** — floating-point wrappers are ALWAYS newly created. |

```java
Integer x = 127; Integer y = 127;
System.out.println(x == y);   // true  (within cache range -128..127)

Integer x2 = 128; Integer y2 = 128;
System.out.println(x2 == y2); // false (outside cache range -> new objects each time)

Boolean b1 = true; Boolean b2 = true;
System.out.println(b1 == b2); // true (Boolean ALWAYS cached)

Double d1 = 10.0; Double d2 = 10.0;
System.out.println(d1 == d2); // false (Double/Float NEVER cached)
```

```java
// Explicitly via new -> ALWAYS a new object, cache bypassed entirely
Integer x = new Integer(10);
Integer y = new Integer(10);
System.out.println(x == y);   // false

// Explicitly via valueOf() -> cache honored
Integer x2 = Integer.valueOf(10);
Integer y2 = Integer.valueOf(10);
System.out.println(x2 == y2); // true (10 is within cache range)
```

> **Rule:** Internally, Autoboxing (`Integer i = 10;`) is implemented via `Integer.valueOf()` — so the cache rule applies identically whether you write `Integer i = 10;` or `Integer i = Integer.valueOf(10);`. Only `new Integer(10)` bypasses the cache (and is deprecated anyway since Java 9).

> ⚠️ **Modern Update:** The `Integer` cache upper bound (default 127) can be **tuned at JVM startup** with `-XX:AutoBoxCacheMax=<N>` (an undocumented/internal HotSpot flag, JDK-internal `sun.misc.VM` / `jdk.internal` property `java.lang.Integer.IntegerCache.high`). This does NOT change the language spec — `Long`, `Short`, `Byte`, `Character`, `Boolean` caches are fixed and not tunable this way.

> **Architect takeaway:** **NEVER** use `==` to compare wrapper objects for value equality — always use `.equals()`, or unbox to primitives first if `null`-safety is guaranteed. This is a top-tier code-review red flag.

### Conclusions on Autoboxing (from source)

1. A **buffer of cached objects** is created per wrapper class **at class-loading time**.
2. Autoboxing first **checks the buffer**; if the value is present, the **cached object is reused**.
3. If not present in the buffer, a **new object is created**.
4. This buffering happens **only** for the ranges listed above; outside these ranges, a new object is always created.
5. Since Autoboxing is internally implemented via `valueOf()`, **the same caching rule applies to explicit `valueOf()` calls**.

---

## 8. Overload Resolution: Widening vs Autoboxing vs Var-args

> **Compiler's precedence order when resolving an overloaded method call:**
> **1. Widening → 2. Autoboxing → 3. Var-arg method** (var-args is the *last resort*, similar to a `default` case in `switch`).

### Case 1: Widening vs Autoboxing → **Widening wins**

```java
class Demo {
    public static void methodOne(long l)     { System.out.println("widening"); }
    public static void methodOne(Integer i)  { System.out.println("autoboxing"); }
    public static void main(String[] args) {
        int x = 10;
        methodOne(x);          // Output: widening
    }
}
```

### Case 2: Widening vs Var-arg → **Widening wins**

```java
class Demo {
    public static void methodOne(long l)      { System.out.println("widening"); }
    public static void methodOne(int... i)    { System.out.println("var-arg method"); }
    public static void main(String[] args) {
        int x = 10;
        methodOne(x);           // Output: widening
    }
}
```

### Case 3: Autoboxing vs Var-arg → **Autoboxing wins**

```java
class Demo {
    public static void methodOne(Integer i)   { System.out.println("Autoboxing"); }
    public static void methodOne(int... i)    { System.out.println("var-arg method"); }
    public static void main(String[] args) {
        int x = 10;
        methodOne(x);            // Output: Autoboxing
    }
}
```

### Case 4: Widening + Autoboxing COMBINED is illegal in one step

```java
class Demo {
    public static void methodOne(Long l) { System.out.println("Long"); }
    public static void main(String[] args) {
        int x = 10;
        methodOne(x);   // COMPILE ERROR:
        // methodOne(java.lang.Long) cannot be applied to (int)
    }
}
```
> **Rule:** `int → long` (widening) **followed by** `long → Long` (autoboxing) is **NOT allowed** in a single implicit conversion chain. However, `int → Integer` (autoboxing) **followed by** `Integer → Object` (widening reference conversion) **IS allowed**:

```java
class Demo {
    public static void methodOne(Object o) { System.out.println("Object"); }
    public static void main(String[] args) {
        int x = 10;
        methodOne(x);    // Output: Object  (Autoboxing int->Integer, then widening reference Integer->Object)
    }
}
```
> **Key asymmetry (frequently tested):** *"Widening followed by Autoboxing" is illegal, but "Autoboxing followed by widening (reference widening)" is legal.*

### Valid/Invalid Declaration Quiz (Classic MCQ Set)

```java
int i = 10;             // ✅ valid
Integer I = 10;          // ✅ valid (autoboxing)
// int i = 10L;          // ❌ CE: possible lossy conversion from long to int
Long l = 10L;             // ✅ valid
// Long l = 10;           // ❌ CE: incompatible types (int -> Long needs widening+autoboxing, illegal)
long l2 = 10;              // ✅ valid (int -> long widening, primitive to primitive)
Object o = 10;              // ✅ valid (autoboxing int->Integer, then widening ref Integer->Object)
double d = 10;                // ✅ valid (int -> double widening)
// Double d2 = 10;             // ❌ CE (int -> Double needs widening+autoboxing, illegal — same as Long case)
Number n = 10;                  // ✅ valid (autoboxing int->Integer, then widening ref Integer->Number)
```

---

## 9. Partial Hierarchy of `java.lang` & `Void`

```
                                   Object
        ┌─────────┬───────────┬───────┼─────────┬──────────┬────────┐
     String  StringBuffer StringBuilder Number  Character  Boolean  Void  ...
                                          │
                          ┌───────┬───────┼───────┬────────┐
                        Byte    Short  Integer   Long    Float, Double
```

**Key facts:**

1. `String`, `StringBuffer`, `StringBuilder`, and **all wrapper classes** are declared `final` (cannot be subclassed).
2. Wrapper classes that are **NOT** children of `Number`: `Boolean`, `Character`.
3. Wrapper classes that are **NOT direct children of `Object`** (they extend `Number` first): `Byte`, `Short`, `Integer`, `Long`, `Float`, `Double`.
4. `Void` is sometimes considered a "wrapper class" too.
5. In addition to `String`, **all wrapper class objects are also immutable**.

### `Void`

- Class representation of the `void` keyword.
- Direct child of `Object`.
- Contains **no instance methods**, only **one `public static final Class<Void> TYPE`** field.
- Used in **Reflection** to check if a method's return type is `void`.

```java
// Reflection example: check whether m1() returns void
if (obj.getClass().getMethod("m1").getReturnType() == Void.TYPE) {
    System.out.println("m1() returns void");
}
```

---

## 10. Modern Java Updates (Java 8 → 21+) Relevant to `java.lang`

The source material predates several important JDK evolutions. As a **Senior Architect**, you are expected to know these:

### 10.1 String Pool relocated to Heap (Java 7, JEP-adjacent change)
- Prior to Java 7: SCP lived in **PermGen** — pooled Strings were effectively **never GC'd** until JVM shutdown (matches the legacy text in §3.2).
- **Java 7+**: The String pool moved into the **regular heap**. Consequently, unreferenced interned Strings **CAN now be garbage collected**. PermGen itself was **removed in Java 8**, replaced by **Metaspace** (native memory, not part of heap, for class metadata).

### 10.2 Compact Strings (JEP 254, **Java 9**)
- Prior to Java 9, `String` internally stored characters as `char[]` (2 bytes/char, always UTF-16).
- Since Java 9, `String` internally uses `byte[]` + a `coder` field:
  - **LATIN1** encoding (1 byte/char) when all characters fit in Latin-1.
  - **UTF16** encoding (2 bytes/char) otherwise.
- **Benefit:** Significant heap memory savings for the (very common) case of ASCII/Latin-1-only strings, with **no source-level API changes** — fully transparent to application code.

### 10.3 New `String` Instance Methods

| Method | Since | Purpose |
|---|---|---|
| `isBlank()` | 11 | `true` if empty or only whitespace (uses `Character.isWhitespace`), stricter/more Unicode-aware than manual `trim().isEmpty()`. |
| `strip()`, `stripLeading()`, `stripTrailing()` | 11 | Unicode-aware whitespace trimming (better than legacy `trim()`, which only strips chars `<= U+0020`). |
| `repeat(int count)` | 11 | Repeats the string `count` times. |
| `lines()` | 11 | Returns a `Stream<String>` split by line terminators. |
| `chars()` / `codePoints()` | 8 | `IntStream` of char/code-point values. |
| `formatted(Object... args)` | 15 | Instance-style equivalent of `String.format(this, args)`. |
| `String.join(CharSequence, CharSequence...)` | 8 | Joins strings with a delimiter. |
| `Text Blocks` (`"""..."""`) | 15 (13/14 preview) | Multi-line string literals without escape-heavy concatenation. |

```java
String s = "  Hello World  ";
System.out.println(s.isBlank());          // false
System.out.println("   ".isBlank());       // true
System.out.println(s.strip());              // "Hello World"
System.out.println("ab".repeat(3));          // ababab
"line1\nline2\nline3".lines().forEach(System.out::println);

String formatted = "Name: %s, Age: %d".formatted("Alice", 30);
System.out.println(formatted); // Name: Alice, Age: 30

// Text Block (Java 15+)
String json = """
    {
      "name": "Alice",
      "age": 30
    }
    """;
System.out.println(json);
```

### 10.4 Wrapper Class Constructor Deprecation (Java 9)
- All `new Integer(...)`, `new Boolean(...)`, `new Long(...)`, etc. constructors are `@Deprecated(since="9", forRemoval=true)`.
- Use `valueOf()` (object) or `parseXxx()` (primitive) instead — see §6.1.

### 10.5 `java.util.Objects` Utility Class (Java 7+, heavily used post-8)
```java
import java.util.Objects;

Objects.equals(a, b);          // null-safe equals
Objects.hashCode(obj);         // null-safe hashCode (returns 0 for null)
Objects.hash(a, b, c);         // combines multiple fields into one hash — replaces manual 31*hash+field patterns
Objects.requireNonNull(obj);   // throws NPE with optional custom message — great for constructor validation
Objects.requireNonNullElse(obj, "default");  // Java 9+
Objects.toString(obj, "N/A"); // null-safe toString with fallback
```

### 10.6 `var` — Local Variable Type Inference (**Java 10**, JEP 286)
```java
var list = new ArrayList<String>();   // inferred as ArrayList<String>
var s = "hello";                       // inferred as String
// Applicable to local variables/for-loop variables/try-with-resources only — NOT fields, method params, or return types.
```

### 10.7 Pattern Matching for `instanceof` (**Java 16**, JEP 394)
- Directly relevant to `equals()` overriding (§2.4) — eliminates the manual cast.
```java
// Old style
if (o instanceof Student) {
    Student s2 = (Student) o;
    ...
}
// Modern style (Java 16+)
if (o instanceof Student s2) {
    // s2 is already cast and in scope
    ...
}
```

### 10.8 `record` — Immutable Data Carriers (**Java 16**, JEP 395; preview in 14/15)
- Directly relevant to §3.6 (custom immutable classes) and §2.4-2.6 (`equals`/`hashCode`/`toString`).
- The compiler **auto-generates**: private final fields, a canonical constructor, accessor methods (`name()`, not `getName()`), and correctly-contracted `equals()`, `hashCode()`, `toString()`.

```java
public record Student(String name, int rollno) { }

// Equivalent hand-written class would need ~40 lines (constructor, accessors,
// equals/hashCode honoring the contract from §2.6, toString like §2.2)

Student s1 = new Student("vijayabhaskar", 101);
Student s2 = new Student("vijayabhaskar", 101);
System.out.println(s1.equals(s2));   // true  (field-by-field, contract-correct)
System.out.println(s1.hashCode() == s2.hashCode());  // true (contract honored automatically)
System.out.println(s1);              // Student[name=vijayabhaskar, rollno=101]
System.out.println(s1.name());       // vijayabhaskar (no "get" prefix)
```
> **Architect note:** Records are implicitly `final` and extend an implicit sealed-like base — they solve most of the "create your own immutable class" boilerplate discussed in §3.6, and guarantee the `equals`/`hashCode` contract from §2.6 correctly, by construction.

### 10.9 Sealed Classes (**Java 17**, JEP 409)
```java
public sealed interface Shape permits Circle, Square { }
public final class Circle implements Shape { }
public final class Square implements Shape { }
```
- Restricts which classes may implement/extend a type — complements `switch` pattern matching (Java 21) for exhaustive type hierarchies.

### 10.10 Pattern Matching for `switch` (**Java 21**, JEP 441, finalized)
```java
Object obj = "hello";
String result = switch (obj) {
    case Integer i -> "int " + i;
    case String s when s.length() > 3 -> "long string " + s;
    case String s -> "short string " + s;
    default -> "unknown";
};
```

### 10.11 Virtual Threads (**Java 21**, JEP 444)
- Directly relevant to §2.11 (`wait/notify`) — for new highly-concurrent code, prefer virtual threads (`Thread.ofVirtual().start(...)`) with structured concurrency APIs over manually managing monitors, though `wait/notify` fundamentals remain unchanged underneath.

### 10.12 `finalize()` Deprecation Path (Java 9 → 18+)
- See §2.10. Deprecated since Java 9; JEP-tracked for eventual removal. Use `Cleaner` or `try-with-resources`/`AutoCloseable`.

### 10.13 Helpful `NullPointerException` Messages (**Java 14**, JEP 358)
- `-XX:+ShowCodeDetailsInExceptionMessages` (default **on** from Java 15) — NPE messages now describe **exactly which** variable/method-call was `null`, e.g.:
  `Cannot invoke "String.length()" because "s" is null` — massively helps debug the Autounboxing NPE trap from §7.

---

## 11. Interview Questions Cheat Sheet

### Conceptual / Theory

1. **Why is `java.lang` not required to be explicitly imported?** — It's implicitly imported by the compiler into every source file.
2. **What is the default implementation of `Object.equals()`?** — Reference comparison (`this == obj`).
3. **What is the default implementation of `Object.hashCode()`?** — JVM/implementation-specific unique integer (native method), not guaranteed to represent memory address.
4. **Why must you override `hashCode()` whenever you override `equals()`?** — To satisfy the equals-hashCode contract (§2.6); breaking it corrupts `HashMap`/`HashSet` behavior silently (no compile/runtime error).
5. **Difference between `==` and `.equals()`?** — Operator vs method; reference vs (potentially) content comparison; see full table in §2.5.
6. **Why does `StringBuffer.equals()` return `false` for two objects with identical content?** — `StringBuffer` never overrides `equals()`; it uses `Object`'s reference-based version. Only `String`, wrapper classes, and Collections override it.
7. **What is the String Constant Pool (SCP), and why does it exist?** — A shared pool of `String` literals to avoid creating duplicate objects for identical content, improving performance/memory. See §3.2.
8. **Why is `String` immutable?** — A direct consequence of SCP: since many references may point to the same pooled object, mutability would corrupt all of them. Immutability is the safeguard.
9. **Where does the `String` pool live — heap or PermGen?** — Since Java 7, it's in the **heap** (previously PermGen pre-Java 7). PermGen itself was removed in Java 8 (replaced by Metaspace).
10. **What does `String.intern()` do?** — Returns the SCP-resident equivalent of a heap String; creates the SCP entry if absent.
11. **Difference between `String`, `StringBuffer`, `StringBuilder`?** — Mutability + thread-safety; see decision matrix in §5.
12. **Why was `StringBuilder` introduced in 1.5 if `StringBuffer` already existed?** — To eliminate the `synchronized`-method performance overhead when thread-safety isn't required.
13. **What is Shallow Cloning vs Deep Cloning?** — Shallow copies primitives directly but shares nested object references; Deep Cloning independently duplicates nested objects too. See §2.8.
14. **Why is `Cloneable` called a marker interface?** — It has zero methods; its mere presence signals to the JVM that cloning is permitted.
15. **What happens if you call `clone()` on a class that doesn't implement `Cloneable`?** — Runtime `CloneNotSupportedException`.
16. **What's wrong with `Object.clone()`/`Cloneable` from a design perspective?** — No enforced deep-copy semantics, exception-based control flow, doesn't work well with `final` fields; *Effective Java* recommends copy constructors/factories instead.
17. **What is `final` vs immutability?** — `final` locks a variable's reference/reassignment; immutability locks an object's internal state. They're orthogonal (§3.7).
18. **What does Autoboxing/Autounboxing mean, and how are they implemented internally?** — Compiler-inserted conversions using `valueOf()` (boxing) and `xxxValue()` (unboxing), introduced in Java 5.
19. **Explain the Integer Cache. Why does `Integer x=127,y=127; x==y` return `true` but `128` returns `false`?** — Because `-128..127` values are pre-cached at class-loading time (§7); autoboxing reuses cached instances within that range; `new Integer()` always bypasses the cache (and is deprecated since Java 9).
20. **Which wrapper classes support caching, and over what ranges?** — `Byte` (always), `Short/Integer/Long` (-128..127), `Character` (0..127), `Boolean` (always); `Float`/`Double` are **never** cached.
21. **Overload resolution precedence: widening vs autoboxing vs var-args?** — Widening > Autoboxing > Var-args (§8).
22. **Can "widening then autoboxing" happen in a single implicit call?** — No — illegal combination. But "autoboxing then reference-widening" (e.g., `int` → `Integer` → `Object`) IS legal.
23. **Which wrapper classes are NOT children of `Number`?** — `Boolean`, `Character`.
24. **Which wrapper classes are NOT DIRECT children of `Object`?** — `Byte`, `Short`, `Integer`, `Long`, `Float`, `Double` (they extend `Number` first).
25. **What is `Void.TYPE` used for?** — Reflection — checking whether a method's declared return type is `void`.
26. **Why is `Object.finalize()` deprecated, and what replaces it?** — Deprecated since Java 9 due to unpredictable timing, performance issues, and security risks; replaced by `try-with-resources`/`AutoCloseable` or `java.lang.ref.Cleaner`.
27. **What are Compact Strings (Java 9)?** — Internal `String` storage switched from always-`char[]` to `byte[]` + coder flag (LATIN1/UTF16), saving memory for ASCII-heavy strings — fully transparent to the API.
28. **How do Java `record`s relate to the "create your own immutable class" pattern?** — Records (Java 16+) auto-generate the constructor, accessors, and a contract-correct `equals()/hashCode()/toString()`, eliminating hand-written immutable-class boilerplate.
29. **What is the classic NPE trap with Autounboxing?** — Unboxing a `null` wrapper (e.g., `int i = someNullInteger;`) throws `NullPointerException` at the point of unboxing — very common in real-world bugs involving `Long`/`Integer` fields from databases/JPA entities.
30. **Difference between `parseXxx()` and `valueOf()`?** — `parseXxx()` returns a **primitive**; `valueOf()` returns a **wrapper object** (and may hit the cache).

### Predict-the-Output / Code-Trace Style

```java
// Q1
String s1 = new String("java");
String s2 = "java";
System.out.println(s1 == s2);          // false
System.out.println(s1.equals(s2));     // true

// Q2
Integer a = 100, b = 100;
Integer c = 200, d = 200;
System.out.println(a == b);            // true  (cached)
System.out.println(c == d);            // false (not cached)

// Q3
final String x = "abc";
String y = x + "def";
String z = "abcdef";
System.out.println(y == z);            // true (compile-time constant folding, x is final+literal)

// Q4
String p = "abc";
String q = new String("abc").intern();
System.out.println(p == q);            // true (intern() returns SCP reference)

// Q5
StringBuffer sb1 = new StringBuffer("test");
StringBuffer sb2 = new StringBuffer("test");
System.out.println(sb1.equals(sb2));   // false (StringBuffer doesn't override equals())
System.out.println(sb1.toString().equals(sb2.toString())); // true

// Q6
Integer i = null;
int j = i;                              // NullPointerException at runtime (autounboxing null)
```

### Quick-Reference Cheat Sheet

| Topic | Golden Rule |
|---|---|
| `equals()` override | Must check `this==o`, `instanceof`, then compare fields. Never throw CCE/NPE. |
| `hashCode()` override | MUST use the same fields as `equals()`. Equal objects ⇒ equal hash (mandatory); unequal objects ⇒ hash may collide (allowed). |
| `String` immutability | Any change → new object; no change → same object reused. |
| SCP | Literals only; `new` always goes to heap; `intern()` bridges heap→SCP. |
< /* end of quick table row */ | |
| `StringBuffer` vs `StringBuilder` | Thread-safety (`synchronized`) is the ONLY functional difference. |
| Wrapper caching | `-128..127` for `Short/Integer/Long`; `0..127` for `Character`; always for `Byte/Boolean`; never for `Float/Double`. |
| Overload precedence | Widening → Autoboxing → Var-args. |
| `clone()` | Requires `Cloneable`; default is shallow; deep clone needs manual override. |
| `finalize()` | Deprecated (Java 9+); use `Cleaner`/`AutoCloseable`. |
| Immutable class recipe | `final` class, `private final` fields, no setters, defensive copies for mutable fields — or just use a `record` (Java 16+). |

---

*Document compiled and technically validated for Senior Java Architect interview preparation — covering both classic SCJP/OCJP fundamentals and modern Java (9–21) evolutions of the `java.lang` package.*
