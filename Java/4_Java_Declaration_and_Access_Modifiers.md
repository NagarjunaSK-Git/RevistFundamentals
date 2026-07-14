# Core Java — Declaration & Access Modifiers
### Senior Java Architect Interview-Prep Study Guide (Chapter 4)

> Source: DURGASOFT Core Java with SCJP/OCJP study material — restructured, expanded,
> and validated for architect-level interview preparation. Every code snippet below
> compiles/runs on a standard JDK (8–21) unless a version note says otherwise.

---

## Table of Contents

1. [Java Source File Structure](#1-java-source-file-structure)
2. [Import Statements](#2-import-statements)
3. [Package Statement](#3-package-statement)
4. [Overall Source File Structure & Ordering Rules](#4-overall-source-file-structure--ordering-rules)
5. [Class-Level (Top-Level) Modifiers](#5-class-level-top-level-modifiers)
6. [Member Modifiers (Access Control)](#6-member-modifiers-access-control)
7. [`final` Variables (Instance / Static / Local)](#7-final-variables-instance--static--local)
8. [`static` Modifier](#8-static-modifier)
9. [`native` Modifier](#9-native-modifier)
10. [`synchronized` Modifier](#10-synchronized-modifier)
11. [`transient` Modifier](#11-transient-modifier)
12. [`volatile` Modifier](#12-volatile-modifier)
13. [Modifier Summary Matrix](#13-modifier-summary-matrix)
14. [Interfaces — Complete Deep Dive](#14-interfaces--complete-deep-dive)
15. [Interface vs Abstract Class vs Concrete Class](#15-interface-vs-abstract-class-vs-concrete-class)
16. [Why Abstract Classes Have Constructors (Deep Dive)](#16-why-abstract-classes-have-constructors-deep-dive)
17. [Interview Q&A Bank (Answered)](#17-interview-qa-bank-answered)
18. [Quick-Reference Cheat Sheet](#18-quick-reference-cheat-sheet)

---

## 1. Java Source File Structure

### Core Rule
A `.java` file can contain **any number of classes**, but **at most one `public` class**.

> **Rule:** If a public class exists, the file name **must** match the public class
> name exactly (case-sensitive), otherwise you get a **compile-time error**.
> If there is **no** public class, the file may be named anything.

### Case 1 — No public class → any file name allowed

```java
// Can be saved as A.java, B.java, C.java, or even Ashok.java — all valid
class A { }
class B { }
class C { }
```

### Case 2 — One public class → file name must match

```java
// Must be saved as B.java
class A { }
public class B { }
class C { }
```
If saved as anything else:
```
error: class B is public, should be declared in a file named B.java
```

### Case 3 — Two public classes in one file → always a compile error

```java
public class B { }
public class C { }   // ERROR even if file is named B.java
```
```
class C is public, should be declared in a file named C.java
```

**Best practice:** One class per file; file name == class name. Improves readability & maintainability.

### Compilation vs Execution — Class Files Are Independent

```java
// Bhaskar.java (file name is irrelevant here — no public class)
class A {
    public static void main(String args[]) {
        System.out.println("A class main method is executed");
    }
}
class B {
    public static void main(String args[]) {
        System.out.println("B class main method is executed");
    }
}
class C {
    public static void main(String args[]) {
        System.out.println("C class main method is executed");
    }
}
class D { }
```

```
D:\Java>javac Bhaskar.java     → generates A.class, B.class, C.class, D.class

D:\Java>java A
A class main method is executed
D:\Java>java B
B class main method is executed
D:\Java>java C
C class main method is executed
D:\Java>java D
Exception in thread "main" java.lang.NoSuchMethodError: main
D:\Java>java Ashok
Exception in thread "main" java.lang.NoClassDefFoundError: Ashok
```

### Key Architect-Level Takeaways
| Statement | Explanation |
|---|---|
| Compilation is at the **file** level | `javac` compiles every class in the file, producing one `.class` per class. |
| Execution is at the **class** level | `java <ClassName>` only cares about that class's `main` method. |
| `NoSuchMethodError: main` | Thrown at **runtime** when the class exists (`.class` found) but has no `main(String[])`. |
| `NoClassDefFoundError` | Thrown at **runtime** when the JVM cannot locate the requested `.class` file at all. |
| An **empty source file** is a 100% valid Java program | `javac Empty.java` compiles successfully with zero output classes. |

---

## 2. Import Statements

### Why Import Is Needed
Without `import`, referring to a class outside `java.lang` forces you to use the **fully qualified name (FQN)** every time:

```java
class Test {
    public static void main(String args[]) {
        ArrayList l = new ArrayList();   // COMPILE ERROR: cannot find symbol
    }
}
```
```
Test.java:3: cannot find symbol
symbol  : class ArrayList
location: class Test
```

**Fix 1 — Fully Qualified Name (verbose, but no import required):**
```java
class Test {
    public static void main(String args[]) {
        java.util.ArrayList l = new java.util.ArrayList();  // compiles fine
    }
}
```

**Fix 2 — Import statement (preferred, improves readability):**
```java
import java.util.ArrayList;
class Test {
    public static void main(String args[]) {
        ArrayList l = new ArrayList();   // compiles fine
    }
}
```

> **Golden Rule:** Whenever you use a **fully qualified name**, an `import` is **not required**.
> Whenever you use an `import`, the FQN is **not required**. They are two alternative solutions to the same problem.

### 2.1 Types of Import Statements

| Type | Syntax | Recommendation |
|---|---|---|
| **Explicit (single-type) import** | `import java.util.ArrayList;` | ✅ Highly recommended — improves readability ("Hi-Tech City" style — precision matters) |
| **Implicit (on-demand) import** | `import java.util.*;` | ❌ Never recommended in production code — reduces readability ("Ameerpet" style — convenience over clarity) |

### 2.2 Case Studies on Import Validity

**Case 1 — Which imports are syntactically meaningful?**

| Statement | Valid? |
|---|---|
| `import java.util;` | ❌ Invalid — can't import a package directly |
| `import java.util.ArrayList.*;` | ❌ Invalid — can't do wildcard on a class |
| `import java.util.*;` | ✅ Valid |
| `import java.util.ArrayList;` | ✅ Valid |

**Case 2 — FQN eliminates need for import entirely:**
```java
class MyArrayList extends java.util.ArrayList { }   // Compiles fine — no import needed
```

**Case 3 — Ambiguity between two normal imports:**
```java
import java.util.*;
import java.sql.*;
class Test {
    public static void main(String args[]) {
        Date d = new Date();   // COMPILE ERROR
    }
}
```
```
Test.java:7: reference to Date is ambiguous,
both class java.sql.Date in java.sql and class java.util.Date in java.util match
```
> Note: `List` has the same ambiguity risk (`java.util.List` vs `java.awt.List`).

**Case 4 — Precedence order the compiler follows to resolve simple class names:**
1. **Explicit import** (highest precedence)
2. **Classes present in the current working directory (default package)**
3. **Implicit (`*`) import** (lowest precedence)

```java
import java.util.Date;   // explicit → wins
import java.sql.*;       // implicit
class Test {
    public static void main(String args[]) {
        Date d = new Date();   // Compiles fine — java.util.Date is used
    }
}
```

**Case 5 — Importing a package imports classes at that level ONLY, not sub-packages:**
```
java.util.regex.Pattern
```
To use `Pattern` directly:

| Import | Works? |
|---|---|
| `import java.*;` | ❌ |
| `import java.util.*;` | ❌ (regex is a sub-package, not included) |
| `import java.util.regex.*;` | ✅ |
| `import java.util.regex.Pattern;` | ✅ |

**Case 6 — Two packages that never require an explicit import:**
1. `java.lang` package (String, Math, System, Object, Thread, Exception, etc.)
2. The **default package** (current working directory)

**Case 7 — Import is a purely compile-time concept:**
> More imports → longer compile time. **Zero impact on runtime performance.**

### 2.3 C `#include` vs Java `import`

| `#include` (C/C++) | `import` (Java) |
|---|---|
| Usable in C & C++ | Usable only in Java |
| Compiler physically copies header code into the current file at compile time | JVM loads the referenced `.class` at runtime, on demand |
| **Static inclusion** | **Dynamic inclusion** |
| Wastes memory (code duplicated per translation unit) | No memory wastage |
| Analogous to JSP `<%@ include file="" %>` (static include) | Analogous to `<jsp:include>` (dynamic include) |

> Java's `import` does **not** load `.class` files at the import line. Loading happens
> **on first actual use** of that class — this is called **"load-on-demand"** or **"load-on-fly"**.

### 2.4 Java 1.5 New Features (Context Reference)
1. For-Each loop
2. Var-args
3. `Queue` interface
4. Generics
5. Autoboxing / Auto-unboxing
6. Co-variant return types
7. Annotations
8. `enum`
9. Static import
10. `StringBuilder`

### 2.5 Static Import

Introduced in Java 1.5. Allows accessing `static` members **without the class name qualifier**.

> **Opinion in industry (and in this material):** Sun/Oracle claims it improves
> readability; most experienced developers consider it a **readability hazard** —
> use it sparingly, if at all.

**Without static import:**
```java
class Test {
    public static void main(String args[]) {
        System.out.println(Math.sqrt(4));
        System.out.println(Math.max(10, 20));
        System.out.println(Math.random());
    }
}
```

**With static import:**
```java
import static java.lang.Math.sqrt;
import static java.lang.Math.*;

class Test {
    public static void main(String args[]) {
        System.out.println(sqrt(4));   // no "Math." prefix
        System.out.println(max(10, 20));
        System.out.println(random());
    }
}
```

### 2.6 What Exactly Is `System.out.println`?

```java
class Test {
    static String name = "bhaskar";
}
```
| Expression | Breakdown |
|---|---|
| `Test.name.length()` | `Test` → class · `name` → static `String` variable in `Test` · `length()` → method in `String` class |
| `System.out.println()` | `System` → class in `java.lang` · `out` → static variable of type `PrintStream` in `System` · `println()` → method in `PrintStream` |

**Static import applied to `System.out`:**
```java
import static java.lang.System.out;
class Test {
    public static void main(String args[]) {
        out.println("hello");
        out.println("hi");
    }
}
```

### 2.7 Ambiguity in Static Imports (Very Common Interview Trap)

```java
import static java.lang.Integer.*;
import static java.lang.Byte.*;

class Test {
    public static void main(String args[]) {
        System.out.println(MAX_VALUE);   // COMPILE ERROR
    }
}
```
```
Test.java:6: reference to MAX_VALUE is ambiguous,
both variable MAX_VALUE in java.lang.Integer and variable MAX_VALUE in java.lang.Byte match
```

> **Important distinction:** Ambiguity between **two classes/interfaces with the same
> name** across packages is *rare* in ordinary imports. But ambiguity between
> **two static members with the same name** across classes is *very common* in
> static imports — this is the #1 argument against overusing static import.

**Precedence order for resolving static members:**
1. Current class's own static members (highest)
2. Explicit static import (`import static pkg.Class.member;`)
3. Implicit static import (`import static pkg.Class.*;`) (lowest)

```java
//import static java.lang.Integer.MAX_VALUE;  ← line 2 (commented)
import static java.lang.Byte.*;

class Test {
    //static int MAX_VALUE = 999;             ← line 1 (commented)
    public static void main(String args[]) throws Exception {
        System.out.println(MAX_VALUE);
    }
}
```
- Uncomment only **line 1** → prints `999` (current class wins).
- Comment out line 1, uncomment **line 2** → prints `2147483647` (Integer, explicit import wins over implicit `Byte.*`).
- Neither uncommented → prints `127` (Byte, the only remaining candidate).

### 2.8 Validity Table for Static Imports

| Statement | Valid? |
|---|---|
| `import java.lang.Math.*;` | ❌ (missing `static` keyword; wrong syntax for normal import too) |
| `import static java.lang.Math.*;` | ✅ |
| `import java.lang.Math;` | ✅ |
| `import static java.lang.Math;` | ❌ (can't static-import a class itself) |
| `import static java.lang.Math.sqrt.*;` | ❌ (`sqrt` is a method, not a type) |
| `import java.lang.Math.sqrt;` | ❌ (normal import can't target a method) |
| `import static java.lang.Math.sqrt();` | ❌ (no parentheses in import statements) |
| `import static java.lang.Math.sqrt;` | ✅ |

**Pattern to memorize:**
```
Normal import starts with →  ClassName;         or  package.*;
Static import starts with →  ClassName.member;  or  ClassName.*;
```

### 2.9 General Import vs Static Import

| Normal Import | Static Import |
|---|---|
| Imports classes/interfaces of a package | Imports **static members** of a specific class |
| Access classes/interfaces via short name (no FQN needed) | Access static members directly (no class-name qualifier needed) |

---

## 3. Package Statement

### Definition
A **package** is an encapsulation mechanism to group related classes/interfaces into a single module (namespace).

### Objectives
1. Resolve naming conflicts
2. Improve modularity
3. Provide access-control based security
4. Universally-accepted naming convention: **reverse of your internet domain name**

```
com.icicibank.loan.housingloan.Account
 └────┬────┘ └──┬─┘ └────┬────┘  └──┬──┘
  domain(rev)  module   submodule   class
```

### Compiling a Package Program

```java
package com.durgajobs.itjobs;
class HydJobs {
    public static void main(String args[]) {
        System.out.println("package demo");
    }
}
```

- **Without `-d`:** `javac HydJobs.java` → `.class` lands in the current working directory (CWD), *not* inside the package folder structure.
- **With `-d`:** `javac -d . HydJobs.java` → `-d` = destination for generated `.class` files; `.` = CWD. The compiler **auto-creates** the folder structure `com/durgajobs/itjobs/HydJobs.class` if it doesn't already exist.
- You can target any valid directory: `javac -d C:\ HydJobs.java` → creates `C:\com\durgajobs\itjobs\HydJobs.class`.
- If the destination itself doesn't exist (e.g., drive `Z:` doesn't exist) → **compile-time error**.

### Executing a Package Program

```
D:\Java>java com.durgajobs.itjobs.HydJobs
```
> At execution time, you **must** supply the fully qualified class name.

### Conclusion 1 — At Most One `package` Statement

```java
package pack1;
package pack2;      // COMPILE ERROR
class A { }
```
```
A.java:2: class, interface, or enum expected
package pack2;
```

### Conclusion 2 — `package` Must Be the First Non-Comment Statement

```java
import java.util.*;
package pack1;      // COMPILE ERROR — import came first
class A { }
```
```
A.java:2: class, interface, or enum expected
package pack1;
```

---

## 4. Overall Source File Structure & Ordering Rules

```
┌─────────────────────────────────────┐
│  At most ONE   → package statement  │   ┐
│  Any number    → import statements  │   │  Order is IMPORTANT
│  Any number    → class/interface/   │   │  (must appear in this sequence)
│                  enum declarations  │   ┘
└─────────────────────────────────────┘
```

**All of the following are valid `Test.java` programs:**

| # | Content | Valid |
|---|---|---|
| 1 | *(completely empty file)* | ✅ |
| 2 | `package pack1;` | ✅ |
| 3 | `import java.util.*;` | ✅ |
| 4 | `package pack1;`<br>`import java.util.*;` | ✅ |
| 5 | `class Test { }` | ✅ |

> **Interview one-liner:** *"An empty `.java` source file is a completely valid Java program."*

---

## 5. Class-Level (Top-Level) Modifiers

Every class declaration communicates 3 things to the JVM:
1. Accessibility (from where can it be used?)
2. Whether it can be subclassed
3. Whether it can be instantiated

### 5.1 Only Applicable Modifiers for Top-Level Classes

```
public | <default> | final | abstract | strictfp
```
Using any other modifier (`private`, `protected`, `static`, `synchronized`, `native`, `transient`, `volatile`) on a **top-level** class → **compile-time error**.

```java
private class Test { }
```
```
Test.java:1: modifier private not allowed here
private class Test
```

### 5.2 Extra Modifiers Allowed for INNER Classes

Inner (nested) classes get **two extra options** beyond the top-level set:

```
Top-level set: public, <default>, final, abstract, strictfp
     +
Inner-only extras: private, protected, static
```

### 5.3 Access Specifier vs Access Modifier
- In older languages (C / C++): `public`, `private`, `protected`, `<default>` were called **access specifiers**; everything else was an **access modifier**.
- **In Java: there is no such distinction.** All of them are uniformly called **modifiers**.

### 5.4 Public Classes

A `public` class is accessible **from anywhere** — inside or outside its package.

```java
// pack1/Test.java
package pack1;
public class Test {
    public void methodOne() {
        System.out.println("test class methodOne is executed");
    }
}
```
```java
// pack2/Test1.java
package pack2;
import pack1.Test;
class Test1 {
    public static void main(String args[]) {
        Test t = new Test();
        t.methodOne();
    }
}
```
```
D:\Java>javac -d . pack1\Test.java
D:\Java>javac -d . pack2\Test1.java
D:\Java>java pack2.Test1
test class methodOne is executed.
```
> If `Test` were **not** `public`, compiling `Test1` would fail with:
> `pack1.Test is not public in pack1; cannot be accessed from outside package`

### 5.5 Default (Package-Level) Classes

A default-access class is accessible **only within its own package** — hence "package-level access."

```java
// pack1/Test.java
package pack1;
class Test {
    public void methodOne() {
        System.out.println("methodOne is executed");
    }
}
```
```java
// pack1/Test1.java
package pack1;
class Test1 {
    public static void main(String args[]) {
        Test t = new Test();
        t.methodOne();     // OK — same package
    }
}
```

### 5.6 `final` Modifier — Applicable to Classes, Methods, and Variables

#### `final` Methods
A `final` method **cannot be overridden** by any subclass.

```java
class Parent {
    public void property() {
        System.out.println("cash+gold+land");
    }
    public final void marriage() {
        System.out.println("subbalakshmi");
    }
}
class Child extends Parent {
    public void marriage() {     // COMPILE ERROR
        System.out.println("thamanna");
    }
}
```
```
Child.java:3: marriage() in Child cannot override marriage() in Parent;
overridden method is final
```

#### `final` Class
A `final` class **cannot be subclassed** — inheritance is blocked entirely.

```java
final class Parent { }
class Child extends Parent { }   // COMPILE ERROR
```
```
Child.java:1: cannot inherit from final Parent
```

> **Important nuance:** Every method inside a `final` class is **implicitly `final`**
> (whether declared so or not) — because there can never be a subclass to override it anyway.
> However, **variables** inside a `final` class are **NOT** implicitly final.

```java
final class Parent {
    static int x = 10;
    static {
        x = 999;    // legal — x is not final
    }
}
```

**Advantage/Disadvantage trade-off:**
- ✅ Advantage: Security (guarantees behavior/structure can't be altered by subclassing).
- ❌ Disadvantage: Loses two core OOP benefits — **polymorphism** (blocked by final methods) and **inheritance** (blocked by final classes). Use `final` on classes/methods only when there's a specific design reason (immutability, security-sensitive APIs).

### 5.7 `abstract` Modifier — Applicable Only to Classes and Methods (NOT Variables)

#### `abstract` Methods
Declaration only, **no body**, must end with `;`.

```java
public abstract void methodOne();      // ✅ valid
public abstract void methodOne(){}     // ❌ invalid — abstract methods cannot have a body
```

The child class is **responsible** for providing the implementation:

```java
abstract class Vehicle {
    public abstract int getNoOfWheels();
}
class Bus extends Vehicle {
    public int getNoOfWheels() { return 7; }
}
class Auto extends Vehicle {
    public int getNoOfWheels() { return 3; }
}
```

#### Illegal Modifier Combinations for Methods (with `abstract`)

`abstract` **never talks about implementation**; any modifier that *does* talk about implementation is illegal alongside it:

```
abstract + final           → ILLEGAL (final implies "cannot override," contradicts abstract's requirement to override)
abstract + static          → ILLEGAL (static methods are resolved at compile time, not overridden)
abstract + synchronized    → ILLEGAL (synchronized requires a method body/lock)
abstract + native          → ILLEGAL (native implies external implementation exists)
abstract + strictfp        → ILLEGAL (strictfp governs floating-point *implementation*)
abstract + private         → ILLEGAL (private methods aren't visible/overridable in subclasses)
```
**All 6 combinations above are illegal for methods.**

#### `abstract` Class

Cannot be instantiated — object creation is blocked.

```java
abstract class Test {
    public static void main(String args[]) {
        Test t = new Test();   // COMPILE ERROR
    }
}
```
```
Test.java:4: Test is abstract; cannot be instantiated
```

#### Abstract Class ↔ Abstract Method Relationship

- If a class has **at least one** abstract method → the class **must** be declared `abstract`.
- The reverse is **not** required: an abstract class **can have zero abstract methods**.
  - Example: `HttpServlet` is abstract but has no abstract methods.
  - Example: Every AWT/Swing **Adapter class** (`MouseAdapter`, `KeyAdapter`) is abstract with zero abstract methods.

```java
class Parent {
    public void methodOne();      // COMPILE ERROR
}
```
```
missing method body, or declare abstract
```

```java
class Parent {
    public abstract void methodOne(){}   // COMPILE ERROR
}
```
```
abstract methods cannot have a body
```

```java
class Parent {
    public abstract void methodOne();    // COMPILE ERROR — class itself isn't abstract
}
```
```
Parent is not abstract and does not override abstract method methodOne() in Parent
```

**Extending an abstract class → must implement ALL abstract methods, or declare the child abstract too:**

```java
abstract class Parent {
    public abstract void methodOne();
    public abstract void methodTwo();
}
class Child extends Parent {
    public void methodOne(){}     // COMPILE ERROR — methodTwo() still unimplemented
}
```
```
Child is not abstract and does not override abstract method methodTwo() in Parent
```
> If `Child` is declared `abstract`, compilation succeeds — but then **grandchild** class must implement `methodTwo()`.

### 5.8 `final` vs `abstract` — Direct Comparison

| Aspect | `abstract` | `final` |
|---|---|---|
| Methods | Must be **overridden** by subclass | Cannot be **overridden** |
| Classes | Must be **subclassed** to be useful | Cannot be **subclassed** |
| Combination | `abstract final` on methods → **ILLEGAL** | `abstract final` on classes → **ILLEGAL** |
| Interaction | A **`final` class cannot contain `abstract` methods** (it can never be implemented) | An **`abstract` class CAN contain `final` methods** |

```java
final class A {
    public abstract void methodOne();   // ❌ INVALID
}

abstract class A {
    public final void methodOne(){}     // ✅ VALID
}
```

### 5.9 `strictfp` Modifier

- Applicable to **classes and methods**, NOT variables.
- Introduced in Java 1.2.
- Floating-point arithmetic results can vary across CPU/platform architectures (extended precision vs standard IEEE 754). `strictfp` forces **strict adherence to the IEEE 754 standard**, guaranteeing **platform-independent** results.

```java
System.out.println(10.0 / 3);
```
| Platform variant | Result |
|---|---|
| P4-style extended precision | `3.33333333333333` |
| P3-style | `3.333333` |
| IEEE 754 (strictfp guaranteed) | `3.333...` (consistent everywhere) |

> If a class is declared `strictfp`, **every concrete (non-abstract) method** in that class automatically follows IEEE 754 rules.

> **Note:** As of Java 17, `strictfp` is a no-op keyword (JEP 306 made strict floating-point semantics the default for all code), but it remains **syntactically valid** and is a classic OCJP/interview topic for legacy-version reasoning.

### 5.10 `abstract` vs `strictfp`

| Combination | Legal? |
|---|---|
| `abstract` + `strictfp` for a **class** | ✅ Legal |
| `abstract` + `strictfp` for a **method** | ❌ Illegal |

```java
public abstract strictfp void methodOne();   // ❌ INVALID (method)

abstract strictfp class Test { }             // ✅ VALID (class)
```

**Reasoning:** `strictfp` talks about *how floating-point implementation behaves*; `abstract` methods have *no implementation at all* — direct contradiction. But a class can carry both flags simultaneously since the class-level `strictfp` just cascades down to whichever concrete methods eventually exist.

---

## 6. Member Modifiers (Access Control)

### 6.1 `public` Members

Accessible from **anywhere** — **but the enclosing class must also be visible**. Always check class-level visibility **first**, then member-level visibility.

```java
// pack1/A.java
package pack1;
class A {                      // default (package) access
    public void methodOne() {
        System.out.println("A class method");
    }
}
```
```java
// pack2/B.java
package pack2;
import pack1.A;
class B {
    public static void main(String args[]) {
        A a = new A();
        a.methodOne();          // COMPILE ERROR
    }
}
```
```
B.java:2: pack1.A is not public in pack1; cannot be accessed from outside package
```
> Even though `methodOne()` is `public`, class `A` itself is only default-access — so the method is unreachable from another package.

### 6.2 Default Members ("Package-Level Access")

Accessible **only within the same package**.

```java
package pack1;
class A {
    void methodOne() {
        System.out.println("methodOne is executed");
    }
}
class B {
    public static void main(String args[]) {
        A a = new A();
        a.methodOne();          // OK — same package
    }
}
```
> If `B` were in `pack2` instead, this would fail even if `import pack1.A;` were used and `A` were public but `methodOne()` had default access.

### 6.3 `private` Members

- Accessible **only within the declaring class**.
- **Not inherited/visible** in child classes at all.
- `private` + `abstract` combination is **illegal for methods** — abstract methods must be visible to child classes to be overridden, but `private` hides them entirely.

### 6.4 `protected` Members

**Rule of thumb: `protected` = `default` + `kids` (visible to subclasses even outside the package).**

- **Same package (anywhere):** accessible via either **child reference** or **parent reference**.
- **Different package:** accessible **only inside child classes**, and **only through a child-class reference** (NOT through a parent-class reference) — this is the classic trap.

```java
// pack1/A.java
package pack1;
public class A {
    protected void methodOne() {
        System.out.println("methodOne is executed");
    }
}
```

**Same-package child class — both parent & child refs work:**
```java
// pack1/B.java
package pack1;
class B extends A {
    public static void main(String args[]) {
        A a = new A();
        a.methodOne();          // ✅ OK (same package)
        B b = new B();
        b.methodOne();          // ✅ OK
        A a1 = new B();
        a1.methodOne();         // ✅ OK — parent reference, same package
    }
}
```

**Cross-package child class — ONLY child reference works:**
```java
// pack2/C.java
package pack2;
import pack1.A;
public class C extends A {
    public static void main(String args[]) {
        A a = new A();
        a.methodOne();          // ❌ COMPILE ERROR — parent ref, cross-package

        C c = new C();
        c.methodOne();          // ✅ OK — child ref, inherited method

        A a1 = new B();         // (hypothetical B from pack1 — also fails)
        a1.methodOne();         // ❌ COMPILE ERROR — parent-typed ref
    }
}
```
```
C.java:7: methodOne() has protected access in pack1.A
```

### 6.5 Comprehensive Access Comparison Table

| Caller's location | `private` | `default` | `protected` | `public` |
|---|:---:|:---:|:---:|:---:|
| 1) Same class | ✅ | ✅ | ✅ | ✅ |
| 2) Child class, same package | ❌ | ✅ | ✅ | ✅ |
| 3) Non-child class, same package | ❌ | ✅ | ✅ | ✅ |
| 4) Child class, different package | ❌ | ❌ | ✅ **(child reference only)** | ✅ |
| 5) Non-child class, different package | ❌ | ❌ | ❌ | ✅ |

**Accessibility ordering (least → most accessible):**
```
private  <  default  <  protected  <  public
```

> **Recommended practice:** variables → `private` (encapsulation); methods → `public` (well-defined API surface), with getters/setters mediating access to state.

---

## 7. `final` Variables (Instance / Static / Local)

### 7.1 Instance Variables (Recap)
- Value **varies per object**; each object gets its own copy.
- JVM auto-assigns **default values** (0, 0.0, false, null) if not explicitly initialized.

```java
class Test {
    int i;
    public static void main(String args[]) {
        Test t = new Test();
        System.out.println(t.i);   // prints 0 (default)
    }
}
```

### 7.2 `final` Instance Variables
- If declared `final`, the JVM provides **NO default value** — you **must** initialize it explicitly, whether or not you actually use it, otherwise **compile-time error**.

```java
class Test {
    final int i;    // COMPILE ERROR
}
```
```
Test.java:1: variable i might not have been initialized
```

**Rule:** A `final` instance variable must be assigned **before constructor completion**. Three legal places:

**(1) At the time of declaration:**
```java
class Test {
    final int i = 10;
}
```

**(2) Inside an instance initializer block:**
```java
class Test {
    final int i;
    { i = 10; }
}
```

**(3) Inside every constructor:**
```java
class Test {
    final int i;
    Test() { i = 10; }
}
```

Assigning anywhere else (e.g., a regular method) → **compile-time error**:
```java
class Test {
    final int i;
    public void methodOne() {
        i = 10;    // COMPILE ERROR
    }
}
```
```
Test.java:5: cannot assign a value to final variable i
```

### 7.3 `final static` Variables

- Normal `static` variables: JVM auto-assigns default values.
- **`final static`**: JVM provides **no default**; must initialize **before class-loading completes**.

**(1) At the time of declaration:**
```java
class Test {
    final static int i = 10;
}
```

**(2) Inside a static initializer block:**
```java
class Test {
    final static int i;
    static { i = 10; }
}
```

Any other location (e.g., inside `main`) → **compile-time error**:
```java
class Test {
    final static int i;
    public static void main(String args[]) {
        i = 10;    // COMPILE ERROR
    }
}
```
```
Test.java:5: cannot assign a value to final variable i
```

### 7.4 Local Variables

- No default value ever provided by the JVM — **must** be initialized explicitly **before use**, regardless of `final`.

```java
class Test {
    public static void main(String args[]) {
        int i;
        System.out.println("hello");   // fine — i is unused
    }
}
```

```java
class Test {
    public static void main(String args[]) {
        int i;
        System.out.println(i);   // COMPILE ERROR — used before assignment
    }
}
```
```
Test.java:5: variable i might not have been initialized
```

> **The only applicable modifier for local variables is `final`.** Using anything else
> (`private`, `public`, `volatile`, `transient`, `static`, etc.) → **compile-time error: "illegal start of expression."**

### 7.5 Summary — Default Value Rules

| Variable type | Default value if not `final`? | Default value if `final`? |
|---|:---:|:---:|
| Instance | ✅ Yes | ❌ No (compile error if unassigned) |
| Static | ✅ Yes | ❌ No (compile error if unassigned) |
| Local | ❌ Never | ❌ Never (same either way) |

### 7.6 Formal Parameters

Formal parameters act like local variables of the method and **can** be declared `final`. A `final` parameter's value **cannot be reassigned** inside the method body.

```java
class Test {
    public static void main(String args[]) {
        methodOne(10, 20);
    }
    public static void methodOne(final int x, int y) {
        //x = 100;      // ❌ COMPILE ERROR if uncommented
        y = 200;        // ✅ OK — y is not final
        System.out.println(x + "...." + y);
    }
}
```
```
Test.java:6: final parameter x may not be assigned
```

---

## 8. `static` Modifier

- Applicable to: **variables, methods, and blocks** (not top-level classes — but **inner classes** CAN be `static`).
- Instance variables → one copy **per object**.
- Static variables → **single copy at class level**, shared across all objects.

```java
class Test {
    static int x = 10;
    int y = 20;
    public static void main(String args[]) {
        Test t1 = new Test();
        t1.x = 888;
        t1.y = 999;
        Test t2 = new Test();
        System.out.println(t2.x + "....." + t2.y);   // 888.....20
    }
}
```
> Modifying `x` via `t1` affects the value seen via `t2` (shared copy). Modifying `y` via `t1` has **no effect** on `t2.y` (separate instance copies).

### 8.1 Static vs Instance Access Rules

- **Instance variables/methods** → accessible only from **instance context** directly; NOT accessible from static context directly.
- **Static variables/methods** → accessible from **both** static and instance contexts directly.

```java
class Test {
    int x = 10;                              // (1)
    static int y = 20;                       // (2)

    public void methodA() {                  // (3) — instance
        System.out.println(x);               // ✅ OK: (1)+(3)
    }
    public static void methodB() {           // (4) — static
        System.out.println(x);               // ❌ COMPILE ERROR: (1)+(4)
    }
}
```
```
non-static variable x cannot be referenced from a static context
```

| Combination | Compiles? |
|---|:---:|
| instance var + instance method | ✅ |
| instance var + static method | ❌ |
| static var + instance method | ✅ |
| static var + static method | ✅ |
| duplicate variable names (`int x` AND `static int x`) in same class | ❌ (`x is already defined`) |
| duplicate method signatures — one instance, one static, same name/params | ❌ (`methodOne() is already defined`) |

> For `static` methods, an implementation **must** exist; abstract methods have **no** implementation → **`static abstract` is illegal for methods.**

### 8.2 Overloading & `main()`

Overloading **is** applicable to static methods including `main`. But the JVM will **only ever invoke** the exact `public static void main(String[] args)` signature automatically. Any other overload must be called **explicitly** like a normal method.

```java
class Test {
    public static void main(String args[]) {
        System.out.println("String[] method is called");
    }
    public static void main(int args[]) {
        System.out.println("int[] method is called");   // only runs if called explicitly
    }
}
```
```
Output: String[] method is called
```

### 8.3 Inheritance & Static Methods (Method Hiding, NOT Overriding)

Static methods (including `main`) **participate in inheritance**. If a child class doesn't define its own `main`, the parent's `main` runs.

```java
class Parent {
    public static void main(String args[]) {
        System.out.println("parent main() method called");
    }
}
class Child extends Parent { }
```
```
D:\Java>java Parent
parent main() method called
D:\Java>java Child
parent main() method called
```

If **both** define `main`, calling `java Child` runs **Child's own** `main` — but this is **method hiding**, not true polymorphic override, because static-method resolution is based on the **reference/class type at compile time**, not dynamic dispatch.

```java
class Parent {
    public static void main(String args[]) {
        System.out.println("parent main() method called");
    }
}
class Child extends Parent {
    public static void main(String args[]) {
        System.out.println("child main() method called");
    }
}
```
```
D:\Java>java Child
child main() method called
```
> **Interview trap:** *"It looks like overriding is applicable to static methods, but it is NOT — it is method hiding."*

---

## 9. `native` Modifier

- Applicable **only to methods** (not variables, not classes).
- Native methods are implemented in a **non-Java language** (typically C/C++ via JNI — Java Native Interface).

### Objectives
1. Improve performance for compute-heavy operations
2. Reuse existing legacy non-Java code
3. Achieve machine/OS-level or hardware-level communication (memory addresses, device drivers)

### Pseudo-code Pattern

```java
class NativeDemo {
    static {
        System.loadLibrary("NativeLibrary");   // 1) Load the native library
    }
    public native void methodOne();            // 2) Native method declaration (ends with ';')
}
class Client {
    public static void main(String args[]) {
        NativeDemo n = new NativeDemo();
        n.methodOne();                         // 3) Invoke the native method
    }
}
```

### Rules
- Since native implementation already exists externally, the Java-side declaration **must end with a semicolon** — no body:
  ```java
  public native void methodOne() { }   // ❌ INVALID
  public native void methodOne();      // ✅ VALID
  ```
- `abstract` + `native` → **illegal** (abstract implies no implementation exists yet; native implies implementation already exists elsewhere — contradiction).
- `native` + `strictfp` → **illegal** (no guarantee the external/legacy language honors the IEEE 754 standard).
- Inheritance, overriding, and overloading concepts **are** applicable to native methods.
- **Advantage:** performance boost, code reuse.
- **Disadvantage:** breaks Java's platform-independence — a native method compiled for Windows won't run on Linux without a matching native library.

---

## 10. `synchronized` Modifier

- Applicable to **methods and blocks** (not variables, not classes).
- Ensures **only one thread at a time** can execute the synchronized method/block **on a given object's monitor lock**.
- **Advantage:** resolves data-inconsistency / race-condition problems in multi-threaded code.
- **Disadvantage:** increases thread waiting time, hurts throughput/performance. Use only when genuinely required for thread safety.
- `abstract` + `synchronized` → **illegal for methods** (synchronized requires an actual lock-acquiring implementation; abstract has none).

```java
class Counter {
    private int count = 0;
    public synchronized void increment() {   // one thread at a time per Counter instance
        count++;
    }
}
```

---

## 11. `transient` Modifier

- Applicable **only to variables** (not methods, not classes).
- Used during **serialization**: marks a field whose value should **not** be persisted (e.g., passwords, sensitive derived data, or non-serializable references).
- At serialization time, the JVM **ignores** the field's actual value and writes the **default value** for that type instead.

```java
class User implements java.io.Serializable {
    String username;
    transient String password;   // will NOT be serialized — default (null) is written instead
}
```

### Special Notes
- **Static variables are not part of object state**, so serialization doesn't touch them anyway — marking a `static` variable `transient` has **no effect**.
- **`final` variables participate directly by their literal value** at compile time in many cases (constant folding) — marking a `final` variable `transient` has **no meaningful impact**.

---

## 12. `volatile` Modifier

- Applicable **only to variables** (not methods, not classes).
- Used when a variable's value is expected to **change frequently**, especially in multi-threaded contexts, to control **visibility** across threads.
- Historically (pre-JMM clarifications), each thread could maintain a local working copy; `volatile` forced all reads/writes to hit the shared **master copy** directly, ensuring visibility of the latest value across threads.
- **Advantage:** helps resolve data-visibility/inconsistency issues between threads.
- **Disadvantage:** can add complexity and (in older JVMs) overhead; today `volatile` is well-understood but should be reserved for simple visibility guarantees — it does **NOT** provide atomicity for compound operations like `i++`.
- `final` + `volatile` → **illegal for variables** (`final` = value never changes; `volatile` = value changes constantly — direct logical contradiction).

```java
class Flag {
    private volatile boolean running = true;   // visibility guaranteed across threads
    public void stop() { running = false; }
    public void run() {
        while (running) { /* work */ }
    }
}
```

> **Modern Java note:** For most concurrency needs today, prefer `java.util.concurrent.atomic.*`
> classes (`AtomicInteger`, `AtomicBoolean`, etc.) or higher-level constructs
> (`java.util.concurrent` locks, executors) over raw `volatile`/`synchronized` where possible —
> but understanding `volatile`/`synchronized` fundamentals remains essential for interviews.

---

## 13. Modifier Summary Matrix

| Modifier | Outer Class | Inner Class | Methods | Variables | Blocks | Interfaces | `enum` | Constructors |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `public` | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| `private` | ❌ | ✅ | ✅ | ✅ | — | ❌ | ❌ | ✅ |
| `protected` | ❌ | ✅ | ✅ | ✅ | — | ❌ | ❌ | ✅ |
| `<default>` | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| `final` | ✅ | ✅ | ✅ | ✅ | — | ❌ | ❌ | ❌ |
| `static` | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | — | ❌ |
| `synchronized` | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| `abstract` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `native` | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `strictfp` | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| `transient` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `volatile` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |

### Golden Conclusions
1. **Constructors:** only `public`, `private`, `protected`, `<default>` apply.
2. **Local variables:** only `final` applies.
3. **Applicable to classes but NOT interfaces:** `final`.
4. **Applicable to classes but NOT `enum`:** `final`, `abstract`.
5. **Applicable to inner classes but NOT outer (top-level) classes:** `public`(same), `protected`, `private`, `static`.
6. **The ONLY modifier applicable exclusively to methods** (nothing else): `native`.
7. **Modifiers applicable ONLY to variables:** `transient`, `volatile`.

---

## 14. Interfaces — Complete Deep Dive

### 14.1 Definitions (Three Complementary Views)

1. **Definition 1 (SRS view):** Any *service requirement specification* is called an interface.
   > Example: Sun/Oracle defines the **JDBC API** (interface); database vendors (Oracle, DB2, MySQL) provide the **driver implementations**.

2. **Definition 2 (Contract view):** From the client's perspective, an interface defines the services it *expects*. From the provider's perspective, it defines the services it *offers*. Hence an interface is a **contract** between client and service provider.
   > Example: An ATM's GUI screen is the contract between the bank (provider) and the customer (client) — both sides agree on the same set of operations.

3. **Definition 3 (Purity view):** Inside an interface, **every method is implicitly `abstract`** whether declared so or not — hence an interface is **100% pure abstract "class."**

> **Summary definition:** Any *service requirement specification*, or any *contract* between client and provider, or a *100% pure abstract class*, is called an interface.

### 14.2 Declaring & Implementing an Interface

**Note 1:** When implementing an interface, you must provide implementations for **every** method — otherwise the implementing class must itself be declared `abstract`, deferring the remaining implementations to its own subclass.

**Note 2:** When implementing an interface method, it **must** be declared `public` — otherwise **compile-time error** (you cannot reduce visibility when overriding/implementing).

```java
interface Interf {
    void methodOne();
    void methodTwo();
}

abstract class ServiceProvider implements Interf {
    public void methodOne() { }
    // methodTwo() intentionally left unimplemented → class MUST be abstract
}

class SubServiceProvider extends ServiceProvider {
    // still missing methodTwo() → COMPILE ERROR
}
```
```
SubServiceProvider is not abstract and does not override
abstract method methodTwo() in Interf
```

### 14.3 `extends` vs `implements`

| Rule | Detail |
|---|---|
| A class can `extends` **only one** class | Single inheritance for classes |
| A class can `implements` **any number** of interfaces | Multiple "inheritance of type" via interfaces |
| A class can `extends` one class **AND** `implements` multiple interfaces **simultaneously** | ✅ |
| An interface can `extends` **any number** of interfaces | Multiple interface inheritance is allowed |

```java
// Multiple interface implementation
interface One { void methodOne(); }
interface Two { void methodTwo(); }
class Three implements One, Two {
    public void methodOne() {}
    public void methodTwo() {}
}

// Class extends + implements simultaneously
class Two2 { public void methodTwo(){} }
class Three2 extends Two2 implements One {
    public void methodOne(){}
}

// Interface extending multiple interfaces
interface Three3 extends One, Two { }
```

### 14.4 `extends`/`implements` Combination Cheat-Sheet

| Expression | Rule |
|---|---|
| `X extends Y` | Both `X`,`Y` are classes **OR** both are interfaces (never mixed) |
| `X extends Y, Z` | `X`, `Y`, `Z` must **all** be interfaces |
| `X extends Y implements Z` | `X`,`Y` are classes; `Z` is an interface |
| `X implements Y, Z` | `X` is a class; `Y`,`Z` are interfaces |
| `X implements Y extends Z` | ❌ **Always a syntax error** — wrong keyword order |

```java
interface One { }
class Two { }
class Three implements One extends Two { }   // ❌ COMPILE ERROR
```
```
Three.java:5: '{' expected
```

**True/False check (frequently asked):**
> *"A class can extend a class and implement an interface but not both simultaneously."* → **FALSE**. A class absolutely can do both at once (`class X extends Y implements Z`).

### 14.5 Interface Methods

Every interface method is **implicitly `public abstract`**, whether declared or not — all four forms below are 100% equivalent:

```java
void methodOne();
public void methodOne();
abstract void methodOne();
public abstract void methodOne();
```

**Forbidden modifiers on interface methods** (all contradict "always public abstract"):
```
private, protected, final, static, synchronized, native, strictfp
```
> *(Note: Since Java 8, interfaces CAN have `default` and `static` methods with bodies, and since Java 9, `private` interface methods are allowed — these are language evolutions beyond this classic pre-8 material. See callout below.)*

**Validity check — which of these interface method declarations is legal (pre-Java-8 classic rule)?**

| # | Declaration | Valid? |
|---|---|:---:|
| 1 | `public void methodOne(){}` | ❌ (body not allowed pre-Java-8) |
| 2 | `private void methodOne();` | ❌ |
| 3 | `public final void methodOne();` | ❌ |
| 4 | `public static void methodOne();` | ❌ (no body given — static interface methods need a body since Java 8) |
| 5 | `public abstract void methodOne();` | ✅ |

> **Java 8+ Update (architect must know this):** Interfaces can now declare:
> - `default` methods (with body, inheritable, overridable) — enables API evolution without breaking implementers.
> - `static` methods (with body, called via `InterfaceName.method()`, NOT inherited by implementers).
> - (Java 9+) `private` and `private static` helper methods for internal code reuse within the interface.
> ```java
> interface Vehicle {
>     void drive();                                  // still implicitly public abstract
>     default void honk() { System.out.println("Beep!"); }   // Java 8 default method
>     static Vehicle createDefault() { return () -> System.out.println("driving"); }  // Java 8 static method
> }
> ```

### 14.6 Interface Variables

- An interface **can** contain variables — used to define **requirement-level constants**.
- Every interface variable is implicitly **`public static final`**, whether declared so or not.

```java
interface Interf {
    int x = 10;
}
```

All the following declarations are 100% equivalent:
```java
int x = 10;
public int x = 10;
static int x = 10;
final int x = 10;
public static int x = 10;
public final int x = 10;
static final int x = 10;
public static final int x = 10;
```

**Forbidden modifiers** (contradict "always public static final"):
```
private, protected, transient, volatile
```

**Mandatory initialization at declaration:**
```java
interface Interf {
    int x;    // COMPILE ERROR — must assign a value
}
```
```
Interf.java:3: = expected
```

**Validity check:**

| # | Declaration | Valid? |
|---|---|:---:|
| 1 | `int x;` | ❌ |
| 2 | `private int x = 10;` | ❌ |
| 3 | `public volatile int x = 10;` | ❌ |
| 4 | `public transient int x = 10;` | ❌ |
| 5 | `public static final int x = 10;` | ✅ |

**Interface variables can be READ but never reassigned by implementers:**

```java
interface Interf { int x = 10; }

class Test implements Interf {
    public static void main(String args[]) {
        x = 20;                          // COMPILE ERROR
        System.out.println("value of x " + x);
    }
}
```
```
Test.java:4: cannot assign a value to final variable x
```

> **Shadowing note:** A local variable of the *same name* declared inside the implementing class's method **hides** the inherited interface constant for that scope:
> ```java
> class Test implements Interf {
>     public static void main(String args[]) {
>         int x = 20;              // local variable shadows Interf.x
>         System.out.println(x);  // prints 20 — the LOCAL variable
>     }
> }
> ```

### 14.7 Interface Naming Conflicts

#### Method Naming Conflicts — Case 1: Identical signature & return type
If two interfaces declare a method with the **same signature and same return type**, the implementing class needs only **ONE** implementation to satisfy both.

```java
interface Left  { void methodOne(); }
interface Right { void methodOne(); }

class Test implements Left, Right {
    public void methodOne() { }   // satisfies BOTH interfaces
}
```

#### Case 2: Same name, different parameters → treated as overloads
```java
interface Left  { void methodOne(); }
interface Right { void methodOne(int i); }

class Test implements Left, Right {
    public void methodOne() { }
    public void methodOne(int i) { }   // separate overloaded method
}
```

#### Case 3: Same signature, DIFFERENT return type → impossible to implement both
```java
interface Left  { void methodOne(); }
interface Right { int methodOne(); }
```
> **No Java class can implement both `Left` and `Right` simultaneously** — a single method cannot have two different return types. This is a **covariant-return-type violation**, not resolvable by any override trick.

> **Q: Can a class implement any number of interfaces simultaneously?**
> **A:** Yes — **except** when two interfaces declare a method with the same signature but different return types.

#### Variable Naming Conflicts — Resolved via Interface-Qualified Access

```java
interface Left  { int x = 888; }
interface Right { int x = 999; }

class Test implements Left, Right {
    public static void main(String args[]) {
        //System.out.println(x);      // ❌ COMPILE ERROR — ambiguous
        System.out.println(Left.x);   // ✅ 888
        System.out.println(Right.x);  // ✅ 999
    }
}
```

### 14.8 Marker Interface (a.k.a. Tag Interface / Ability Interface)

**Definition:** An interface with **zero methods**; simply *implementing* it grants the class a special "ability" that the JVM recognizes and honors internally.

**Classic examples:**
```
Serializable, Cloneable, RandomAccess, SingleThreadModel (deprecated), EventListener
```

| Interface | Ability granted |
|---|---|
| `Serializable` | Object can be converted to a byte stream (sent over network / saved to file) |
| `Cloneable` | Object can be exactly duplicated via `Object.clone()` |
| `SingleThreadModel` *(deprecated Servlet API)* | Servlet processes only one request at a time → thread safety |

**Q: Without any methods, how does the object gain the ability?**
**A:** The **JVM internally** checks (via `instanceof`) whether the object's class implements the marker interface, and grants/enables the corresponding special behavior at runtime.

**Q: Why does the JVM do it this way instead of a regular method call?**
**A:** To **reduce programming complexity** — the developer doesn't need to implement any logic; simply "opting in" via the marker interface is enough.

**Q: Can you create your own marker interface?**
**A:** Yes — but it only has real effect if some framework/JVM-level code explicitly checks for it via `instanceof`. Custom examples: `Sleepable`, `Jumpable` (require accompanying processing logic elsewhere to actually do anything).

```java
interface Auditable { }   // custom marker interface — no methods

class Order implements Auditable { }

// Elsewhere in framework code:
if (obj instanceof Auditable) {
    // apply audit-logging behavior
}
```

### 14.9 Adapter Class

**Problem:** If an interface has many methods and a class implements it directly, **every single method** must be implemented — even ones irrelevant to that use case. This bloats code and hurts readability.

```java
interface X {
    void m1(); void m2(); void m3(); void m4(); void m5();
}
class Test implements X {
    public void m3() { System.out.println("m3() called"); }
    public void m1() {}   // boilerplate — not needed by this class
    public void m2() {}   // boilerplate
    public void m4() {}   // boilerplate
    public void m5() {}   // boilerplate
}
```

**Solution — Adapter class:** A concrete-but-abstract "middle" class that implements the interface and provides **empty default bodies** for every method. Subclasses of the adapter only override what they actually need.

```java
abstract class AdapterX implements X {
    public void m1() {}
    public void m2() {}
    public void m3() {}
    public void m4() {}
    public void m5() {}
}

class Test extends AdapterX {
    public void m3() { System.out.println("m3() called"); }   // only override what's needed
}
```

**Real-world example — Servlet API:**
```
javax.servlet.Servlet          (interface — 5 methods: init, service, destroy, getServletInfo, getServletConfig)
        ↑
javax.servlet.GenericServlet   (abstract class — Adapter — implements ALL methods with defaults/utility logic)
        ↑
YourServlet                    (concrete class — overrides ONLY service())
```

```java
// Direct interface implementation — verbose
class MyServlet implements Servlet {
    public void init(ServletConfig config) {}
    public void destroy() {}
    public void service(ServletRequest req, ServletResponse res) { /* actual logic */ }
    public String getServletInfo() { return null; }
    public ServletConfig getServletConfig() { return null; }
}

// Using the Adapter (GenericServlet) — clean
class MyServlet1 extends GenericServlet {
    public void service(ServletRequest req, ServletResponse res) { /* actual logic */ }
}
```

> `GenericServlet` is a textbook **Adapter class** for the `Servlet` interface.
> Similarly `MouseAdapter`, `KeyAdapter`, `WindowAdapter` in AWT/Swing are adapters for `MouseListener`, `KeyListener`, `WindowListener`.

---

## 15. Interface vs Abstract Class vs Concrete Class

### 15.1 When to Choose Which

| Situation | Choose |
|---|---|
| You only know the **requirement specification** — zero implementation knowledge | **Interface** |
| You know **part** of the implementation (partial) | **Abstract class** |
| You know the **complete** implementation, ready to provide the service | **Concrete class** |

```
Servlet (Interface)
    ↑
GenericServlet (Abstract Class)
    ↑
HttpServlet (Abstract Class)
    ↑
MailSendingServlet (Concrete Class)
```

### 15.2 Interface vs Abstract Class — Feature-by-Feature

| Feature | Interface | Abstract Class |
|---|---|---|
| When to use | Pure requirement spec, no implementation knowledge | Partial implementation known |
| Method default nature | Always `public abstract` (unless Java 8+ `default`/`static`) | No restriction — can be any combination |
| Forbidden method modifiers | `private, protected, final, static, synchronized, native, strictfp` (classic pre-8 rule) | None — no restrictions |
| Variable default nature | Always `public static final` | No restriction |
| Forbidden variable modifiers | `private, protected, transient, volatile` | None |
| Variable initialization | Mandatory at declaration | Optional at declaration |
| Static / instance blocks | ❌ Not allowed | ✅ Allowed |
| Constructors | ❌ Not allowed | ✅ Allowed |
| Multiple inheritance of type | ✅ A class can implement many interfaces | ❌ A class can extend only one abstract class |
| Object creation cost/overhead scenario | Extremely lightweight — no forced state | Can carry initialized state via constructor logic |

### 15.3 Performance & Design Rationale

```
interface X { ... }              abstract class X { ... }
class Test implements X { ... }  class Test extends X { ... }

Test t = new Test();             Test t = new Test();
// e.g., ~2 sec (lightweight)    // e.g., ~20 sec (heavier init chain, illustrative)
```
> While extending `X`, a class **cannot extend any other class** — you lose flexibility.
> While implementing `X` (interface), the class **can still extend another class** — you retain full inheritance flexibility.

**Rule of thumb:** *"If everything is abstract (pure requirement, zero implementation), always prefer an interface over an abstract class."* Using an abstract class purely as a substitute for an interface is considered **misuse of the abstract-class role** — analogous to *"recruiting an IAS officer to do sweeping duty."*

---

## 16. Why Abstract Classes Have Constructors (Deep Dive)

This is one of the most frequently asked **architect-level trick questions**.

### 16.1 The Purpose of a Constructor
- A constructor's job is to **initialize** an object — it does **NOT** create the object.
- Object creation happens via the `new` operator; **initialization** happens afterward via the constructor.

```
Student s = new Student("Ravi", 30, 6, 60, 101, 70);
                ↑                            ↑
          Object creation             Initialization
           (by `new`)                 (by constructor)
```

- The object already exists (memory allocated, default field values assigned) **before** the constructor body finishes executing — proven by the fact you can access `this.hashCode()` **inside** the constructor:

```java
class Test {
    Test() {
        System.out.println(this);              // Test@6e3d60
        System.out.println(this.hashCode());   // 7224672 — object identity already exists!
    }
    public static void main(String[] args) {
        Test t = new Test();
    }
}
```

### 16.2 Constructor Chaining: Parent Constructor Runs, But NO Separate Parent Object Is Created

```java
class P {
    P() { System.out.println(this.hashCode()); }   // 7224672
}
class C extends P {
    C() { System.out.println(this.hashCode()); }   // 7224672 — SAME hash!
}
class Test {
    public static void main(String[] args) {
        C c = new C();
        System.out.println(c.hashCode());            // 7224672 — SAME hash!
    }
}
```
> **All three hash codes are identical** — proving there is only **ONE object** in memory.
> The parent constructor executes purely to **initialize the parent-inherited portion**
> of that single child object — a separate "parent object" is never created.

### 16.3 Real-World Example — Why Abstract Class Constructors Matter

```java
abstract class Person {
    String name;
    int age;
    int height;
    int weight;

    public Person(String name, int age, int height, int weight) {
        this.name = name;
        this.age = age;
        this.height = height;
        this.weight = weight;
    }
}

class Student extends Person {
    int rollno;
    int marks;

    public Student(String name, int age, int height, int weight, int rollno, int marks) {
        super(name, age, height, weight);   // delegates to abstract Person's constructor
        this.rollno = rollno;
        this.marks = marks;
    }
}

class Demo {
    public static void main(String[] args) {
        Student s = new Student("Ravi", 30, 6, 60, 101, 70);
        // name/age/height/weight → initialized by Person's constructor (via super())
        // rollno/marks           → initialized by Student's own constructor
    }
}
```

**Without** the abstract class's constructor, `Student` would have to **manually re-assign** `name`, `age`, `height`, `weight` itself, duplicating logic that rightfully belongs to `Person` — defeating the purpose of code reuse via inheritance.

### 16.4 Why Interfaces Can NEVER Have Constructors (But Abstract Classes Can)

| Reasoning step | Interface | Abstract Class |
|---|---|---|
| Purpose of a constructor | Initialize **instance variables** | Same |
| Can it have instance variables? | ❌ NO — every interface variable is implicitly `public static final` (i.e., always static, never instance) | ✅ YES — ordinary instance variables allowed |
| Mandatory init at declaration? | Yes — so there's never an "uninitialized" state to fix via a constructor | No — can defer init to a constructor |
| **Conclusion** | No instance-variable initialization ever needed → **constructor concept is irrelevant/inapplicable** | Instance variables often need constructor-driven initialization for the *child's benefit* → **constructor is essential** |

### 16.5 "But Interfaces and Abstract Classes Can Both Contain Only Abstract Methods — So Why Not Just Always Use Abstract Class?"

**Technically possible, but a bad practice**, because:
1. **Flexibility loss:** `class Test extends X` blocks any further inheritance (`extends` is single-parent only). `class Test implements X` leaves the `extends` slot free for a real superclass.
2. **Semantic misuse:** Abstract classes are meant for **partial implementation reuse**; using one purely as a "100% abstract contract" abuses its role — reserved for interfaces by design intent.
3. **API evolution:** Interfaces (Java 8+) support `default`/`static` methods for smoother versioning without breaking every implementer.

### 16.6 True/False Rapid-Fire (Common in Interviews)

| # | Statement | Verdict |
|---|---|:---:|
| 1 | The purpose of a constructor is to create an object. | ❌ False |
| 2 | The purpose of a constructor is to initialize an object, NOT create it. | ✅ True |
| 3 | Once the constructor completes, only then does object creation complete. | ❌ False |
| 4 | The object is created first, and then the constructor executes. | ✅ True |
| 5 | `new` creates the object; the constructor initializes it. | ✅ True |
| 6 | We can't create an object for an abstract class directly, but we can indirectly. | ❌ False (never possible, directly or indirectly) |
| 7 | Creating a child object automatically creates an internal parent object too. | ❌ False |
| 8 | Creating a child object automatically executes the abstract (parent) class's constructor. | ✅ True |
| 9 | Creating a child object creates a separate parent object. | ❌ False |
| 10 | Parent constructor executes, but no separate parent object is created — it initializes the SAME child object. | ✅ True |
| 11 | Because we can never instantiate an abstract class, constructors are inapplicable to it. | ❌ False — constructors ARE applicable and DO run (for the child's benefit) |
| 12 | Interfaces can contain constructors. | ❌ False |

---

## 17. Interview Q&A Bank (Answered)

> A curated, **answered** version of the classic OCJP/SCJP question bank found in this chapter — organized by topic for rapid review.

**Q1. Which modifiers are allowed for top-level classes?**
`public`, `<default>`, `final`, `abstract`, `strictfp`.

**Q2. Can a top-level class be declared `static`, `private`, or `protected`?**
No — those are valid only for **inner (nested)** classes.

**Q3. What extra modifiers are applicable to inner classes vs. outer classes?**
`private`, `protected`, `static` (in addition to the standard top-level set).

**Q4. What is a `final` class?**
A class that cannot be subclassed — `extends` on it is a compile-time error.

**Q5. Difference between `final`, `finally`, and `finalize`?**
- `final` → modifier (non-inheritable class, non-overridable method, non-reassignable variable).
- `finally` → block that always executes after a `try` (whether or not an exception occurred), used for cleanup.
- `finalize()` → deprecated `Object` method historically invoked by the GC before reclaiming an object (removed in modern JDKs — use `try-with-resources`/`Cleaner` instead).

**Q6. Is every method in a `final` class implicitly `final`?**
Yes — since the class can never be subclassed, all its methods are effectively `final` whether declared so or not.

**Q7. Is every variable in a `final` class implicitly `final`?**
No — `final` on the class does not cascade to its variables.

**Q8. What is an abstract class?**
A class that cannot be instantiated, typically containing one or more abstract methods (though zero is also legal), meant to be extended.

**Q9. What is an abstract method?**
A method with only a signature/declaration (ending in `;`), no body — implementation is deferred to subclasses.

**Q10. If a class has at least one abstract method, must the class be `abstract`?**
Yes, mandatorily.

**Q11. If a class has zero abstract methods, can it still be declared `abstract`?**
Yes — e.g., `HttpServlet`, adapter classes.

**Q12. When extending an abstract class, must you implement every abstract method?**
Yes — unless you also declare your subclass `abstract`, deferring the obligation further down the hierarchy.

**Q13. Can a `final` class contain an abstract method?**
No — contradiction (abstract requires overriding; final blocks subclassing entirely).

**Q14. Can an abstract class contain `final` methods?**
Yes — perfectly legal.

**Q15. Example of an abstract class with zero abstract methods?**
`HttpServlet`, any GUI Adapter class (`MouseAdapter`), `AbstractList` (partial implementation).

**Q16. Which modifier combos are legal for methods?**
✅ `public static`; ❌ `static abstract`; ❌ `abstract final`; ❌ `final synchronized` is actually **legal** (careful — `final` blocks override, `synchronized` just adds locking, no contradiction); ❌ `synchronized native` is legal too (native method can still be synchronized); ❌ `native abstract` (illegal).
*(Tip: the only truly illegal combos with `abstract` are: final, static, synchronized, native, strictfp, private.)*

**Q17. Which modifier combos are legal for classes?**
✅ `public final`; ❌ `final abstract`; ✅ `abstract strictfp`; ✅ `strictfp public`.

**Q18. Difference between abstract class and interface?**
See [Section 15.2](#152-interface-vs-abstract-class--feature-by-feature) full comparison table.

**Q19. What is the `strictfp` modifier?**
Forces IEEE 754–compliant floating-point arithmetic for platform-independent results (no-op as of Java 17+, but syntactically retained).

**Q20. Can a variable be declared `strictfp`?**
No — only classes and methods.

**Q21. Is `abstract strictfp` legal for classes or methods?**
Legal for **classes**; illegal for **methods**.

**Q22. Can you override a `native` method?**
Yes — inheritance, overriding, and overloading all apply normally to native methods.

**Q23. Difference between instance and static variables?**
Instance → one copy per object; Static → one shared copy per class.

**Q24. Difference between a general static variable and a `final static` variable?**
General static gets a JVM default value automatically; `final static` requires explicit initialization before class-loading completes (at declaration or in a static block), or it's a compile error.

**Q25. Which modifiers apply to local variables?**
Only `final`.

**Q26. When are static variables created?**
At **class-loading time** (once per classloader, before any object of that class is instantiated).

**Q27. Memory locations of instance, local, and static variables?**
- Instance variables → **Heap** (as part of the object).
- Static variables → **Method Area / Metaspace** (class-level metadata region, per classloader).
- Local variables → **Stack** (per thread, per method-call frame).

**Q28. Can `main()` be overloaded?**
Yes, but the JVM only auto-invokes `public static void main(String[] args)`; other overloads must be called manually.

**Q29. Can static methods be overridden?**
No — they can only be **hidden** (method hiding), resolved statically based on reference type, not dynamically dispatched.

**Q30. What is the `native` keyword and where is it applicable?**
Denotes a method implemented outside Java (typically C/C++ via JNI); applicable only to methods.

**Q31. Main advantage of `native`?**
Performance improvement + reuse of existing legacy code + low-level/hardware access.

**Q32. How does `native` affect Java's platform-independent nature?**
It **breaks** it — the native library must be recompiled per target platform/architecture.

**Q33. How do you declare a native method?**
`public native void methodOne();` — body-less declaration ending in a semicolon, paired with `System.loadLibrary(...)` in a static block.

**Q34. Can an abstract method have a body?**
No — compile-time error ("abstract methods cannot have a body").

**Q35. What is `synchronized` and where can it be applied?**
Thread-safety modifier applicable to methods and blocks.

**Q36. Advantages/disadvantages of `synchronized`?**
Advantage: resolves data-inconsistency across threads. Disadvantage: increases thread wait time, hurts throughput.

**Q37. Which modifiers are considered "most dangerous" (i.e., most likely to hurt performance/design if misused)?**
`synchronized` and `volatile` (performance overhead), and `final`/`abstract` overuse (loses OOP flexibility).

**Q38. What is serialization?**
Converting an object's state into a byte stream (for persistence or network transfer) via `ObjectOutputStream.writeObject()`.

**Q39. What is deserialization?**
Reconstructing an object from its serialized byte-stream form via `ObjectInputStream.readObject()`.

**Q40. Which classes achieve serialization/deserialization?**
`java.io.ObjectOutputStream` and `java.io.ObjectInputStream`.

**Q41. What is the `Serializable` interface and its methods?**
A **marker interface** (`java.io.Serializable`) with **zero methods** — implementing it simply signals to the JVM that instances are eligible for serialization.

**Q42. What is a marker interface? Give an example.**
An interface with no methods that grants a special JVM-recognized ability upon implementation. Examples: `Serializable`, `Cloneable`, `RandomAccess`.

**Q43. Without any method in `Serializable`, how does an object gain serializability?**
The JVM internally checks `instanceof Serializable` before allowing `writeObject()` to proceed; it's a signal/flag, not behavior.

**Q44. Purpose and advantage of `transient`?**
Excludes a field's actual value from serialization (writes the type's default instead) — useful for security-sensitive fields (e.g., passwords) or non-serializable references.

**Q45. Can every Java object be serialized?**
No — only objects whose class (and all non-transient field types, recursively) implement `Serializable`. Otherwise, `NotSerializableException` is thrown at runtime.

**Q46. Can a method or a class be declared `transient`?**
No — only variables.

**Q47. Impact of declaring a `static` variable `transient`?**
None — static variables aren't part of object state, so serialization never touches them regardless.

**Q48. Impact of declaring a `final` variable `transient`?**
Minimal/no practical impact — final fields (especially compile-time constants) are typically inlined/participate by value directly.

**Q49. What is a `volatile` variable?**
A variable whose reads/writes are guaranteed visible across threads immediately, bypassing thread-local caching of its value.

**Q50. Can a class or method be declared `volatile`?**
No — only variables.

**Q51. Advantage/disadvantage of `volatile`?**
Advantage: resolves visibility-based data-inconsistency across threads without full locking overhead. Disadvantage: adds conceptual complexity; does **not** provide atomicity for compound read-modify-write operations (e.g., `count++` is still not thread-safe even if `count` is `volatile`).

---

### Interfaces — Additional Q&A

**Q52. What is an interface?**
A 100% abstract type representing a service contract; every field is `public static final`, every (classic) method is `public abstract`.

**Q53. Can a Java class implement any number of interfaces?**
Yes — except when two interfaces declare a method with identical signature but different return types (unresolvable).

**Q54. Difference between `extends` and `implements`?**
`extends` → class-to-class (single) or interface-to-interface (multiple) inheritance.
`implements` → class-to-interface(s), any number allowed.

**Q55. Why does an abstract class need a constructor if we can never instantiate it directly?**
Because subclass objects still need the parent's instance-variable initialization logic executed — the constructor runs *for the child object's benefit*, not to create a standalone parent object. (Full explanation in [Section 16](#16-why-abstract-classes-have-constructors-deep-dive).)

**Q56. What is an Adapter class and its usage?**
A concrete-but-abstract class implementing an interface with empty default method bodies, allowing subclasses to override only the methods they actually need — reduces boilerplate. Example: `GenericServlet` for `Servlet`, `MouseAdapter` for `MouseListener`.

**Q57. If both interface and abstract class can have only abstract methods, why do we need interfaces at all?**
Because using an abstract class purely for 100% abstraction (a) blocks the implementing class's single `extends` slot, and (b) misuses the abstract class's intended role of *partial implementation*. Interfaces are lighter, support multiple implementation, and (Java 8+) evolve safely via `default`/`static` methods.

---

## 18. Quick-Reference Cheat Sheet

### Top-Level Class Modifiers
```
Allowed:    public, <default>, final, abstract, strictfp
Not Allowed: private, protected, static, native, synchronized, transient, volatile
Inner-class extras: private, protected, static
```

### Member Access Levels (least → most visible)
```
private < <default> < protected < public
```

### Illegal Modifier Pairs (memorize these!)
```
abstract + final           → illegal (methods & implicitly classes)
abstract + static          → illegal (methods)
abstract + synchronized    → illegal (methods)
abstract + native          → illegal (methods)
abstract + strictfp        → illegal (METHODS only; LEGAL for classes)
abstract + private         → illegal (methods)
final    + abstract        → illegal (classes)
final    + volatile        → illegal (variables)
native   + strictfp        → illegal (methods)
private  + abstract        → illegal (methods)
```

### Default Value Behavior
```
Instance var (non-final)  → JVM default provided
Static var   (non-final)  → JVM default provided
Local var                 → NEVER a default — must assign before use
final instance/static var → NEVER a default — must assign explicitly
```

### `final` Variable — Legal Initialization Points
```
Instance final:  (1) at declaration  (2) instance block  (3) every constructor
Static   final:  (1) at declaration  (2) static block
Local    final:  before first use (only one legal assignment ever)
```

### Interface Defaults (classic, pre-Java-8)
```
Methods   → always public abstract
Variables → always public static final (must initialize at declaration)
Forbidden method modifiers:  private, protected, final, static, synchronized, native, strictfp
Forbidden variable modifiers: private, protected, transient, volatile
No constructors, no static/instance blocks allowed (pre-8 rule for member content)
```

### Interfaces — Java 8+ Enhancements (bonus, beyond source material)
```
default methods   → have a body, inherited, overridable
static methods    → have a body, called via InterfaceName.method(), NOT inherited
private methods   → (Java 9+) internal helper methods within the interface only
```

### Package & Import Essentials
```
- At most ONE package statement per file, and it must be the first non-comment line.
- Any number of import statements, always AFTER package, BEFORE type declarations.
- java.lang and the default package never require an explicit import.
- Explicit import > classes in current directory > implicit (*) import — precedence order.
- Static import precedence: current class members > explicit static import > implicit static import.
```

### Static vs Instance Quick Rules
```
Instance member accessible directly only from instance context.
Static member accessible directly from BOTH static and instance context.
Static methods are "hidden," never truly overridden (no dynamic dispatch).
```

### Serialization-Related Modifiers
```
transient → field excluded from serialized byte stream (default value written instead)
static    → never serialized (not part of object state) — transient has no extra effect
final     → participates by value; transient has negligible effect
```

---

## Appendix: Consolidated "Only Applicable To…" List

| Modifier(s) | Exclusively applicable to |
|---|---|
| `native` | Methods (nothing else) |
| `transient`, `volatile` | Variables (nothing else) |
| `final` (for local scope) | Local variables (only legal local-variable modifier) |
| `strictfp` | Classes & methods (not variables) |
| `abstract` | Classes & methods (not variables) |
| `synchronized` | Methods & blocks (not variables/classes) |
| Constructor modifiers | `public`, `private`, `protected`, `<default>` only |

**End of Study Guide — Chapter 4: Declaration and Access Modifiers**
