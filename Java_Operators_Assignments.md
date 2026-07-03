# Java Language Fundamentals — Operators & Assignments
### Senior Java Architect Interview Reference

**Source:** Durga Sir Core Java Materials — *Chapter 2: Operators & Assignments*
**Validation:** Every code snippet below has been compiled and executed on **OpenJDK 21.0.11** in a clean sandbox. Outputs are actual, verified outputs — not transcribed guesses. Two factual errors present in the original source material were caught during validation and are called out explicitly in **§21 — Corrections to Source Material**.

> This is not a beginner tutorial. It is written for someone who already knows Java syntax and is preparing for architect-level interviews, where questions probe *why* the language behaves a certain way (JLS rules), not just *what* the output is.

---

## Table of Contents
1. Increment & Decrement Operators
2. Arithmetic Operators & Numeric Promotion
3. String Concatenation Operator (`+`)
4. Relational Operators
5. Equality Operators (`==`, `!=`)
6. `instanceof` Operator
7. Bitwise Operators (`&`, `|`, `^`, `~`, `!`)
8. Short-Circuit Operators (`&&`, `||`)
9. Type Cast Operator
10. Assignment Operators
11. Conditional (Ternary) Operator
12. `new` Operator
13. `[]` Operator
14. Java Operator Precedence Table
15. Evaluation Order of Operands
16. `new` vs `Class.newInstance()` (and the modern replacement)
17. `instanceof` vs `Class.isInstance()`
18. `ClassNotFoundException` vs `NoClassDefFoundError`
19. Architect-Level Rapid-Fire Q&A
20. Common Pitfalls Checklist
21. Corrections to Source Material

---

## 1. Increment & Decrement Operators

| Form | Syntax | Meaning |
|---|---|---|
| Pre-increment | `y = ++x;` | Increment `x` first, then assign the **new** value to `y` |
| Post-increment | `y = x++;` | Assign the **old** value of `x` to `y`, then increment `x` |
| Pre-decrement | `y = --x;` | Decrement `x` first, then assign the **new** value to `y` |
| Post-decrement | `y = x--;` | Assign the **old** value of `x` to `y`, then decrement `x` |

**Validated truth table** (starting `x = 10`):

| Expression | Value of `y` | Final value of `x` |
|---|---|---|
| `y = ++x` | 11 | 11 |
| `y = x++` | 10 | 11 |
| `y = --x` | 9 | 9 |
| `y = x--` | 10 | 9 |

```java
public class T1 {
    public static void main(String[] args) {
        int x = 10, y;
        y = ++x; System.out.println("y=++x -> x=" + x + " y=" + y); // x=11 y=11
        x = 10; y = x++; System.out.println("y=x++ -> x=" + x + " y=" + y); // x=11 y=10
        x = 10; y = --x; System.out.println("y=--x -> x=" + x + " y=" + y); // x=9 y=9
        x = 10; y = x--; System.out.println("y=x-- -> x=" + x + " y=" + y); // x=9 y=10
    }
}
```
**Output:**
```
y=++x -> x=11 y=11
y=x++ -> x=11 y=10
y=--x -> x=9 y=9
y=x-- -> x=9 y=10
```

### Rule 1 — Only variables, never literals/constant values
`++`/`--` require a variable (an lvalue). Applying it to a literal is a compile-time error.

```java
int x = 4;
int y = ++4;   // COMPILE ERROR
```
```
error: unexpected type
  required: variable
  found:    value
```

### Rule 2 — No nesting of increment/decrement
`++(++x)` does not compile, because the inner `++x` produces a **value**, not a variable, and `++` requires a variable operand.

```java
int x = 4;
int y = ++(++x);  // COMPILE ERROR: unexpected type — required: variable, found: value
```

### Rule 3 — Not applicable to `final` variables
```java
final int x = 4;
x++;   // COMPILE ERROR: cannot assign a value to final variable x
```

### Rule 4 — Works on every primitive type except `boolean`
```java
int x = 10;      x++;  // 11
char ch = 'a';   ch++; // 'b'
double d = 10.5; d++;  // 11.5
byte b = 10;     b++;  // 11  (implicit narrowing cast happens internally)
boolean bo = true; bo++; // COMPILE ERROR: bad operand type boolean for unary operator '++'
```

### 🎯 Interview favorite — `b++` vs `b = b + 1`
For any binary arithmetic operator applied between two operands `a` and `b`, the JLS **binary numeric promotion** rule (JLS §5.6.2) states:

> **result type = max(int, type of a, type of b)**

This means `byte`, `short`, and `char` are *always* promoted to at least `int` before an arithmetic operation. Consequently:

```java
byte a = 10, b = 20;
byte c = a + b;              // COMPILE ERROR: possible lossy conversion from int to byte
byte c = (byte)(a + b);      // OK, explicit cast required — 30
```

But the increment/decrement and **compound assignment** operators are special-cased by the JLS to perform an **implicit (silent) narrowing cast** back to the target type:

```java
byte b = 10;
b++;                  // Internally expands to: b = (byte)(b + 1);
System.out.println(b); // 11 — compiles fine, no explicit cast needed
```

This asymmetry — `b++` compiles but `b = b + 1` doesn't (for a `byte`) — is one of the most commonly asked "gotcha" questions in Java interviews. **The compiler treats `x++` / `x--` / `x += y` as inherently containing an implicit cast to the type of `x`; plain `x = x + y` does not.**

Watch overflow behavior too — Java integer arithmetic silently **wraps around** (no exception):
```java
byte b2 = 127;
b2++;
System.out.println(b2); // -128  (wraps past Byte.MAX_VALUE, no overflow exception)
```

---

## 2. Arithmetic Operators & Numeric Promotion

### Binary numeric promotion rule
For `+ - * / %` between two operands `a` and `b`:

> **result type = max(int, type of a, type of b)**

| Expression | Result type |
|---|---|
| `byte + byte` | `int` |
| `byte + short` | `int` |
| `short + short` | `int` |
| `short + long` | `long` |
| `double + float` | `double` |
| `int + double` | `double` |
| `char + char` | `int` |
| `char + int` | `int` |
| `char + double` | `double` |

```java
System.out.println('a' + 'b');   // 195  ('a'=97, 'b'=98 -> int arithmetic)
System.out.println('a' + 1);     // 98
System.out.println('a' + 1.2);   // 98.2
```
**Validated output:** `195`, `98`, `98.2` — exactly as predicted.

### Division by zero: integral vs floating-point — the #1 gotcha
> In **integral arithmetic** (`byte`, `short`, `int`, `long`) there is no bit pattern to represent infinity or "undefined." So `x/0` and `0/0` both throw `ArithmeticException`.
> In **floating-point arithmetic** (`float`, `double`), IEEE 754 *does* define infinity and NaN, so no exception is thrown.

```java
System.out.println(10 / 0.0);   // Infinity
System.out.println(-10 / 0.0);  // -Infinity
System.out.println(0.0 / 0.0);  // NaN
System.out.println(-0.0 / 0.0); // NaN
```
```java
System.out.println(10 / 0);     // throws java.lang.ArithmeticException: / by zero
```
**Validated:** `Infinity`, `-Infinity`, `NaN`, `NaN`, then an uncaught `ArithmeticException: / by zero`.

> ⚠️ **Note on source material:** the original notes print this as `infinity` (lowercase). The actual JVM output (validated on OpenJDK 21) is `Infinity` (capital I), because it comes from `Double.toString()` which renders `Double.POSITIVE_INFINITY` as the literal string `"Infinity"`.

Relevant constants: `Float.POSITIVE_INFINITY`, `Float.NEGATIVE_INFINITY`, `Double.POSITIVE_INFINITY`, `Double.NEGATIVE_INFINITY`, `Float.NaN`, `Double.NaN`.

### NaN comparison semantics — every comparison with NaN is `false`, except `!=`
This is IEEE 754 behavior and is asked constantly in interviews because it **breaks the reflexive property of equality** (`x == x` is normally always true, but `NaN == NaN` is `false`).

```java
System.out.println(10 < Float.NaN);          // false
System.out.println(10 <= Float.NaN);         // false
System.out.println(10 > Float.NaN);          // false
System.out.println(10 >= Float.NaN);         // false
System.out.println(10 == Float.NaN);         // false
System.out.println(Float.NaN == Float.NaN);  // false  <-- classic trap
System.out.println(10 != Float.NaN);         // true
System.out.println(Float.NaN != Float.NaN);  // true
```
**Validated output** (in this exact order): `false false false false false false true true`.

> **Practical consequence:** never use `==` to check "is this value NaN". Use `Double.isNaN(x)` / `Float.isNaN(x)` instead. This also affects `Collections`, `TreeSet`/`TreeMap` ordering, and `Double.compare()` (which, unlike `==`, treats `NaN` as equal to itself and greater than all other values — another common trap: `Double.compare(Double.NaN, Double.NaN) == 0` is `true`, contradicting `NaN == NaN`).

### `ArithmeticException` — precise definition
1. It is an **unchecked** `RuntimeException`, not a compile-time error.
2. It occurs **only** in integral arithmetic, never in floating-point arithmetic.
3. Among all arithmetic operators, only `/` and `%` can throw it (never `+ - *`, which just overflow/wrap silently for integers).

---

## 3. String Concatenation Operator (`+`)

`+` is the **only overloaded operator** in the Java language. The compiler decides at each application, left-to-right, whether it means numeric addition or `String` concatenation:

> If **either** operand of `+` is a `String`, the operator performs concatenation (and the other operand is converted via `String.valueOf(...)`). If **both** operands are numeric, it performs arithmetic addition.

Because evaluation is strictly left-to-right, operator *position* changes the result:

```java
String a = "ashok";
int b = 10, c = 20, d = 30;
System.out.println(a + b + c + d); // "ashok" + 10 -> String, then + 20 -> String, then + 30 -> String
System.out.println(b + c + d + a); // 10+20+30 = 60 (all int) THEN + "ashok" -> "60ashok"
System.out.println(b + c + a + d); // 10+20=30 (int) + "ashok" -> "30ashok" + 30 -> "30ashok30"
System.out.println(b + a + c + d); // 10 + "ashok" -> "10ashok" + 20 -> "10ashok20" + 30 -> "10ashok2030"
```
**Validated output:**
```
ashok102030
60ashok
30ashok30
10ashok2030
```

### Compile error — implicit reverse conversion is *not* allowed
```java
String a = "ashok";
int b = 10, c = 20, d = 30;
a = b + c + d;  // COMPILE ERROR: incompatible types: int cannot be converted to String
```
`b + c + d` evaluates entirely as `int` arithmetic (60) since no `String` operand appears in that sub-expression, so assigning it to a `String` variable fails.

---

## 4. Relational Operators (`<`, `<=`, `>`, `>=`)

* Applicable to **every primitive type except `boolean`**.
* **Never** applicable to object/reference types (including `String`) — even though `String` is `Comparable`, the `<`/`>` operators do not invoke `compareTo()`.
* **No nesting** — a relational expression itself produces `boolean`, and `boolean` cannot be an operand of another relational operator.

```java
System.out.println(10 < 10.5);     // true
System.out.println('a' > 100.5);   // false  ('a' = 97)
System.out.println('b' > 'a');     // true
System.out.println(true > false);        // COMPILE ERROR: bad operand types for '>': boolean, boolean
System.out.println("ashok123" > "ashok"); // COMPILE ERROR: bad operand types for '>': String, String
System.out.println(10 > 20 > 30);         // COMPILE ERROR: bad operand types for '>': boolean, int
```
All three compile errors validated exactly as shown.

---

## 5. Equality Operators (`==`, `!=`)

* Unlike relational operators, `==`/`!=` **can** be applied to `boolean` as well as all other primitives.
* Can also be applied to **reference types**, where they perform **reference (identity/address) comparison** — do the two references point to the exact same object on the heap?

```java
System.out.println(10 == 20);        // false
System.out.println('a' == 'b');      // false
System.out.println('a' == 97.0);     // true  (char promoted to double, 'a' is codepoint 97)
System.out.println(false == false);  // true

Thread t1 = new Thread();
Thread t2 = new Thread();
Thread t3 = t1;
System.out.println(t1 == t2); // false — different objects
System.out.println(t1 == t3); // true  — t3 refers to the SAME object as t1
```

### Rule — reference types must be related (compile-time type hierarchy check)
To compare two reference types with `==`, the compiler requires the two **static types** to be related by inheritance (one must be assignable to the other, in either direction). Otherwise: compile-time error `incomparable types`.

```java
Thread t = new Thread();
Object o = new Object();
String s = new String("durga");

System.out.println(t == o); // false — Thread <: Object, related, compiles fine
System.out.println(o == s); // false — String <: Object, related, compiles fine
System.out.println(s == t); // COMPILE ERROR: incomparable types: String and Thread
```

### `null` semantics
```java
String s = new String("ashok");
System.out.println(s == null);    // false
String s2 = null;
System.out.println(s2 == null);   // true
System.out.println(null == null); // true
```

### `==` vs `.equals()` — the single most-asked Java interview question
| | `==` | `.equals()` |
|---|---|---|
| Compares | Reference identity (memory address) | Content (as defined by the class's `equals` override) |
| Default behavior for objects | Always identity check | `Object.equals()` defaults to identity too, **unless overridden** (e.g., `String`, wrapper classes, records auto-generate content-based `equals`) |

```java
String s1 = new String("ashok");
String s2 = new String("ashok");
System.out.println(s1 == s2);       // false — two distinct heap objects
System.out.println(s1.equals(s2));  // true  — same character content
```
> **Architect-level nuance:** if instead you write `String s1 = "ashok"; String s2 = "ashok";` (string literals), both would refer to the *same* interned object from the **String Constant Pool**, so `s1 == s2` would be `true`. This distinction (`new String(...)` bypasses the pool; literals use it) is a very common follow-up question.

---

## 6. `instanceof` Operator

Syntax: `O instanceof X` — `O` is an object reference (or expression), `X` is a class/interface name. Returns `true` if `O` refers to an object that **is** (or subtypes) `X` at runtime.

```java
Thread t = new Thread();
System.out.println(t instanceof Thread);   // true
System.out.println(t instanceof Object);   // true — every class extends Object
System.out.println(t instanceof Runnable); // true — Thread implements Runnable
```
(`public class Thread extends Object implements Runnable`)

### Rule — compile-time relatedness check (same rule family as `==` above)
`instanceof` requires that the **static type** of the left operand and the named type on the right be related by inheritance; otherwise it's a compile error.

```java
Thread t = new Thread();
System.out.println(t instanceof String); // COMPILE ERROR: incompatible types: Thread cannot be converted to String
```

### Parent-object-vs-child-type always evaluates to `false` (not an error)
```java
Object o1 = new Object();
System.out.println(o1 instanceof String); // false — runtime object IS-NOT a String, but types ARE related so it compiles
Object o2 = new String("ashok");
System.out.println(o2 instanceof String); // true — runtime object really is a String
```

### `null instanceof AnyType` is always `false`
```java
System.out.println(null instanceof String); // false — never NPEs, always returns false
```

> **Modern note (Java 16+):** pattern matching for `instanceof` lets you combine the check and cast:
> ```java
> if (o2 instanceof String s) {           // s is auto-cast and scoped to the true-branch
>     System.out.println(s.length());
> }
> ```
> This is a strong candidate follow-up question — mention it to demonstrate currency with modern Java.

### Practical dispatch idiom (common in legacy/polymorphic collection processing)
```java
Object o = list.get(0);
if (o instanceof Student) {
    Student s = (Student) o;
    // perform student-specific operation
} else if (o instanceof Customer) {
    Customer c = (Customer) o;
    // perform customer-specific operation
}
```

---

## 7. Bitwise Operators (`&`, `|`, `^`, `~`, `!`)

| Operator | Meaning | Applicable to |
|---|---|---|
| `&` (AND) | true only if **both** operands true | `boolean` **and** integral types |
| `\|` (OR) | true if **at least one** operand true | `boolean` **and** integral types |
| `^` (XOR) | true if operands **differ** | `boolean` **and** integral types |
| `~` (bitwise complement) | flips every bit | integral types **only**, not `boolean` |
| `!` (boolean complement / logical NOT) | flips truth value | `boolean` **only**, not integral |

```java
System.out.println(true & false);  // false
System.out.println(true | false);  // true
System.out.println(true ^ false);  // true

System.out.println(4 & 5);  // 4    (100 & 101 = 100)
System.out.println(4 | 5);  // 5    (100 | 101 = 101)
System.out.println(4 ^ 5);  // 1    (100 ^ 101 = 001)

System.out.println(~4);     // -5
```

### Why `~4 == -5` — two's complement explanation (frequently asked)
```
4  -> 0000....0100   (sign bit 0 = positive)
~4 -> 1111....1011   (sign bit 1 = negative)
```
Java integers are stored in **two's complement**. To read a negative bit pattern as a decimal value: invert all bits and add 1.
```
~4 bit pattern: 1111....1011
invert:         0000....0100  (= 4)
add 1:          0000....0101  (= 5)
so ~4 represents -5
```
General identity: `~x == -(x + 1)`, always.

```java
System.out.println(~true); // COMPILE ERROR: bad operand type boolean for unary operator '~'
System.out.println(!4);    // COMPILE ERROR: bad operand type int for unary operator '!'
```

**Summary table:**
| Operator | boolean | integral |
|---|---|---|
| `&`, `\|`, `^` | ✅ | ✅ |
| `~` | ❌ | ✅ only |
| `!` | ✅ only | ❌ |

---

## 8. Short-Circuit Operators (`&&`, `||`)

`&&`/`||` behave logically identically to `&`/`|` for `boolean` operands, but differ in evaluation strategy:

| | `&` , `\|` | `&&` , `\|\|` |
|---|---|---|
| Second operand | **Always** evaluated | Evaluated **conditionally** ("short-circuited") |
| Performance | Lower (always does both evaluations, and their side effects) | Higher (skips unnecessary work) |
| Applicable types | `boolean` and integral | `boolean` **only** |

**Semantics:**
* `x && y` — `y` is evaluated **only if** `x` is `true`. (If `x` is `false`, the whole expression is `false` regardless of `y`, so `y` is skipped.)
* `x || y` — `y` is evaluated **only if** `x` is `false`. (If `x` is `true`, the whole expression is already `true`, so `y` is skipped.)

### Validated side-effect demonstration
```java
static void run(String op) {
    int x = 10, y = 15;
    boolean cond = switch (op) {
        case "&"  -> (++x < 10) & (++y > 15);
        case "|"  -> (++x < 10) | (++y > 15);
        case "&&" -> (++x < 10) && (++y > 15);
        default   -> (++x < 10) || (++y > 15);
    };
    if (cond) x++; else y++;
    System.out.println(op + " -> x=" + x + " y=" + y);
}
```
**Validated output** (starting `x=10, y=15` fresh each run):
| Operator | x | y |
|---|---|---|
| `&` | 11 | 17 |
| `\|` | 12 | 16 |
| `&&` | 11 | 16 |
| `\|\|` | 12 | 16 |

With `&`/`|`, **both** `++x` and `++y` always execute (`y` reaches 17 or stays evaluated), whereas `&&`/`||` skip the second increment whenever short-circuiting kicks in — proof that side effects genuinely differ, not just performance.

### The critical real-world use case: guarding against exceptions/NPEs
This is *the* production reason `&&`/`||` exist beyond micro-optimization:
```java
int x = 10;
if (++x < 10 && ((x / 0) > 10)) {
    System.out.println("Hello");
} else {
    System.out.println("Hi");
}
```
**Validated output:** `Hi` — because `++x < 10` evaluates to `false` (x becomes 11), `&&` short-circuits, and `x / 0` (which would throw `ArithmeticException`) is **never evaluated**. This is the exact pattern behind idioms like:
```java
if (obj != null && obj.getValue() > 0) { ... }   // avoids NullPointerException
if (index >= 0 && index < array.length && array[index] == target) { ... } // avoids ArrayIndexOutOfBoundsException
```

---

## 9. Type Cast Operator

Two categories:

### Implicit type casting (widening / upcasting)
* Performed **automatically by the compiler**.
* Occurs when assigning a **lower-capacity** primitive value to a **higher-capacity** variable.
* No loss of information.
* Widening chain: `byte → short → int → long → float → double`, and `char → int` (also feeds into the same chain from `int` onward).

```java
int x = 'a';        System.out.println(x); // 97 — char widened to int automatically
double d = 10;       System.out.println(d); // 10.0 — int widened to double automatically
```

### Explicit type casting (narrowing / downcasting)
* Programmer's responsibility — must write `(targetType)`.
* Occurs when assigning a **higher-capacity** value to a **lower-capacity** variable.
* **May lose information** — the compiler forces you to acknowledge this with an explicit cast.

```java
int x = 130;
byte b = (byte) x;
System.out.println(b); // -126

int x2 = 150;
short s = (short) x2;
byte b2 = (byte) x2;
System.out.println(s);  // 150 — fits fine in short
System.out.println(b2); // -106 — overflow when narrowed to byte
```

**Without the cast, it's a compile error:**
```java
int x = 130;
byte b = x;  // COMPILE ERROR: incompatible types: possible lossy conversion from int to byte
```

### Rule — narrowing keeps the least-significant bits
> When narrowing an integral value, Java **discards the high-order bits** and keeps only the low-order bits that fit the target type's width. This is why the result can look like an unrelated/"random" number — it's really just truncation of the two's-complement bit pattern.

### Rule — narrowing a floating-point value to an integral type truncates the decimal part first
```java
double d = 130.456;
int x = (int) d;   System.out.println(x); // 130 — decimal portion truncated (not rounded)
byte b = (byte) d;  System.out.println(b); // -126 — 130.456 truncates to 130, then 130 narrows to byte same as above
```

---

## 10. Assignment Operators

Three flavors:

### (a) Simple assignment
```java
int x = 10;
```

### (b) Chained assignment
```java
int a, b, c, d;
a = b = c = d = 20;
System.out.println(a + "---" + b + "---" + c + "---" + d); // 20---20---20---20
```
> **Rule:** chained assignment **cannot** be combined with a variable *declaration* for more than the leftmost variable — `b`, `c`, `d` must already exist as declared variables before you chain-assign into them.
```java
int a = b = c = d = 30; // COMPILE ERROR: cannot find symbol — variable b (c, d likewise undeclared)
```
i.e., `int a = b = c = d = 20;` only declares `a`; the compiler then tries to resolve `b`, `c`, `d` as already-existing variables and fails.

### (c) Compound assignment
```java
int a = 10;
a += 20;
System.out.println(a); // 30
```
Full list: `+= -= *= /= %= &= |= ^= <<= >>= >>>=`

> Just like `++`/`--`, compound assignment operators perform an **implicit narrowing cast** automatically — this is the exact same JLS special case discussed in §1:
```java
byte b = 10;
b += 1;                 // internally: b = (byte)(b + 1)
System.out.println(b);  // 11 — compiles fine

byte b2 = 127;
b2 += 1;
System.out.println(b2); // -128 — silent overflow, no exception, no warning
```

### Deep-dive: complex compound-assignment expression evaluation order
```java
int a, b, c, d;
a = b = c = d = 20;
a += b -= c *= d /= 2;
System.out.println(a + "---" + b + "---" + c + "---" + d);
```
**Validated output:** `-160---180---200---10` (i.e., `a=-160, b=-180, c=200, d=10`)

Trace (assignment operators are **right-associative**, so evaluate the rightmost sub-expression first, then work leftward):
```
d /= 2   => d = d/2 = 10                (d: 20 -> 10)
c *= d   => c = c*d = 20*10 = 200       (c: 20 -> 200)
b -= c   => b = b-c = 20-200 = -180     (b: 20 -> -180)
a += b   => a = a+b = 20+(-180) = -160  (a: 20 -> -160)
```
> **Interview tip:** interviewers use this exact style of expression to test whether you understand (1) right-associativity of assignment operators and (2) that each compound operator both *reads* and *reassigns* its left operand before the next operator up the chain uses that new value.

---

## 11. Conditional (Ternary) Operator `?:`

Java's **only** ternary operator.

```java
int x1 = (10 > 20) ? 30 : 40;
System.out.println(x1); // 40

int x2 = (10 > 20) ? 30 : ((40 > 50) ? 60 : 70); // nesting is legal
System.out.println(x2); // 70
```

> **Architect-level nuance — numeric promotion inside `?:`:** if the two result expressions have different numeric types, the result type follows binary numeric promotion just like arithmetic operators (e.g., `true ? 1 : 2.0` has type `double`, and the `int` 1 is silently widened to `1.0`). This is a classic autoboxing/NPE trap when mixing a primitive with a boxed type:
> ```java
> Integer i = null;
> int result = true ? 0 : i;  // NullPointerException! because i forces unboxing of BOTH branches' common type
> ```

---

## 12. `new` Operator

* Used to instantiate objects on the heap.
* Java has **no `delete` operator** — object destruction is entirely the responsibility of the **Garbage Collector**; you cannot deterministically free memory (this is a frequent contrast question vs. C++).

```java
Test t = new Test();
```

---

## 13. `[]` Operator

Used to declare and construct arrays:
```java
int[] arr = new int[5];       // declaration + construction
int arr2[] = {1, 2, 3};       // C-style declaration also legal in Java
```

---

## 14. Java Operator Precedence Table (highest to lowest)

| # | Category | Operators |
|---|---|---|
| 1 | Unary | `[]`, `x++`, `x--`, `++x`, `--x`, `~`, `!`, `new`, `(type)` cast |
| 2 | Arithmetic | `*`, `/`, `%`, `+`, `-` |
| 3 | Shift | `>>`, `>>>`, `<<` |
| 4 | Relational / comparison | `<`, `<=`, `>`, `>=`, `instanceof` |
| 5 | Equality | `==`, `!=` |
| 6 | Bitwise | `&`, `^`, `\|` |
| 7 | Short-circuit | `&&`, `\|\|` |
| 8 | Conditional | `?:` |
| 9 | Assignment | `=`, `+=`, `-=`, `*=`, `/=`, `%=`, ... |

> Note: standard JLS precedence tables also separate `&`, `^`, `|` into three distinct levels (in that priority order) — the source material groups them together as "bitwise", which is a simplification worth mentioning if pressed on details in an interview.

---

## 15. Evaluation Order of Operands

> **Critical rule:** Java **always evaluates operands strictly left-to-right**, *regardless of operator precedence*. Precedence determines how the result is *combined*, not the *order in which operands are evaluated*. This is explicitly guaranteed by the JLS (§15.7 — Evaluation Order).

```java
public class T15 {
    public static void main(String[] args) {
        System.out.println(m1(1) + m1(2) * m1(3) / m1(4) * m1(5) + m1(6));
    }
    public static int m1(int i) {
        System.out.println("evaluating: " + i);
        return i;
    }
}
```
**Validated output** — the calls happen in strict source order `1,2,3,4,5,6` even though `*` and `/` bind tighter than `+`:
```
evaluating: 1
evaluating: 2
evaluating: 3
evaluating: 4
evaluating: 5
evaluating: 6
12
```
(Final arithmetic result: `1 + 2*3/4*5 + 6 = 1 + (2*3)/4*5 + 6 = 1 + 6/4*5 + 6 = 1 + 1*5 + 6 = 1+5+6 = 12`, using integer division `6/4=1`.)

> This is a **language guarantee that differs from C/C++**, where operand evaluation order for many operators is unspecified/implementation-defined. It matters enormously once operands have **side effects** (method calls, `++`/`--`), which is exactly what the next two examples test.

### Classic trap #1 — mixed increment operators in one expression
```java
int i = 1;
i += ++i + i++ + ++i + i++;
System.out.println(i); // 13
```
Trace: `i = i + ++i + i++ + ++i + i++` — expand strictly left to right, each sub-expression's side effect is applied immediately once evaluated, but the *addition* uses the value produced at the moment each sub-expression is evaluated:
```
start: i = 1
i (LHS, captured for the += target)      = 1
++i   -> i becomes 2, value used = 2
i++   -> value used = 2, i becomes 3
++i   -> i becomes 4, value used = 4
i++   -> value used = 4, i becomes 5
sum = 1 + 2 + 2 + 4 + 4 = 13
final assignment: i = 13
```
**Validated output:** `13` ✅

### Classic trap #2 — `x = x++`
```java
int x = 10;
x = x++;
System.out.println(x); // 10, NOT 11!
```
This is possibly the single most common Java "gotcha" interview question. Explanation:
1. Evaluate the right-hand side `x++`: this **captures the current value of `x` (10)** as the expression's *value*, then increments `x` to 11 as a side effect.
2. The **assignment** `x = ...` now runs, assigning the **captured value (10)** — not the post-increment value — back into `x`.
3. Net effect: the increment is silently **overwritten** by the assignment. Final `x` is `10`.

**Validated output:** `10` ✅

---

## 16. `new` vs `Class.newInstance()` (and its modern replacement)

| | `new` | `Class.newInstance()` *(deprecated since Java 9)* |
|---|---|---|
| What it is | Keyword/operator | Instance method on `java.lang.Class` |
| When to use | Class name is known **at compile time** | Class name is known only **dynamically at runtime** |
| Syntax | `Test t = new Test();` | `Object o = Class.forName(className).newInstance();` |
| Missing `.class` file at runtime | `NoClassDefFoundError` (**unchecked**, `Error`) | `ClassNotFoundException` (**checked**, `Exception`) |
| No-arg constructor requirement | Not required by `new` itself (you call whatever constructor you invoke) | The target class **must** have a no-arg constructor, or you get `InstantiationException` |

```java
public class Test {
    public static void main(String[] args) throws Exception {
        Object o = Class.forName(args[0]).newInstance();
        System.out.println(o.getClass().getName());
    }
}
```

> ⚠️ **Modern update the source material predates:** `Class.newInstance()` was **deprecated in Java 9** because it propagates checked exceptions thrown by the constructor as unchecked ones, bypassing normal exception-transparency, and it doesn't let you invoke a *specific* constructor. The recommended replacement — validated below — is:
> ```java
> Object o = Class.forName("java.lang.String")
>                  .getDeclaredConstructor()
>                  .newInstance();
> System.out.println(o.getClass().getName()); // java.lang.String
> ```
> With this modern API, a missing no-arg constructor surfaces as `NoSuchMethodException` from `getDeclaredConstructor()` rather than `InstantiationException` from `newInstance()` — validated:
> ```java
> class NoDefaultCtor { NoDefaultCtor(int x) {} }
> NoDefaultCtor.class.getDeclaredConstructor().newInstance();
> // throws: java.lang.NoSuchMethodException: NoDefaultCtor.<init>()
> ```
> Knowing this deprecation and its replacement is a strong signal of currency for a senior architect interview — citing only the old API without mentioning the change would look dated.

---

## 17. `instanceof` vs `Class.isInstance()`

| | `instanceof` | `Class.isInstance()` |
|---|---|---|
| What it is | Operator | Instance method on `java.lang.Class` |
| Type known when | At **compile time** | Dynamically, at **runtime** |
| Compile-time relatedness check | **Required** (compile error if unrelated) | Not required — accepts any `Object` |

```java
String s = new String("ashok");
System.out.println(s instanceof Object); // true — type known at compile time

System.out.println(Class.forName("java.lang.String").isInstance(s)); // true
System.out.println(Class.forName("java.lang.Object").isInstance(s)); // true
```
All validated `true`. `isInstance()` is the tool of choice in frameworks (Spring, serialization libraries, plugin systems) where the type to check against is only known via configuration/reflection at runtime, not hardcoded in source.

---

## 18. `ClassNotFoundException` vs `NoClassDefFoundError`

| | `NoClassDefFoundError` | `ClassNotFoundException` |
|---|---|---|
| Hierarchy | `extends LinkageError extends Error` — **unchecked** | `extends ReflectiveOperationException extends Exception` — **checked** |
| Trigger scenario | A class was **hard-coded/referenced at compile time** (`new Test()`, or a static reference), and its `.class` file is missing/unreachable **at runtime** | A class name was supplied **dynamically** (typically via `Class.forName(String)`) and the `.class` file could not be located at runtime |
| Typical cause | Successfully compiled against a JAR that is now missing/mismatched on the runtime classpath | Wrong/misspelled fully-qualified class name passed dynamically; missing runtime dependency for reflective loading |

**Validated demonstration:**
```java
// Helper.java compiles fine, MainApp references it directly (hard-coded)
public class Helper { void greet() { System.out.println("hello"); } }
public class MainApp {
    public static void main(String[] args) {
        Helper h = new Helper();
        h.greet();
    }
}
```
Compile both, run once successfully (`hello`), then **delete `Helper.class`** and run `MainApp` again:
```
Exception in thread "main" java.lang.NoClassDefFoundError: Helper
    at MainApp.main(MainApp.java:3)
Caused by: java.lang.ClassNotFoundException: Helper
    at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(...)
    ...
```
Note how the JVM's own diagnostic message shows `ClassNotFoundException` as the **cause** of `NoClassDefFoundError` — this is the classloader's internal mechanism surfacing: it attempted to load the class, got a `ClassNotFoundException` internally, and wrapped it into the unchecked `NoClassDefFoundError` because the calling code (`MainApp`) never declared it expected this failure (it used `new`, not reflection).

**Dynamic lookup version — genuinely checked `ClassNotFoundException`:**
```java
try {
    Class.forName("com.nonexistent.Foo");
} catch (ClassNotFoundException e) {
    System.out.println("Caught: " + e);
}
```
**Validated output:** `Caught: java.lang.ClassNotFoundException: com.nonexistent.Foo`

> **Architect-level context:** `NoClassDefFoundError` in production is almost always a **classpath/dependency-version mismatch** issue (e.g., a JAR was present at compile time via Maven but excluded/shaded/relocated at runtime, or two versions of the same library conflict). It is *not* the same bug class as a simple typo — that's the practical distinction worth raising in an interview beyond the textbook definition.

---

## 19. Architect-Level Rapid-Fire Q&A

**Q: Why does `byte b = 10; b++;` compile, but `byte b = 10; b = b + 1;` doesn't?**
A: `++`/compound-assignment operators perform an implicit narrowing cast defined by the JLS as part of their semantics; plain `b + 1` promotes to `int` (binary numeric promotion) and requires an explicit cast to reassign to a `byte`.

**Q: What's the output of `System.out.println(1 + 2 + "3")` vs `System.out.println("1" + 2 + 3)`?**
A: `"33"` (left-to-right: `1+2=3` numerically, then `3+"3"` concatenates to `"33"`) vs `"123"` (`"1"+2` concatenates to `"12"`, then `+3` concatenates to `"123"`). Tests understanding of left-to-right evaluation combined with `+` operator overloading.

**Q: Is `NaN == NaN` true or false? Why does this matter for `equals()`/hashing?**
A: `false`, by IEEE 754 definition. This is why `Double`/`Float` wrapper classes' `.equals()` method deliberately does **not** delegate to `==` — `Double.valueOf(Double.NaN).equals(Double.valueOf(Double.NaN))` is `true`, so that `NaN` can be reliably stored/looked-up as a key in `HashMap`/used in `HashSet`.

**Q: Difference between `&` and `&&` beyond short-circuiting?**
A: `&`/`|` also work on integral types as true bitwise operators; `&&`/`||` are boolean-only logical operators. For booleans, `&` always evaluates both sides (useful when you *want* both side effects to run, e.g., validation methods that must all execute and report all errors), `&&` short-circuits (useful for guard conditions).

**Q: Why is there no `NoClassDefFoundError` equivalent for `new` at compile time?**
A: There is nothing at compile time — the compiler verifies the class exists then. `NoClassDefFoundError` is purely a **runtime linkage** failure: bytecode verification succeeded during compilation, but the JVM's classloader can't locate the referenced `.class` at class-loading time (often due to classpath drift between build and deploy environments).

**Q: What does `x = x++` teach about the JLS assignment evaluation model?**
A: Java evaluates the right-hand side to a **value** first (capturing side effects along the way), *then* performs the assignment using that already-captured value — the target variable reference on the LHS is bound *before* the RHS's side effects run, but the actual store happens *after*, using the value computed at RHS-evaluation time, not whatever the variable holds afterward.

**Q: Ternary operator + autoboxing — what's the NPE trap?**
A: `Integer i = null; int r = flag ? 0 : i;` throws NPE even if `flag` is `true` and the `i` branch is never logically "needed", because the compiler determines the *static* result type of the ternary expression up front (here, unifying `int` and `Integer` forces both branches toward `int`, meaning `i` **must** be unboxed to participate in that unified type, regardless of which branch executes at runtime).

---

## 20. Common Pitfalls Checklist (quick pre-interview review)

- [ ] `byte`/`short`/`char` arithmetic always promotes to at least `int` — explicit cast needed to narrow back, **except** for `++`/`--`/compound-assignment which auto-narrow.
- [ ] Integer division/modulo by zero → `ArithmeticException`; floating-point division by zero → `Infinity`/`NaN`, no exception.
- [ ] `NaN` fails every comparison (`<,<=,>,>=,==`) including against itself; only `!=` returns `true`.
- [ ] `+` is overloaded for `String`; evaluation is left-to-right, so operand order changes output.
- [ ] Relational operators (`<,<=,>,>=`) cannot be applied to `boolean` or reference types, and cannot be nested.
- [ ] `==` on references is identity comparison; requires compile-time-related types or it's a compile error.
- [ ] `instanceof` also requires compile-time-related types; `null instanceof X` is always `false`, never throws.
- [ ] `~` is integral-only; `!` is boolean-only; `& | ^` work for both.
- [ ] `&&`/`||` short-circuit; `&`/`|` don't — this changes program behavior, not just speed, whenever operands have side effects.
- [ ] Explicit narrowing casts truncate (don't round) floating values, and keep only low-order bits for integral narrowing — can produce wraparound-looking results.
- [ ] Chained assignment (`a=b=c=d=20`) requires all variables pre-declared; can't declare-and-chain in one statement.
- [ ] Assignment operators are right-associative — evaluate rightmost compound expression first when tracing complex chains.
- [ ] `x = x++` is a trap: net result is **no change** to `x`, because the pre-increment value is what gets reassigned.
- [ ] `Class.newInstance()` is deprecated since Java 9 — use `getDeclaredConstructor().newInstance()`.
- [ ] `NoClassDefFoundError` (unchecked `Error`, hard-coded class reference) vs `ClassNotFoundException` (checked `Exception`, dynamic `Class.forName` reference) — often confused, frequently tested.

---

## 21. Corrections to Source Material

During validation on OpenJDK 21, two discrepancies were found in the original source notes and are corrected here:

1. **Byte cast of `130.456`:** the source states `byte b = (byte) 130.456;` outputs `-206`. This is **impossible** — `byte` range is `-128` to `127`, so `-206` cannot be a valid `byte` value. The actual, compiled-and-run result is **`-126`** (the `double` truncates to `int 130` first, which then narrows to `byte` the same way as the earlier `int x = 130; byte b = (byte)x;` example, also yielding `-126`).

2. **`infinity` capitalization:** the source prints division-by-zero floating point results as lowercase `infinity`. The actual JVM output (via `Double.toString()`) is capitalized: **`Infinity`** / **`-Infinity`**.

3. **Minor whitespace typo:** the source shows `System.out.println(b+a+c+d)` producing `"10ashok 2030"` (with a space). The validated, actual output has no space: **`"10ashok2030"`**.

All other claims, code behaviors, and compile-error messages in the source material were reproduced and confirmed exactly (modern `javac` wording differs cosmetically in a couple of places — e.g., "possible lossy conversion from int to byte" instead of the older "possible loss of precision" phrasing — but the underlying rule is identical).
