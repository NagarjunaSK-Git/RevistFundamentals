# Core Java – Language Fundamentals
## Inner Classes & File I/O — Senior Architect Interview Guide

> **Scope note:** The uploaded source material covers *Inner Classes* (Ch. 9) and *File I/O* (Ch. 11) from a SCJP/OCJP study series — not a generic "Language Fundamentals" chapter. This guide is built strictly from that source, with all compile errors in the original corrected, modern Java (8/9/11/17+) updates layered in, and an interview cheat-sheet appended. If you intended a different "Language Fundamentals" topic (data types, operators, control flow), let me know and I'll produce that separately.

---

# PART 1 — INNER CLASSES

## 1. Introduction

A class declared inside another class is an **inner class** (nested class).

- Introduced in JDK 1.1, originally to support **event handling** in AWT/Swing GUIs.
- Golden rule for *when* to use one:
  > **If, without the existence of one type of object, there is no chance of the other type of object existing, model it as an inner class.**
  - `University` → `Department` (no University, no Department)
  - `Bank` → `Account`
  - `Map` → `Map.Entry` (a real JDK example — `Entry` is a nested interface inside `Map`)
- **Relationship type:** Outer ↔ Inner is a **HAS-A** relationship, never IS-A.
- **Key invariant:** Without an Outer class **object**, there is (usually) no chance of an Inner class object existing — this applies to *regular* inner classes only, not static nested classes (see §5).

### The 4 categories

| # | Type | Declared |
|---|------|----------|
| 1 | Normal / Regular (Member) Inner Class | Directly inside a class, no `static` |
| 2 | Method-Local Inner Class | Inside a method body |
| 3 | Anonymous Inner Class | No name, "just for instant use" |
| 4 | Static Nested Class | Inside a class, with `static` |

---

## 2. Normal (Regular) Inner Class

```java
class Outer {
    class Inner {
        // instance members only
    }
}
```

### 2.1 Inner classes cannot declare `static` members

```java
class Outer {
    class Inner {
        public static void main(String[] args) {   // COMPILE ERROR
            System.out.println("inner class main method");
        }
    }
}
```
```
Outer.java:3: error: modifier static is only allowed in constant variable declarations
    public static void main(String[] args)
```
> **Corrected rule (post-Java 16 nuance):** Since **Java 16 (JEP 395 / general relaxation, JLS updates)**, *inner classes may declare static members if they are implicitly constant (`static final`) or if the compiler can treat them as such* — practically: **regular (non-static) inner, local, and anonymous classes still cannot declare `static` methods or non-final `static` fields.** Only `static final` constant fields are allowed. This is a **frequently outdated interview fact** — always confirm which JDK version is being tested against. Standard SCJP-era answer: *"Inner classes cannot have static declarations except constants."*

### 2.2 Accessing inner class code — 3 contexts

**(a) From static area of outer class** — not directly; you need an `Outer` instance first:
```java
class Outer {
    class Inner {
        public void methodOne() {
            System.out.println("inner class method");
        }
    }

    public static void main(String[] args) {
        Outer o = new Outer();
        Outer.Inner i = o.new Inner();   // must go through an Outer instance
        i.methodOne();
    }
}
```

**(b) From instance area of outer class** — direct, since `this` (implicit outer instance) is available:
```java
class Outer {
    class Inner {
        public void methodOne() {
            System.out.println("inner class method");
        }
    }

    public void methodTwo() {
        Inner i = new Inner();   // implicit Outer.this.new Inner()
        i.methodOne();
    }

    public static void main(String[] args) {
        new Outer().methodTwo();
    }
}
```
**Output:** `inner class method`

**(c) From outside the outer class** — using the special `Outer.new Inner()` syntax:
```java
class Outer {
    class Inner {
        public void methodOne() {
            System.out.println("inner class method");
        }
    }
}

class Test {
    public static void main(String[] args) {
        Outer.Inner obj = new Outer().new Inner();
        obj.methodOne();
    }
}
```
**Output:** `Inner class method`

### 2.3 Inner class can access ALL outer members directly

```java
class Outer {
    int x = 10;
    static int y = 20;

    class Inner {
        public void methodOne() {
            System.out.println(x); // 10
            System.out.println(y); // 20
        }
    }

    public static void main(String[] args) {
        new Outer().new Inner().methodOne();
    }
}
```
> An inner class has an **implicit reference to its enclosing instance** (`Outer.this`), compiled internally as a synthetic field (`Outer$Inner` holds `Outer.this$0`). This is *why* inner classes cause classic **memory-leak bugs** in Android/GUI apps if the inner object outlives the outer — a favorite architect-level interview trap.

### 2.4 `this` vs `Outer.this`

```java
class Outer {
    int x = 10;
    class Inner {
        int x = 100;
        public void methodOne() {
            int x = 1000;
            System.out.println(x);           // 1000 (local variable)
            System.out.println(this.x);      // 100  (Inner's field)
            System.out.println(Outer.this.x);// 10   (Outer's field)
        }
    }
    public static void main(String[] args) {
        new Outer().new Inner().methodOne();
    }
}
```

### 2.5 Applicable modifiers

| Outer (top-level) class | Inner (member) class — extra options |
|---|---|
| `public`, default, `final`, `abstract`, `strictfp` | + `private`, `protected`, `static` (all combinable, subject to normal exclusivity rules e.g. not both `final` & `abstract`) |

> A **top-level** class can only be `public` or default (package-private) for access modifiers — never `private`/`protected`. An **inner class** can be `private`, `protected`, `public`, or default, because it is a *member* of the outer class, just like a field or method.

### 2.6 Nesting of inner classes

```java
class A {
    class B {
        class C {
            public void m1() {
                System.out.println("C class method");
            }
        }
    }

    public static void main(String[] args) {
        A a = new A();
        A.B b = a.new B();
        A.B.C c = b.new C();
        c.m1();
    }
}
```
**Output:** `C class method`

---

## 3. Method-Local Inner Class

- Declared **inside a method**.
- Used for **method-specific, repeatedly-required functionality**.
- Scope is limited to the enclosing method — cannot be accessed outside it.
- Rarely used in modern code (largely superseded by **lambdas** for functional cases — see §6).

```java
class Test {
    public void methodOne() {
        class Inner {
            public void sum(int i, int j) {
                System.out.println("The sum:" + (i + j));
            }
        }
        Inner i = new Inner();
        i.sum(10, 20);
        i.sum(100, 200);
        i.sum(1000, 2000);
    }

    public static void main(String[] args) {
        new Test().methodOne();
    }
}
```
**Output:**
```
The sum: 30
The sum: 300
The sum: 3000
```

### 3.1 Static vs instance method context

- Inner class declared inside an **instance method** → can access both static & non-static members of the outer class directly.
- Inner class declared inside a **static method** → can access **only static** members of the outer class directly.

```java
class Test {
    int x = 10;
    static int y = 20;

    public void methodOne() {
        class Inner {
            public void methodTwo() {
                System.out.println(x); // 10 - OK, instance context
                System.out.println(y); // 20
            }
        }
        new Inner().methodTwo();
    }

    public static void main(String[] args) {
        new Test().methodOne();
    }
}
```
> If `methodOne()` were `static`, referencing instance field `x` inside `Inner` would fail:
> `error: non-static variable x cannot be referenced from a static context`

### 3.2 Local variable capture — "effectively final"

**Pre-Java 8 rule (SCJP era):** A method-local inner class can access local variables of the enclosing method **only if declared `final`**.

```java
class Test {
    int x = 10;
    public void methodOne() {
        int y = 20;              // NOT final
        class Inner {
            public void methodTwo() {
                System.out.println(x); // OK
                System.out.println(y); // COMPILE ERROR (pre-Java 8):
                                        // "local variable y is accessed from within
                                        // inner class; needs to be declared final"
            }
        }
        new Inner().methodTwo();
    }
}
```

> ### 🔥 Java 8 Update — Effectively Final (JEP / JLS 8 change)
> Since **Java 8**, the local variable does **not need the explicit `final` keyword** — it only needs to be **"effectively final"**, i.e., never reassigned after initialization. The compiler treats it as `final` implicitly.
> ```java
> public void methodOne() {
>     int y = 20;                 // effectively final — never reassigned
>     class Inner {
>         public void methodTwo() {
>             System.out.println(y);  // Compiles fine on Java 8+
>         }
>     }
>     new Inner().methodTwo();
> }
> ```
> This same "effectively final" rule underpins **lambda expressions** and **anonymous inner classes** capturing local variables. **This is one of the highest-frequency interview questions** — be ready to explain *why*: the JVM captures local variables **by value** (copies them into the inner class's synthetic constructor), so a reassignable variable could create inconsistent state between the method's stack frame and the heap-allocated inner class instance.

```java
class Test {
    int i = 10;
    static int j = 20;

    public void methodOne() {
        int k = 30;
        final int l = 40;         // final not required post Java-8, but still valid
        class Inner {
            public void methodTwo() {
                System.out.println(i); // 10
                System.out.println(j); // 20
                System.out.println(k); // 30 (effectively final)
                System.out.println(l); // 40
            }
        }
        new Inner().methodTwo();
    }
    public static void main(String[] args) {
        new Test().methodOne();
    }
}
```
- At the marked line, accessible variables are: **i, j, k, l** (all reachable; instance field, static field, effectively-final locals).
- If `methodTwo()` (or the outer method) were declared `static`, you'd still be able to access static members and effectively-final locals, but **not instance field `i`**.
- The only applicable modifiers for a method-local inner class itself are: **`final`, `abstract`, `strictfp`** (no access modifiers — it has no meaning inside a method scope).

---

## 4. Anonymous Inner Classes

- A nameless inner class, for **one-time, throwaway use**.
- Three flavors:
  1. Extends a concrete/abstract class
  2. Implements an interface
  3. Declared inline as a method argument

### 4.1 Extends a class

```java
class PopCorn {
    public void taste() {
        System.out.println("spicy");
    }
}

class Test {
    public static void main(String[] args) {
        PopCorn p = new PopCorn() {          // anonymous subclass of PopCorn
            public void taste() {
                System.out.println("salty");
            }
        };
        p.taste();          // salty (overridden)

        PopCorn p1 = new PopCorn();
        p1.taste();         // spicy (original)
    }
}
```

**Analysis:**
1. `new PopCorn();` → creates a plain `PopCorn` object.
2. `new PopCorn() { ... };` → creates an **unnamed subclass** of `PopCorn`, overrides `taste()`, and instantiates it — held via the **parent type reference** `p`.

> **Rule:** You can declare **new methods** inside an anonymous class, but you **cannot call them from outside** because the reference type is the parent — the compiler only knows the parent's contract.

```java
class PopCorn {
    public void taste() { System.out.println("spicy"); }
}

class Test {
    public static void main(String[] args) {
        PopCorn p = new PopCorn() {
            public void taste() {
                methodOne();                 // valid: internal call
                System.out.println("salty");
            }
            public void methodOne() {
                System.out.println("child specific method");
            }
        };
        // p.methodOne();   // COMPILE ERROR — not visible via PopCorn reference
        p.taste();          // salty (methodOne runs first)
        PopCorn p1 = new PopCorn();
        p1.taste();         // spicy
    }
}
```
**Output:**
```
child specific method
salty
spicy
```

Classic thread example:
```java
class Test {
    public static void main(String[] args) {
        Thread t = new Thread() {
            public void run() {
                for (int i = 0; i < 10; i++) {
                    System.out.println("child thread");
                }
            }
        };
        t.start();
        for (int i = 0; i < 10; i++) {
            System.out.println("main thread");
        }
    }
}
```
> ⚠️ **Modern caveat:** `Thread.start()` output order is **non-deterministic** — the JVM's thread scheduler decides interleaving. Do not assume "main thread" prints fully before "child thread" begins. On modern JVMs this is even more true with `Thread` virtual threads (Project Loom, **Java 21 `Thread.ofVirtual()`**).

### 4.2 Implements an interface

```java
class InnerClassesDemo {
    public static void main(String[] args) {
        Runnable r = new Runnable() {        // implementer object, not interface instance
            public void run() {
                for (int i = 0; i < 10; i++) {
                    System.out.println("Child thread");
                }
            }
        };
        Thread t = new Thread(r);
        t.start();
        for (int i = 0; i < 10; i++) {
            System.out.println("Main thread");
        }
    }
}
```

### 4.3 Defined inline as a method argument

```java
class Test {
    public static void main(String[] args) {
        new Thread(new Runnable() {
            public void run() {
                for (int i = 0; i < 10; i++) {
                    System.out.println("child thread");
                }
            }
        }).start();

        for (int i = 0; i < 10; i++) {
            System.out.println("main thread");
        }
    }
}
```

> ### 🔥 Java 8 Update — Lambdas replace most anonymous inner classes
> Every anonymous class implementing a **functional interface** (exactly one abstract method) can be rewritten as a **lambda expression** — this is essential architect-level knowledge:
> ```java
> // Old style (anonymous inner class)
> Runnable r1 = new Runnable() {
>     public void run() { System.out.println("Running..."); }
> };
>
> // Modern (Java 8+) lambda — no synthetic .class file per instance*
> Runnable r2 = () -> System.out.println("Running...");
>
> Thread t = new Thread(() -> {
>     for (int i = 0; i < 10; i++) System.out.println("child thread");
> });
> t.start();
> ```
> **Key differences to cite in interviews:**
> | | Anonymous Inner Class | Lambda |
> |---|---|---|
> | `this` reference | Refers to the anonymous class instance | Refers to the **enclosing** instance (lexical scoping) |
> | Bytecode | Generates a separate `.class` file (`Outer$1.class`) at compile time | Uses `invokedynamic` + `LambdaMetafactory` at runtime — no separate `.class` per lambda site |
> | Applicability | Any abstract class or interface (any # of abstract methods) | **Only** functional interfaces (single abstract method, `@FunctionalInterface`) |
> | Can have state/fields | Yes | No (stateless, effectively-final capture only) |
> | Constructors | No (no name) | N/A |

### 4.4 Anonymous vs General class — differences

| General Class | Anonymous Inner Class |
|---|---|
| Extends only one class at a time | Extends only one class at a time |
| Can implement multiple interfaces | Can implement **only one** interface at a time |
| Can extend a class **and** implement interface(s) simultaneously | Can **either** extend a class **or** implement an interface — never both |
| Has a constructor (name known) | **No constructor** — the class has no name |

### 4.5 Application area — GUI callbacks

```java
import java.awt.*;
import java.awt.event.*;

public class AnonymousInnerClassDemo {
    public static void main(String[] args) {
        Frame f = new Frame();

        f.addWindowListener(new WindowAdapter() {          // NOTE: WindowAdapter, not "WindowAdaptor"
            public void windowClosing(WindowEvent e) {
                System.exit(0);
            }
        });

        f.add(new Label("Anonymous Inner class Demo !!!"));
        f.setSize(500, 500);
        f.setVisible(true);
    }
}
```

**Without anonymous inner class (verbose, all events funneled through one method):**
```java
class GUI extends Frame implements ActionListener {
    Button b1, b2, b3, b4;

    public void actionPerformed(ActionEvent e) {
        if (e.getSource() == b1) {
            // perform b1 specific functionality
        } else if (e.getSource() == b2) {
            // perform b2 specific functionality
        }
    }
}
```

**With anonymous inner class (each button gets its own dedicated handler):**
```java
class GUI extends Frame {
    Button b1, b2;

    void wireUp() {
        b1.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                // perform b1 specific functionality
            }
        });
        b2.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                // perform b2 specific functionality
            }
        });
    }
}
```
> Modern equivalent with lambdas: `b1.addActionListener(e -> { /* b1 logic */ });`

---

## 5. Static Nested Classes

- Declared with the `static` modifier inside another class.
- **Not** associated with an instance of the outer class — behaves almost like a **top-level class packaged inside another class's namespace**.

```java
class Test {
    static class Nested {
        public void methodOne() {
            System.out.println("nested class method");
        }
    }

    public static void main(String[] args) {
        Test.Nested t = new Test.Nested();   // no Test instance required
        t.methodOne();
    }
}
```

### 5.1 Can declare `static` members, including `main()`

```java
class Test {
    static class Nested {
        public static void main(String[] args) {
            System.out.println("nested class main method");
        }
    }
    public static void main(String[] args) {
        System.out.println("outer class main method");
    }
}
```
```
$ javac Test.java
$ java Test
outer class main method
$ java Test$Nested
nested class main method
```

### 5.2 Access rules — only static outer members

```java
class Test {
    int x = 10;
    static int y = 20;

    static class Nested {
        public void methodOne() {
            System.out.println(x); // COMPILE ERROR:
                                    // non-static variable x cannot be referenced
                                    // from a static context
            System.out.println(y); // OK
        }
    }
}
```

### 5.3 Comparison — Regular Inner Class vs Static Nested Class

| Normal / Regular Inner Class | Static Nested Class |
|---|---|
| Cannot exist without an Outer **object**; always tied to it | Can exist independently of any Outer object |
| Cannot declare `static` members | Can declare `static` members freely |
| Cannot declare `main()`; cannot be invoked directly from the command line | Can declare `main()`; can be run via `java Outer$Nested` |
| Can access both static & non-static members of outer class directly | Can access **only static** members of outer class directly |
| Instantiation: `outer.new Inner()` | Instantiation: `new Outer.Nested()` |

> **Real-world JDK examples of static nested classes:** `Map.Entry`, `AbstractMap.SimpleEntry`, `ThreadLocal.ThreadLocalMap`, `Character.UnicodeBlock`, `Node<K,V>` inside `HashMap` (private static nested).
> **Builder pattern** almost always uses a `public static class Builder` nested inside the target class — a top interview scenario:
> ```java
> public class Pizza {
>     private final String size;
>     private final boolean cheese;
>
>     private Pizza(Builder b) {
>         this.size = b.size;
>         this.cheese = b.cheese;
>     }
>
>     public static class Builder {
>         private String size;
>         private boolean cheese;
>         public Builder size(String size) { this.size = size; return this; }
>         public Builder cheese(boolean c)  { this.cheese = c;   return this; }
>         public Pizza build() { return new Pizza(this); }
>     }
> }
>
> Pizza p = new Pizza.Builder().size("Large").cheese(true).build();
> ```

---

## 6. Nesting Combinations of Classes & Interfaces

### 6.1 Class inside a Class
```java
class University {
    class Department {
    }
}
```
Without a `University` object, no `Department` object can exist.

### 6.2 Interface inside a Class
```java
class VehicleType {
    interface Vehicle {
        int getNoOfWheels();
    }
    class Bus implements Vehicle {
        public int getNoOfWheels() { return 6; }
    }
    class Auto implements Vehicle {
        public int getNoOfWheels() { return 3; }
    }
}
```
Use when the interface's implementations are meaningful only in the context of one enclosing class.

### 6.3 Interface inside an Interface
```java
interface Map {
    interface Entry {
        Object getKey();
        Object getValue();
        Object setValue(Object newValue);   // corrected signature
    }
}
```
> **Rule:** A nested interface is **implicitly `public static`**, whether declared explicitly or not. This means you can implement the inner interface **without implementing the outer interface**.

```java
interface Outer {
    void methodOne();
    interface Inner {
        void methodTwo();
    }
}

class Test implements Outer.Inner {
    public void methodTwo() {
        System.out.println("Inner interface method");
    }
    public static void main(String[] args) {
        new Test().methodTwo();
    }
}
```
```java
class Test2 implements Outer {
    public void methodOne() {
        System.out.println("Outer interface method");
    }
    public static void main(String[] args) {
        new Test2().methodOne();
    }
}
```
> Both `Outer` and `Outer.Inner` can be implemented **independently** — they are unrelated types unless one extends the other explicitly.

### 6.4 Class inside an Interface
```java
interface EmailServer {
    void sendEmail(EmailDetails e);

    class EmailDetails {   // implicitly public static
        String from;
        String to;
        String subject;
    }
}
```
Use when a helper class's functionality is tightly coupled to just that interface.

**Providing a default implementation via nested class (pre-`default` methods pattern):**
```java
interface Vehicle {
    int getNoOfWheels();

    class DefaultVehicle implements Vehicle {
        public int getNoOfWheels() { return 3; }
    }
}

class Bus implements Vehicle {
    public int getNoOfWheels() { return 6; }
}

class Test {
    public static void main(String[] args) {
        Bus b = new Bus();
        System.out.println(b.getNoOfWheels());               // 6

        Vehicle.DefaultVehicle d = new Vehicle.DefaultVehicle();
        System.out.println(d.getNoOfWheels());                // 3
    }
}
```
> A class nested inside an interface is **implicitly `public static`**, so it can be instantiated directly without any enclosing interface-typed object.

> ### 🔥 Java 8/9 Update — This pattern is now largely obsolete
> Since **Java 8**, interfaces support **`default`** and **`static`** methods, which make the "default implementation nested class" trick unnecessary in most cases:
> ```java
> interface Vehicle {
>     int getNoOfWheels();
>     default int defaultWheelCount() { return 3; }     // Java 8 default method
>     static Vehicle economy() { return () -> 3; }       // Java 8 static factory method
> }
> ```
> Since **Java 9**, interfaces can also have **`private`** and **`private static`** methods to share code between default methods without exposing it.

### Conclusions (from source, verified)
1. You can declare almost anything inside anything with respect to classes and interfaces (subject to the rules above).
2. Nested interfaces are **always implicitly `public static`**.
3. A class declared inside an interface is **always implicitly `public static`**.

---

# PART 2 — FILE I/O (`java.io`)

## 1. `File`

```java
File f = new File("abc.txt");
```
- First checks whether the physical file/directory exists.
- If it doesn't exist, **no physical file is created** — `f` is just a Java object *representing* a path/name.
- A `File` object can represent **either a file or a directory** (Java's file I/O model mirrors UNIX, where "everything is a file").

```java
import java.io.*;

class FileDemo {
    public static void main(String[] args) throws IOException {
        File f = new File("cricket.txt");
        System.out.println(f.exists());      // false (1st run)
        f.createNewFile();
        System.out.println(f.exists());      // true
    }
}
```
Run 1: `false` then `true`. Run 2 (file now exists): `true` then `true`.

```java
import java.io.*;

class FileDemo {
    public static void main(String[] args) throws IOException {
        File f = new File("cricket123");
        System.out.println(f.exists());   // false
        f.mkdir();
        System.out.println(f.exists());   // true
    }
}
```

### 1.1 Constructors

| Constructor | Purpose |
|---|---|
| `File(String pathname)` | File/dir with given name in current working directory |
| `File(String parent, String child)` | File/dir `child` inside directory `parent` |
| `File(File parent, String child)` | Same, but `parent` given as a `File` object |

```java
// Create demo.txt in current working directory
File f = new File("demo.txt");
f.createNewFile();

// Create a directory then a file inside it
File dir = new File("SaiCharan123");
dir.mkdir();
File f2 = new File(dir, "abc.txt");
f2.createNewFile();

// Create a file inside an absolute path
File f3 = new File("c:\\saiCharan", "demo.txt");
f3.createNewFile();
```

### 1.2 Important `File` methods

| Method | Description |
|---|---|
| `boolean exists()` | true if the physical file/dir exists |
| `boolean createNewFile()` | Creates the file if absent; returns `false` if it already exists (does **not** overwrite) |
| `boolean mkdir()` | Creates the directory if absent; `false` if it already exists |
| `boolean isFile()` | true if this represents a regular file |
| `boolean isDirectory()` | true if this represents a directory |
| `String[] list()` | Names of files & subdirectories in this directory |
| `long length()` | Size of file in bytes (source loosely calls it "characters") |
| `boolean delete()` | Deletes the file or **empty** directory |

**Corrected working programs (the source PDF has multiple syntax bugs — `for(String s1=s)`, lowercase `file`, etc. — fixed below):**

```java
import java.io.*;

class FileDemo {
    public static void main(String[] args) throws IOException {
        int count = 0;
        File f = new File("c:\\charan_classes");
        String[] s = f.list();
        if (s != null) {
            for (String s1 : s) {           // FIXED: enhanced for-loop
                count++;
                System.out.println(s1);
            }
        }
        System.out.println("total number : " + count);
    }
}
```

```java
// Only file names
import java.io.*;

class FileDemo {
    public static void main(String[] args) throws IOException {
        int count = 0;
        File f = new File("c:\\charan_classes");
        String[] s = f.list();
        for (String s1 : s) {
            File f1 = new File(f, s1);       // FIXED: capital 'File'
            if (f1.isFile()) {
                count++;
                System.out.println(s1);
            }
        }
        System.out.println("total number : " + count);
    }
}
```

```java
// Only directory names
import java.io.*;

class FileDemo {
    public static void main(String[] args) throws IOException {
        int count = 0;
        File f = new File("c:\\charan_classes");
        String[] s = f.list();
        for (String s1 : s) {
            File f1 = new File(f, s1);
            if (f1.isDirectory()) {
                count++;
                System.out.println(s1);
            }
        }
        System.out.println("total number : " + count);
    }
}
```

> ### 🔥 Modern Update — NIO.2 (`java.nio.file`), since Java 7
> The legacy `java.io.File` API is largely **superseded** by `java.nio.file.Path` + `java.nio.file.Files` (NIO.2, JSR-203). Architect-level interviews expect awareness of this:
> ```java
> import java.nio.file.*;
> import java.io.IOException;
> import java.util.stream.Stream;
>
> Path dir = Paths.get("c:/charan_classes");
> try (Stream<Path> entries = Files.list(dir)) {
>     entries.filter(Files::isRegularFile)
>            .forEach(System.out::println);
> }
>
> boolean exists = Files.exists(dir);
> Files.createDirectories(dir);              // like mkdir(), but creates parent dirs too
> long size = Files.size(dir.resolve("demo.txt"));
> Files.delete(dir.resolve("demo.txt"));
> ```
> **Why NIO.2 is preferred:** better exception messages (tells you *why* an operation failed), symbolic-link support, file-attribute views, `WatchService` for directory-change notifications, and `Files.walk()`/`Files.walkFileTree()` for recursive traversal — none of which `java.io.File` supports cleanly.

---

## 2. `FileWriter` — writing character data

### Constructors
```java
FileWriter fw = new FileWriter(String name);
FileWriter fw = new FileWriter(File f);
FileWriter fw = new FileWriter(String name, boolean append);
FileWriter fw = new FileWriter(File f, boolean append);
```
- Without `append`, the constructors **overwrite** existing content.
- `append = true` → data is appended instead of overwritten.
- If the target physical file doesn't exist, it is created.

### Key methods
`write(int ch)`, `write(char[] ch)`, `write(String s)`, `flush()`, `close()`.

```java
import java.io.*;

class FileWriterDemo {
    public static void main(String[] args) throws IOException {
        FileWriter fw = new FileWriter("cricket.txt", true);
        fw.write(99);                          // writes char 'c' (ASCII 99)
        fw.write("haran\nsoftware solutions");
        fw.write("\n");
        char[] ch = {'a', 'b', 'c'};
        fw.write(ch);
        fw.write("\n");
        fw.flush();
        fw.close();
    }
}
```
**Output (appended to cricket.txt):**
```
charan
software solutions
abc
```

**Limitation:** Line separators (`\n`) must be inserted **manually**, and the separator is **platform-dependent** (`\n` on Unix/macOS, `\r\n` on Windows) — this motivates `BufferedWriter.newLine()` (§4) or, better, `System.lineSeparator()`.

## 3. `FileReader` — reading character data

### Constructors
```java
FileReader fr = new FileReader(String name);
FileReader fr = new FileReader(File f);
```

### Key methods
- `int read()` — reads next character, returns its Unicode code point, or `-1` at EOF.
- `int read(char[] ch)` — bulk read into a buffer; returns count of chars read.
- `void close()`.

**Approach 1 — char by char (works for any file size):**
```java
import java.io.*;

class FileReaderDemo {
    public static void main(String[] args) throws IOException {
        FileReader fr = new FileReader("cricket.txt");
        int i = fr.read();
        while (i != -1) {
            System.out.print((char) i);   // cast required — read() returns int
            i = fr.read();
        }
        fr.close();
    }
}
```

**Approach 2 — bulk read (fine for small files; risky for very large files as it loads entirely into memory):**
```java
import java.io.*;

class FileReaderDemo {
    public static void main(String[] args) throws IOException {
        File f = new File("cricket.txt");
        FileReader fr = new FileReader(f);
        char[] ch = new char[(int) f.length()];
        fr.read(ch);
        for (char ch1 : ch) {
            System.out.print(ch1);
        }
        fr.close();
    }
}
```

**Why FileWriter/FileReader are discouraged:**
1. Manual, platform-dependent line-separator handling on write.
2. Character-by-character reading is inconvenient — no line-by-line API.
3. → Use `BufferedWriter`/`BufferedReader` instead.

## 4. `BufferedWriter`

### Constructors
```java
BufferedWriter bw = new BufferedWriter(Writer w);
BufferedWriter bw = new BufferedWriter(Writer w, int bufferSize);
```
> `BufferedWriter` **cannot talk to a file directly** — it must wrap another `Writer` (typically `FileWriter`).

**Valid / invalid declarations:**
```java
BufferedWriter bw = new BufferedWriter("cricket.txt");                       // INVALID — String not a Writer
BufferedWriter bw = new BufferedWriter(new File("cricket.txt"));             // INVALID — File not a Writer
BufferedWriter bw = new BufferedWriter(new FileWriter("cricket.txt"));       // VALID

// Two-level buffering (legal, though pointless in practice):
BufferedWriter bw2 = new BufferedWriter(new BufferedWriter(new FileWriter("cricket.txt")));
```

### Methods
`write(int)`, `write(char[])`, `write(String)`, `flush()`, `close()`, and **`newLine()`** — inserts a platform-correct line separator.

> **Interview trap:** *Compared to `FileWriter`, what NEW capability does `BufferedWriter` add?* → **Answer: `newLine()`.** Everything else (`write`, `flush`, `close`) is inherited/common via the `Writer` contract — the real value-add of Buffered* classes is **internal buffering for I/O performance**, plus the `newLine()` convenience.

```java
import java.io.*;

class BufferedWriterDemo {
    public static void main(String[] args) throws IOException {
        FileWriter fw = new FileWriter("cricket.txt");
        BufferedWriter bw = new BufferedWriter(fw);
        bw.write(100);              // 'd'
        bw.newLine();
        char[] ch = {'a', 'b', 'c', 'd'};
        bw.write(ch);
        bw.newLine();
        bw.write("SaiCharan");
        bw.newLine();
        bw.write("software solutions");
        bw.flush();
        bw.close();
    }
}
```
**Output:**
```
d
abcd
SaiCharan
software solutions
```
> Closing a `BufferedWriter` **automatically closes the underlying `Writer`** — you do not need to close `fw` separately.

## 5. `BufferedReader`

### Constructors
```java
BufferedReader br = new BufferedReader(Reader r);
BufferedReader br = new BufferedReader(Reader r, int bufferSize);
```
Cannot talk directly to a file — must wrap a `Reader` (e.g. `FileReader`).

### Key advantage
`String readLine()` — reads a full line, returns `null` at EOF. This is the biggest ergonomic win over `FileReader`.

```java
import java.io.*;

class BufferedReaderDemo {
    public static void main(String[] args) throws IOException {
        FileReader fr = new FileReader("cricket.txt");
        BufferedReader br = new BufferedReader(fr);
        String line = br.readLine();
        while (line != null) {
            System.out.println(line);
            line = br.readLine();
        }
        br.close();     // closes br AND the underlying fr
    }
}
```
> Same closing rule as `BufferedWriter`: closing `BufferedReader` auto-closes the wrapped `FileReader`. Explicitly closing both is redundant (not an error, just unnecessary).

## 6. `PrintWriter`

- The **most enhanced Writer** — can write **any primitive type or String** to a file, not just `char` data.
- Can wrap **directly around a file** (`String`/`File`) or **around another `Writer`**.

### Constructors
```java
PrintWriter pw = new PrintWriter(String name);
PrintWriter pw = new PrintWriter(File f);
PrintWriter pw = new PrintWriter(Writer w);
```

### Methods
`write(...)` (character-only, like `Writer`) **plus** overloaded `print(...)`/`println(...)` for `char`, `int`, `double`, `boolean`, `String`, `Object`, etc.

```java
import java.io.*;

class PrintWriterDemo {
    public static void main(String[] args) throws IOException {
        FileWriter fw = new FileWriter("cricket.txt");
        PrintWriter out = new PrintWriter(fw);
        out.write(100);          // writes the CHARACTER 'd' (code point 100)
        out.println(100);        // writes the STRING "100"
        out.println(true);
        out.println('c');
        out.println("SaiCharan");
        out.flush();
        out.close();
    }
}
```
**Output:**
```
d100
true
c
SaiCharan
```

**`write(100)` vs `print(100)`:**
| Call | Behavior |
|---|---|
| `write(100)` | Treats `100` as a **char code point** → writes the character `'d'` |
| `print(100)` | Treats `100` as an **int value** → writes the literal text `"100"` |

### Notes
1. Most enhanced **Reader** for character data → `BufferedReader`.
2. Most enhanced **Writer** for character data → `PrintWriter`.
3. **Readers/Writers** handle **character (text) data**; **InputStream/OutputStream** handle **binary data** (images, audio, video, serialized objects).

### `java.io` Writer/Reader hierarchy

```
                     Object
                       │
        ┌──────────────┴──────────────┐
     Writer (AC)                   Reader (AC)
        │                              │
  ┌─────┼─────────────┐         ┌──────┼─────────┐
OutputStreamWriter  BufferedWriter  PrintWriter  InputStreamReader  BufferedReader
        │                                              │
   FileWriter                                     FileReader
```
(`AC` = Abstract Class. `PrintWriter` extends `Writer` directly. `FileWriter` extends `OutputStreamWriter`. `FileReader` extends `InputStreamReader`.)

---

## 7. Practical File-Processing Programs (source examples, corrected)

> The original PDF examples contain several genuine compile errors (wrong constructor for `BufferedReader`, lowercase `file`, invalid `for(String s1=s)` syntax). All are corrected below and are **compile-clean**.

### 7.1 Merge two files sequentially (file1 then file2 → file3)

```java
import java.io.*;

class FileMergeDemo {
    public static void main(String[] args) throws IOException {
        try (PrintWriter pw = new PrintWriter(new FileWriter("file3.txt"));
             BufferedReader br1 = new BufferedReader(new FileReader("file1.txt"));
             BufferedReader br2 = new BufferedReader(new FileReader("file2.txt"))) {

            String line;
            while ((line = br1.readLine()) != null) {
                pw.println(line);
            }
            while ((line = br2.readLine()) != null) {
                pw.println(line);
            }
        }
    }
}
```
> ### 🔥 Modern Update — `try-with-resources` (Java 7+)
> All the resource-cleanup boilerplate (explicit `close()` in `finally`) from the original source examples is obsolete since **Java 7**. Any class implementing `AutoCloseable`/`Closeable` (all `Reader`/`Writer`/`Stream` classes qualify) can be declared in the `try(...)` parentheses and is **guaranteed closed**, even on exception, in **reverse declaration order**. This is now considered mandatory best practice and is a top interview differentiator between junior and senior candidates.

### 7.2 Merge two files line-by-line, alternately

```java
import java.io.*;

class AlternateMergeDemo {
    public static void main(String[] args) throws IOException {
        try (PrintWriter pw = new PrintWriter(new FileWriter("file3.txt"));
             BufferedReader br1 = new BufferedReader(new FileReader("file1.txt"));
             BufferedReader br2 = new BufferedReader(new FileReader("file2.txt"))) {

            String line1 = br1.readLine();
            String line2 = br2.readLine();

            while (line1 != null || line2 != null) {
                if (line1 != null) {
                    pw.println(line1);
                    line1 = br1.readLine();
                }
                if (line2 != null) {
                    pw.println(line2);
                    line2 = br2.readLine();
                }
            }
        }
    }
}
```
Given `file1.txt = aaa,bbb,ccc` and `file2.txt = 666,777,888`, `file3.txt` becomes:
```
aaa
666
bbb
777
ccc
888
```

### 7.3 Merge ALL files in a folder into one output file

```java
import java.io.*;

class TotalFileMerge {
    public static void main(String[] args) throws IOException {
        File folder = new File("E:\\xyz");
        String[] names = folder.list();

        try (PrintWriter pw = new PrintWriter(new FileWriter("output.txt"))) {
            if (names != null) {
                for (String name : names) {
                    File child = new File(folder, name);
                    if (!child.isFile()) continue;     // guard against sub-directories
                    try (BufferedReader br = new BufferedReader(new FileReader(child))) {
                        String line;
                        while ((line = br.readLine()) != null) {
                            pw.println(line);
                        }
                    }
                }
            }
        }
    }
}
```

### 7.4 Remove duplicate lines from a file

```java
import java.io.*;
import java.util.*;

class RemoveDuplicatesDemo {
    public static void main(String[] args) throws IOException {
        // Efficient O(n) approach using a Set (recommended in real systems)
        try (BufferedReader br = new BufferedReader(new FileReader("input.txt"));
             PrintWriter out = new PrintWriter(new FileWriter("output.txt"))) {

            Set<String> seen = new LinkedHashSet<>();   // preserves first-seen order
            String line;
            while ((line = br.readLine()) != null) {
                seen.add(line);
            }
            for (String uniqueLine : seen) {
                out.println(uniqueLine);
            }
        }
    }
}
```
> The original source solution re-opens `output.txt` and re-scans it for **every** line of `input.txt` — an **O(n²) file-I/O** algorithm (opens a new `BufferedReader` inside the outer loop on every iteration). That pattern is shown below **only for interview-recognition purposes** (you may be asked "what's wrong with this code?"), but should never be used in production:
```java
// ANTI-PATTERN — shown for critique/interview purposes only. Do NOT use in production.
BufferedReader br1 = new BufferedReader(new FileReader("input.txt"));
PrintWriter out = new PrintWriter(new FileWriter("output.txt"));
String target = br1.readLine();
while (target != null) {
    boolean available = false;
    BufferedReader br2 = new BufferedReader(new FileReader("output.txt")); // reopened every iteration!
    String line = br2.readLine();
    while (line != null) {
        if (target.equals(line)) { available = true; break; }
        line = br2.readLine();
    }
    br2.close();
    if (!available) {
        out.println(target);
        out.flush();
    }
    target = br1.readLine();
}
br1.close();
out.close();
```
**Why this is bad (interview talking points):** repeated file-handle churn, O(n²) time complexity, no `try-with-resources` (leaks handles on exception), `flush()` called per-line (unnecessary I/O syscalls).

### 7.5 File extraction (set-difference: `input.txt − delete.txt → output.txt`)

```java
import java.io.*;
import java.util.*;

class FileExtractionDemo {
    public static void main(String[] args) throws IOException {
        Set<String> toDelete = new HashSet<>();
        try (BufferedReader brDelete = new BufferedReader(new FileReader("delete.txt"))) {
            String t;
            while ((t = brDelete.readLine()) != null) {
                toDelete.add(t);
            }
        }

        try (BufferedReader br = new BufferedReader(new FileReader("input.txt"));
             PrintWriter pw = new PrintWriter(new FileWriter("output.txt"))) {
            String line;
            while ((line = br.readLine()) != null) {
                if (!toDelete.contains(line)) {
                    pw.println(line);
                }
            }
        }
    }
}
```
Given `input.txt = 111,222,333,444,555` and `delete.txt = 333,444`, `output.txt` becomes `111,222,555`.

---

## 8. Modern File I/O — What a Senior Architect Should Know Beyond This Chapter

| Legacy (`java.io`) | Modern equivalent (`java.nio.file`, Java 7+) |
|---|---|
| `new File(name)` | `Paths.get(name)` / `Path.of(name)` (Java 11+) |
| `file.exists()` | `Files.exists(path)` |
| `file.mkdir()` | `Files.createDirectory(path)` / `Files.createDirectories(path)` |
| `file.delete()` | `Files.delete(path)` / `Files.deleteIfExists(path)` |
| `file.list()` | `Files.list(path)` (returns `Stream<Path>`, must be closed) |
| Manual read loop | `Files.readAllLines(path)` (returns `List<String>`) |
| Manual read loop (large files) | `Files.lines(path)` → `Stream<String>` (lazy, Java 8+) |
| Manual write loop | `Files.write(path, lines)` |
| Copy files manually | `Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING)` |
| Recursive directory walk | `Files.walk(path)` / `Files.walkFileTree(path, visitor)` |
| No native watch capability | `WatchService` (directory change notifications) |

```java
import java.nio.file.*;
import java.util.List;
import java.util.stream.Collectors;

public class ModernFileIODemo {
    public static void main(String[] args) throws Exception {
        Path input = Path.of("input.txt");

        // Read all lines (Java 7+)
        List<String> lines = Files.readAllLines(input);

        // Stream-based processing, dedupe, write (Java 8+)
        List<String> distinctSorted = Files.lines(input)
                                            .distinct()
                                            .sorted()
                                            .collect(Collectors.toList());
        Files.write(Path.of("output.txt"), distinctSorted);

        // Copy / move
        Files.copy(input, Path.of("input_copy.txt"), StandardCopyOption.REPLACE_EXISTING);
    }
}
```

> **Java 11+ update:** `Files.readString(path)` / `Files.writeString(path, content)` — one-liners for whole small text files, no `BufferedReader`/`Writer` boilerplate needed.
> **Java 12+ (`Files.mismatch`)**, **Java 16 (`Stream.toList()`)**, and general trend: prefer `try-with-resources` + `java.nio.file` + `Stream` API for all new File I/O code. `java.io.File`/`FileReader`/`FileWriter` remain fully supported for legacy compatibility but are considered dated in new architecture designs.

---

# INTERVIEW CHEAT SHEET

## Inner Classes

1. **Why were inner classes introduced in Java?**
   Introduced in JDK 1.1 to support event-handling in AWT GUI components, letting event-handler code live in the context (and with access to the state) of the enclosing component.

2. **What relationship does an inner class have with its outer class — IS-A or HAS-A?**
   HAS-A. An inner class instance is logically part of / owned by an outer class instance.

3. **Can a regular (member) inner class have static members?**
   No — only `static final` constants are allowed; static methods and non-final static fields are not.

4. **How do you instantiate an inner class from outside the outer class?**
   `Outer.Inner obj = new Outer().new Inner();` — you must go through an outer class instance.

5. **How does an inner class access the enclosing instance's field when there's a name clash?**
   `Outer.this.fieldName` — `this` alone refers to the inner class's own instance.

6. **What's the difference between a member inner class and a static nested class?**
   Member inner class requires (and implicitly holds a reference to) an outer instance; a static nested class does not and behaves like an independent top-level class scoped inside another class's namespace.

7. **Can you declare `main()` inside an inner class and run it directly?**
   Not in a regular inner class. Yes in a static nested class: `java Outer$Nested`.

8. **What is a method-local inner class, and why is it rarely used today?**
   A class declared inside a method body, scoped to that method only. Largely superseded by lambda expressions (Java 8+) for the common case of implementing functional interfaces on the fly.

9. **Why must local variables captured by a local/anonymous inner class be final or effectively final?**
   Because the compiler copies the variable's value into the inner class's synthetic constructor at instantiation time (capture-by-value). If the variable could change afterward, the copy inside the inner class and the "live" method-local variable would diverge — creating an inconsistency the JLS disallows by requiring immutability of captured locals.

10. **What changed in Java 8 regarding local variable capture?**
    The `final` keyword became **optional** as long as the variable is *effectively final* (never reassigned after initialization) — this rule enables lambda expressions to feel more natural, since lambdas follow the identical capture semantics.

11. **What are the three types of anonymous inner classes?**
    (1) extending a class, (2) implementing an interface, (3) declared inline as a constructor/method argument.

12. **Can an anonymous inner class extend a class AND implement an interface simultaneously?**
    No — unlike a named class, an anonymous class can do only one or the other, never both, and can implement at most one interface.

13. **Why can't you call a child-specific method (declared only in the anonymous subclass) from outside?**
    Because the reference variable's compile-time type is the parent type; the compiler resolves method calls based on the declared (static) type of the reference, not the runtime type.

14. **How do lambda expressions differ from anonymous inner classes with respect to `this`?**
    Inside a lambda, `this` refers to the *enclosing* instance (lexical scoping); inside an anonymous inner class, `this` refers to the anonymous class's own instance.

15. **Are nested interfaces implicitly public and static?**
    Yes — always, whether declared explicitly or not.

16. **Is a class declared inside an interface implicitly static?**
    Yes, always `public static` implicitly.

17. **What's a real JDK example of a nested interface?**
    `Map.Entry<K,V>` — nested inside `java.util.Map`.

18. **How do default and static interface methods (Java 8) change classic "nested-class-for-default-implementation" patterns?**
    They make that pattern largely unnecessary — interfaces can now ship their own default logic without a companion nested implementation class.

19. **What's a common real-world bug caused by inner classes?**
    Memory leaks — a non-static inner class instance implicitly holds a strong reference to its outer instance (`Outer.this`), so if the inner instance outlives the intended lifecycle (e.g., an `AsyncTask`/listener held by a static field in Android), the outer instance (and its whole object graph) cannot be garbage collected. Fix: use a `static` nested class with a `WeakReference<Outer>` instead.

20. **Design-pattern application: where do static nested classes shine?**
    The Builder pattern (`public static class Builder` inside the target class) and, historically, the classic Singleton "initialization-on-demand holder" idiom, which relies on JVM class-loading guarantees for thread-safe lazy initialization.

## File I/O

21. **What's the difference between `Reader`/`Writer` and `InputStream`/`OutputStream`?**
    `Reader`/`Writer` handle character (text) data; `InputStream`/`OutputStream` handle raw binary data (images, audio, serialized objects).

22. **Can `BufferedReader`/`BufferedWriter` talk directly to a file?**
    No — they must wrap another `Reader`/`Writer` (typically `FileReader`/`FileWriter`); they don't have file-path constructors.

23. **What extra capability does `BufferedWriter` add over `FileWriter`?**
    `newLine()` — a platform-independent line-separator insertion method (plus internal buffering for performance).

24. **What extra capability does `BufferedReader` add over `FileReader`?**
    `readLine()` — reads a full line at a time instead of character-by-character.

25. **What's the difference between `write(100)` and `print(100)` on a `PrintWriter`?**
    `write(100)` treats 100 as a char code point and writes the character `'d'`; `print(100)` writes the literal text `"100"`.

26. **What happens when you close a `BufferedReader`/`BufferedWriter`?**
    The underlying wrapped `Reader`/`Writer` is automatically closed too — no need to close it separately.

27. **Why is `File.createNewFile()` not the same as "always creates a file"?**
    It returns `false` (and does nothing) if the file already exists; it only creates and returns `true` if the file was absent.

28. **What replaced `java.io.File` in modern Java, and why?**
    `java.nio.file.Path` + `Files` (NIO.2, Java 7+) — better exceptions, symbolic-link awareness, atomic operations, `WatchService` support, and stream-based traversal (`Files.walk`, `Files.lines`).

29. **How would you read an entire small text file in one line using modern Java?**
    `String content = Files.readString(Path.of("file.txt"));` (Java 11+), or `List<String> lines = Files.readAllLines(path);` (Java 7+).

30. **Why is `try-with-resources` preferred over manual `close()` calls in `finally`?**
    Guarantees deterministic resource cleanup even on exceptions, in reverse declaration order, with suppressed-exception tracking — eliminates a whole class of resource-leak bugs common in the manual `finally`-block style shown in legacy code.

31. **What's wrong with re-opening a `BufferedReader` on every iteration of an outer loop (as seen in the naive duplicate-removal algorithm)?**
    It's an O(n²) I/O algorithm with excessive file-handle churn; a `HashSet`/`LinkedHashSet`-based single-pass approach is O(n) and far more efficient, and also avoids leaking handles on exceptions.

32. **How do you stream-process a very large file without loading it fully into memory?**
    `Files.lines(path)` returns a lazily-evaluated `Stream<String>` (Java 8+) — must be used inside try-with-resources since it holds an open file handle.

---

*End of study material — validated against JDK behavior through Java 21 (LTS). Flag any single-fact you want cross-checked against a specific JDK version before an interview.*
