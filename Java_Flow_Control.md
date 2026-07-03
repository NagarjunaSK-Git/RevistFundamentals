# Core Java — Language Fundamentals: Flow Control
### Study Material for Senior Java Architect Interview Preparation

> Source basis: DURGASOFT Core Java (SCJP/OCJP) Chapter 3 — *Flow Control*, extended and corrected with modern JLS behavior (Java 5 → Java 21) and interview-oriented annotations for architect-level depth.

---

## Table of Contents
1. Introduction to Flow Control
2. Selection Statements — `if-else`
3. Selection Statements — `switch`
4. Iterative Statements — `while`
5. Iterative Statements — `do-while`
6. Iterative Statements — `for`
7. Iterative Statements — `for-each` (Enhanced for)
8. `Iterable` vs `Iterator`
9. Transfer Statements — `break`, `continue`, Labeled statements
10. The "do-while + continue" trap
11. Modern Java Enhancements to Flow Control (Java 5 – 21)
12. Architect-Level Interview Questions
13. Quick Reference Cheat Sheet

---

## 1. Introduction to Flow Control

**Definition:** Flow control describes the order in which statements are executed at runtime. By default, the JVM executes statements **sequentially, top to bottom**. Flow-control constructs let a program deviate from that sequence based on conditions, repetition needs, or the need to jump out of a construct.

```
                         Flow Control
                              |
      -----------------------------------------------------
      |                       |                             |
Selection Statements   Iterative Statements          Transfer Statements
  1. if-else              1. while()                   1. break
  2. switch                2. do-while()                2. continue
                            3. for()                     3. return
                            4. forEach() (1.5+)          4. try-catch-finally
                                                          5. assert (1.4+)
```

**Architect note:** `return`, `try-catch-finally`, and `assert` are technically transfer/flow-control-adjacent constructs (they alter control flow and are frequently tested alongside `break`/`continue` in interviews), even though the source material focuses primarily on `break`/`continue`. Keep them in mind when discussing "all ways to transfer control in Java."

---

## 2. Selection Statements — `if-else`

### Syntax
```java
if (b)          // b MUST be of type boolean (or Boolean, via unboxing)
{
    // action if b is true
} else {
    // action if b is false
}
```

### Key Rule
The condition **must evaluate to `boolean`** (or auto-unbox from `Boolean`). Unlike C/C++, Java does **not** allow integer-to-boolean implicit conversion.

```java
public class ExampleIf {
    public static void main(String[] args) {
        int x = 0;
        if (x) {                 // COMPILE-TIME ERROR
            System.out.println("hello");
        }
    }
}
```
```
ExampleIf.java:4: error: incompatible types: int cannot be converted to boolean
if(x)
```

### Assignment vs Comparison — The Classic Trap
```java
int x = 10;
if (x = 20) { ... }     // COMPILE ERROR: int cannot be converted to boolean
```
```java
boolean b = false;
if (b = true) {          // LEGAL — assignment expression whose VALUE is boolean
    System.out.println("hello");   // prints "hello"
}
```
> **Interview gold:** In Java, `if (x = 20)` fails to compile because assignment of an `int` produces an `int`, and `int` cannot substitute for `boolean`. This is *precisely* the safety net C/C++ lacks (where `if (x = 20)` silently compiles and is a notorious bug source). Only `boolean`-typed assignment expressions like `if (b = true)` compile — and this is a favorite "gotcha" MCQ.

### Optional Braces & Optional `else`
- Both the `else` branch and `{}` are optional.
- Without `{}`, only **one statement** may follow `if`, and it **must not be a declaration statement** (local variable declarations are not "statements" in the grammar sense usable here).

```java
if (true)
    System.out.println("hello");     // OK — single statement
```

```java
if (true)
    int x = 10;   // COMPILE ERROR — declaration cannot be a lone dependent statement
```
```
error: '.class' expected / not a statement
```

Wrapping in `{}` fixes it:
```java
if (true) {
    int x = 10;   // OK, compiles cleanly, but x is unused (no output)
}
```

### The Empty Statement (`;`)
```java
if (true);   // ';' is a valid, legal "empty statement" in Java — produces no output
```
This compiles successfully and does nothing — a subtle bug source when a developer accidentally places `;` right after an `if(...)`.

### Dependent vs Independent Statements
```java
if (true)
    System.out.println("hello");   // <-- dependent on if (executes only if true)
    System.out.println("hi");      // <-- NOT dependent; always executes (indentation lies!)
```
**Architect note:** Indentation is cosmetic in Java. Only the statement immediately after `if` (no braces) is conditional. This is why static-analysis tools (Checkstyle, SonarQube "Rule: Missing curly braces") flag brace-less `if` blocks — mis-indentation like the above is a classic production bug (related to Apple's historical "goto fail" style defects, though that was C).

---

## 3. Selection Statements — `switch`

### Why switch over if-else?
When there are many discrete options, `switch` improves **readability** over long `if-else-if` chains and can be optimized by the JVM into a `tableswitch` or `lookupswitch` bytecode instruction for O(1)/O(log n) dispatch (vs. O(n) sequential `if-else` comparisons) — a good architect-level point on performance.

### Syntax
```java
switch (x) {
    case 1:
        action1;
    case 2:
        action2;
    // ...
    default:
        defaultAction;
}
```

### Allowed Switch Argument Types (evolution across versions)
| Java Version | Allowed `switch` Argument Types |
|---|---|
| ≤ 1.4 | `byte`, `short`, `char`, `int` |
| 1.5 | + wrapper classes `Byte`, `Short`, `Character`, `Integer` (via autoboxing/unboxing) and `enum` |
| 1.7 | + `String` |
| 14 (standard)/21 | + **switch expressions**, **pattern matching for switch** (sealed types, records, `null` labels) — see Section 11 |

> `long`, `float`, `double`, and `boolean` are **never** valid switch types — because `switch` fundamentally compares discrete/enumerable values (bytecode-level `tableswitch`/`lookupswitch` needs int-comparable keys internally), and floating-point equality/`boolean` aren't suited to this dispatch model.

### Structural Rules
- **Curly braces `{}` are mandatory** for switch (unlike `if`/loops where they're optional).
- Both `case` and `default` are **optional**.
- Every statement inside `switch` **must belong to some `case` or `default`** — standalone/independent statements directly inside the switch body are illegal.

```java
switch (x) {
    System.out.println("hello");  // COMPILE ERROR: not under any case/default
}
```
```
error: case, default, or '}' expected
```

### Case Labels Must Be Compile-Time Constants
```java
int x = 10, y = 20;
switch (x) {
    case 10:
        System.out.println("10");
    case y:                      // COMPILE ERROR — y is not a constant
        System.out.println("20");
}
```
```
error: constant expression required
```
Declaring `y` as `final` (with a constant initializer known at compile time) fixes it:
```java
final int y = 20;   // now a "compile-time constant" (constant variable)
switch (x) {
    case 10: System.out.println("10");
    case y:  System.out.println("20");  // legal now
}
```

**Architect nuance:** A `final` variable is a *compile-time constant* **only if** it is initialized with a constant expression at the point of declaration (JLS §4.12.4 "constant variable"). `final int y = someMethodCall();` would **not** qualify, and would still fail as a case label.

### Switch Argument vs Case Label — Expressions Allowed Differently
The **switch argument** can be *any* runtime expression; the **case label** must be a *constant* expression.
```java
int x = 10;
switch (x + 1) {          // runtime expression — perfectly legal
    case 10:
    case 10 + 20:          // constant expression — legal
    case 10 + 20 + 30:     // constant expression — legal
}
// No output (x+1 = 11, doesn't match any case, no default) — compiles & runs fine.
```

### Case Label Range Must Fit the Switch Argument's Type
```java
byte b = 10;
switch (b) {
    case 10:   System.out.println("10");
    case 100:  System.out.println("100");
    case 1000: System.out.println("1000");   // COMPILE ERROR
}
```
```
error: incompatible types: possible lossy conversion from int to byte
```
`1000` exceeds the range of `byte` (-128 to 127), so it cannot be a valid case label for a `byte` switch.

### Duplicate Case Labels Are Forbidden
Values that are *equal after implicit widening/promotion* count as duplicates — even if written in different literal forms (int literal vs char literal):
```java
int x = 10;
switch (x) {
    case 97:  System.out.println("97");
    case 99:  System.out.println("99");
    case 'a': System.out.println("100");   // 'a' == 97 -> duplicate of case 97
}
```
```
error: duplicate case label
case 'a':
```

### Case Label Rules Summary (Diagram)
| Rule | Detail |
|---|---|
| 1 | Case label must be a compile-time constant |
| 2 | Expressions allowed, but must be a compile-time constant *expression* |
| 3 | Value must be within the range of the switch argument's type |
| 4 | Duplicate case labels not allowed |

### Fall-Through Behavior
> **Definition:** Once a `case` matches, execution continues into **every subsequent statement** (regardless of `case` boundaries) until it hits a `break` or the end of the `switch` block. This is called **fall-through**.

```java
int x = 0;
switch (x) {
    case 0:
        System.out.println("0");
    case 1:
        System.out.println("1");
        break;
    case 2:
        System.out.println("2");
    default:
        System.out.println("default");
}
```
| x | Output |
|---|---|
| 0 | `0` then `1` (falls through 0→1, stops at break) |
| 1 | `1` (stops at break) |
| 2 | `2` then `default` |
| 3 | `default` (no case matches; jumps straight to default) |

**Use-case for fall-through:** defining a *common action* shared by multiple cases (e.g., grouping weekend days, or multiple equivalent HTTP status handling) without duplicating code:
```java
switch (day) {
    case SATURDAY:
    case SUNDAY:
        System.out.println("Weekend");
        break;
    default:
        System.out.println("Weekday");
}
```

### `default` Case Rules
- `default` can appear **at most once**.
- `default` executes **only if no other case matches**.
- `default` can be placed **anywhere** in the switch body (not necessarily last) — but convention (and virtually every style guide, Checkstyle rule, and code review standard) places it last.
- **Critical gotcha:** if `default` is *not* the last case, fall-through still applies positionally — statements below `default` (even under other case labels) still execute if `default` is reached and there's no `break`.

```java
int x = 0;
switch (x) {
    default:
        System.out.println("default");
    case 0:
        System.out.println("0");
        break;
    case 1:
        System.out.println("1");
    case 2:
        System.out.println("2");
}
```
| x | Output |
|---|---|
| 0 | `0` (matches case 0 directly, break stops it) |
| 1 | `1` then `2` (fall-through) |
| 2 | `2` |
| 3 | `default` then `0` (no match → default runs → falls through into case 0's body → break stops it) |

---

## 4. Iterative Statements — `while`

**When to use:** Best suited when the **number of iterations is not known in advance** and depends on a runtime condition (e.g., `ResultSet.next()`, `Iterator.hasNext()`, `Enumeration.hasMoreElements()`).

```java
while (rs.next()) { ... }
while (e.hasMoreElements()) { ... }
while (itr.hasNext()) { ... }
```

### Rule: Argument Must Be `boolean`
```java
while (1) { ... }   // COMPILE ERROR — incompatible types: int -> boolean
```

### Optional Braces
Without `{}`, only one non-declarative statement is allowed (same rule as `if`).
```java
while (true)
    System.out.println("hello");   // legal, infinite loop
```
```java
while (true)
    int x = 10;   // COMPILE ERROR: not a statement
```

### The Empty While
```java
while (true);   // legal infinite empty loop (spins CPU, no output) — classic accidental busy-wait bug
```

### Unreachable Statement Detection — The Compiler's Static Flow Analysis
Java's compiler performs **definite/unreachable statement analysis** using **constant folding** on `while`/`for`/`do-while` conditions — but the rules for *what counts as unreachable* differ meaningfully between constant literal `true`/`false`, `final` variables, and plain variables.

| Condition Type | Compiler Behavior |
|---|---|
| Literal `while(true)` with no `break` | Statement AFTER the loop = **unreachable — compile error** |
| Literal `while(false)` | Loop BODY itself = **unreachable — compile error** (loop never runs) |
| `while(a < b)` with plain (non-final) variables | Compiler CANNOT prove it's infinite → statement after loop is reachable, compiles fine (even though at runtime, if `a<b` never changes, it *is* infinite) |
| `while(a < b)` with **both `a` and `b` declared `final`** with constant initializers | Compiler substitutes the constant values and evaluates the expression at compile-time → behaves like literal `true` → statement after is unreachable → **compile error** |

```java
// Case 1: literal true, no break -> statement after loop is unreachable
while (true) {
    System.out.println("hello");
}
System.out.println("hi");   // COMPILE ERROR: unreachable statement
```

```java
// Case 2: literal false -> loop body itself is unreachable
while (false) {
    System.out.println("hello");   // COMPILE ERROR: unreachable statement
}
System.out.println("hi");
```

```java
// Case 3: plain variables -> compiler can't prove infinite-ness -> compiles fine
int a = 10, b = 20;
while (a < b) {
    System.out.println("hello");
}
System.out.println("hi");   // reachable as far as the compiler is concerned; compiles OK
                              // (runtime: infinite loop since a, b never change)
```

```java
// Case 4: BOTH final -> compiler treats a<b as a compile-time constant expression (true)
final int a = 10, b = 20;
while (a < b) {
    System.out.println("hello");
}
System.out.println("hi");   // COMPILE ERROR: unreachable statement
```

```java
// Case 5: only ONE final -> compiler still can't fully constant-fold -> compiles fine
final int a = 10;
while (a < 20) {              // 'a' substituted, but this is still evaluated as a constant expr since literal 20 too
    System.out.println("hello");
}
System.out.println("hi");    // Per JLS this actually IS unreachable too, since a<20 is a constant expression
```

> **Compiler rule (JLS §14.21 restated simply):**
> - If **every operand** in the loop condition is a compile-time constant (`final` with constant initializer, or a literal), the compiler evaluates the expression **at compile time** and applies unreachability rules exactly as if you'd typed `true`/`false` literally.
> - If **at least one operand is a non-final variable**, the compiler defers the check to runtime — it does *not* attempt to prove infinite loops in general (this is provably undecidable — the Halting Problem), so it conservatively assumes the statement after the loop **could** be reached, and compiles without error.

This is a **frequently asked OCPJP/architect trick question**: *"Why does replacing a variable with `final` suddenly cause a compile error two lines later?"* — Answer: because it turns a runtime-only decidable condition into a compile-time-constant one, activating unreachable-code analysis.

---

## 5. Iterative Statements — `do-while`

**When to use:** When the loop body must execute **at least once**, regardless of the condition (classic use: menu-driven console programs, input validation retry loops).

### Syntax
```java
do {
    // ----------
    // ----------
} while (b);   // <-- trailing semicolon is MANDATORY
```

### Optional Braces
Without `{}`, exactly one non-declarative statement is allowed between `do` and `while`.
```java
do
    System.out.println("hello");
while (true);    // infinite "hello"
```

```java
do;
while (true);    // legal — empty statement as body; compiles, infinite empty loop
```

```java
do
    int x = 10;      // COMPILE ERROR — declaration not allowed as lone body statement
while (true);
```

### Nested/Ambiguous-Looking `do-while`
```java
do
    while (true)
        System.out.println("hello");
while (true);
```
This is **legal** — the inner `while(true) System.out.println("hello");` is itself a single (compound) statement acting as the `do` loop's body. Effectively an infinite inner `while` inside an outer `do-while` whose own condition is never even reached at runtime (though it compiles fine, since the inner loop is a plain non-final-variable-free literal loop with no code after it inside the do-block to prove unreachable).

```java
do
while (true);
```
This, however, **fails to compile** — `do` requires *some* statement body followed by the keyword `while`; here the parser sees `do` immediately followed by `while`, which isn't a valid statement in that position — producing `error: while expected` / `illegal start of expression`. (There must be a statement, even if just `;`, between `do` and the terminating `while(...)`.)

### Unreachable Statement Rules for `do-while`
Applies analogously to `while`, based on whether the condition is compile-time constant:

```java
do {
    System.out.println("hello");
} while (true);
System.out.println("hi");   // COMPILE ERROR: unreachable
```

```java
do {
    System.out.println("hello");
} while (false);
System.out.println("hi");   // OK — loop runs once, then reachable. Output: hello \n hi
```

```java
int a = 10, b = 20;
do {
    System.out.println("hello");
} while (a < b);            // plain vars -> not proven infinite by compiler
System.out.println("hi");   // compiles fine (though runtime-infinite)
```

```java
int a = 10, b = 20;
do {
    System.out.println("hello");
} while (a > b);            // false at runtime -> loop executes exactly once
System.out.println("hi");
// Output: hello \n hi
```

```java
final int a = 10, b = 20;
do {
    System.out.println("hello");
} while (a < b);            // both final, compile-time-constant TRUE
System.out.println("hi");   // COMPILE ERROR: unreachable
```

```java
final int a = 10, b = 20;
do {
    System.out.println("hello");
} while (a > b);            // compile-time-constant FALSE
System.out.println("hi");   // OK — compiles and runs: hello, hi
```

**Key architect distinction vs `while`:** In `while(false)`, the **loop body** is unreachable (never runs even once). In `do-while(false)`, the body **is** reachable (guaranteed one execution) — only code *after* an infinite `do-while(true)` becomes unreachable.

---

## 6. Iterative Statements — `for`

**When to use:** Best suited when the **number of iterations is known in advance**.

### Syntax & Execution Order
```java
for (initialization; condition; increment/decrement) {
    body
}
```

Execution sequence (classically diagrammed as ①②③④…):
```
① init  →  ② condition check  →  ③ body  →  ④ increment  →  ⑤ condition check  →  ⑥ body  →  ⑦ increment ...
```
i.e., **init runs once**, then repeatedly: **check → body → increment**, until the condition is false.

### Section 1: Initialization
- Executes **exactly once**, before the loop starts.
- Can declare multiple loop variables, but **all must be of the same type**:
```java
for (int i = 0, j = 0; ...; ...) { }              // valid
for (int i = 0, boolean b = true; ...; ...) { }   // INVALID — mixed types
for (int i = 0, int j = 0; ...; ...) { }          // INVALID — redundant 'int' repeated is a syntax error
```
- Can hold **any valid Java statement**, not just declarations — including method calls like `System.out.println(...)`:
```java
int i = 0;
for (System.out.println("hello u r sleeping"); i < 3; i++) {
    System.out.println("no boss, u only sleeping");
}
```
```
Output:
hello u r sleeping
no boss, u only sleeping
no boss, u only sleeping
no boss, u only sleeping
```
(The init section printed once; the loop iterated 3 times based on `i<3`.)

### Section 2: Conditional Check
- Must be a `boolean` expression.
- **Optional** — if omitted, the compiler treats it as `true` (infinite loop by default).

### Section 3: Increment/Decrement
- Can contain any valid Java statement, including method calls:
```java
int i = 0;
for (System.out.println("hello"); i < 3; System.out.println("hi")) {
    i++;
}
```
```
Output:
hello
hi
hi
hi
```

### All Three Sections Are Independent and Optional
```java
for ( ; ; ) {
    System.out.println("hello");   // infinite loop — completely valid, common idiom
}
```

### Unreachable Statement Analysis in `for`
Same compile-time-constant logic as `while`/`do-while` applies to the **condition section**:

```java
for (int i = 0; true; i++) {
    System.out.println("hello");
}
System.out.println("hi");   // COMPILE ERROR: unreachable
```

```java
for (int i = 0; false; i++) {   // body itself unreachable
    System.out.println("hello");   // COMPILE ERROR: unreachable
}
System.out.println("hi");
```

```java
for (int i = 0; ; i++) {         // empty condition == compiler-inserted 'true'
    System.out.println("hello");
}
System.out.println("hi");        // COMPILE ERROR: unreachable (empty condition treated as literal true)
```

```java
int a = 10, b = 20;
for (int i = 0; a < b; i++) {    // plain variables — compiler can't prove infinite
    System.out.println("hello");
}
System.out.println("hi");        // compiles fine (runtime infinite loop though)
```

```java
final int a = 10, b = 20;
for (int i = 0; a < b; i++) {    // both final -> constant-folds to 'true'
    System.out.println("hello");
}
System.out.println("hi");        // COMPILE ERROR: unreachable
```

---

## 7. Iterative Statements — `for-each` (Enhanced For Loop)

- Introduced in **Java 1.5**.
- Best suited to iterate over the elements of **arrays** and **Collections** when the index/position is not needed.

### Syntax
```java
for (ElementType item : target) {
    // ...
}
```

### Single-Dimensional Array Example
```java
int[] a = {10, 20, 30, 40, 50};

// Normal for loop
for (int i = 0; i < a.length; i++) {
    System.out.println(a[i]);
}

// Enhanced for loop
for (int x : a) {
    System.out.println(x);
}
```
Both print `10 20 30 40 50`.

### Two-Dimensional Array Example
```java
int[][] a = {{10, 20, 30}, {40, 50}};

// Normal for loop
for (int i = 0; i < a.length; i++) {
    for (int j = 0; j < a[i].length; j++) {
        System.out.println(a[i][j]);
    }
}

// Enhanced for loop
for (int[] x : a) {
    for (int y : x) {
        System.out.println(y);
    }
}
```

### Not Every `for` Has an Equivalent `for-each`
```java
for (int i = 0; i < 10; i++) {
    System.out.println("hello");
}
```
This **cannot** be meaningfully rewritten as a `for-each` — there is no target array/collection to iterate; `for-each` is not a general-purpose loop, only an element-retrieval loop.

### Key Limitations of `for-each`
- It is **not a general-purpose loop replacement** — no access to the loop index/counter directly, no ability to modify the underlying array element via the loop variable (the loop variable is a copy for primitives, and for object references, mutating the object is fine but reassigning the reference does not affect the collection/array), and cannot control direction.
- Using normal `for`, you can traverse **left-to-right or right-to-left**. Using `for-each`, you can **only traverse left-to-right** (in the iteration order defined by the `Iterable`'s `iterator()`).
- You typically **cannot safely remove elements** from a `List` while inside a `for-each` (throws `ConcurrentModificationException`) — must use `Iterator.remove()` explicitly instead.

---

## 8. `Iterable` vs `Iterator`

### Requirement
The **target** of a `for-each` loop must be an **`Iterable`** object (or an array — arrays get special-cased support directly in the bytecode/compiler, they don't literally implement `Iterable`).

- An object is "iterable" **iff its class implements `java.lang.Iterable`**.
- `Iterable` was introduced in **Java 1.5** and defines exactly **one method**:
```java
public interface Iterable<T> {
    Iterator<T> iterator();
}
```
- Every `Collection` (`List`, `Set`, `Queue`, etc.) already implements `Iterable` (since `Collection extends Iterable`).

### Difference Table

| Aspect | `Iterable` | `Iterator` |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Introduced | Java 1.5 | Java 1.2 |
| Purpose | Marks a type as "can be iterated"; enables `for-each` | Actually walks the elements one by one |
| Methods | 1 method: `iterator()` | 3 methods: `hasNext()`, `next()`, `remove()` (default `remove()` throws `UnsupportedOperationException` unless overridden) |
| Relationship | `Iterable.iterator()` **returns** an `Iterator` | Obtained *from* an `Iterable` |
| Usage context | `for (T x : iterableObj)` | `while (itr.hasNext()) { itr.next(); }` |

**Architect note (currency beyond the source PDF):** Since Java 8, `Iterable` also has two **default methods**: `forEach(Consumer<? super T> action)` and `spliterator()` — enabling `collection.forEach(System.out::println)` and paving the way for the Streams API's parallel traversal via `Spliterator`. This is a natural follow-up question interviewers ask after "Iterable vs Iterator."

---

## 9. Transfer Statements

### `break` Statement
`break` is legal **only** in three contexts:
1. **Inside `switch`** — to stop fall-through.
2. **Inside loops** (`while`, `do-while`, `for`, `for-each`) — to exit the loop early based on a condition.
3. **Inside labeled blocks** — to jump out of an arbitrary labeled block.

```java
// Inside switch
switch (x) {
    case 0:
        System.out.println("hello");
        break;
    case 1:
        System.out.println("hi");
}
```

```java
// Inside loop
for (int i = 0; i < 10; i++) {
    if (i == 5) break;
    System.out.println(i);
}
// Output: 0 1 2 3 4
```

```java
// Inside labeled block (NOT a loop — just a plain block with a label!)
int x = 10;
l1: {
    System.out.println("begin");
    if (x == 10)
        break l1;              // jumps to just after the l1 block
    System.out.println("end"); // skipped
}
System.out.println("hello");
// Output: begin \n hello
```
> **Architect insight:** A labeled block need not be a loop at all — `break label;` works on *any* labeled statement, including plain `{ }` blocks. This is a lesser-known but legal Java feature, occasionally used to simulate a poor-man's "goto forward" for early-exit logic without deep nesting.

**Anywhere else → compile error:**
```java
int x = 10;
if (x == 10)
    break;   // COMPILE ERROR: break outside switch or loop
```

### `continue` Statement
Skips the **rest of the current iteration** and proceeds to the next iteration. Legal **only inside loops** — using it elsewhere is a compile-time error (`continue outside of loop`).

```java
int x = 2;
for (int i = 0; i < 10; i++) {
    if (i % x == 0)
        continue;
    System.out.println(i);
}
// Output: 1 3 5 7 9
```

```java
if (x == 10)
    continue;   // COMPILE ERROR: continue outside of loop
```

### Labeled `break` / `continue`
In **nested loops**, plain `break`/`continue` only affects the **innermost** enclosing loop. To target an outer loop, use a **label**.

```
l1:
for (...) {
    ...
    l2:
    for (...) {
        ...
        l3:
        for (...) {
            ...
            break l1;      // exits ALL THREE loops
            break l2;      // exits l2 and l3 (the two outer-of-l3 loops)
            break l3;      // exits only l3 (same as plain 'break')
            ...
        }
        ...
    }
    ...
}
```

### Worked Example — plain vs labeled `break`/`continue`
```java
l1:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (i == j)
            break;                 // or: continue; / break l1; / continue l1;
        System.out.println(i + "........." + j);
    }
}
```

| Statement used | Output |
|---|---|
| `break;` (plain) | `1.........0` `2.........0` `2.........1` |
| `break l1;` | *(no output — exits immediately on first i==j match, i.e., i=0,j=0)* |
| `continue;` (plain) | `0.........1` `0.........2` `1.........0` `1.........2` `2.........0` `2.........1` |
| `continue l1;` | `1.........0` `2.........0` `2.........1` |

**How to reason about it (architect mental model):**
- Plain `break`/`continue` → affects the **nearest enclosing loop only** (here, the inner `j` loop).
- `break l1;` → terminates the **entire outer loop structure** immediately, skipping all remaining iterations of both loops.
- `continue l1;` → skips the rest of the **inner loop's current pass** *and* the rest of the **outer loop's current iteration's remaining code**, jumping straight to the outer loop's increment/next-condition-check (`i++` then re-check `i<3`).

---

## 10. The "Do-While + `continue`" Trap — *Most Dangerous Combination*

This is explicitly called out in the source material as **"the most dangerous combination"** because `continue` inside a `do-while` does **not** skip to the top of the loop body (as it appears to) — it skips directly to the **condition check** (`while(...)`), which is at the *bottom*. Since side effects (like increments) placed *after* the `continue` but *before* `while(...)` get skipped on a `continue`, but the condition itself may still contain further side-effecting expressions (like `++x`), this produces highly counter-intuitive results.

```java
class Test {
    public static void main(String[] args) {
        int x = 0;
        do {
            ++x;
            System.out.println(x);
            if (++x < 5)
                continue;
            ++x;
            System.out.println(x);
        } while (++x < 10);
    }
}
```

**Trace it carefully:**

| Iteration | `++x` (line1) | print | `++x<5`? (increments x) | if true → `continue` (skip rest of body) | `++x` line (if reached) | print (if reached) | `++x<10` (loop condition, ALWAYS runs) |
|---|---|---|---|---|---|---|---|
| 1 | x=1 | prints `1` | x=2, 2<5 true → continue | skip to while check | — | — | x=3, 3<10 true → loop again |
| 2 | x=4 | prints `4` | x=5, 5<5 false → don't continue | x=6 | prints `6` | x=7, 7<10 true → loop again |
| 3 | x=8 | prints `8` | x=9, 9<5 false | x=10 | prints `10` | x=11, 11<10 false → loop ends |

**Output:**
```
1
4
6
8
10
```

> **Why this is dangerous in real code:** Developers assume `continue` in a `do-while` re-evaluates the loop "from the top" the way it conceptually does in `while`/`for`. In reality, `continue` in **any** loop jumps to the loop's own update/condition-check step — for `for`, that's the increment expression *then* the condition; for `while`/`do-while`, that's directly the condition. In a `do-while`, since the condition is physically at the *bottom*, any code between the `continue` and the `while(...)` clause is silently skipped, while the condition expression itself (if it has side effects, e.g., `++x < 10`) *still executes*. This is a strong argument, at the architect/code-review level, for **banning side-effecting expressions inside loop conditions** (`++x < 10` should be `x++; ... ; if (x >= 10) break;`-style, or better, avoid mutation-in-condition entirely) — enforceable via static analysis (SonarQube rule "loop conditions should not contain assignments/increments").

---

## 11. Modern Java Enhancements to Flow Control (Beyond the Source Material — Java 5→21)

The source PDF reflects Java 1.4/1.5-era exam content (SCJP/OCJP). For architect-level interviews in 2026, you are expected to also know the following evolutions:

### a) `String` in `switch` (Java 7)
```java
String day = "MON";
switch (day) {
    case "MON": System.out.println("Monday"); break;
    default: System.out.println("Other");
}
```
Internally the compiler desugars this to a `hashCode()`-based `tableswitch` + `equals()` verification (to handle collisions), **not** a direct string comparison at the bytecode level.

### b) Switch Expressions & Arrow Syntax (Java 14, JEP 361)
```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY               -> 7;
    case THURSDAY, SATURDAY    -> 8;
    case WEDNESDAY             -> 9;
};
```
- No fall-through with `->` syntax (each arm is independent).
- Can be used as an **expression** producing a value, assignable directly.
- `yield` keyword returns a value from a `{}` block arm:
```java
int result = switch (x) {
    case 1 -> 10;
    default -> {
        int temp = x * 2;
        yield temp;
    }
};
```
- Exhaustiveness is enforced by the compiler for `enum`/`sealed` switch expressions (no `default` needed if all cases are covered).

### c) Pattern Matching for `switch` (Java 21, JEP 441 — finalized)
```java
static String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "positive int: " + i;
        case Integer i             -> "non-positive int: " + i;
        case String s               -> "string of length " + s.length();
        case null                   -> "it's null";
        default                     -> "something else";
    };
}
```
- `switch` can now match on **type patterns**, use **guarded patterns** (`when` clause), and explicitly handle `case null`.
- Works beautifully with **records** for deconstruction patterns:
```java
record Point(int x, int y) {}

static String locate(Object o) {
    return switch (o) {
        case Point(int x, int y) when x == 0 && y == 0 -> "origin";
        case Point(int x, int y) -> "point at " + x + "," + y;
        default -> "not a point";
    };
}
```

### d) `instanceof` Pattern Matching (Java 16) — related flow-control simplification
```java
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase());
}
```
Eliminates the old cast-after-check idiom, and the compiler tracks *definite assignment/flow scoping* of `s` based on the `if`/`&&` structure — an interesting compiler-flow-analysis topic in its own right.

### e) Enhanced `for` + Streams as an Architectural Alternative
While `for-each` remains foundational, architects should be ready to discuss **when to prefer the Streams API** (`collection.stream().filter(...).forEach(...)`) over imperative loops — for readability, composability, and (with `.parallelStream()`) potential parallel execution, versus the raw performance/predictability/debuggability advantages of classic loops.

---

## 12. Architect-Level Interview Questions

**Q1. Why doesn't `if(x = 5)` compile in Java, unlike C/C++?**
Because assignment expressions in Java evaluate to the type of the assigned value (`int` for `int x`), and `if` requires strictly `boolean`. Java has no implicit int→boolean conversion. This is a deliberate language safety decision to eliminate a whole class of accidental-assignment bugs. `if (b = true)` *does* compile only because `b` is itself `boolean`.

**Q2. Explain "fall-through" in `switch` and one legitimate design use for it.**
Fall-through means execution continues past a matched case into subsequent cases until a `break` or the switch ends. A legitimate use is grouping multiple case labels that should trigger identical behavior (e.g., `case SATURDAY: case SUNDAY:` both leading into the same "weekend" logic) without duplicating code.

**Q3. Why does adding `final` to loop-boundary variables sometimes turn working code into a compile error ("unreachable statement")?**
Because the compiler performs compile-time constant folding on loop conditions. Plain variables make the condition's truth value undecidable at compile time (Halting Problem territory), so the compiler defers to runtime and doesn't flag anything as unreachable. When all operands become `final` with constant initializers, the JLS treats the whole expression as a compile-time constant, and the compiler can statically prove the loop is (or isn't) infinite, activating unreachable-code diagnostics for code after it.

**Q4. What's the actual difference between `Iterable` and `Iterator`, and why does Java split these into two interfaces instead of one?**
`Iterable` (java.lang, 1.5) is a capability marker — "this type can produce an iterator" — with a single factory method `iterator()`. `Iterator` (java.util, 1.2) is the actual cursor/state-holder that performs traversal via `hasNext()`/`next()`/`remove()`. Splitting them allows a single `Iterable` object (e.g., a `List`) to spawn **multiple independent iterators** simultaneously (each with its own traversal position/state), which would be impossible if the collection itself held the single cursor. This also enables the `for-each` sugar to work uniformly across any type without exposing iteration state directly on the collection.

**Q5. Why can't you always rewrite a `for` loop as a `for-each` loop?**
`for-each` is not a general-purpose loop; it strictly retrieves elements from an `Iterable`/array in forward order, exposing no index, no way to iterate right-to-left, no ability to skip/step arbitrarily, and no way to safely mutate the underlying structure mid-iteration (removal throws `ConcurrentModificationException` unless done via `Iterator.remove()`). A `for` loop like `for(int i=0;i<10;i++) System.out.println("hello");` has no target collection/array to iterate over at all — there's nothing to convert.

**Q6. Explain the "do-while + continue" danger with an example, and how would you prevent this bug class in code review?**
`continue` in a `do-while` jumps straight to the trailing `while(condition)` check, skipping any code between the `continue` and that check — but the condition itself still executes (including any side effects like `++x`). Developers often assume `continue` restarts from the top, leading to subtly wrong iteration counts. Mitigation: disallow side-effecting/mutating expressions inside loop conditions (enforced via static analysis), and prefer explicit boolean condition variables evaluated once per iteration with all mutation confined to the loop body.

**Q7. Where exactly can `break` legally appear in Java, and what happens if you use it elsewhere?**
Only inside: (a) a `switch` block (to halt fall-through), (b) any loop (`for`, `while`, `do-while`, `for-each`) to exit early, and (c) a labeled statement/block (`break label;`) to exit that specific block, even a non-loop `{}` block. Anywhere else it's a compile-time error: `break outside switch or loop`.

**Q8. What allowed types can be used as a `switch` argument, and how has that evolved?**
Up to 1.4: `byte`, `short`, `char`, `int`. From 1.5: their wrapper classes (`Byte`, `Short`, `Character`, `Integer` via autoboxing) and `enum`. From 1.7: `String` (desugared to hashCode-based dispatch with equals() verification). `long`, `float`, `double`, `boolean` are never allowed. Since Java 14/21, `switch` can also be used as an **expression** with pattern matching over arbitrary reference types (including `sealed` hierarchies and records), vastly broadening its role beyond simple discrete dispatch.

**Q9. Why must a `switch` case label be a "compile-time constant," and what qualifies a `final` variable as one?**
The compiler must resolve each case label to a fixed value to build the underlying `tableswitch`/`lookupswitch` dispatch table (essentially a jump table) at compile/class-file-generation time — it cannot defer to runtime like an `if-else` chain can. A `final` variable qualifies as a "constant variable" (JLS §4.12.4) only if it's of a primitive type or `String`, and initialized with a constant expression at declaration — e.g., `final int y = 20;` qualifies, but `final int y = someMethod();` does not, and using the latter as a case label still fails to compile.

**Q10. In a labeled nested-loop scenario, explain precisely what `continue outerLabel;` does versus `break outerLabel;`.**
`break outerLabel;` immediately terminates the entire labeled loop (and any loops nested within it), transferring control to the statement immediately following that labeled loop. `continue outerLabel;` does **not** terminate the outer loop — it abandons the remainder of the current iteration of *all* inner loops **and** the remainder of the current iteration's body in the outer loop, then proceeds directly to the outer loop's own increment/re-check step, continuing the outer loop's iteration cycle.

---

## 13. Quick Reference Cheat Sheet

| Construct | Braces Optional? | Condition Type | `break` allowed? | `continue` allowed? | Runs body at least once? |
|---|---|---|---|---|---|
| `if-else` | Yes | `boolean` only | No (unless in enclosing loop/switch) | No | — |
| `switch` | **No (mandatory)** | byte/short/char/int/wrapper/enum/String/expr(14+) | Yes (native use) | No | — |
| `while` | Yes | `boolean` | Yes | Yes | No |
| `do-while` | Yes | `boolean` | Yes | Yes (jumps to trailing condition!) | **Yes** |
| `for` | Yes | `boolean` (default `true` if omitted) | Yes | Yes | No |
| `for-each` | Yes | N/A (`Iterable`/array target) | Yes | Yes | No |

**Unreachable-statement quick rule:**
> Compiler flags unreachable code only when the loop condition is a **compile-time constant expression** (`true`/`false` literal, or all operands `final` with constant initializers). Plain-variable conditions are never flagged, even if truly infinite at runtime.

**Fall-through quick rule:**
> `switch` always falls through on match unless a `break` (or `return`/`throw`/`continue` in a loop context, or reaching the switch's closing `}`) halts it. Arrow-syntax (`->`) switch expressions (Java 14+) do **not** fall through.

**Break/Continue legality quick rule:**
> `break` → switch, loop, or labeled block.
> `continue` → loop only (never switch, never a bare labeled block).

**Case label validity checklist:**
> ✅ Compile-time constant → ✅ Within range of switch argument's type → ✅ No duplicates (after any implicit widening/char-to-int comparison) → ✅ `default` at most once.

---

*Study material compiled and technically validated against JLS-documented behavior (Java SE Language Specification) and verified compiler outputs (javac) referenced from DURGASOFT Core Java Chapter 3: Flow Control, extended with Java 7–21 language evolution for senior architect interview readiness.*
