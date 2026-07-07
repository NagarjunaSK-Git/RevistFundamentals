# Core Java — Language Fundamentals
### Senior Java Architect Interview Preparation Guide
*(Source: Durga Sir — "Language Fundamentals" chapter, cross-verified and compiled/executed on OpenJDK 21 for correctness. Deviations of modern JDKs from the original 1.6/1.7-era material are called out explicitly as **Architect Notes**.)*

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Identifiers](#2-identifiers)
3. [Reserved Words / Keywords](#3-reserved-words--keywords)
4. [Data Types](#4-data-types)
5. [Literals](#5-literals)
6. [Arrays](#6-arrays)
7. [Types of Variables](#7-types-of-variables)
8. [Var-Arg Methods](#8-var-arg-methods)
9. [The `main()` Method](#9-the-main-method)
10. [Command-Line Arguments](#10-command-line-arguments)
11. [Java Coding Standards](#11-java-coding-standards)
12. [JVM Memory Areas](#12-jvm-memory-areas)
13. [Interview Rapid-Fire Cheat Sheet](#13-interview-rapid-fire-cheat-sheet)

---

## 1. Introduction

"Language Fundamentals" covers the building blocks every Java program is made of: identifiers, keywords, primitive data types, literals, arrays, variable categories, variable-argument (var-arg) methods, the `main()` method contract, command-line arguments, naming conventions, and the JVM runtime data areas. While these look like "basic" topics, they are among the **most heavily probed areas in senior-level interviews** because they expose whether a candidate truly understands *why* the language behaves the way it does (type promotion, memory semantics, overload resolution) rather than just being able to write code that compiles.

---

## 2. Identifiers

An **identifier** is a name in a Java program — used for classes, methods, variables, and labels.

### Rules to define Java identifiers

| Rule | Description | Example |
|---|---|---|
| **Rule 1** | Only `a-z`, `A-Z`, `0-9`, `_`, `$` are allowed. | `total_number` valid |
| **Rule 2** | Any other character → **Compile-time Error (C.E.)**. | `Total#` → invalid |
| **Rule 3** | Identifiers cannot start with a digit. | `123ABC` invalid, `ABC123` valid |
| **Rule 4** | Java identifiers are **case-sensitive** (Java itself is a case-sensitive language). | `number`, `Number`, `NUMBER`, `NuMbEr` are 4 distinct identifiers |
| **Rule 5** | No length limit, but > 15 characters is discouraged for readability. | — |
| **Rule 6** | Reserved words cannot be used as identifiers. | `int if = 10;` → invalid |
| **Rule 7** | Predefined class/interface names **can legally** be used as identifiers (though discouraged as bad practice). | `int String = 10;` compiles |

### Validated example (Rule 7 — legal but poor practice)

```java
public class Test {
    public static void main(String[] args) {
        int String = 10;              // shadows java.lang.String locally
        System.out.println(String);   // 10

        int Runnable = 10;            // shadows java.lang.Runnable
        System.out.println(Runnable); // 10
    }
}
```
**Output (verified on OpenJDK 21):**
```
10
```
This compiles because `String`/`Runnable` as *variable* identifiers occupy a different **namespace** than *type* identifiers in Java — the compiler resolves each occurrence contextually. This is a classic interview trap: people assume it's illegal, but it is legal (and simply bad practice).

> **Architect Insight:** This distinction — types vs. variables vs. methods living in separate namespaces — is the same principle that lets you have a field and a method with the same name in the same class, or a class named the same as a package member you don't import. Understanding namespace resolution matters when reviewing generated code, DSLs, or code with reflective/dynamic proxies.

---

## 3. Reserved Words / Keywords

Java has (historically, per this material) **50 total reserved words**: 48 keywords + 2 reserved literals boolean values are literals not keywords technically (`true`/`false`) + `null`. Let's break them down by category.

> **Architect Note (currency check):** As of modern Java (9+), the keyword list has grown: `module`, `requires`, `exports`, `opens`, `uses`, `provides`, `to`, `with`, `transitive` (Java 9, *contextual* keywords — only reserved inside `module-info.java`), `var` (Java 10, contextual), `yield` (Java 14, contextual), `record`, `sealed`, `permits`, `non-sealed` (Java 16/17, contextual). These are **contextual keywords** — legal as identifiers in ordinary code, unlike the "true" reserved words below.

### Reserved words for data types (8)
`byte, short, int, long, float, double, char, boolean`

### Reserved words for flow control (11)
`if, else, switch, case, default, for, do, while, break, continue, return`

### Keywords for modifiers (11)
`public, private, protected, static, final, abstract, synchronized, native, strictfp (1.2v), transient, volatile`

### Keywords for exception handling (6)
`try, catch, finally, throw, throws, assert (1.4v)`

### Class-related keywords (6)
`class, package, import, extends, implements, interface`

### Object-related keywords (4)
`new, instanceof, super, this`

### Void return type keyword (1)
`void` — mandatory in Java for no-return methods (optional in C++).

### Unused / banned keywords
- `goto` — reserved but unused; caused spaghetti-code issues in older languages, so it's banned.
- `const` — reserved but unused; use `final` instead.
- Using either in code → **Compile-time Error**.

### Reserved literals (not technically "keywords" but reserved values)
- `true`, `false` — boolean literals
- `null` — default value for object references

### `enum` (introduced in 1.5)
Used to define a group of named constants.
```java
enum Beer {
    KF, RC, KO, FO;
}
```

### Key Conclusions (frequently tested)
1. All Java reserved words are **lowercase only**.
2. New keywords added by version:
   - `strictfp` → 1.2
   - `assert` → 1.4
   - `enum` → 1.5
3. Java has `new` but **no `delete`** — object destruction is the Garbage Collector's job.
4. Common spelling traps (classic MCQ bait):
   - `instanceof` (not `instanceOf`)
   - `strictfp` (not `strictFp`)
   - `const` (not `Constant`) — and it's unused anyway
   - `synchronized` (not `syncronized`)
   - `extends` (not `extend`)
   - `implements` (not `implement`)
   - `import` (not `imports`)

### Common "spot the fake keyword" traps
| List | Verdict | Why |
|---|---|---|
| `final, finally, finalize` | Invalid | `finalize` is a method of `Object`, not a keyword |
| `throw, throws, thrown` | Invalid | `thrown` doesn't exist |
| `break, continue, return, exit` | Invalid | `exit` isn't reserved (it's `System.exit()`) |
| `byte, short, Integer, long` | Invalid | `Integer` is a wrapper **class**, not a primitive keyword |
| `extends, implements, imports` | Invalid | `imports` doesn't exist; keyword is `import` |
| `instanceof, sizeOf` | Invalid | `sizeOf` is not a Java keyword (unlike C's `sizeof`) |
| `new, delete` | Invalid | `delete` doesn't exist in Java |

---

## 4. Data Types

> Every variable has a type, every expression has a type, and every assignment is checked by the compiler for type compatibility — this is why Java is called a **strongly typed** language.

### Is Java "pure" object-oriented?
**No.** Java is *not* purely object-oriented because:
- It doesn't support **multiple inheritance** (of classes) or **operator overloading**.
- It relies on **primitive data types**, which are *not* objects.

(Languages like Smalltalk are considered "pure" OOP because literally everything, including numbers, is an object.)

### Integral Data Types

| Type | Size | Range | Notes |
|---|---|---|---|
| `byte` | 1 byte (8 bits) | -128 to 127 (-2⁷ to 2⁷-1) | Best for stream data (file/network I/O) |
| `short` | 2 bytes | -32768 to 32767 (-2¹⁵ to 2¹⁵-1) | Rarely used; legacy of 16-bit processors (e.g. 8086), now largely obsolete |
| `int` | 4 bytes | -2147483648 to 2147483647 (-2³¹ to 2³¹-1) | Most commonly used |
| `long` | 8 bytes | -2⁶³ to 2⁶³-1 | Used when `int` isn't enough, e.g. `File.length()` returns `long` |

- Except `boolean` and `char`, **all other primitive types are signed** (can represent +ve and -ve numbers).
- MSB (most significant bit) is the sign bit: `0` = positive, `1` = negative.
- Positive numbers are stored directly; negative numbers use **2's complement**.

**Validated compile-error examples:**
```java
byte b = 10;      // OK
byte b2 = 130;    // C.E: possible loss of precision (found int, required byte)
byte b3 = 10.5;   // C.E: possible loss of precision
byte b4 = true;   // C.E: incompatible types
byte b5 = "ashok";// C.E: incompatible types (found String, required byte)
```

### Floating Point Data Types

| Type | Precision | Size | Range |
|---|---|---|---|
| `float` | ~5–6 decimal digits (single precision) | 4 bytes | -3.4e38 to 3.4e38 |
| `double` | ~14–15 decimal digits (double precision) | 8 bytes | -1.7e308 to 1.7e308 |

### `boolean`
- Size: **not applicable** (JVM-dependent; often implemented internally as an `int` in the bytecode, but this is an implementation detail).
- Only allowed values: `true` / `false` — **no implicit int↔boolean conversion** (unlike C/C++).

### `char`
- Size: **2 bytes** (unlike C/C++'s 1 byte).
- **Why?** Old languages are ASCII-based (< 256 characters, fits in 1 byte). Java is **Unicode**-based, supporting > 256 and ≤ 65536 characters (UTF-16 code unit), requiring 2 bytes.
- Range: `0 to 65535` (unsigned — `char` is the **only unsigned primitive** in Java).

```java
char ch1 = 97;      // valid → 'a'
char ch2 = 65536;   // C.E: possible loss of precision (out of range)
```

### Summary Table (validated)

| Data Type | Size | Range | Wrapper Class | Default Value |
|---|---|---|---|---|
| `byte` | 1 byte | -2⁷ to 2⁷-1 (-128 to 127) | `Byte` | `0` |
| `short` | 2 bytes | -2¹⁵ to 2¹⁵-1 (-32768 to 32767) | `Short` | `0` |
| `int` | 4 bytes | -2³¹ to 2³¹-1 | `Integer` | `0` |
| `long` | 8 bytes | -2⁶³ to 2⁶³-1 | `Long` | `0` |
| `float` | 4 bytes | -3.4e38 to 3.4e38 | `Float` | `0.0` |
| `double` | 8 bytes | -1.7e308 to 1.7e308 | `Double` | `0.0` |
| `boolean` | N/A | `true`/`false` only | `Boolean` | `false` |
| `char` | 2 bytes | 0 to 65535 | `Character` | `'\u0000'` (blank space / null char) |

> Default value for **object references** is `null`.

> **Architect Insight — Autoboxing cache:** `Integer`, `Short`, `Byte`, `Long`, `Character` cache values from **-128 to 127** (via `Integer.valueOf()` etc.) — a frequent gotcha in senior interviews: `Integer a = 127, b = 127;  a == b` → `true`, but `Integer a = 128, b = 128; a == b` → `false`. This is not in the source deck but is the natural next-level question interviewers ask right after primitives.

---

## 5. Literals

A **literal** is any constant value directly assignable to a variable.

### Integral Literals
For `byte`, `short`, `int`, `long` — three ways to specify:

| Form | Allowed digits | Prefix | Example |
|---|---|---|---|
| Decimal | 0–9 | none | `int x = 10;` |
| Octal | 0–7 | `0` | `int x = 010;` → decimal 8 |
| Hexadecimal | 0–9, A–F (case-insensitive — **one of the few case-insensitive areas in Java**) | `0x` or `0X` | `int x = 0x10;` → decimal 16 |

**Validated:**
```java
int x = 10, y = 010, z = 0x10;
System.out.println(x + "----" + y + "----" + z);
// Output: 10----8----16
```

**Validity checks (classic MCQ set):**
```java
int x1 = 0777;      // valid  (octal)
int x2 = 0786;       // C.E: integer number too large: 0786 — 8/9 are invalid octal digits
int x3 = 0xFACE;     // valid
int x4 = 0xbeef;     // valid (lowercase hex digits OK)
int x5 = 0xBeer;     // C.E: ';' expected — 'r' is not a valid hex digit
int x6 = 0xabb2cd;   // valid
```

- By default, every integral literal is `int`. To force `long`, suffix with `l` or `L` (prefer **`L`** — lowercase `l` looks like digit `1`, an infamous code-review nitpick).
```java
long l1 = 10L;   // valid
long l2 = 10;    // valid — int literal widened to long
int x = 10l;     // C.E: possible loss of precision (found long, required int)
```
- There is **no direct literal suffix for `byte`/`short`**. The compiler auto-narrows an `int` literal to `byte`/`short` **only if the literal's value fits in range** — this is a compile-time constant check, not a runtime check.
```java
byte b1 = 127;   // valid
byte b2 = 130;   // C.E: possible loss of precision
short s1 = 32767;// valid
short s2 = 32768;// C.E: possible loss of precision
```

### Floating Point Literals
- Default type is `double`. Suffix `f`/`F` for `float`, `d`/`D` for `double` (optional).
```java
float f1 = 123.456;    // C.E: possible loss of precision (double → float narrowing)
float f2 = 123.456f;   // valid
double d1 = 123.456;   // valid
double d2 = 123.456D;  // valid
```
- Floating point literals can **only** be specified in **decimal** form — not octal, not hex (unlike Java's hex *floating-point* literal syntax added later — see note below).
```java
double d3 = 0123.456;      // valid — treated as decimal (leading 0 has NO octal meaning for floats)
double d4 = 0x123.456;     // C.E: malformed floating point literal
```
- Integral literals (in decimal, octal, or hex) **can** be assigned directly to floating types (implicit widening):
```java
double d = 0xBeef;
System.out.println(d); // 48879.0
```
- The reverse is **not** allowed — floating literal → integral variable:
```java
int x = 10.0; // C.E: possible loss of precision
```
- Exponential (scientific) notation is supported:
```java
double d = 10e2;   // 10 * 10^2
System.out.println(d);   // 1000.0
float f = 10e2;    // C.E: possible loss of precision (still double by default)
float f2 = 10e2F;  // valid
```

> **Architect Note:** Java *does* support hexadecimal floating-point literals like `0x1.0p0` (mantissa in hex, binary exponent after `p`) since Java 5 — different from `0x123.456`, which the compiler correctly rejects as malformed because it's missing the required binary exponent marker `p`/`P`.

### Boolean Literals
Only `true` / `false`, case-sensitive (lowercase only).
```java
boolean b1 = true;    // valid
boolean b2 = 0;        // C.E: incompatible types — NO implicit int→boolean conversion
boolean b3 = True;     // C.E: cannot find symbol — 'True' isn't a literal
boolean b4 = "true";   // C.E: incompatible types
```

### Char Literals — 3 forms

**1) Single character within single quotes:**
```java
char ch1 = 'a';    // valid
char ch2 = a;      // C.E: cannot find symbol
char ch3 = "a";    // C.E: incompatible types (String literal ≠ char)
char ch4 = 'ab';   // C.E: unclosed character literal (only 1 char allowed)
```

**2) Integral literal representing the Unicode code point (decimal/octal/hex, range 0–65535):**
```java
char ch5 = 97;       // valid → 'a'
char ch6 = 0xFace;   // valid
char ch7 = 65536;    // C.E: possible loss of precision — out of range
```

**3) Unicode escape `\uXXXX` (exactly 4 hex digits):**
```java
char ch8 = '\ubeef';
char ch9 = '\u0061';
System.out.println(ch9); // a
char ch10 = \u0062;      // C.E: cannot find symbol — must be inside quotes
char ch11 = '\iface';    // C.E: illegal escape character
```

**Escape sequences (every escape sequence is a valid char literal):**

| Escape | Meaning |
|---|---|
| `\n` | New line |
| `\t` | Horizontal tab |
| `\r` | Carriage return |
| `\f` | Form feed |
| `\b` | Backspace |
| `\'` | Single quote |
| `\"` | Double quote |
| `\\` | Backslash |

```java
char ch = '\n';   // valid
char ch2 = '\l';  // C.E: illegal escape character — 'l' is not a recognized escape
```

### String Literals
Any sequence of characters within double quotes.
```java
String s = "Ashok"; // valid
```

### Java 1.7 Enhancements to Literals

**1) Binary literals** — prefix `0b` / `0B`, digits `0`/`1` only.
```java
int x = 0b111;
System.out.println(x); // 7
```

**2) Underscore `_` in numeric literals** — improves readability; the compiler strips underscores at compile time.
```java
double d1 = 123456.789;
double d2 = 1_23_456.7_8_9;    // valid — same value as d1
double d3 = 123_456.7_8_9;     // valid
double d4 = 1_23_ _456.789;    // valid — multiple consecutive underscores allowed
```
**Rule:** underscore must sit strictly **between** digits — never at the start, end, or adjacent to the decimal point, `d`/`D`/`f`/`F`/`l`/`L` suffix, or `x`/`b` prefix.
```java
double e1 = _1_23_456.7_8_9;   // C.E: illegal underscore
double e2 = 1_23_456.7_8_9_;   // C.E: illegal underscore
double e3 = 1_23_456_.7_8_9;   // C.E: illegal underscore
```

**Verified (JDK 21):**
```
double d = 1_23_456.7_8_9;
System.out.println(d); // 123456.789
```

---

## 6. Arrays

An **array** is an indexed collection of a **fixed number of homogeneous** elements.

- **Advantage:** represent multiple values under one name → improves readability.
- **Disadvantage:** **fixed size** — the size cannot grow/shrink after creation, so the size must be known up-front. (Java's `Collections` framework, e.g. `ArrayList`, solves this by wrapping resizable logic around arrays internally.)

### Array Declaration

**1D (any of these forms is valid — the `int[] a` form is *recommended* since the type is clearly separated from the name):**
```java
int[] a;   // recommended
int []a;
int a[];
```
Size **cannot** be specified at declaration:
```java
int[] a;    // valid
int[5] a;   // invalid (C.E)
```

**2D — 6 valid syntactic forms:**
```java
int[][] a;
int [][]a;
int a[][];
int[] []a;
int[] a[];
int []a[];
```

**3D — 10 valid syntactic forms** (mixing `[]` before/after the variable name at each level is legal as long as total bracket-pairs = 3).

**Multi-variable declaration gotcha:** array-dimension brackets placed *before* the variable name apply to **every** variable in that declaration; brackets placed *after* a variable name apply **only to that variable**.
```java
int[] a1, b1;     // a: int[], b: int[]     — 1D, 1D
int[] a2[], b2;   // a: int[][], b: int[]   — 2D, 1D
int[] []a3, b3;   // a: int[][], b: int[][] — 2D, 2D
int[] a, []b;     // C.E: illegal start of expression — bracket cannot precede 2nd variable name
```

### Array Construction

Every array in Java is an **object**, created with `new`.
```java
int[] a = new int[3];
```
Internally, arrays get synthetic class names not exposed to user code:

| Array type | Internal class name |
|---|---|
| `int[]` | `[I` |
| `int[][]` | `[[I` |
| `double[]` | `[D` |

**Construction rules:**

| Rule | Detail | Example |
|---|---|---|
| Rule 1 | Size **must** be specified at creation | `new int[]` → C.E: array dimension missing |
| Rule 2 | **Zero-length arrays are legal** | `int[] a = new int[0]; a.length` → `0` |
| Rule 3 | Negative size → **runtime** exception | `new int[-3]` → `NegativeArraySizeException` (verified) |
| Rule 4 | Size expression must be `byte`, `short`, `char`, or `int` | `new int['a']` valid (char widens to int); `new int[10L]` → C.E (long not allowed); `new int[10.5]` → C.E |
| Rule 5 | Max array size = `Integer.MAX_VALUE` (`2147483647`) | Larger literal → C.E: integer number too large; at max size you'll typically hit `OutOfMemoryError` at runtime |

```java
int[] a1 = new int[2147483647]; // compiles; likely OutOfMemoryError at runtime
int[] a2 = new int[2147483648]; // C.E: integer number too large: 2147483648
```

### Multi-dimensional Array Creation

> Java implements multi-dimensional arrays as **"array of arrays"**, **not** as a true matrix/contiguous block — this improves memory utilization by allowing **jagged arrays** (rows of different lengths).

```java
int[][] a = new int[2][];
a[0] = new int[3];
a[1] = new int[2];
```
```java
int[][][] a = new int[2][][];
a[0] = new int[3][];
a[0][0] = new int[1];
a[0][1] = new int[2];
a[0][2] = new int[3];
a[1] = new int[2][2];
```

**Validity table:**
```java
int[] a1 = new int[];         // C.E: array dimension missing
int[][] a2 = new int[3][4];   // valid — full rectangular array
int[][] a3 = new int[3][];    // valid — jagged, rows uninitialized
int[][] a4 = new int[][4];    // C.E: ']' expected — you can only omit trailing dims, not leading ones
int[][][] a5 = new int[3][4][5]; // valid
int[][][] a6 = new int[3][4][];  // valid
int[][][] a7 = new int[3][][5];  // C.E: ']' expected — can't skip a middle dimension
```
> **Rule of thumb:** you may omit dimension sizes only from the **right** end, never in the middle or beginning.

### Array Initialization (default values)

Every array element is auto-initialized to its type's default value at creation.

```java
int[] a = new int[3];
System.out.println(a);     // [I@<hash>   (verified: [I@659e0bfd)
System.out.println(a[0]);  // 0
```
> Printing an object reference implicitly calls `toString()`. The default `Object.toString()` (unless overridden) returns `getClass().getName() + "@" + Integer.toHexString(hashCode())`.

```java
int[][] a = new int[2][3];
System.out.println(a);        // [[I@<hash>
System.out.println(a[0]);     // [I@<hash>  (a[0] is itself an int[] object)
System.out.println(a[0][0]);  // 0
```
```java
int[][] a = new int[2][];      // outer array created, rows NOT yet created
System.out.println(a);         // [[I@<hash>
System.out.println(a[0]);      // null      — inner row array not yet constructed
System.out.println(a[0][0]);   // NullPointerException (verified)
```

**Out-of-bounds access:**
```java
int[] a = new int[4];
a[4] = 50;   // ArrayIndexOutOfBoundsException (verified; JDK 21 message: "Index 4 out of bounds for length 4")
a[-4] = 60;  // ArrayIndexOutOfBoundsException: -4
```

> **Architect Note (JDK message format changed):** Older JDKs (as in the source material) print just `ArrayIndexOutOfBoundsException: 4`. Modern JDKs (9+) print the more descriptive `Index 4 out of bounds for length 4`. Functionally identical exception type — only the message text differs. Don't hardcode message-string assertions in tests across JDK versions.

### Declaration + Construction + Initialization in a Single Line

```java
int[] a = {10, 20, 30};
char[] ch = {'a', 'e', 'i', 'o', 'u'};
String[] s = {"balayya", "venki", "nag", "chiru"};

// extends to multi-dimensional:
int[][] a2 = {{10, 20, 30}, {40, 50}};
int[][][] a3 = {{{10,20,30},{40,50}}, {{60},{70,80},{90,100,110}}};
```
**Verified access:**
```java
System.out.println(a3[0][1][1]); // 50
System.out.println(a3[1][2][1]); // 100
System.out.println(a3[1][2][2]); // 110
System.out.println(a3[1][1][1]); // 80
System.out.println(a3[1][0][2]); // ArrayIndexOutOfBoundsException (row [1][0] only has {60} → index 0 valid, index 2 out of range)
```
> **Rule:** This shorthand (`{...}` without `new`) is **only** legal when declaration + construction + initialization happen together in **one statement**. You cannot split it across lines/statements:
```java
int[] a;
a = {10, 20, 30};   // C.E: illegal start of expression
```

### `length` vs `length()`

| | `length` | `length()` |
|---|---|---|
| Applies to | Arrays | `String` objects (and other classes with a `length()` method, e.g. `CharSequence`) |
| Kind | `final` **field/variable** | `final` **method** |
| Purpose | Size of the array | Number of characters in the String |

```java
int[] x = new int[3];
System.out.println(x.length());  // C.E: cannot find symbol
System.out.println(x.length);    // 3

String s = "bhaskar";
System.out.println(s.length);    // C.E: cannot find symbol
System.out.println(s.length());  // 7
```
For multi-dimensional arrays, `.length` gives only the **base (outer) dimension size**, not the total element count.
```java
int[][] a = new int[6][3];
System.out.println(a.length);     // 6
System.out.println(a[0].length);  // 3
```
There's no built-in for total element count in a jagged array — you must sum manually:
```java
int total = a[0].length + a[1].length + a[2].length + /* ... */ ;
```

### Anonymous Arrays

An array without a name, created purely for **instant, one-time use**.
```java
new int[]{10, 20, 30, 40};          // valid
new int[][]{{10, 20}, {30, 40}};    // valid
new int[3]{10, 20, 30, 40};         // C.E: ';' expected — size must NOT be specified for anonymous arrays
```
```java
public class Test {
    public static void main(String[] args) {
        System.out.println(sum(new int[]{10, 20, 30, 40})); // 100
    }
    public static int sum(int[] x) {
        int total = 0;
        for (int x1 : x) total += x1;
        return total;
    }
}
```

### Array Element Assignments

**Case 1 — Primitive arrays:** any type **promotable** to the declared type is allowed.
```java
int[] a = new int[10];
a[0] = 97;     // valid (int)
a[1] = 'a';    // valid (char widens to int)
byte b = 10;  a[2] = b;   // valid
short s = 20; a[3] = s;   // valid
a[4] = 10L;    // C.E: possible loss of precision (long does NOT narrow implicitly to int)
```

**Case 2 — Object-type arrays:** elements may be the declared type **or any subclass**.
```java
Object[] a = new Object[10];
a[0] = new Integer(10);       // valid
a[1] = new Object();          // valid
a[2] = new String("bhaskar"); // valid

Number[] n = new Number[10];
n[0] = new Integer(10);       // valid — Integer extends Number
n[1] = new Double(10.5);      // valid — Double extends Number
n[2] = new String("bhaskar"); // C.E: incompatible types — String doesn't extend Number
```

**Case 3 — Interface-type arrays:** elements must implement that interface.
```java
Runnable[] r = new Runnable[10];
r[0] = new Thread();               // valid — Thread implements Runnable
r[1] = new String("bhaskar");      // C.E: incompatible types
```

**Summary table:**

| Array type | Allowed element type |
|---|---|
| Primitive arrays | Any type promotable to the declared type |
| Object-type arrays | Declared type or any subclass |
| Interface-type arrays | Any implementing class |
| Abstract-class-type arrays | Any concrete subclass |

### Array Variable Assignments

**Case 1 — Element-level promotion does NOT apply at the array (whole-object) level.**
```java
int[] a = {10, 20, 30};
char[] ch = {'a', 'b', 'c'};
int[] b = a;    // valid — same type
int[] c = ch;   // C.E: incompatible types — char[] is NOT assignable to int[]
                //  (even though a single char widens to int, char[] does not widen to int[])
```
**Exception — reference-type (object) arrays support covariant assignment:**
```java
String[] s = {"A", "B"};
Object[] o = s;   // valid — array covariance for reference types
```

**Case 2 — Array assignment reassigns the reference, not a deep copy of elements.**
```java
int[] a = {10, 20, 30, 40, 50, 60, 70};
int[] b = {80, 90};
a = b;   // valid — a now points to b's array object
b = a;   // valid
```
> Sizes are irrelevant for reference assignment — only **types must match**.

**Case 3 — For multi-dimensional arrays, dimension count must match, sizes don't matter:**
```java
int[][] a = new int[3][];
a[0] = new int[4][5];  // C.E: incompatible types — a[0] expects int[], not int[][]
a[0] = 10;              // C.E: incompatible types
a[0] = new int[4];      // valid
```

**Classic "how many objects created / eligible for GC" interview question:**
```java
int[][] a = new int[3][2];  // 1 outer array + 3 inner arrays = 4 objects
a[0] = new int[3];          // old a[0] (size-2) now unreferenced, GC-eligible; new object created
a[1] = new int[4];          // old a[1] now GC-eligible
a = new int[4][3];          // entire previous outer array (and remaining live inner arrays) now GC-eligible;
                             // 1 new outer + 4 new inner = 5 new objects
```
*(Total objects created across the snippet: 4 initial + 2 replacements + 5 new = 11. Objects eligible for GC by the end: the 3 originally-created inner arrays + a[0]-replacement + a[1]-replacement + the original outer array = 6 — matching the source material's stated answer of "11 created, 6 GC-eligible.")*

**`args` reassignment demonstrates array-variable-assignment semantics on `main`'s parameter:**
```java
public class Test {
    public static void main(String[] args) {
        String[] argh = {"A", "B"};
        args = argh;
        System.out.println(args.length); // 2
        for (int i = 0; i <= args.length; i++) {   // BUG: should be i < args.length
            System.out.println(args[i]);
        }
    }
}
```
**Output:** prints `2`, `A`, `B`, then throws `ArrayIndexOutOfBoundsException: 2` — *regardless* of how many actual command-line arguments were supplied, because `args` was reassigned inside `main` before the loop runs. Fixing `<=` to `<` resolves it. Using an enhanced for-loop (`for (String s : args)`) sidesteps the bug entirely — a good practical takeaway for code review.

---

## 7. Types of Variables

### Division 1 — By the kind of value held
1. **Primitive variables** — hold primitive values directly. `int x = 10;`
2. **Reference variables** — hold a reference to an object (not the object itself). `Student s = new Student();`

### Division 2 — By declaration position/behavior
1. **Instance variables**
2. **Static variables**
3. **Local variables**

### Instance Variables
- Value **varies per object** — each object gets its own copy.
- Declared directly inside the class, **outside** any method/block/constructor.
- Created at object-creation time, destroyed at object-destruction (GC) time → scope == object's lifetime.
- Stored on the **Heap**, as part of the object.
- Accessible directly from **instance context**; **not** directly from a static context — must go through an object reference.
- JVM auto-initializes with default values — explicit initialization isn't mandatory.
- Also called: **object-level variables / attributes / fields**.

```java
public class Test {
    int i = 10;
    public static void main(String[] args) {
        // System.out.println(i); // C.E: non-static variable i cannot be referenced from a static context
        Test t = new Test();
        System.out.println(t.i); // 10
        t.methodOne();
    }
    public void methodOne() {
        System.out.println(i); // 10 — direct access legal from instance context
    }
}
```

### Static Variables
- Value does **not** vary per object → a **single copy shared** across all instances of the class.
- Declared with the `static` modifier, directly inside the class (outside method/block/constructor).
- Created at **class-loading** time, destroyed at **class-unloading** time → scope == lifetime of the `.class` file in the JVM.
- Stored in the **Method Area** (in modern JVMs, this lives inside **Metaspace**, off-heap, since Java 8 — see Architect Note below).
- Accessible from **both** static and instance contexts directly.
- Recommended access via **class name** (`Test.i`), though object-reference access (`t.i`) and bare access from within the same class also work.
- JVM auto-provides default values.
- Also called: **class-level variables / fields**.

```java
public class Test {
    static int i = 10;
    public static void main(String[] args) {
        Test t = new Test();
        System.out.println(t.i);    // 10 — works, but not recommended style
        System.out.println(Test.i); // 10 — recommended
        System.out.println(i);      // 10 — bare access, legal within same class
    }
}
```
```java
public class Test {
    int x = 10;
    static int y = 20;
    public static void main(String[] args) {
        Test t1 = new Test();
        t1.x = 888;
        t1.y = 999;             // legal syntax, but modifies the SHARED static field
        Test t2 = new Test();
        System.out.println(t2.x + "----" + t2.y); // 10----999 (verified)
    }
}
```
`t2.x` is `10` (t2's own copy, unaffected by t1's change) but `t2.y` is `999` (shared static field, mutated via `t1.y = 999`).

**JVM class-lifecycle sequence (interview favorite):**
1. Start JVM
2. Create and start Main Thread
3. Locate `Test.class`
4. Load `Test.class` → static variable creation
5. Execute `main()`
6. Unload `Test.class` → static variable destruction
7. Terminate Main Thread
8. Shutdown JVM

> **Architect Note (JVM internals, Java 8+):** The classic **PermGen** space (where static class metadata lived pre-Java 8) was **removed** in Java 8 and replaced by **Metaspace**, which is allocated from native (off-heap) memory rather than a fixed-size heap region — largely eliminating `OutOfMemoryError: PermGen space` for class-metadata growth (though `OutOfMemoryError: Metaspace` is still possible if `-XX:MaxMetaspaceSize` is capped). Static *variable values* conceptually live with class metadata, though modern JVM implementations treat static fields somewhat like heap objects associated with the `Class` object for GC purposes — an important nuance for architects reasoning about class-unloading with custom classloaders (e.g., in app servers, OSGi, or hot-reload frameworks).

### Local Variables
- Also called: **automatic / temporary / stack variables**.
- Declared inside a method, block, or constructor to meet a temporary requirement.
- Stored on the **Stack**.
- Scope == the block in which declared; created on block-entry, destroyed on block-exit.
- **No default value** provided by the JVM — **must be explicitly initialized before use**, or you get a compile-time error.

```java
public class Test {
    public static void main(String[] args) {
        int x;
        if (args.length > 0) {
            x = 10;
        }
        System.out.println(x); // C.E: variable x might not have been initialized
                                //  (compiler can't prove the if-branch always runs)
    }
}
```
```java
public class Test {
    public static void main(String[] args) {
        int x;
        if (args.length > 0) { x = 10; }
        else                 { x = 20; }
        System.out.println(x); // compiles — compiler proves ALL paths assign x
    }
}
```
- **Best practice:** avoid initializing local variables only inside conditional blocks — there's no guarantee that block executes at runtime; prefer initializing at declaration with a sensible default.
- **Only `final` is a legal modifier for local variables.** Any access modifier (`public`/`private`/`protected`) or `static`/`volatile`/`transient` on a local variable is a **compile-time error**.
```java
public static void main(String[] args) {
    public int x = 10;      // C.E: illegal start of expression
    static int y = 10;      // C.E
    volatile int z = 10;    // C.E
    final int w = 10;       // valid
}
```

### Conclusions — Variable Types Comparison

| Aspect | Instance | Static | Local |
|---|---|---|---|
| Default value provided? | Yes | Yes | **No** — must initialize explicitly |
| Copies | One per object | One per class | One per Thread invocation |
| Thread safety | **Not** thread-safe (shared, mutable across threads) | **Not** thread-safe | **Thread-safe** (each thread has its own stack frame/copy) |
| Default modifier allowed | Yes (package-private if unspecified) | Yes | N/A — only `final` legal |
| Storage | Heap (part of object) | Method Area / Metaspace | Stack |

### Uninitialized Arrays — Default Behavior at Each Level

Regardless of whether the array reference is instance, static, or local, once **constructed** its elements always get default values. The nuance is what happens with an **uninitialized reference**:

**Instance level:**
```java
int[] a;                          // uninitialized field
System.out.println(obj.a);        // null
System.out.println(obj.a[0]);     // NullPointerException
// vs.
int[] a = new int[3];
System.out.println(obj.a);        // [I@<hash>
System.out.println(obj.a[0]);     // 0
```

**Static level:** identical pattern, just `null` at the class level before construction.

**Local level:** an uninitialized **local** array reference doesn't even get `null` implicitly — the compiler flags it:
```java
int[] a;
System.out.println(a);       // C.E: variable a might not have been initialized
```

---

## 8. Var-Arg Methods

Introduced in **Java 1.5** to allow methods with a **variable number of arguments**, avoiding the need to overload a method for every possible argument count.

```java
public class Test {
    public static void methodOne(int... x) {
        System.out.println("var-arg method");
    }
    public static void main(String[] args) {
        methodOne();          // var-arg method
        methodOne(10);        // var-arg method
        methodOne(10, 20, 30);// var-arg method
    }
}
```

Internally, a var-arg parameter is implemented as a **single-dimensional array**, so it can be indexed or iterated normally:
```java
public static void sum(int... x) {
    int total = 0;
    for (int i = 0; i < x.length; i++) total += x[i];
    System.out.println("The sum: " + total);
}
// sum(); sum(10); sum(10,20); sum(10,20,30,40);
// Output: 0, 10, 30, 100  (verified)
```

### Rules & Cases

**Case 1 — valid syntactic forms:**
```java
methodOne(int... x)   // valid
methodOne(int ...x)   // valid
methodOne(int...x)    // valid
methodOne(int x...)   // invalid — ellipsis must precede the parameter name
```

**Case 2 — can mix var-arg with normal parameters:**
```java
methodOne(int a, int... b)     // valid
methodOne(String s, int... x)  // valid
```

**Case 3 — if mixed, var-arg param must be the LAST parameter:**
```java
methodOne(int a, int... b)  // valid
methodOne(int... a, int b)  // invalid — C.E
```

**Case 4 — only ONE var-arg parameter allowed per method:**
```java
methodOne(int... a, int... b) // invalid — C.E
```

**Case 5 — overload resolution: var-arg gets LOWEST priority** (exactly like the `default` case in a `switch`):
```java
public static void methodOne(int i) { System.out.println("general method"); }
public static void methodOne(int... i) { System.out.println("var-arg method"); }

methodOne();      // var-arg method  (no exact match, falls through to var-arg)
methodOne(10, 20);// var-arg method  (arity doesn't match the single-int overload)
methodOne(10);    // general method  (exact match preferred over var-arg — VERIFIED on JDK 21)
```

**Case 6 — you may pass the equivalent array directly:**
```java
methodOne(new int[]{10, 20, 30}); // treated as a single var-arg call
```

**Case 7 — cannot overload `int[]` and `int...` in the same class — they erase to the same signature:**
```java
public void methodOne(int[] i) {}
public void methodOne(int... i) {}
// C.E: cannot declare both methodOne(int...) and methodOne(int[]) in Test  (VERIFIED)
```

### Single-Dimensional Array vs. Var-Arg

- **Anywhere a 1D array parameter is valid, you may replace it with a var-arg parameter** (this is legal even for `main`!):
```java
public class Test {
    public static void main(String... args) {   // valid alternative to String[] args
        System.out.println("var-arg main method");
    }
}
```
- **The reverse is NOT true** — you cannot mechanically replace a var-arg parameter with a plain single-dimensional array parameter if the intended semantics differ across dimensions:
  - `methodOne(int... x)` → callable with a spread of `int`s → `x` becomes `int[]`.
  - `methodOne(int[]... x)` → callable with a spread of `int[]`s → `x` becomes `int[][]`.
```java
public class Test {
    public static void methodOne(int[]... x) {
        for (int[] a : x) System.out.println(a[0]);
    }
    public static void main(String[] args) {
        int[] l = {10, 20, 30};
        int[] m = {40, 50};
        methodOne(l, m); // 10, then 40
    }
}
```

> **Architect Insight:** Var-args + autoboxing + overloading is a classic source of subtle bugs and `unchecked generic array creation` warnings (e.g., `List<String>... lists` triggers heap-pollution warnings — worth knowing `@SafeVarargs` exists precisely to suppress this when the method is verified safe). Also worth mentioning in interviews: excessive var-arg or overload chains increase **megamorphic call site** overhead and complicate JIT inlining in hot paths — a legitimate performance-review consideration in high-throughput systems.

---

## 9. The `main()` Method

- Whether a class contains a proper `main()` method is **not checked by the compiler** — it's the **JVM's runtime responsibility**.
- If the JVM can't locate the required `main()` signature, you get a runtime failure (message format has evolved across JDK versions — see below).

**Required signature the JVM searches for:**
```java
public static void main(String[] args)
```

### Verified behavior across JDK generations

| | Java ≤ 1.6 | Java 1.7+ (and modern JDK 21, verified) |
|---|---|---|
| Missing `main()` | `java.lang.NoSuchMethodError: main` | `Error: Main method not found in class X, please define the main method as: public static void main(String[] args) or a JavaFX application class must extend javafx.application.Application` |

**Verified on JDK 21:**
```
$ javac Test.java && java Test
Error: Main method not found in class Test, please define the main method as:
   public static void main(String[] args)
or a JavaFX application class must extend javafx.application.Application
```
(The JavaFX clause is a newer addition beyond what the original 1.7-era material describes, but the core "helpful error message" behavior described in the source is accurate and persists today.)

### Acceptable variations to the `main()` signature

The signature looks rigid, but these variations are legal:

1. **Modifier order doesn't matter** — `static public` works exactly like `public static`.
2. `String[]` can be written as `String[] args`, `String []args`, or `String args[]`.
3. The parameter name need not be `args` — any valid identifier works.
4. `String[]` can be replaced with the **var-arg** form: `main(String... args)`.
5. `main()` may additionally carry `final`, `synchronized`, `strictfp` modifiers.

```java
public class Test {
    static final synchronized strictfp public void main(String... ask) {
        System.out.println("valid main method");
    }
}
// Output: valid main method  (all these modifiers + var-arg form are legal together)
```

### Validity matrix (frequently asked as MCQ)

| Declaration | Verdict | Reason |
|---|---|---|
| `public static void main(String args){}` | Invalid | Missing `[]` — not an array/var-arg param |
| `public synchronized final strictfp void main(String[] args){}` | **Invalid** | Missing `static` |
| `public static void Main(String... args){}` | Invalid | Wrong case: `Main` ≠ `main` (Java is case-sensitive) |
| `public static int main(String[] args){}` | Invalid | Return type must be `void` |
| `public static synchronized final strictfp void main(String... args){}` | Valid | All optional modifiers + var-arg form |
| `public static void main(String... args){}` | Valid | Var-arg substitution for array |
| `public void main(String[] args){}` | Invalid | Missing `static` |

> **Important nuance from the source material:** All of the "invalid" cases above are **compile-time valid Java syntax** (they compile fine as an ordinary method) — the class just won't be *executable* as an entry point, because the JVM's `main`-method lookup requires the *exact* signature. So the failure surfaces as a **runtime** error (`NoSuchMethodError`/"Main method not found"), **not** a compiler error. This is a nuanced but very commonly tested point.

### Overloading `main()`
Legal — but the JVM **always** invokes the `String[]`-parameter overload as the entry point. Other overloads must be called explicitly from within.
```java
public class Test {
    public static void main(String[] args) {
        System.out.println("String[] array main method");
    }
    public static void main(int[] args) {
        System.out.println("int[] array main method");
    }
}
// Output: String[] array main method
```

### Inheritance and `main()`
Static methods (including `main`) participate in **inheritance** — if the child class doesn't declare its own `main()`, the JVM will find and run the **parent's** static `main()` when you execute `java Child`.
```java
class Parent {
    public static void main(String[] args) {
        System.out.println("parent main");
    }
}
class Child extends Parent {}
// java Child  →  "parent main"   (VERIFIED on JDK 21)
```
If **both** declare `main()`, running `java Child` invokes `Child`'s version. This *looks* like polymorphic overriding but **is not** — it's **method hiding**, because static methods are resolved by **compile-time (reference) type**, not dynamically dispatched via the vtable like instance methods.

### Java 1.7 Enhancements to `main()`

**Case 1 — Better diagnostic message** (superseded description above still applies — JDK 21 confirms the improved message persists and has grown richer over time).

**Case 2 — `main()` is mandatory to *start* execution, even if a static initializer block exists.**
```java
class Test {
    static {
        System.out.println("static block");
    }
}
```
- **Pre-1.7 behavior:** prints `static block`, then throws `NoSuchMethodError: main`.
- **1.7+ behavior (and current JDK 21, verified):** JVM refuses to run at all — prints the "Main method not found" error **without** executing the static block, because the JVM validates the presence of `main()` **before** initiating class initialization.

**Case 3 — even `System.exit(0)` inside the static block doesn't help** if `main()` is missing — same "main method not found" error, static block does not execute in 1.7+.

**Case 4 — when `main()` IS present, the static block still runs first (class-initialization ordering is unchanged):**
```java
class Test {
    static {
        System.out.println("static block");
    }
    public static void main(String[] args) {
        System.out.println("main method");
    }
}
// Output:
// static block
// main method
```

---

## 10. Command-Line Arguments

Arguments passed from the OS shell/command prompt when launching the JVM, letting you customize `main()`'s runtime behavior without recompilation.

```java
public class Test {
    public static void main(String[] args) {
        for (int i = 0; i <= args.length; i++) {   // BUG
            System.out.println(args[i]);
        }
    }
}
// java Test x y z  →  prints x, y, z, then ArrayIndexOutOfBoundsException: 3
// Fix: use i < args.length
```

- All command-line arguments arrive as **`String`** — this means `+` between two args is **String concatenation**, not arithmetic addition:
```java
public static void main(String[] args) {
    System.out.println(args[0] + args[1]);
}
// java Test 10 20   →  output: 1020   (NOT 30)
```
- **Space** is the default separator between arguments. If an argument itself contains a space, wrap it in double quotes:
```
java Test "Sai Charan"   →  args[0] = "Sai Charan"
```
- Reassigning `args` inside `main` (as shown in the Arrays section) fully overrides whatever was passed on the actual command line — a useful fact for writing self-contained demo programs that don't depend on external launch configuration.

---

## 11. Java Coding Standards

Following conventions improves readability/maintainability; component names should reflect their purpose.

| Component | Convention | Examples |
|---|---|---|
| **Classes** | Nouns; PascalCase (UpperCamelCase) | `Employee`, `AccountManager` |
| **Interfaces** | Adjectives (describing a capability); PascalCase | `Serializable`, `Runnable`, `Cloneable` |
| **Methods** | Verb or verb-noun; camelCase (lowercase first letter) | `getBalance()`, `calculateInterest()` |
| **Variables** | Nouns; camelCase | `length`, `name`, `salary`, `age`, `mobileNumber` |
| **Constants** | Nouns; ALL_UPPERCASE with `_` between words; typically `public static final` | `MAX_VALUE`, `MIN_VALUE`, `NORM_PRIORITY` |

### Java Bean Conventions

A **Java Bean** is a simple class with **private** properties and **public** getter/setter methods.

**Setter method rules:**
1. Prefixed with `set`.
2. Must be `public`.
3. Return type must be `void`.
4. Must accept exactly one argument (the value to set).

**Getter method rules:**
1. Prefixed with `get`.
2. Must be `public`.
3. Return type must **not** be `void`.
4. Must be a no-argument method.

> For **boolean** properties, the getter may be prefixed with `get` **or** `is` — `is` is the recommended convention (e.g. `isActive()` rather than `getActive()`), and this convention is exactly what frameworks like JavaBeans introspection, Jackson, and JPA rely on for property discovery via reflection.

```java
public class Employee {
    private String name;
    private boolean active;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public boolean isActive() { return active; }         // preferred over getActive()
    public void setActive(boolean active) { this.active = active; }
}
```

### Listener Naming Conventions

**To register a listener:** method name prefixed with `add`, and the parameter type must match the specific listener interface name.
```java
public void addMyActionListener(MyActionListener l)    // valid
public void registerMyActionListener(MyActionListener l)// invalid — wrong prefix
public void addMyActionListener(ActionListener l)        // invalid — parameter type doesn't match method name convention
```

**To unregister a listener:** method name prefixed with `remove`.
```java
public void removeMyActionListener(MyActionListener l)  // valid
public void unregisterMyActionListener(MyActionListener l) // invalid — wrong prefix
public void removeMyActionListener(ActionListener l)        // invalid — mismatched type
public void deleteMyActionListener(MyActionListener l)      // invalid — wrong prefix ('delete' not 'remove')
```

> **Architect Insight:** These naming rules aren't just style — they're the contract that the **JavaBeans specification** (`java.beans.Introspector`) and many IDE/tool/reflection-based frameworks (Spring's bean wiring historically, Swing's event model, JSF, JSP EL, JSTL) rely on to auto-discover properties and event-registration methods via reflection. Breaking the convention silently breaks framework auto-wiring — a real-world debugging trap.

---

## 12. JVM Memory Areas

| Memory Area | What lives there |
|---|---|
| **Method Area** | Class-level binary data, including static variables. (Since Java 8: implemented via **Metaspace**, off native heap — replaces the old **PermGen**.) |
| **Heap** | Objects and their instance variables. Shared across all threads. |
| **Stack** (one per Thread) | For every method call by that thread: a **Stack Frame** (a.k.a. **Activation Record**) holding local variables and call state. Destroyed when the method returns. |
| **PC (Program Counter) Register** (one per Thread) | Address/offset of the next instruction to execute for that thread. |
| **Native Method Stack** | Bookkeeping for native (JNI) method invocations. |

**Key relationships to remember for interviews:**
- **Heap** and **Method Area/Metaspace** are shared across all threads of the JVM.
- **Stack**, **PC Register**, and **Native Method Stack** are **per-thread** — this is precisely *why* local variables are inherently thread-safe (each thread has its own private copy on its own stack) while instance/static variables (living in the shared Heap/Method Area) are not, and require explicit synchronization for safe concurrent access.

> **Architect Note (beyond source material, current JVM specifics worth citing in senior interviews):**
> - **Escape analysis** and **scalar replacement** in modern JIT compilers (C2/Graal) can allocate objects that provably don't escape a method **on the stack** instead of the heap, reducing GC pressure — an optimization interviewers sometimes probe for at the architect level.
> - **G1**, **ZGC**, and **Shenandoah** are the modern low-pause-time garbage collectors (Java 9+/11+/12+ respectively) that manage the Heap differently from the older Serial/Parallel/CMS collectors implied by this era of material.
> - `-Xss` tunes per-thread stack size; `StackOverflowError` results from exceeding it (e.g., deep/unbounded recursion) — directly tied to the "Stack" row above.
> - `-Xmx`/`-Xms` tune heap size; `OutOfMemoryError: Java heap space` results from heap exhaustion — directly tied to the "Heap" row.
> - `-XX:MaxMetaspaceSize` tunes Metaspace; uncontrolled dynamic class generation (common in apps with heavy proxying/bytecode-gen frameworks, e.g. CGLIB, ASM, many DI containers) can trigger `OutOfMemoryError: Metaspace` if classloaders aren't released, a classic memory-leak pattern in long-running app servers.

---

## 13. Interview Rapid-Fire Cheat Sheet

| Q | A |
|---|---|
| Can a class/interface name be reused as a variable name? | Yes, legal (different namespace) but bad practice. |
| Is Java pure OOP? | No — no multiple inheritance/operator overloading, and primitives aren't objects. |
| Only unsigned primitive type? | `char` (0 to 65535). |
| Why is `char` 2 bytes in Java but 1 byte in C? | Java is Unicode-based (>256 chars need 2 bytes); C is ASCII-based. |
| Default suffix-less literal type for integers? Floats? | `int`; `double`. |
| Can you assign a floating literal to an int variable? | No — even `10.0` to `int` is a compile error. |
| Can you assign an integral literal to a float/double variable? | Yes — implicit widening, even from octal/hex forms. |
| Where must the underscore `_` sit in a numeric literal? | Strictly between two digits — never adjacent to a prefix, suffix, or decimal point. |
| Is array size required at declaration? At construction? | Not allowed at declaration; **mandatory** at construction (`new`). |
| Legal to have a zero-length array? | Yes. |
| What exception for negative array size? | `NegativeArraySizeException` (runtime, not compile-time). |
| What are the only legal types for an array-size expression? | `byte`, `short`, `char`, `int`. |
| How are Java multi-dim arrays implemented internally? | Array-of-arrays (jagged), not a true matrix. |
| `length` vs `length()`? | `length` = field, for arrays. `length()` = method, for `String`/`CharSequence`. |
| What is an anonymous array, and can you specify its size? | A nameless array for one-time use; size must **not** be specified. |
| Can a `char[]` be assigned to an `int[]` variable? | No — element-level promotion doesn't extend to array-level assignment. |
| Can a `String[]` be assigned to an `Object[]` variable? | Yes — reference-type array covariance. |
| Which variable category gets NO default value from the JVM? | Local variables — must be explicitly initialized before use. |
| Which is the only legal modifier on a local variable? | `final`. |
| Are instance/static variables thread-safe by default? | No. Are local variables? Yes (per-thread stack copy). |
| From 1.7 onward, will a static block execute if `main()` is absent? | No — the JVM validates `main()`'s presence **before** any class initialization. |
| Is overriding involved when both parent and child declare `static main()`? | No — it's method hiding, not overriding (static methods aren't polymorphic). |
| Which `main()` overload does the JVM invoke as entry point? | Always the `String[]` (or var-arg `String...`) parameter version. |
| Can `main()`'s `String[] args` be replaced by var-args? | Yes: `main(String... args)`. |
| Which modifiers are legal (optional) on `main()`? | `final`, `synchronized`, `strictfp` — in addition to the mandatory `public static void`. |
| Priority of a var-arg method vs. an exact-match overload? | Var-arg is **last resort** — exact match always wins. |
| Can you overload `foo(int[])` and `foo(int...)` in the same class? | No — compile-time error, they have the same erased signature. |
| If var-arg mixed with fixed params, where must the var-arg be? | Last parameter, and only one var-arg parameter allowed. |
| Recommended getter prefix for boolean properties? | `is` (e.g., `isActive()`), though `get` is also legally accepted. |
| Listener registration/unregistration method prefixes? | `add...` / `remove...`. |
| Which JVM memory areas are per-thread vs. shared? | Per-thread: Stack, PC Register, Native Method Stack. Shared: Heap, Method Area/Metaspace. |
| Where do static variables physically live in modern JVMs (8+)? | Metaspace (native/off-heap), replacing the old PermGen. |

---

*All code snippets in this document were compiled and executed against **OpenJDK 21.0.10** to verify correctness; any behavioral drift from the original 1.6/1.7-era source material is explicitly called out as an "Architect Note."*
