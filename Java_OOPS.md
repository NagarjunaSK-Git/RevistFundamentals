# CORE JAVA — LANGUAGE FUNDAMENTALS & OOPS
### Senior Java Architect — Interview Preparation Study Material
*(Consolidated, technically validated, and updated with modern Java (8→21+) notes)*

---

## TABLE OF CONTENTS
1. Data Hiding, Abstraction, Encapsulation
2. Tightly Encapsulated Classes
3. IS-A Relationship (Inheritance)
4. HAS-A Relationship (Composition / Aggregation)
5. Method Signature
6. Polymorphism — Overloading
7. Polymorphism — Overriding
8. Method Hiding
9. Overloading vs Overriding — Master Table
10. Static Control Flow
11. Instance Control Flow
12. Constructors — Complete Rules
13. Coupling & Cohesion
14. Object Type Casting
15. Ways to Create Objects in Java
16. Singleton Classes & Factory Method Pattern
17. Modern Java Updates Relevant to OOPS
18. Interview Q&A Cheat Sheet

---

## 1. DATA HIDING, ABSTRACTION, ENCAPSULATION

### 1.1 Data Hiding
- Internal data of a class should **not** be directly accessible from outside the class.
- Implemented using the **`private`** access modifier.
- **Primary advantage:** Security.
- **Best practice:** Data members should *always* be declared `private`.

```java
class Account {
    private double balance;   // outside person can't access this directly
    // access only through controlled methods (getters/setters)
}
```

> **Architect's note:** Data hiding is a *language-level* mechanism (private modifier). It is the foundation on which Encapsulation is built. Don't confuse it with Abstraction — hiding *data* is different from hiding *implementation logic*.

### 1.2 Abstraction
- **Definition:** Hiding internal implementation and highlighting only the set of services offered.
- Implemented using **abstract classes** and **interfaces**.
- Real-world analogy: ATM GUI — the bank shows you *what* services are available (withdraw, deposit) without showing you *how* they are implemented internally.

**Advantages of Abstraction:**
| # | Advantage |
|---|-----------|
| 1 | Security — internal implementation is not exposed |
| 2 | Enhancement becomes easy without affecting end users |
| 3 | Provides flexibility to end users |
| 4 | Improves maintainability |
| 5 | Improves modularity |
| 6 | Improves ease of use |

### 1.3 Encapsulation
- **Definition:** Binding data and the corresponding methods that operate on that data into a single unit.
- **Formula (frequently asked in interviews):**
  ```
  Encapsulation = Data Hiding + Abstraction
  ```
- A Java class that follows *both* data hiding and abstraction is called an **encapsulated class**.
- Rule of thumb: every data member → `private`; every member → getter & setter.

```java
class Account {
    private double balance;

    public double getBalance() {
        // validate user
        return balance;
    }

    public void setBalance(double balance) {
        // validate user
        this.balance = balance;
    }
}
```

**Advantages:**
1. Achieves security.
2. Enhancement becomes very easy.
3. Improves maintainability and modularity.
4. Provides flexibility to use the system easily.

**Disadvantage:** Increases the length of code and (marginally) slows down execution due to method-call indirection (getter/setter) versus direct field access. In practice, JIT inlining largely negates this cost in modern JVMs.

---

## 2. TIGHTLY ENCAPSULATED CLASS

> **Definition:** A class is *tightly encapsulated* **if and only if** every variable of that class is declared `private` — **regardless of** whether getter/setter methods exist, and **regardless of** whether those methods are `public`. These checks (getter/setter existence, access modifier of methods) are irrelevant to the tight-encapsulation test.

```java
class Account {
    private double balance;
    public double getBalance() {
        return balance;
    }
}   // Tightly encapsulated (balance is private; getter's visibility doesn't matter)
```

### Worked Examples (classic interview trap)

```java
class A {
    private int x = 10;      // valid → tightly encapsulated
}
class B extends A {
    int y = 20;              // invalid → NOT tightly encapsulated (y not private)
}
class C extends A {
    private int z = 30;      // valid on its own, BUT...
}
```

```java
class A { int x = 10; }              // not tightly encapsulated
class B extends A { private int y = 20; }  // not (because A isn't)
class C extends B { private int z = 30; }  // not (because A isn't)
```

> **CRITICAL RULE:** *If the parent class is not tightly encapsulated, then **no** child class is tightly encapsulated* — even if every variable declared in the child itself is private. Tight encapsulation is evaluated across the **entire inheritance chain**.

**Architect tip:** Java **`record`** types (Java 16+) are tightly encapsulated by construction — all fields are implicitly `private final`, with public accessors generated automatically. This makes records a natural fit whenever "tight encapsulation + immutability" is the design goal (DTOs, value objects).

```java
record Point(int x, int y) { }   // x, y are private final; getters x(), y() auto-generated
```

---

## 3. IS-A RELATIONSHIP (INHERITANCE)

- Also known as **Inheritance**.
- Implemented using the **`extends`** keyword.
- **Main advantage:** Reusability.

```java
class Parent {
    public void methodOne() { }
}
class Child extends Parent {
    public void methodTwo() { }
}

class Test {
    public static void main(String[] args) {
        Parent p = new Parent();
        p.methodOne();
        // p.methodTwo();          // C.E: cannot find symbol — methodTwo() not in Parent

        Child c = new Child();
        c.methodOne();              // valid — inherited
        c.methodTwo();              // valid

        Parent p1 = new Child();    // valid — upcasting
        p1.methodOne();             // valid
        // p1.methodTwo();          // C.E: cannot find symbol — reference type governs compile-time check

        // Child c1 = new Parent(); // C.E: incompatible types — cannot downcast implicitly
    }
}
```

### Conclusion (frequently tested)
1. Whatever the Parent has, is by default available to the Child — but whatever the Child has is **not** by default available to the Parent. Hence, using a **Child reference**, we can call both Parent and Child methods. Using a **Parent reference**, we can call *only* Parent methods (not Child-specific methods).
2. A Parent class reference **can** hold a Child class object (upcasting), but through that reference we can only call methods present in the Parent class.
3. A Child class reference **cannot** hold a Parent class object (this is a compile-time error: incompatible types).

### Practical Example — Loan Hierarchy
```java
class Loan {
    // common methods required for any type of loan
}
class HousingLoan extends Loan {
    // Housing loan specific methods
}
class EducationLoan extends Loan {
    // Education loan specific methods
}
```

### Object as the Universal Root
- `Object` class acts as the root for **all** Java classes.
- `Throwable` class acts as the root for the **exception hierarchy** (`Exception` and `Error` both extend `Throwable`).

```
                Object
               /        \
           String       Throwable
                        /        \
                  Exception     Error
                     |
              RuntimeException, IOException, SQLException...
```

### Multiple Inheritance
- **Definition:** Having more than one Parent class at the same level.
- **Java does NOT support multiple inheritance of classes.**

```java
class A {}
class B {}
class C extends A, B {}   // INVALID — Java allows extending only ONE class
```

**Why?** — To avoid the classic **ambiguity ("Diamond") problem**:

```
Parent1.methodOne()      Parent2.methodOne()
        \                  /
         \                /
        C.methodOne();  --> Ambiguity: which methodOne() to inherit?
```

- **Interfaces CAN support multiple inheritance** because (traditionally) they only declared method signatures with no implementation — hence no ambiguity.

```java
interface Inter1 { void methodOne(); }
interface Inter2 { void methodOne(); }
interface Inter3 extends Inter1, Inter2 { }   // valid multiple inheritance of interfaces

class Test implements Inter3 {
    public void methodOne() {
        System.out.println("This is methodOne()");
    }
}
```

> ⚠️ **MODERN JAVA UPDATE (Java 8+):** Interfaces can now have **`default`** and **`static`** methods with bodies. This *reintroduces* the potential for the "Diamond Problem" in a limited form. Java resolves it with these rules:
> 1. **Classes win over interfaces** — an inherited/overridden method from a superclass always takes priority over any default method from an interface.
> 2. If two interfaces provide conflicting default methods with the same signature, the implementing class **must explicitly override** the method (compile-time error otherwise), optionally calling a specific interface's version via `InterfaceName.super.methodName()`.

```java
interface Inter1 { default void greet() { System.out.println("Inter1"); } }
interface Inter2 { default void greet() { System.out.println("Inter2"); } }

class Impl implements Inter1, Inter2 {
    @Override
    public void greet() {
        Inter1.super.greet();   // explicit disambiguation required
        Inter2.super.greet();
    }
}
```

### Object Class Hierarchy Rules
- If a class doesn't `extends` any other class → it is the **direct child** of `Object`.
- If a class extends another class → it is an **indirect child** of `Object`, forming **multilevel inheritance**.

```java
class B {}
class A extends B {}   // valid → multilevel inheritance (A → B → Object)
```

### Cyclic Inheritance
**Cyclic inheritance is NOT allowed in Java.**

```java
class A extends B {}   // C.E: cyclic inheritance involving A
class B extends A {}
```
```java
class A extends A {}   // C.E: cyclic inheritance involving A
```

---

## 4. HAS-A RELATIONSHIP (Composition / Aggregation)

- Also known as **Composition** or **Aggregation**.
- No specific keyword — implemented mostly via the `new` operator (an object reference as an instance field).
- **Main advantage:** Reusability.
- **Main disadvantage:** Increases dependency between components — creates maintenance problems (tight coupling).

```java
class Engine {
    // engine specific functionality
}
class Car {
    Engine e = new Engine();   // Car HAS-A Engine
}
```

### Composition vs Aggregation

| Aspect | Composition | Aggregation |
|---|---|---|
| Association strength | **Strong** | **Weak** |
| Lifecycle dependency | Contained object **cannot exist** without the container | Contained object **can exist** independently of the container |
| Example | `University` — `Department` (departments destroyed when university is destroyed) | `Department` — `Professor` (professor exists even if department closes) |
| Ownership | Container **holds** contained objects directly | Container just **holds a reference** to contained objects |

```java
// Composition example
class Department { /* ... */ }
class University {
    private final List<Department> departments = new ArrayList<>(); // owns lifecycle
}

// Aggregation example
class Professor { /* ... */ }
class Department {
    private List<Professor> professors;  // merely references, doesn't own lifecycle
    Department(List<Professor> professors) { this.professors = professors; }
}
```

> **Architect's note:** This is a UML/OOAD concept mapped onto Java. In real design discussions, distinguishing Composition vs Aggregation matters for **object lifecycle management**, especially in JPA/Hibernate (`CascadeType.ALL` vs no cascade), and in DDD (Aggregate Roots).

---

## 5. METHOD SIGNATURE

> **Definition:** In Java, a method signature consists of the **method name followed by argument types** (in order).

```java
public void methodOne(int i, float f);
// Signature: methodOne(int, float)
```

**Critical rules:**
1. **Return type is NOT part of the method signature.**
2. The **compiler uses the method signature** while resolving method calls (at compile time for overloading).
3. Within the same class, two methods **cannot** have the same signature — even if return types differ — otherwise **compile-time error**.

```java
class Test {
    public void methodOne() { }
    public int methodOne() { return 10; }
}
// Output: Compile time error — methodOne() is already defined in Test
```

```java
class Test {
    public void m1(double d) { }
    public void m2(int i) { }
    public static void main(String[] ar) {
        Test t = new Test();
        t.m1(10.5);
        t.m2(10);
        t.m3(10.5);   // C.E: cannot find symbol — method m3(double)
    }
}
```

---

## 6. POLYMORPHISM — OVERLOADING (Compile-Time / Static / Early Binding)

> **Definition:** Same name with different forms.
> "A boy starts love with the word *friendship*; a girl ends love with the same word *friendship* — the word is the same, but with different attitudes. This is polymorphism." *(classic Durga-ism, frequently quoted in interviews for a light-hearted intro to the concept.)*

### 6.1 Diagram — Polymorphism Types

```
                        Polymorphism
                       /             \
     Compile-time/static/early-binding   Runtime/dynamic/late-binding
        /              \                          \
   Overloading     Method Hiding                Overriding
```

### 6.2 Rules
1. Two methods are **overloaded** if and only if they have the **same name** but **different argument types** (or different order/number of arguments).
2. Historically in **C**, you could NOT overload — every distinct behavior needed a distinct name (`abs()`, `labs()`, `fabs()`). Lack of overloading increases programming complexity.
3. **Java supports overloading** — multiple methods, same name, different argument lists.
4. **Conclusion:** In overloading, the **compiler** resolves the method call based on the **reference type** (declared type), NOT the runtime object. Hence overloading = compile-time polymorphism = static polymorphism = early binding.

```java
class Test {
    public void methodOne() {
        System.out.println("no-arg method");
    }
    public void methodOne(int i) {
        System.out.println("int-arg method");
    }
    public void methodOne(double d) {
        System.out.println("double-arg method");
    }
    public static void main(String[] args) {
        Test t = new Test();
        t.methodOne();       // no-arg method
        t.methodOne(10);     // int-arg method
        t.methodOne(10.5);   // double-arg method
    }
}
```

### 6.3 Automatic Promotion in Overloading

- If the compiler can't find an **exact match**, it does NOT immediately raise a compile error.
- It **promotes** the argument to the next wider type, level by level, and re-checks.
- If, after all possible promotions, no match is found → **compile-time error**.

**Promotion Ladder:**
```
byte  ─┐
       ├─→ int ─→ long ─→ float ─→ double
char  ─┘
short ───→ int ─→ long ─→ float ─→ double
```

```java
class Test {
    public void methodOne(int i)   { System.out.println("int-arg method"); }
    public void methodOne(float f) { System.out.println("float-arg method"); }

    public static void main(String[] args) {
        Test t = new Test();
        t.methodOne('a');    // int-arg method   (char → int promotion)
        t.methodOne(10L);    // float-arg method (long → float promotion)
        // t.methodOne(10.5); // C.E: cannot find symbol — double can't be
                              // implicitly narrowed to int or float
    }
}
```

### 6.4 Case Studies (High-Frequency Interview Traps)

**Case 2 — String vs Object (exact match wins; child type wins over parent type)**
```java
class Test {
    public void methodOne(String s) { System.out.println("String version"); }
    public void methodOne(Object o) { System.out.println("Object version"); }
    public static void main(String[] args) {
        Test t = new Test();
        t.methodOne("arun");        // String version (exact match)
        t.methodOne(new Object());  // Object version (exact match)
        t.methodOne(null);          // String version (String is more specific/derived than Object)
    }
}
```
> **Rule:** While resolving overloaded methods, an **exact match always gets highest priority**. Among non-exact matches, the **child type wins over the parent type**.

**Case 3 — Ambiguous null (siblings, not parent-child)**
```java
class Test {
    public void methodOne(String s)       { System.out.println("String version"); }
    public void methodOne(StringBuffer s) { System.out.println("StringBuffer version"); }
    public static void main(String[] args) {
        Test t = new Test();
        t.methodOne("arun");                     // String version
        t.methodOne(new StringBuffer("sai"));    // StringBuffer version
        // t.methodOne(null);   // C.E: reference to methodOne is ambiguous
    }
}
```
> `String` and `StringBuffer` are **siblings** under `Object` — neither is more specific than the other, so `null` is ambiguous.

**Case 4 — Ambiguous overload with reversed parameter order**
```java
class Test {
    public void methodOne(int i, float f) { System.out.println("int-float method"); }
    public void methodOne(float f, int i) { System.out.println("float-int method"); }
    public static void main(String[] args) {
        Test t = new Test();
        t.methodOne(10, 10.5f);     // int-float method
        t.methodOne(10.5f, 10);     // float-int method
        // t.methodOne(10, 10);     // C.E: reference to methodOne is ambiguous
                                     // (both methodOne(int,float) & methodOne(float,int) match via promotion)
        // t.methodOne(10.5f,10.5f);// C.E: cannot find symbol — no matching signature at all
    }
}
```

**Case 5 — Varargs get the LEAST priority (like `default` in a switch)**
```java
class Test {
    public void methodOne(int i)     { System.out.println("general method"); }
    public void methodOne(int... i)  { System.out.println("var-arg method"); }
    public static void main(String[] args) {
        Test t = new Test();
        t.methodOne();        // var-arg method (only varargs matches — 0 args)
        t.methodOne(10, 20);  // var-arg method (only varargs matches — 2 args)
        t.methodOne(10);      // general method (exact match wins over varargs)
    }
}
```

**Case 6 — Overloading is decided by reference type; runtime object plays NO role**
```java
class Animal {}
class Monkey extends Animal {}
class Test {
    public void methodOne(Animal a) { System.out.println("Animal version"); }
    public void methodOne(Monkey m) { System.out.println("Monkey version"); }
    public static void main(String[] args) {
        Test t = new Test();
        Animal a = new Animal();
        t.methodOne(a);              // Animal version

        Monkey m = new Monkey();
        t.methodOne(m);              // Monkey version

        Animal a1 = new Monkey();    // upcasting
        t.methodOne(a1);             // Animal version — reference type (Animal) decides, NOT runtime object (Monkey)!
    }
}
```
> ⭐ **This is one of the single most-asked interview questions**: *"In overloading, method resolution is always based on the reference type at compile time — the runtime object never plays a role."*

---

## 7. POLYMORPHISM — OVERRIDING (Runtime / Dynamic / Late Binding)

### 7.1 Definition
- Whatever the Parent has, by default, is available to the Child through inheritance. If the Child is not satisfied with the Parent's implementation, it can **redefine** that method — this is called **overriding**.
- The Parent's method is called the **overridden method**.
- The Child's method is called the **overriding method**.

```java
class Parent {
    public void property() { System.out.println("cash+land+gold"); }
    public void marry()    { System.out.println("subbalakshmi"); }   // overridden method
}
class Child extends Parent {
    public void marry()    { System.out.println("3sha/4me/9tara/anushka"); }  // overriding method
}
class Test {
    public static void main(String[] args) {
        Parent p = new Parent();
        p.marry();               // subbalakshmi (parent method)

        Child c = new Child();
        c.marry();                // Child version (child method)

        Parent p1 = new Child();  // upcasting
        p1.marry();                // Child version — JVM resolves based on RUNTIME OBJECT!
    }
}
```

> **Conclusion:** In overriding, method resolution is **always** taken care of by the **JVM based on the runtime object** — this is called **Dynamic Method Dispatch**. Hence overriding = runtime polymorphism = dynamic polymorphism = late binding.
>
> **Note:** In overriding, the runtime object plays the role — the reference type is a "dummy" (irrelevant to which method body executes).

### 7.2 Rules for Overriding

**Rule 1 — Method name and arguments must match exactly (same signature).**

**Rule 2 — Return type:**
- Until Java 1.4: return types **must be identical**.
- **From Java 1.5 onwards: co-variant return types are allowed.** The child's method return type can be a **subtype** of the parent's return type.

```java
class Parent {
    public Object methodOne() { return null; }
}
class Child extends Parent {
    public String methodOne() { return null; }   // valid from 1.5+ — String IS-A Object
}
```
> **Co-variant return types apply ONLY to object/reference types — NOT to primitives.**
```
Parent return type:  Object → Number → String → double
Child  return type:  String → Integer → Object(✗) → int(✗)
```
(`String` is a subtype of `Object` ✓; `Integer` is a subtype of `Number` ✓; `Object` is NOT a subtype of `String` ✗; primitive covariance doesn't exist, `int` cannot "override" `double` ✗.)

**Rule 3 — Private methods are NOT visible in child classes → overriding does not apply to private methods.**
- You *can* declare a method with the same signature as a private parent method in the child — it's **valid Java**, but it is **NOT overriding** (it's an independent method).

```java
class Parent {
    private void methodOne() {}     // not visible to Child
}
class Child extends Parent {
    private void methodOne() {}     // valid, but NOT overriding
}
```

**Rule 4 — `final` methods in the Parent cannot be overridden in the Child.**
```java
class Parent {
    public final void methodOne() {}
}
class Child extends Parent {
    public void methodOne() {}   // C.E: methodOne() in Child cannot override methodOne() in Parent; overridden method is final
}
```
- However, a **non-final** parent method **can** be overridden as `final` in the child.
- `native` methods **can** be overridden in child classes.

**Rule 5 — `abstract` methods MUST be overridden (implemented) in a concrete child class.**
```java
abstract class Parent {
    public abstract void methodOne();
}
class Child extends Parent {
    public void methodOne() { }   // mandatory implementation
}
```
- A **non-abstract** method **can** be re-declared as `abstract` in the child (stops the parent implementation from propagating to further subclasses) — provided the child class itself is declared `abstract`.
```java
class Parent {
    public void methodOne() {}
}
abstract class Child extends Parent {
    public abstract void methodOne();   // valid — re-abstracting
}
```

**Rule 6 — `synchronized` and `strictfp` modifiers place NO restriction on overriding** (you can freely add/remove them).

```
final       → nonfinal   ✗ (not allowed)
nonfinal    → final      ✓
native      → nonnative  ✓
abstract    → nonabstract ✓
Synchronized→ nonSynchronized ✓
strictfp    → nonstrictfp ✓
```

**Rule 7 — While overriding, you CANNOT reduce the scope of the access modifier** (you CAN widen it).

```java
class Parent {
    public void methodOne() {}
}
class Child extends Parent {
    protected void methodOne() {}   // C.E: attempting to assign weaker access privileges; was public
}
```

**Access modifier widening rules:**
```
public     → public only
protected  → protected / public
<default>  → <default> / protected / public
private    → overriding concept not applicable at all
```
> Ordering: `private < default < protected < public`

### 7.3 Overriding & Checked vs Unchecked Exceptions

**Checked vs Unchecked Exceptions:**
- **Checked exceptions** are checked by the compiler for smooth execution at runtime.
- **Unchecked exceptions** are NOT checked by the compiler.
- `RuntimeException` (& its subclasses) and `Error` (& its subclasses) are **unchecked**. Everything else under `Throwable` is **checked**.

```
                     Object
                        │
                    Throwable
                   /          \
             Exception        Error
              /      \        /       \
      RuntimeException  IOException  LinkageError  VirtualMachineError
         │                  │                          │
   ArithmeticException  FileNotFoundException   OutOfMemoryError, StackOverflowError
   NullPointerException
   IndexOutOfBoundsException
   ClassCastException
   IllegalArgumentException → NumberFormatException
```

**Overriding Rule:** If the child class's overriding method throws a **checked** exception, the parent's overridden method **must** throw that same checked exception (or its superclass) — otherwise compile-time error. **No such restriction for unchecked exceptions.**

```java
class Parent {
    public void methodOne() {}
}
class Child extends Parent {
    public void methodOne() throws Exception {}   // C.E: overridden method does not throw java.lang.Exception
}
```

**Quick-reference validity table:**

| # | Parent | Child | Valid? |
|---|--------|-------|--------|
| 1 | `throws Exception` | (no throws) | ✅ Valid |
| 2 | (no throws) | `throws Exception` | ❌ Invalid |
| 3 | `throws Exception` | `throws Exception` | ✅ Valid |
| 4 | `throws IOException` | `throws Exception` | ❌ Invalid (Exception is broader than IOException) |
| 5 | `throws IOException` | `throws EOFException, FileNotFoundException` | ✅ Valid (both are subtypes of IOException) |
| 6 | `throws IOException` | `throws EOFException, InterruptedException` | ❌ Invalid (`InterruptedException` isn't related to IOException) |
| 7 | `throws IOException` | `throws EOFException, ArithmeticException` | ✅ Valid (ArithmeticException is unchecked — no restriction) |
| 8 | (no throws) | `throws ArithmeticException, NullPointerException, ClassCastException, RuntimeException` | ✅ Valid (all unchecked) |

### 7.4 Overriding w.r.t. Static Methods

**Case 1:** A **static** method cannot be overridden as **non-static**.
```java
class Parent { public static void methodOne() {} }
class Child extends Parent { public void methodOne() {} }
// C.E: methodOne() in Child cannot override methodOne() in Parent; overridden method is static
```

**Case 2:** Similarly, a **non-static** method cannot be overridden as **static**.

**Case 3:** static → static is **valid**, BUT it is NOT overriding — it's called **Method Hiding**.
```java
class Parent { public static void methodOne() {} }
class Child extends Parent { public static void methodOne() {} }   // valid — method hiding
```

### 7.5 Overriding w.r.t. Var-arg Methods
A var-arg method should be overridden **only** with a var-arg method. If overridden with a "normal" method, it becomes **overloading**, not overriding.

```java
class Parent {
    public void methodOne(int... i) { System.out.println("parent class"); }
}
class Child extends Parent {                        // overloading, NOT overriding
    public void methodOne(int i)   { System.out.println("child class"); }
}
class Test {
    public static void main(String[] args) {
        Parent p = new Parent();
        p.methodOne(10);           // parent class

        Child c = new Child();
        c.methodOne(10);           // child class

        Parent p1 = new Child();
        p1.methodOne(10);          // parent class  ← reference-type based (because it's OVERLOADING here)
    }
}
```
If the Child's method were also var-arg (`methodOne(int... i)`), this becomes **true overriding**, and `p1.methodOne(10)` would print `"child class"` (runtime-object based).

### 7.6 Overriding w.r.t. Variables
> **Overriding concept is NOT applicable to variables.** Variable resolution is **always** done by the compiler based on the **reference type** — never based on the runtime object.

```java
class Parent { int x = 888; }
class Child extends Parent { int x = 999; }
class Test {
    public static void main(String[] args) {
        Parent p = new Parent();
        System.out.println(p.x);          // 888

        Child c = new Child();
        System.out.println(c.x);          // 999

        Parent p1 = new Child();
        System.out.println(p1.x);         // 888 ← reference-type based, NOT runtime-object based!
    }
}
```
> Note: This behavior is unchanged regardless of `static`/non-static combinations on either variable — **variable resolution is always by reference type.**

---

## 8. METHOD HIDING

All rules of Method Hiding are exactly the same as Overriding **except**:

| Overriding | Method Hiding |
|---|---|
| Both Parent & Child methods must be **non-static** | Both Parent & Child methods must be **static** |
| Method resolution done by **JVM** based on **runtime object** | Method resolution done by **compiler** based on **reference type** |
| = Runtime/dynamic polymorphism (late binding) | = Compile-time/static polymorphism (early binding) |

```java
class Parent {
    public static void methodOne() { System.out.println("parent class"); }
}
class Child extends Parent {
    public static void methodOne() { System.out.println("child class"); }
}
class Test {
    public static void main(String[] args) {
        Parent p = new Parent();
        p.methodOne();              // parent class

        Child c = new Child();
        c.methodOne();              // child class

        Parent p1 = new Child();
        p1.methodOne();             // parent class  ← reference-type based (compile-time resolution)
    }
}
```
> **Contrast:** If both methods were **non-static** (true overriding), the last line would print `"child class"` because JVM resolves based on the runtime object.

---

## 9. OVERLOADING vs OVERRIDING — MASTER COMPARISON TABLE

| Property | Overloading | Overriding |
|---|---|---|
| Method names | Must be same | Must be same |
| Argument type | Must be different (at least order) | Must be exactly same (including order) |
| Method signature | Must be different | Must be same |
| Return type | No restrictions | Must be same until 1.4; from 1.5 onwards co-variant return types allowed |
| `private`, `static`, `final` methods | Can be overloaded | Cannot be overridden |
| Access modifiers | No restrictions | Widening allowed; weakening NOT allowed |
| `throws` clause | No restrictions | Checked exceptions in child must match/be subtype of parent's; no restriction on unchecked |
| Method resolution | Always by **compiler** based on **reference type** | Always by **JVM** based on **runtime object** |
| Also known as | Compile-time / static / early-binding polymorphism | Runtime / dynamic / late-binding polymorphism |

> **Interview gold nugget:** *"In overloading, check only the method name (same) and arguments (different) — nothing else matters. In overriding, you must check EVERYTHING: method name, arguments, return type, throws clause, access modifiers, etc."*

### Combined Rule-Checking Exercise
Given: `Parent: public void methodOne(int i) throws IOException`

| Child declaration | Verdict |
|---|---|
| `public void methodOne(int i)` | ✅ Valid (overriding — dropping a checked exception is allowed) |
| `private void methodOne() throws Exception` | ✅ Valid (overloading — different signature entirely) |
| `public native void methodOne(int i)` | ✅ Valid (overriding — native doesn't restrict) |
| `public static void methodOne(double d)` | ✅ Valid (overloading — different signature) |
| `public static void methodOne(int i)` | ❌ C.E — cannot override with `static`; overriding method is static |
| `public static abstract void methodOne(float f)` | ❌ C.E — illegal combination of modifiers: abstract & static; AND Child not abstract / doesn't override abstract method |

---

## 10. STATIC CONTROL FLOW

**Sequence of Events (Class Loading Time)** — this happens **once**, when the class is loaded by the JVM:
1. **Identification** of static members — from top to bottom.
2. **Execution** of static variable assignments and static blocks — from top to bottom (in the order they're written).
3. **Execution** of the `main()` method.

```java
class Base {
    static int i = 10;                                  // ① identified, ⑦ executed
    static {                                             // ② identified
        methodOne();                                     // ⑧ executed — indirect read of j (not yet assigned; RIWO state, default 0)
        System.out.println("first static block");        // ⑩
    }
    public static void main(String[] args) {              // ③ identified
        methodOne();                                     // ⑬ — j now = 20
        System.out.println("main method");                // ⑮
    }
    public static void methodOne() {                       // ④ identified
        System.out.println(j);                              // ⑨ prints 0 (RIWO), ⑭ prints 20
    }
    static {                                               // ⑤ identified
        System.out.println("second static block");        // ⑪
    }
    static int j = 20;                                     // ⑥ identified, ⑫ executed
}
```
**Output:**
```
0
first static block
second static block
20
main method
```

### 10.1 Read-Indirectly-Write-Only (RIWO) State
- Within a static block, if a variable is read **directly** before its declaration — this is a **"direct read"**.
- If a method is called, and *inside that method* the variable is read — this is an **"indirect read"**.
- **A variable in RIWO state can be read INDIRECTLY, but NOT DIRECTLY** — direct read causes a compile-time error: **"illegal forward reference."**

```java
class Test {
    static int i = 10;
    static {
        System.out.println(i);   // 10 — direct read AFTER declaration is fine
        System.exit(0);
    }
}
```
```java
class Test {
    static {
        System.out.println(i);   // C.E: illegal forward reference
    }
    static int i = 10;
}
```
```java
class Test {
    static {
        methodOne();              // valid — indirect read
    }
    public static void methodOne() {
        System.out.println(i);    // valid indirect read — prints 0 (default value; RIWO)
    }
    static int i = 10;
}
// Output: Runtime — prints "0"
// Then: NoSuchMethodError: main (because no main() method defined!)
```

### 10.2 Static Control Flow — Parent-to-Child Relationship
Whenever we execute a **Child class**, the JVM automatically performs:
1. **Identification** of static members from **Parent → Child**.
2. **Execution** of static variable assignments and static blocks from **Parent → Child**.
3. **Execution** of the **Child class's** `main()` method.

> **CRITICAL NOTE:** When we load a **Child** class, the **Parent** class is *automatically* loaded first (its static init runs). But when we load the **Parent** class directly, the **Child** class is **NOT** automatically loaded.

```java
class Base {
    static int j = 10;
    static { methodOne(); System.out.println("base static block"); }
    public static void main(String[] args) { methodOne(); System.out.println("base main"); }
    public static void methodOne() { System.out.println(j); }
}
class Derived extends Base {
    static int x = 100;
    static { methodTwo(); System.out.println("derived first static block"); }
    public static void main(String[] args) { methodTwo(); System.out.println("derived main"); }
    public static void methodTwo() { System.out.println(y); }
    static { System.out.println("derived second static block"); }
    static int y = 200;
}
```
Running `java Derived`:
```
0
Base static block
0
Derived first static block
Derived second static block
200
Derived main
```
Running `java Base` (Derived is NOT loaded at all):
```
0
Base static block
20
Base main
```

### 10.3 Static Blocks — Key Facts
- Static blocks execute at **class-loading time**.
- A class can have **multiple** static blocks — executed **top to bottom**.
- Use cases: loading native libraries, JDBC driver self-registration with `DriverManager`.

```java
class Driver {
    static {
        // Register this driver with DriverManager
    }
}
```

**Can you print without `main()`?** — YES, using a static block:
```java
class Google {
    static {
        System.out.println("hello i can print");
        System.exit(0);
    }
}
```

> ⚠️ **JAVA VERSION UPDATE — IMPORTANT & OFTEN OUTDATED IN OLD MATERIAL:**
> The original material states this rule is *"applicable until 1.6 — from 1.7 onwards `main()` is mandatory to RUN a java program."*
>
> **Modern clarification:** This is about **launching** a class via `java ClassName`. From **JDK 7 onward**, the `java` launcher requires a valid `public static void main(String[] args)` method to be present as an **entry point**; if absent, you get a runtime error like:
> ```
> Error: Main method not found in class Google, please define the main method as:
>    public static void main(String[] args)
> ```
> The **class still loads and its static blocks still execute** in older behavior descriptions, but modern `java` launchers perform the main-method check **before** invoking static initializers in many JDK versions — behavior has been refined release-to-release. **Always verify on the exact JDK version in a proper interview/lab environment** — don't state this as an absolute rule without version context.
>
> **Java 21 Preview / Java 25 feature — "Implicitly Declared Classes and Instance Main Methods" (JEP 445 / finalized under later JEPs):** Modern Java (21+ as preview, later finalized) allows simplified `main` methods without a class wrapper for beginner/script-style programs:
> ```java
> void main() {
>     System.out.println("Hello, World!"); // valid in Java 21+ preview / 25+ (JEP 445/463/477 evolution)
> }
> ```
> This doesn't change the class-loading/static-block fundamentals above — it's a **launch-protocol convenience feature**, not a change to OOP semantics.

---

## 11. INSTANCE CONTROL FLOW

**Sequence of Events (EVERY object creation — NOT one-time)**:
1. **Identification** of instance members — top to bottom.
2. **Execution** of instance variable assignments and instance blocks — top to bottom.
3. **Execution** of the constructor.

> **KEY DIFFERENCE vs Static Control Flow:**
> - **Static control flow** = ONE-TIME activity (executed at class-loading time).
> - **Instance control flow** = executed **EVERY TIME** an object is created (NOT one-time).

```java
class Parent {
    int x = 10;
    { methodOne(); System.out.println("Parent first instance block"); }
    Parent() { System.out.println("parent class constructor"); }
    public static void main(String[] args) {
        Parent p = new Parent();
        System.out.println("parent class main method");
    }
    public void methodOne() { System.out.println(y); }
    int y = 20;
}
```
Output:
```
0
first instance block
second instance block
Parent class constructor
main method
```
(Note: `y` is read indirectly before its assignment → prints default `0`, i.e., RIWO for instance context too.)

### Instance Control Flow — Parent to Child Relationship
Whenever a **Child** object is created:
1. Identification of instance members from **Parent → Child**.
2. Execution of instance variable assignments & instance blocks **only in Parent**.
3. Execution of the **Parent's constructor**.
4. Execution of instance variable assignments & instance blocks **in Child**.
5. Execution of the **Child's constructor**.

```java
class Parent {
    int x = 10;
    { methodOne(); System.out.println("Parent first instance block"); }
    Parent() { System.out.println("parent class constructor"); }
    public void methodOne() { System.out.println(y); }
    int y = 20;
}
class Child extends Parent {
    int i = 100;
    { methodTwo(); System.out.println("Child first instance block"); }
    Child() { System.out.println("Child class constructor"); }
    public void methodTwo() { System.out.println(j); }
    { System.out.println("Child second instance block"); }
    int j = 200;
}
```
`java Child` output:
```
0
Parent first instance block
parent class constructor
0
Child first instance block
Child second instance block
Child class constructor
```

> **Architect's note:** Object creation is the **most costly operation in Java** — never create objects without a specific requirement. This is foundational reasoning behind object pooling, caching (e.g., `Integer.valueOf()` caching -128 to 127), and Flyweight pattern usage.

### Accessing Instance Members from Static Context
```java
class Test {
    int i = 10;
    public static void main(String[] args) {
        System.out.println(i);   // C.E: non-static variable i cannot be referenced from a static context
    }
}
```
- **Instance members** cannot be accessed directly from a **static area** (at the time static area executes, instance members may not yet be identified/allocated).
- **Static members** CAN be accessed from anywhere directly — because they are identified at class-loading time.
- From an **instance area**, instance members CAN be accessed directly.

---

## 12. CONSTRUCTORS — COMPLETE RULES

### 12.1 Purpose
- Object creation alone is insufficient — the object must be **initialized** to respond correctly.
- A **constructor** is the piece of code that automatically executes at object-creation time to perform this initialization.

```java
class Student {
    String name;
    int rollno;
    Student(String name, int rollno) {   // Constructor
        this.name = name;
        this.rollno = rollno;
    }
    public static void main(String[] args) {
        Student s1 = new Student("vijayabhaskar", 101);
        Student s2 = new Student("bhaskar", 102);
    }
}
```

### 12.2 Constructor vs Instance Block

| Aspect | Constructor | Instance Block |
|---|---|---|
| Execution order | Instance block executes FIRST, then constructor | Executes before constructor |
| Purpose | Initialization of the object | Any activity needed on every object creation (other than pure init) |
| Arguments | Can take arguments | **Cannot** take arguments |
| Replaceable by other? | No | No |

```java
class Test {
    static int count = 0;
    { count++; }                    // instance block
    Test() {}
    Test(int i) {}
    public static void main(String[] args) {
        Test t1 = new Test();
        Test t2 = new Test(10);
        Test t3 = new Test();
        System.out.println(count);  // 3 — instance block runs for EVERY object regardless of constructor used
    }
}
```

### 12.3 Rules to Write Constructors
1. **Name of the constructor must match the class name exactly.**
2. **Return type concept is NOT applicable to constructors** — not even `void`. If you accidentally add a return type, the compiler treats it as a **normal method** (no compile error) — it's simply not a constructor anymore.

```java
class Test {
    void Test() {}   // This is a METHOD, not a constructor!
}
```
3. It is legal (but poor practice) to have a method with the same name as the class.
4. **Only applicable modifiers for constructors:** `public`, `<default>`, `private`, `protected`. Any other modifier (e.g., `static`, `final`, `abstract`, `synchronized`) → compile-time error.

```java
class Test {
    static Test() {}   // C.E: modifier static not allowed here
}
```

### 12.4 Default Constructor
1. Constructor concept applies to **every** Java class — including **abstract** classes.
2. If the programmer writes **no** constructor, the **compiler generates a default (no-arg) constructor**.
3. If the programmer writes **at least one** constructor (of any arity), the compiler generates **NO** default constructor. → Every class has *either* a compiler-generated constructor *or* programmer-written ones, **never both simultaneously**.

**Prototype of default constructor:**
1. It is always a **no-argument** constructor.
2. Access modifier = same as the class's access modifier (rule applies to `public`/default only).
3. Contains exactly one line: `super();` — a no-arg call to the superclass constructor.

| Programmer's code | Compiler-generated code |
|---|---|
| `class Test { }` | `class Test { Test() { super(); } }` |
| `public class Test { }` | `public class Test { public Test() { super(); } }` |
| `class Test { void Test(){} }` (a method, not constructor!) | `class Test { Test() { super(); } void Test(){} }` |
| `class Test { Test(int i) {} }` | No default generated (programmer wrote one); becomes `Test(int i) { super(); }` |
| `class Test { Test(int i){ this(); } Test(){} }` | `Test(int i){ this(); } Test(){ super(); }` |

### 12.5 `super()` vs `this()`

The **first line** of every constructor must be either `super()` or `this()`. If neither is written explicitly, the compiler **always inserts `super()`**.

**Case 1:** `super()`/`this()` must be the **first line** — anywhere else → compile-time error.
```java
class Test {
    Test() {
        System.out.println("constructor");
        super();   // C.E: Call to super must be first statement in constructor
    }
}
```

**Case 2:** You can use `super()` OR `this()` but **NOT both**.
```java
class Test {
    Test() {
        super();
        this();   // C.E: Call to this must be first statement in constructor
    }
}
```

**Case 3:** `super()`/`this()` can be used **ONLY inside a constructor**.
```java
class Test {
    public void methodOne() {
        super();  // C.E: Call to super must be first statement in constructor
    }
}
```
→ You can only call a constructor directly from **another constructor**.

**`super()`/`this()` vs `super`/`this` keywords:**

| `super()`, `this()` | `super`, `this` |
|---|---|
| Constructor calls | Keywords |
| Invoke super class / current class constructors directly | Refer to parent class / current class instance members |
| Must be first line inside constructor only; else compile error | Usable anywhere in instance area (NOT in static area); else compile error |

```java
class Test {
    public static void main(String[] args) {
        System.out.println(super.hashCode());
        // C.E: Non-static variable super cannot be referenced from a static context
    }
}
```

### 12.6 Overloaded Constructors
A class can have multiple constructors with different argument lists — these are **overloaded constructors**.

```java
class Test {
    Test(double d)  { System.out.println("double-argument constructor"); }
    Test(int i)     { this(10.5); System.out.println("int-argument constructor"); }
    Test()          { this(10); System.out.println("no-argument constructor"); }
    public static void main(String[] args) {
        Test t1 = new Test();      // double-arg → int-arg → no-arg constructor (chained)
        Test t2 = new Test(10);    // double-arg → int-arg constructor
        Test t3 = new Test(10.5);  // double-arg constructor
    }
}
```

- **Constructors are NOT inherited** — a Parent's constructor is NOT, by default, available to the Child. Hence **Inheritance and Overriding concepts do NOT apply to constructors.** But constructors **CAN be overloaded**.
- Constructors are valid inside **any** class including **abstract** classes, but **NOT inside interfaces** (traditional interfaces have no instance state to construct; this remains true even with default/static methods in modern Java — interfaces still cannot have constructors).

```java
class Test { Test() {} }                 // valid
abstract class Test { Test() {} }        // valid
interface Test1 { Test1() {} }           // INVALID
```

**Why can an abstract class have a constructor if you can't instantiate it?**
> The abstract class's constructor executes for **every child class object's creation**, to initialize the portion of state defined in the abstract (parent) class.

```java
abstract class Parent {
    Parent() {
        System.out.println(this.hashCode());   // "this" here refers to the CHILD object!
    }
}
class Child extends Parent {
    Child() {
        System.out.println(this.hashCode());
    }
}
class Test {
    public static void main(String[] args) {
        Child c = new Child();
        System.out.println(c.hashCode());   // all THREE hash codes are identical
    }
}
```
> **Key Insight:** *"Whenever we create a Child object, the Parent class constructor is executed — but a separate Parent OBJECT is NOT created."* `this` inside the Parent's constructor refers to the single Child object being constructed.

### 12.7 Recursive Constructor Invocation
> **Recursive method calls are always a RUNTIME exception (StackOverflowError), whereas recursive constructor invocation is a COMPILE-TIME error.**

```java
// Recursive method call → compiles fine, fails at runtime
class Test {
    public static void methodOne() { methodTwo(); }
    public static void methodTwo() { methodOne(); }
    public static void main(String[] args) {
        methodOne();
        System.out.println("hello");
    }
}
// R.E: StackOverflowError
```

```java
// Recursive constructor invocation → COMPILE-TIME error
class Test {
    Test(int i) { this(); }
    Test()      { this(10); }
    public static void main(String[] args) {
        System.out.println("hello");
    }
}
// C.E: recursive constructor invocation
```

**Compiler's Responsibilities Summary:**
1. Check whether the programmer wrote any constructor — if not, generate a default constructor.
2. Check whether the constructor's first line is `super()` or `this()` — if neither, generate (insert) `super()`.
3. Check for the possibility of recursive constructor invocation — if detected, raise a compile-time error.

### 12.8 Constructor Chaining — Parent/Child Interplay Traps

**Case 2 — Parent has only argument constructors → Child breaks:**
```java
class Parent {
    Parent(int i) {}
}
class Child extends Parent {}
// C.E: cannot find symbol — constructor Parent()
// (default constructor of Child implicitly calls super(), but Parent has no no-arg constructor!)
```
> **Best practice:** Whenever you write ANY argument-constructor in a class, it's highly recommended to **also** write a no-argument constructor — to avoid breaking subclasses relying on implicit `super()`.

**Case 3 — Checked exceptions in constructors propagate the same rule as methods:**
```java
class Parent {
    Parent() throws java.io.IOException {}
}
class Child extends Parent {}
// C.E: Unreported exception java.io.IOException in default constructor
```
```java
class Parent {
    Parent() throws java.io.IOException {}
}
class Child extends Parent {
    Child() throws Exception {
        super();
    }
}
// Valid — Child's constructor throws Exception (superclass of IOException)
```
> **Rule:** If a Parent class constructor throws a checked exception, the Child's constructor **must** throw the same checked exception (or its superclass).

---

## 13. COUPLING & COHESION

### 13.1 Coupling
> **Definition:** The degree of **dependency** between components.

```java
class A { static int i = B.j; }
class B extends A { static int j = C.methodOne(); }
class C extends B { public static int methodOne() { return D.k; } }
class D extends C { static int k = 10; }
```
> The above components are **tightly coupled** — heavy interdependency.

**Disadvantages of Tight Coupling:**
1. You cannot modify one component without affecting the others → enhancement becomes difficult.
2. Reduces maintainability.
3. Doesn't promote reusability.

> ✅ **Best practice:** Always aim for **LOOSE coupling**.

### 13.2 Cohesion
> **Definition:** For every component, maintain a **clear, well-defined, single responsibility**. Such a component is said to have **high cohesion**.

```
TotalServlet.java (login + validation + inbox + compose + forward + ...)  → LOW cohesion
             vs
login.jsp → ValidateServlet → inbox.jsp / compose.jsp / error.jsp        → HIGH cohesion
```

**Advantages of High Cohesion:**
1. Modify any component without affecting others → easy enhancement.
2. Improves maintainability.
3. Promotes reusability (e.g., a `ValidateServlet` can be reused wherever validation is needed).

> ✅ **Golden rule:** *"Always follow LOOSE coupling and HIGH cohesion."* — This is the SOLID / clean-architecture design mantra every Senior/Architect-level candidate must articulate clearly.

---

## 14. OBJECT TYPE CASTING

- A Parent reference **can** hold a Child object, but through the Parent reference, we **cannot** call Child-specific methods directly.

```java
Object o = new String("ashok");        // valid — upcasting
System.out.println(o.hashCode());       // valid — hashCode() exists in Object
System.out.println(o.length());
// C.E: cannot find symbol — method length(), location: class java.lang.Object
```

- Similarly, an **interface** reference can hold an implementing class's object:
```java
Runnable r = new Thread();
```

### Type-Casting Syntax
```
A b = (C) d;
    │      │  └── name of reference variable/object
    │      └───── class/interface (the cast target type)
    └──────────── class/interface (declared type of "b")
```

### 14.1 Compile-Time Checking Rules

**Rule 1:** The type of `d` and the cast type `C` must have SOME relationship (Parent-Child in either direction, or identical type) — otherwise: **"inconvertible types"** compile error.

```java
Object o = new String("bhaskar");
StringBuffer sb = (StringBuffer) o;    // valid at compile time (Object ↔ StringBuffer are related via inheritance)
```
```java
String s = new String("bhaskar");
StringBuffer sb = (StringBuffer) s;
// C.E: inconvertible types — found: java.lang.String, required: java.lang.StringBuffer
// (String and StringBuffer have NO inheritance relationship — both are direct Object subclasses, siblings)
```

**Rule 2:** The cast target type `C` must be either the same type as, or a **derived type** of, the *declared* type `A` — otherwise **"incompatible types"** compile error.

```java
Object o = new String("bhaskar");
StringBuffer sb = (StringBuffer) o;   // valid — StringBuffer IS-A derived-relation candidate via Object
```
```java
Object o = new String("bhaskar");
StringBuffer sb = (String) o;
// C.E: incompatible types — found: java.lang.String, required: java.lang.StringBuffer
// (assigning a String-typed cast result to a StringBuffer reference — mismatched target types)
```

### 14.2 Runtime Checking Rule
> The **underlying (actual runtime) object type** must be either the same as, or a derived type of, the cast target type `C` — otherwise: **`java.lang.ClassCastException`** at runtime.

```java
Object o = new String("bhaskar");
StringBuffer sb = (StringBuffer) o;
// Compiles fine (satisfies both compile-time rules), but at RUNTIME:
// java.lang.ClassCastException: class java.lang.String cannot be cast to class java.lang.StringBuffer
```

**Multi-Level Hierarchy Example:**
```
                    Object
                  /        \
              Base1        Base2
             /     \       /     \
       Derived1  Derived2 Derived3 Derived4
```
```java
Base1 b = new Derived2();       // valid
Object o = (Base1) b;           // valid
Object o1 = (Base2) o;          // C.E: inconvertible types — Base1 & Base2 are siblings, unrelated
Object o2 = (Base2) b;          // C.E: inconvertible types
Base2 b1 = (Base1)(new Derived1());  // C.E: incompatible types (Base1 not related to Base2)
Base2 b2 = (Base2)(new Derived3());  // valid — Derived3 IS-A Base2
Base2 b3 = (Base2)(new Derived1());  // C.E: inconvertible types — Derived1 is under Base1, unrelated to Base2
```

> **Key architect insight:** Type casting does **NOT** create a new object. It merely provides a **different type of reference variable** (usually a Parent-type view) to the **SAME underlying object**.

```java
String s = new String("bhaskar");
Object o = (Object) s;
System.out.println(s == o);   // true — same object, different reference "view"
```

### 14.3 Overriding + Casting Interaction (Runtime Object Wins)
```java
class A { public void methodOne() { System.out.println("A"); } }
class B extends A { public void methodOne() { System.out.println("B"); } }
class C extends B { public void methodOne() { System.out.println("C"); } }
// This is OVERRIDING — resolution is by RUNTIME OBJECT, casting the reference has NO effect:
C c = new C();
c.methodOne();               // C
((B) c).methodOne();         // C
((A) ((B) c)).methodOne();   // C
```

### 14.4 Method Hiding + Casting Interaction (Reference Type Wins)
```java
class A { public static void methodOne() { System.out.println("A"); } }
class B extends A { public static void methodOne() { System.out.println("B"); } }
class C extends B { public static void methodOne() { System.out.println("C"); } }
// This is METHOD HIDING (static methods) — resolution is by REFERENCE TYPE:
C c = new C();
c.methodOne();               // C
((B) c).methodOne();         // B
((A) ((B) c)).methodOne();   // A
```

### 14.5 Variables + Casting Interaction (Reference Type Wins — ALWAYS)
```java
class A { int x = 777; }
class B extends A { int x = 888; }
class C extends B { int x = 999; }
C c = new C();
System.out.println(c.x);                 // 999
System.out.println(((B) c).x);            // 888
System.out.println(((A) ((B) c)).x);       // 777
```
> Variable resolution is **always** based on reference type — whether variables are `static` or non-static does not change this.

---

## 15. WAYS TO CREATE OBJECTS IN JAVA (Classic IIQ)

1. **`new` operator:**
   ```java
   Test t = new Test();
   ```
2. **Reflection — `Class.newInstance()` (deprecated) / `Constructor.newInstance()` (preferred, modern):**
   ```java
   // Legacy (deprecated since Java 9):
   Test t = (Test) Class.forName("Test").newInstance();

   // Modern replacement (Java 9+):
   Test t = (Test) Class.forName("Test").getDeclaredConstructor().newInstance();
   ```
3. **`clone()` — via the `Cloneable` interface:**
   ```java
   Test t1 = new Test();
   Test t2 = (Test) t1.clone();
   ```
4. **Factory methods:**
   ```java
   Runtime r = Runtime.getRuntime();
   DateFormat df = DateFormat.getInstance();
   ```
5. **Deserialization:**
   ```java
   FileInputStream fis = new FileInputStream("abc.ser");
   ObjectInputStream ois = new ObjectInputStream(fis);
   Test t = (Test) ois.readObject();
   ```

> ⭐ **MODERN JAVA ADDITIONS (worth mentioning in a Senior Architect interview):**
> 6. **Records** (Java 16+) create objects via a compact canonical constructor, but conceptually still fall under `new` — worth mentioning as a modern *tightly-encapsulated, immutable* object-creation idiom:
>    ```java
>    record Point(int x, int y) {}
>    Point p = new Point(10, 20);
>    ```
> 7. **`MethodHandles` / `VarHandle`** — lower-level, high-performance object/field manipulation APIs (java.lang.invoke), often used in frameworks instead of classic Reflection for performance reasons.
> 8. **Dependency Injection containers** (Spring, CDI) internally use reflection/bytecode generation (CGLIB, ByteBuddy) to create proxy objects — an architect should be aware object creation in enterprise apps is frequently mediated by a container, not raw `new`.

---

## 16. SINGLETON CLASSES & FACTORY METHOD PATTERN

### 16.1 Singleton Classes
> **Definition:** A Java class that allows creation of **only one object**.

**JDK Examples:** `Runtime`, `ActionServlet` (Struts), `ServiceLocator`, `BusinessDelegate` (J2EE patterns).

```java
Runtime r1 = Runtime.getRuntime();   // getRuntime() is a FACTORY METHOD
Runtime r2 = Runtime.getRuntime();
Runtime r3 = Runtime.getRuntime();

System.out.println(r1 == r2);   // true
System.out.println(r1 == r3);   // true — all three references point to the SAME single object
```

### 16.2 Creating Your Own Singleton Class
**Ingredients:** `private` constructor + `private static` variable + `public static` factory method.

```java
class Test {
    private static Test t = null;
    private Test() {}                 // private constructor — blocks external instantiation
    public static Test getTest() {    // factory method
        if (t == null) {
            t = new Test();
        }
        return t;
    }
}
class Client {
    public static void main(String[] args) {
        System.out.println(Test.getTest().hashCode());  // same hashCode every time
        System.out.println(Test.getTest().hashCode());
    }
}
```

> ⚠️ **THREAD-SAFETY WARNING (Interview Must-Know — Not in Original Text but Essential for Senior Architects):**
> The classic lazy-initialization singleton shown above is **NOT thread-safe**. In a multi-threaded environment, two threads could both pass the `if (t == null)` check simultaneously and create **two** instances. Modern, production-grade singleton implementations:
>
> **Option A — Synchronized method (simple, some throughput cost):**
> ```java
> public static synchronized Test getTest() {
>     if (t == null) t = new Test();
>     return t;
> }
> ```
> **Option B — Double-Checked Locking (better throughput):**
> ```java
> private static volatile Test t = null;   // volatile is CRITICAL here
> public static Test getTest() {
>     if (t == null) {
>         synchronized (Test.class) {
>             if (t == null) {
>                 t = new Test();
>             }
>         }
>     }
>     return t;
> }
> ```
> **Option C — Initialization-on-demand holder idiom (thread-safe, no synchronization overhead, JVM-guaranteed):**
> ```java
> class Test {
>     private Test() {}
>     private static class Holder {
>         private static final Test INSTANCE = new Test();
>     }
>     public static Test getTest() {
>         return Holder.INSTANCE;
>     }
> }
> ```
> **Option D — Enum Singleton (Joshua Bloch's recommended approach — serialization-safe, reflection-safe):**
> ```java
> enum Test {
>     INSTANCE;
>     public void doSomething() { /* ... */ }
> }
> // Usage: Test.INSTANCE.doSomething();
> ```
> The **enum singleton** is widely considered the **best practice** because the JVM guarantees a single instance per enum constant, handles serialization correctly by default, and is immune to reflection-based instantiation attacks (unlike private-constructor classes, which CAN still be broken via `Constructor.setAccessible(true)`).

### 16.3 Multi-ton Classes (double-ton, triple-ton, etc.)
```java
class Test {
    private static Test t1 = null;
    private static Test t2 = null;
    private Test() {}
    public static Test getTest() {
        if (t1 == null) { t1 = new Test(); return t1; }
        else if (t2 == null) { t2 = new Test(); return t2; }
        else {
            return (Math.random() < 0.5) ? t1 : t2;   // Math.random(): 0.0 <= x < 1.0
        }
    }
}
```

### 16.4 "Cannot create Child, but class isn't `final`" trick
```java
class Parent {
    private Parent() {}   // ALL constructors private → no subclass can call super()
}
// No class can extend Parent successfully — compiler enforces:
// "Parent() has private access in Parent" if a Child tries super()
```
> **Note:** Whenever we create a Child object, the Parent constructor automatically executes — but a **separate Parent object is NOT created.**

### 16.5 Factory Method
> **Definition:** A method that, when called via the **class name**, **returns an object of that same class**.

```java
Runtime r = Runtime.getRuntime();          // getRuntime() is a factory method
DateFormat df = DateFormat.getInstance();  // getInstance() is a factory method
```
> Use factory methods when object creation needs to happen **under specific constraints** (e.g., pooling, singleton enforcement, caching, or returning subtype instances based on input parameters — the classic GoF **Factory Method Pattern**).

---

## 17. MODERN JAVA UPDATES RELEVANT TO OOPS (Beyond the Legacy Source Material)

Since this source material reflects pre-Java-8 era Core Java (SCJP/OCJP), a Senior Architect must be aware of these OOP-relevant evolutions:

| Feature | JDK | OOP Relevance |
|---|---|---|
| **`default` & `static` methods in interfaces** | 8 | Interfaces can now carry implementation → partial multiple-inheritance-of-behavior; requires explicit disambiguation (`Interface.super.method()`) on conflicts |
| **Functional Interfaces & Lambdas** | 8 | Enables treating behavior as data; underpins Streams API; interacts with polymorphism via method references |
| **`var` (local variable type inference)** | 10 | Does NOT change static typing — `var` is inferred at compile time; still subject to all overloading/overriding rules above |
| **Sealed Classes/Interfaces (`sealed`, `permits`)** | 17 (final) | Restricts which classes can `extend`/`implement` a type — directly controls IS-A relationship at compile time; complements exhaustive pattern matching in `switch` |
| **Records (`record`)** | 16 (final) | Auto-generates a *tightly encapsulated*, immutable class with `private final` fields, canonical constructor, accessors, `equals()`/`hashCode()`/`toString()` |
| **Pattern Matching for `instanceof`** | 16 (final) | `if (obj instanceof String s) { ... }` — combines type-check + cast in one step, reducing classic casting boilerplate discussed in Section 14 |
| **Pattern Matching for `switch`** | 21 (final) | Enables type-based dispatch resembling polymorphic behavior without method overriding, useful with sealed hierarchies |
| **Virtual Threads (Project Loom)** | 21 (final) | Not OOP per se, but changes concurrency-design guidance around synchronized blocks/singleton patterns (avoid pinning virtual threads with heavy `synchronized` blocks) |
| **Deprecation of `Class.newInstance()`** | 9 | Use `Constructor.newInstance()` — throws more specific/checked exceptions; `Class.newInstance()` is deprecated and its rough-edge exception behavior is a known pitfall |
| **Implicitly Declared Classes / Instance main methods** | 21 preview → 25 (evolving) | Simplifies teaching/scripting Java — does not alter deep OOP semantics covered in this material |

### Sealed Classes Example (Modern equivalent for controlled inheritance discussed in Section 3)
```java
public sealed interface Shape permits Circle, Square, Triangle {}

public final class Circle implements Shape { /* ... */ }
public final class Square implements Shape { /* ... */ }
public final class Triangle implements Shape { /* ... */ }

// Any other class attempting `implements Shape` → compile-time error
```
> This is the modern, compiler-enforced way to control "IS-A" relationships at the design level — arguably a refined counterpart to the classic "single inheritance to avoid ambiguity" discussion in Section 3.

---

## 18. INTERVIEW Q&A CHEAT SHEET

**Q1. What is the difference between Data Hiding, Abstraction, and Encapsulation?**
> Data Hiding = `private` fields (security). Abstraction = hiding implementation, exposing services (abstract classes/interfaces). Encapsulation = Data Hiding + Abstraction, i.e., binding data & behavior into one unit.

**Q2. Is a class with all `private` fields but `public` getters/setters tightly encapsulated?**
> Yes — getter/setter presence and their access modifiers are irrelevant to the tight-encapsulation check. Only the field's `private` status matters.

**Q3. If Parent is not tightly encapsulated, can Child be?**
> No. Tight encapsulation must hold across the entire inheritance chain.

**Q4. Why doesn't Java support multiple inheritance of classes?**
> To avoid the ambiguity ("diamond") problem when two parent classes provide conflicting implementations of the same method signature.

**Q5. Why can interfaces support multiple inheritance, but this changed slightly in Java 8+?**
> Traditional interfaces only declared abstract methods (no implementation → no ambiguity). Java 8's `default` methods reintroduced possible conflicts, resolved by mandatory explicit override using `Interface.super.method()`.

**Q6. Difference between Composition and Aggregation?**
> Composition = strong association; contained object cannot exist without the container (e.g., University-Department). Aggregation = weak association; contained object can exist independently (e.g., Department-Professor).

**Q7. Is return type part of the method signature?**
> No. Method signature = method name + argument types only.

**Q8. Overloading is resolved by ___, Overriding is resolved by ___?**
> Overloading → compiler, based on **reference type** (compile-time / static / early binding).
> Overriding → JVM, based on **runtime object** (runtime / dynamic / late binding, via Dynamic Method Dispatch).

**Q9. Can you overload the `main()` method?**
> Yes — you can have multiple `main()` methods with different signatures, but the JVM will only invoke `public static void main(String[] args)` as the entry point.

**Q10. Can a `private` method be overridden?**
> No — private methods are not visible in child classes, so overriding doesn't apply. You CAN declare a method with the same signature in a child, but it's an independent method, not an override.

**Q11. Can a `static` method be overridden?**
> No — static-to-static "override" is actually **Method Hiding**, resolved at compile time by reference type, not JVM dynamic dispatch.

**Q12. What happens when overriding a var-arg method with a normal (fixed-arg) method?**
> It becomes **overloading**, not overriding — var-arg signatures must be overridden with var-arg signatures to remain true overriding.

**Q13. Are variables subject to overriding?**
> No — variable resolution is always done by the compiler based on the reference (declared) type, never the runtime object.

**Q14. What is the rule for checked exceptions in overriding?**
> If the child's overriding method throws a checked exception, the parent's method must throw that same checked exception (or a superclass of it). No restriction applies to unchecked exceptions.

**Q15. Can you reduce the visibility of an overridden method?**
> No — overriding can only widen (or keep same) access; narrowing (e.g., `public` → `protected`) causes a compile-time error.

**Q16. Difference between `ArrayList al = new ArrayList()` and `List l = new ArrayList()`?**
> Use `ArrayList` reference when you know the exact runtime type and need ArrayList-specific methods (e.g., `ensureCapacity()`, `trimToSize()`). Use `List` reference for loose coupling/polymorphism when the exact list implementation may vary (`LinkedList`, `Vector`, `Stack`, etc.) — you lose access to implementation-specific methods but gain flexibility (Program to Interface, not Implementation — a core SOLID principle).

**Q17. What causes `ClassCastException` vs a compile-time casting error?**
> Compile-time errors ("inconvertible types" / "incompatible types") occur when there's NO possible inheritance relationship between the declared and target types. `ClassCastException` occurs at RUNTIME when the compile-time check passes (types are related in the class hierarchy) but the actual object's runtime type is incompatible with the cast target.

**Q18. Name 5 ways to create an object in Java.**
> `new` operator; Reflection (`Class.forName().getDeclaredConstructor().newInstance()`); `clone()`; Factory methods (`Runtime.getRuntime()`); Deserialization (`ObjectInputStream.readObject()`). (Bonus: records via canonical constructor, DI container proxies.)

**Q19. Is the classic lazy-init Singleton pattern thread-safe?**
> No — without synchronization/volatile, race conditions can create multiple instances. Use Double-Checked Locking, the Initialization-on-Demand Holder idiom, or (best practice, per Joshua Bloch) an `enum` singleton.

**Q20. What is a Factory Method?**
> A method invoked via the class name that returns an object of that same class type, typically used to enforce creation constraints (e.g., singleton enforcement, object pooling, or polymorphic instantiation based on input).

**Q21. Explain Coupling vs Cohesion, and the recommended design target.**
> Coupling = degree of inter-component dependency (aim: LOW/loose). Cohesion = how well-defined and single-purpose each component's responsibility is (aim: HIGH). Best practice: **loose coupling + high cohesion**.

**Q22. Why is `this` inside an abstract class's constructor referring to the Child object?**
> Because when a Child object is instantiated, no separate Parent object is created — the Parent's constructor merely initializes the Parent-defined portion of the SAME (Child) object being built. `this` always refers to the object currently under construction, which is the Child instance.

**Q23. What's the difference between recursive method calls and recursive constructor invocation in terms of error detection?**
> Recursive method calls compile fine but throw `StackOverflowError` at runtime. Recursive constructor invocation (via `this()`) is caught by the compiler as a **compile-time error**.

**Q24. How does Java 8+ change the "interfaces avoid ambiguity" argument from Section 3?**
> Default methods introduce implementation into interfaces, which can conflict when a class implements two interfaces with the same default method signature. Java forces the implementing class to override and resolve the conflict explicitly — it doesn't silently pick one (unlike some other languages), preserving type safety.

**Q25. What are co-variant return types, and what's the primitive limitation?**
> From Java 1.5+, an overriding method's return type can be a subtype of the parent method's return type (e.g., `Object` → `String`). This ONLY works for reference/object types — primitives have no covariance (`double` cannot "become" `int` in an override).

---

*End of Study Material — Language Fundamentals / OOPS (Core Java, Senior Architect Interview Track)*
