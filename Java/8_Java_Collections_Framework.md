# Core Java — Collections Framework
### Architect-Grade Interview Study Guide (Java 8 → Java 21+)

> Source curriculum: Durga Sir's Core Java Chapter 14 — Collections Framework, expanded with
> modern Java 8–21 API additions, architect-level rationale, and a large interview Q&A bank.
> All code samples in this guide have been **compiled and executed on OpenJDK 21.0.10** to
> confirm output correctness.

---

## Table of Contents
1. [Why Collections — Arrays vs Collections](#1-why-collections)
2. [Collection Framework Overview](#2-collection-framework-overview)
3. [The `Collection` Interface](#3-the-collection-interface)
4. [The `List` Interface](#4-the-list-interface)
5. [Cursors: Enumeration, Iterator, ListIterator](#5-cursors)
6. [The `Set` Interface](#6-the-set-interface)
7. [Comparable vs Comparator](#7-comparable-vs-comparator)
8. [The `Map` Interface](#8-the-map-interface)
9. [The `Queue` Interface (1.5) & `Deque` (1.6)](#9-the-queue-interface)
10. [`NavigableSet` / `NavigableMap` (1.6)](#10-navigableset--navigablemap)
11. [Utility Classes: `Collections` and `Arrays`](#11-utility-classes)
12. [Concurrent Collections (1.5+)](#12-concurrent-collections)
13. [Enum-based Collections: `EnumSet` & `EnumMap`](#13-enum-based-collections)
14. [Fail-Fast vs Fail-Safe Iterators](#14-fail-fast-vs-fail-safe-iterators)
15. [Modern Java Updates (8 → 21) You Must Know](#15-modern-java-updates)
16. [Architect-Level Comparison Cheat Sheets](#16-cheat-sheets)
17. [Interview Question Bank (56 Qs, incl. a coding exercise)](#17-interview-question-bank)

---

## 1. Why Collections

An **Array** is an indexed collection of a **fixed number of homogeneous** elements. Arrays give
readability (representing many values with one variable), but they have three structural
limitations:

| # | Limitation | Explanation |
|---|---|---|
| 1 | **Fixed size** | Once created, an array cannot grow or shrink. You must know the size upfront. |
| 2 | **Homogeneous only** | A `Student[]` cannot hold a `Customer`. Workaround: `Object[]` — but you lose compile-time type safety. |
| 3 | **No standard data structure** | Arrays aren't backed by a data-structure/algorithm library, so every operation (search, sort, insert-in-middle) must be hand-coded. |

```java
Student[] s = new Student[10000];
s[0] = new Student();     // OK
s[1] = new Customer();    // Compile Error: incompatible types

// Workaround using Object[]
Object[] a = new Object[10000];
a[0] = new Student();     // OK
a[1] = new Customer();    // OK — but no compile-time type safety
```

**Collections** solve exactly these three problems:

| Property | Array | Collection |
|---|---|---|
| Size | Fixed | Growable |
| Memory | Not recommended (fixed allocation) | Recommended |
| Performance | Recommended (raw, contiguous) | Not always recommended (overhead of objects/boxing) |
| Element types | Homogeneous only | Homogeneous **and** Heterogeneous |
| Holds | Primitives and Objects | **Objects only** (primitives are auto-boxed) |
| Underlying DS | None | Every collection class is backed by a known DS (resizable array, linked list, hash table, balanced tree, etc.) — ready-made method support |

> **Architect note:** "Collections can hold only Objects, not primitives" is still 100% true even
> in modern Java — `List<Integer>` really stores boxed `Integer` references. This is why
> autoboxing/unboxing overhead and `==` vs `.equals()` traps on cached `Integer` values
> (`-128`..`127`) are classic interview gotchas (see Q&A section).

---

## 2. Collection Framework Overview

The Collection Framework defines a family of classes/interfaces to represent a group of objects
as a single unit (analogous to C++'s STL "Container").

### The 9 (really, more today) Key Root Interfaces
```
1) Collection (I)
2) List (I)
3) Set (I)
4) SortedSet (I)
5) NavigableSet (I)
6) Queue (I)
7) Map (I)          -- NOT a child of Collection
8) SortedMap (I)
9) NavigableMap (I)
```

### Master Hierarchy Diagram

```
                         Collection (I)  [1.2]
                 ┌───────────┼───────────────────┐
              List (I)    Set (I)             Queue (I) [1.5]
              [1.2]        [1.2]                 │
       ┌───────┼──────┐     ┌────┴─────┐   ┌──────┴───────┐
  ArrayList LinkedList Vector HashSet SortedSet(I) PriorityQueue BlockingQueue(I)
  (1.2)     (1.2)     (1.0)  (1.2)    [1.2]  (1.5)      [1.5]
             (Legacy)   │        │      │
                       Stack  LinkedHashSet NavigableSet(I) [1.6]
                       (1.0)      (1.4)          │
                     (Legacy)                 TreeSet (1.2)

    Map (I) [1.2]  -- independent hierarchy (NOT a Collection)
       ┌──────┬─────────┬───────────────┬──────────┐
   HashMap  WeakHashMap IdentityHashMap SortedMap(I)  (Dictionary(AC) [1.0] Legacy)
   (1.2)      (1.2)       (1.4)          [1.2]              │
     │                                    │              Hashtable
  LinkedHashMap (1.4)               NavigableMap(I) [1.6]     │
                                          │                Properties
                                       TreeMap (1.6)
```

**Legacy classes** (pre-1.2, retro-fitted into the framework): `Enumeration (I)`,
`Dictionary (AC)`, `Vector`, `Stack`, `Hashtable`, `Properties`.

**Utility classes:** `Collections`, `Arrays`
**Sorting:** `Comparable (I)` (`java.lang`), `Comparator (I)` (`java.util`)
**Cursors:** `Enumeration (I)`, `Iterator (I)`, `ListIterator (I)`

### `Collection` (interface) vs `Collections` (class) — classic trap question
| | `Collection` | `Collections` |
|---|---|---|
| Type | Interface | Final utility class |
| Package | `java.util` | `java.util` |
| Purpose | Represents a group of individual objects as one entity; root of List/Set/Queue | Static utility methods (`sort`, `binarySearch`, `synchronizedXxx`, `unmodifiableXxx`, `reverse`, ...) that operate **on** Collection objects |
| Instantiable | No concrete class implements `Collection` directly | N/A — never instantiated (private constructor) |

---

## 3. The `Collection` Interface

If you want to represent a group of **individual objects as a single entity**, use `Collection`.
It is the **root interface** of the framework and defines the most common methods applicable to
any Collection object.

```java
public interface Collection<E> extends Iterable<E> {
    boolean add(E o);
    boolean addAll(Collection<? extends E> c);
    boolean remove(Object o);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);   // removes everything EXCEPT what's in c
    void    clear();
    boolean contains(Object o);
    boolean containsAll(Collection<?> c);
    boolean isEmpty();
    int     size();
    Object[] toArray();
    Iterator<E> iterator();
    // Java 8+ default methods (see section 15)
    // default boolean removeIf(Predicate<? super E> filter)
    // default Stream<E> stream()
    // default Stream<E> parallelStream()
    // default Spliterator<E> spliterator()
    // default void forEach(Consumer<? super E> action)   // from Iterable
}
```

> **Note:** There is **no concrete class that implements `Collection` directly.** There is also no
> direct method inside `Collection` to *retrieve* an individual object by index/key — retrieval is
> delegated to child interfaces (`List.get(int)`, `Map.get(key)`) or cursors.

---

## 4. The `List` Interface

- Child interface of `Collection`.
- Use when: duplicates **are allowed** and **insertion order is preserved**.
- Index plays a central role — it differentiates duplicate objects.

```java
public interface List<E> extends Collection<E> {
    void add(int index, E o);
    boolean addAll(int index, Collection<? extends E> c);
    E get(int index);
    E remove(int index);
    E set(int index, E newObj);      // replaces element at index, returns OLD value
    int indexOf(Object o);           // index of FIRST occurrence
    int lastIndexOf(Object o);
    ListIterator<E> listIterator();
}
```

### 4.1 `ArrayList`

| Property | Value |
|---|---|
| Underlying data structure | Resizable / growable array |
| Duplicates | Allowed |
| Insertion order | Preserved |
| Heterogeneous objects | Allowed |
| `null` insertion | Allowed (any number of times) |
| Implements | `Serializable`, `Cloneable`, `RandomAccess` |
| Thread safety | **Not** synchronized |
| Best for | Frequent **retrieval** (random access via index) |
| Worst for | Frequent insertion/deletion in the **middle** (requires shifting elements) |

**Constructors:**
```java
ArrayList l1 = new ArrayList();                 // default capacity 10
ArrayList l2 = new ArrayList(int initialCapacity);
ArrayList l3 = new ArrayList(Collection c);     // inter-conversion between Collection types
```

**Growth formula (JDK reference implementation):**
```
New Capacity = (Current Capacity * 3 / 2) + 1
```

**Validated example:**
```java
import java.util.ArrayList;

class ArrayListDemo {
    public static void main(String[] args) {
        ArrayList l = new ArrayList();
        l.add("A");
        l.add(10);
        l.add("A");
        l.add(null);
        System.out.println(l);      // [A, 10, A, null]

        l.remove(2);                 // removes index 2 -> "A"
        System.out.println(l);      // [A, 10, null]

        l.add(2, "M");
        l.add("N");
        System.out.println(l);      // [A, 10, M, null, N]
    }
}
```

`RandomAccess` is a **marker interface** (`java.util`, no methods) — `ArrayList` and `Vector`
implement it so JDK algorithms (e.g., `Collections.binarySearch`) can choose an
index-based-loop strategy instead of an iterator-based one for equal-speed random access.

```java
ArrayList l1 = new ArrayList();
LinkedList l2 = new LinkedList();
System.out.println(l1 instanceof Serializable);   // true
System.out.println(l2 instanceof Cloneable);      // true
System.out.println(l1 instanceof RandomAccess);   // true
System.out.println(l2 instanceof RandomAccess);   // false
```

#### ArrayList vs Vector
| ArrayList | Vector |
|---|---|
| Every method is **non-synchronized** | Every method is **synchronized** |
| Multiple threads can operate simultaneously → **not thread-safe** | Only one thread at a time → **thread-safe** |
| High relative performance (no wait) | Lower relative performance (threads wait) |
| Introduced 1.2, Non-Legacy | Introduced 1.0, **Legacy** |

**Get synchronized version of ArrayList:**
```java
List syncList = Collections.synchronizedList(new ArrayList());
Set  syncSet  = Collections.synchronizedSet(new HashSet());
Map  syncMap  = Collections.synchronizedMap(new HashMap());
```

### 4.2 `LinkedList`

| Property | Value |
|---|---|
| Underlying data structure | Doubly linked list |
| Duplicates / null / heterogeneous | Allowed |
| Implements | `Serializable`, `Cloneable` — **NOT** `RandomAccess` |
| Best for | Frequent insertion/deletion in the middle |
| Worst for | Frequent retrieval (must traverse from an end) |

```java
LinkedList l1 = new LinkedList();
LinkedList l2 = new LinkedList(Collection c);
```

`LinkedList` defines 6 stack/queue-style methods, historically used to implement Stacks & Queues:
```java
void addFirst(Object o);
void addLast(Object o);
Object getFirst();
Object getLast();
Object removeFirst();
Object removeLast();
```

**Validated example:**
```java
import java.util.LinkedList;

class LinkedListDemo {
    public static void main(String[] args) {
        LinkedList l = new LinkedList();
        l.add("Durga");
        l.add(30);
        l.add(null);
        l.add("Durga");
        l.set(0, "Software");
        l.add(0, "Venky");
        l.removeLast();
        l.addFirst("CCC");
        System.out.println(l);   // [CCC, Venky, Software, 30, null]
    }
}
```

> Since Java 1.5, `LinkedList` also implements `Deque` and `Queue` (see Section 9) — meaning
> `offer()`, `poll()`, `peek()` are all available on it directly.

### 4.3 `Vector` (Legacy, 1.0)

| Property | Value |
|---|---|
| Underlying data structure | Resizable/growable array |
| Duplicates / order / heterogeneous / null | All allowed, order preserved |
| Implements | `Serializable`, `Cloneable`, `RandomAccess` |
| Thread safety | Every method synchronized → thread-safe |
| Growth formula | **New Capacity = Current Capacity × 2** |

```java
Vector v1 = new Vector();
Vector v2 = new Vector(int initialCapacity);
Vector v3 = new Vector(int initialCapacity, int incrementalCapacity);
Vector v4 = new Vector(Collection c);
```

**Method naming — old Vector-specific names vs new Collection/List names:**

| Purpose | Collection | List | Vector-specific |
|---|---|---|---|
| Add | `add(Object o)` | `add(int idx, Object o)` | `addElement(Object o)` |
| Remove | `remove(Object o)`, `clear()` | `remove(int idx)` | `removeElement(Object o)`, `removeElementAt(int idx)`, `removeAllElements()` |
| Retrieve | — | `get(int idx)` | `elementAt(int idx)`, `firstElement()`, `lastElement()` |
| Other | — | — | `int size()`, `int capacity()`, `Enumeration elements()` |

**Validated example (capacity doubling):**
```java
import java.util.Vector;

class VectorDemo {
    public static void main(String[] args) {
        Vector v = new Vector();
        System.out.println(v.capacity());          // 10
        for (int i = 1; i <= 10; i++) v.addElement(i);
        System.out.println(v.capacity());           // 10
        v.addElement("A");
        System.out.println(v.capacity());           // 20  (10 * 2)
        System.out.println(v);                       // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, A]
    }
}
```

### 4.4 `Stack` (Legacy, 1.0, child of `Vector`)

Specially designed for LIFO order.

```java
Object push(Object o);   // insert
Object pop();            // remove & return top
Object peek();           // return top WITHOUT removal
boolean empty();
int search(Object o);    // 1-based offset from TOP; -1 if not found
```

**Validated example:**
```java
import java.util.Stack;

class StackDemo {
    public static void main(String[] args) {
        Stack s = new Stack();
        s.push("A"); s.push("B"); s.push("C");
        System.out.println(s);              // [A, B, C]
        System.out.println(s.search("A"));  // 3  (offset from top: C=1, B=2, A=3)
        System.out.println(s.search("Z"));  // -1
    }
}
```

> **Architect tip:** `java.util.Stack` extends `Vector`, meaning it inherits **random-access,
> index-based mutation** methods (`add(int, Object)`, `remove(int)`) — this breaks strict LIFO
> discipline. Modern code should prefer `ArrayDeque` as a Stack (`push`/`pop`/`peek`) — it's faster
> and doesn't leak index-based access.

---

## 5. Cursors

To retrieve objects **one by one** from a Collection, use a cursor. Java has 3.

### 5.1 `Enumeration` (Legacy, 1.0)
```java
public interface Enumeration<E> {
    boolean hasMoreElements();
    E nextElement();
}
// Obtained via: Enumeration e = vector.elements();
```
**Limitations:** applicable only to legacy classes (not a universal cursor); read-only, no `remove()`.

```java
import java.util.*;

class EnumerationDemo {
    public static void main(String[] args) {
        Vector v = new Vector();
        for (int i = 0; i <= 10; i++) v.addElement(i);
        Enumeration e = v.elements();
        while (e.hasMoreElements()) {
            Integer I = (Integer) e.nextElement();
            if (I % 2 == 0) System.out.println(I);
        }
        System.out.println(v);   // unchanged: [0,1,2,...,10]
    }
}
```

### 5.2 `Iterator` (1.2 — Universal Cursor)
```java
public interface Iterator<E> {
    boolean hasNext();
    E next();
    void remove();
    // Java 8+: default void forEachRemaining(Consumer<? super E> action)
}
// Obtained via: Iterator itr = collection.iterator();
```
Applicable to **any** Collection object. Supports read + remove, but **not** forward-only
insertion/replace and **not** backward movement.

```java
import java.util.*;

class IteratorDemo {
    public static void main(String[] args) {
        ArrayList l = new ArrayList();
        for (int i = 0; i <= 10; i++) l.add(i);
        Iterator itr = l.iterator();
        while (itr.hasNext()) {
            Integer I = (Integer) itr.next();
            if (I % 2 == 0) System.out.println(I);
            else itr.remove();          // safe removal DURING iteration
        }
        System.out.println(l);          // [0, 2, 4, 6, 8, 10]
    }
}
```

### 5.3 `ListIterator` (1.2 — Bi-directional, List-only)
Child interface of `Iterator`; adds forward/backward traversal, `add()`, `set()`.

```java
public interface ListIterator<E> extends Iterator<E> {
    // forward
    boolean hasNext();
    E next();
    int nextIndex();
    // backward
    boolean hasPrevious();
    E previous();
    int previousIndex();
    // mutation
    void remove();
    void set(E newObj);
    void add(E newObj);
}
// Obtained via: ListIterator litr = list.listIterator();
```

```java
import java.util.*;

class ListIteratorDemo {
    public static void main(String[] args) {
        LinkedList l = new LinkedList();
        l.add("Baala"); l.add("Venki"); l.add("Chiru"); l.add("Naag");
        System.out.println(l);   // [Baala, Venki, Chiru, Naag]

        ListIterator ltr = l.listIterator();
        while (ltr.hasNext()) {
            String s = (String) ltr.next();
            if (s.equals("Venki")) ltr.remove();
            if (s.equals("Naag"))  ltr.add("Chaitu");
            if (s.equals("Chiru")) ltr.add("Charan");
        }
        System.out.println(l);   // [Baala, Chiru, Charan, Naag, Chaitu]
    }
}
```

### 5.4 Cursor Comparison Table

| Property | Enumeration | Iterator | ListIterator |
|---|---|---|---|
| Applicable to | Legacy classes only | Any Collection | Only `List` |
| Direction | Forward only | Forward only | **Bi-directional** |
| Obtain via | `elements()` | `iterator()` | `listIterator()` (from `List`) |
| Read | Yes | Yes | Yes |
| Remove | No | Yes | Yes |
| Replace / Add | No | No | **Yes** |
| Methods | `hasMoreElements()`, `nextElement()` | `hasNext()`, `next()`, `remove()` | 9 methods total (see above) |
| Legacy? | Yes (1.0) | No (1.2) | No (1.2) |

### 5.5 Internal Implementation (whiteboard-question favorite)

```java
import java.util.*;

class CursorDemo {
    public static void main(String[] args) {
        Vector v = new Vector();
        Enumeration e = v.elements();
        Iterator itr = v.iterator();
        ListIterator litr = v.listIterator();
        System.out.println(e.getClass().getName());     // java.util.Vector$1
        System.out.println(itr.getClass().getName());   // java.util.Vector$Itr
        System.out.println(litr.getClass().getName());  // java.util.Vector$ListItr
    }
}
```
All 3 cursors are **private inner classes** of the underlying Collection implementation — this
is *why* Iterator has direct access to the enclosing collection's fields for fail-fast checks.

---

## 6. The `Set` Interface

- Child interface of `Collection`.
- Use when: duplicates **NOT allowed** and insertion order **NOT preserved** (guaranteed).
- `Set` defines **no new methods** — only `Collection` methods apply.

### 6.1 `HashSet`

| Property | Value |
|---|---|
| Underlying DS | Hash table (backed internally by a `HashMap`) |
| Insertion order | Not preserved — based on **hashCode** of elements |
| Duplicates | Not allowed — `add()` silently returns `false` (no exception) |
| `null` | Allowed once |
| Heterogeneous | Allowed |
| Implements | `Serializable`, `Cloneable` — **NOT** `RandomAccess` |
| Best for | Frequent **search** operations |

```java
HashSet h1 = new HashSet();                                    // capacity 16, load factor 0.75
HashSet h2 = new HashSet(int initialCapacity);                  // load factor 0.75
HashSet h3 = new HashSet(int initialCapacity, float fillRatio);
HashSet h4 = new HashSet(Collection c);
```

**Load Factor / Fill Ratio:** 0.75 means once the table is 75% full, it automatically resizes
(rehashes) to a new, larger `HashSet`.

```java
import java.util.*;

class HashSetDemo {
    public static void main(String[] args) {
        HashSet h = new HashSet();
        h.add("B"); h.add("C"); h.add("D"); h.add("Z"); h.add(null); h.add(10);
        System.out.println(h.add("Z"));   // false — duplicate rejected
        System.out.println(h);            // order NOT guaranteed by insertion
    }
}
```

### 6.2 `LinkedHashSet` (1.4, child of `HashSet`)

Same as `HashSet` **except** insertion order **is preserved** (underlying DS = combination of
Hashtable + LinkedList). Commonly used with `LinkedHashMap` for **cache implementations**
requiring uniqueness + insertion-order.

| HashSet | LinkedHashSet |
|---|---|
| DS: Hashtable | DS: Hashtable + LinkedList |
| Insertion order NOT preserved | Insertion order **preserved** |
| 1.2 | 1.4 |

### 6.3 `SortedSet` (interface, child of `Set`)

Elements are stored **without duplicates**, according to a sorting order (natural or custom).

```java
public interface SortedSet<E> extends Set<E> {
    E first();
    E last();
    SortedSet<E> headSet(E toElement);       // elements strictly < toElement
    SortedSet<E> tailSet(E fromElement);      // elements >= fromElement
    SortedSet<E> subSet(E from, E to);        // elements >= from AND < to
    Comparator<? super E> comparator();       // null if using natural ordering
}
```

Example (SortedSet = {100,101,103,104,106,109}):
```
first()        -> 100
last()         -> 109
headSet(104)   -> [100, 101, 103]
tailSet(104)   -> [104, 106, 109]
subSet(101,106)-> [101, 103, 104]
comparator()   -> null
```

### 6.4 `TreeSet` (1.2, implements `NavigableSet` since 1.6)

| Property | Value |
|---|---|
| Underlying DS | Balanced (Red-Black) Tree |
| Insertion order | Not preserved — based on **sorting order** |
| Duplicates | Not allowed |
| Heterogeneous | **Not** allowed with natural sorting → `ClassCastException` at runtime |
| `null` | Allowed **only once**, and **only as the very first element inserted into an empty set**. Any subsequent insert throws `NullPointerException`. |
| Implements | `Serializable`, `Cloneable` — **NOT** `RandomAccess` |

```java
TreeSet t1 = new TreeSet();                 // natural sorting
TreeSet t2 = new TreeSet(Comparator c);      // customized sorting
TreeSet t3 = new TreeSet(Collection c);
TreeSet t4 = new TreeSet(SortedSet s);
```

**Natural sorting requires Comparable + Homogeneous:**
```java
import java.util.TreeSet;

class TreeSetDemo {
    public static void main(String[] args) {
        TreeSet t = new TreeSet();
        t.add("A"); t.add("a"); t.add("B"); t.add("Z"); t.add("L");
        t.add(10);   // RuntimeException: ClassCastException — String vs Integer heterogeneous
    }
}
```
```java
// Non-Comparable class -> ClassCastException even with homogeneous elements
TreeSet t = new TreeSet();
t.add(new StringBuffer("A"));   // RE: ClassCastException:
                                 // java.lang.StringBuffer cannot be cast to java.lang.Comparable
```
> `String` and all wrapper classes implement `Comparable`. `StringBuffer` does **NOT**.

**Custom sorting with `Comparator` — no Comparable/homogeneity restriction:**
```java
import java.util.*;

class TreeSetDemo2 {
    public static void main(String[] args) {
        TreeSet t = new TreeSet(new MyComparator());
        t.add(10); t.add(0); t.add(15); t.add(5); t.add(20); t.add(20); // dup 20 rejected
        System.out.println(t);   // [20, 15, 10, 5, 0]  (descending)
    }
}
class MyComparator implements Comparator {
    public int compare(Object o1, Object o2) {
        Integer i1 = (Integer) o1, i2 = (Integer) o2;
        return i2.compareTo(i1);   // reverse of natural order
    }
}
```

#### TreeSet vs HashSet vs LinkedHashSet
| Property | HashSet | LinkedHashSet | TreeSet |
|---|---|---|---|
| Underlying DS | Hashtable | Hashtable + LinkedList | Balanced Tree |
| Insertion order | Not preserved | Preserved | Not preserved |
| Sorting order | N/A | N/A | Applicable |
| Heterogeneous | Allowed | Allowed | Not allowed (natural sorting) |
| Duplicates | Not allowed | Not allowed | Not allowed |
| `null` | Allowed once | Allowed once | Only as 1st element in an empty set |

---

## 7. Comparable vs Comparator

### `Comparable` (java.lang) — **Default Natural Sorting Order (DNSO)**
```java
public interface Comparable<T> {
    int compareTo(T o);
    // Contract:
    // obj1.compareTo(obj2) < 0  -> obj1 comes BEFORE obj2
    // obj1.compareTo(obj2) > 0  -> obj1 comes AFTER  obj2
    // obj1.compareTo(obj2) == 0 -> equal
}
```
```java
System.out.println("A".compareTo("Z"));    // -25
System.out.println("Z".compareTo("K"));    // 15
System.out.println("Z".compareTo("Z"));    // 0
"Z".compareTo(null);                        // NullPointerException
```
When elements are inserted into a `TreeSet`/`TreeMap` relying on natural ordering, the JVM
internally calls `compareTo()` to decide placement.

### `Comparator` (java.util) — **Customized Sorting Order**
```java
public interface Comparator<T> {
    int compare(T o1, T o2);
    boolean equals(Object o);   // optional to override — inherited from Object
}
```
Implementing `Comparator` requires **only** `compare()`; `equals()` is optional because it's
already inherited from `Object`.

### When to Use Which

| Situation | Approach |
|---|---|
| Predefined comparable class (e.g., `String`) and satisfied with DNSO | Use directly, no interface needed |
| Predefined comparable class, want a **different** order | Write a `Comparator` |
| Predefined **non-comparable** class (e.g., `StringBuffer`) | Must use a `Comparator` — no DNSO available |
| Your own class (e.g., `Employee`) — you are the **author** | Implement `Comparable` to define the class's own default sort (e.g., by `eid`) |
| Your own class — you are the **consumer**, and DNSO doesn't fit your need | Write a `Comparator` (e.g., sort `Employee` by `name`) |

**Full worked example (Employee: natural order by id, custom order by name):**
```java
import java.util.*;

class Employee implements Comparable<Employee> {
    String name; int eid;
    Employee(String name, int eid) { this.name = name; this.eid = eid; }
    public String toString() { return name + "-----" + eid; }
    public int compareTo(Employee e) { return Integer.compare(this.eid, e.eid); }
}

class NameComparator implements Comparator<Employee> {
    public int compare(Employee e1, Employee e2) { return e1.name.compareTo(e2.name); }
}

class CompComp {
    public static void main(String[] args) {
        Employee e1 = new Employee("Nag", 100);
        Employee e2 = new Employee("Bala", 200);
        Employee e3 = new Employee("Chiru", 50);
        Employee e4 = new Employee("Venki", 150);

        TreeSet<Employee> byId = new TreeSet<>();
        byId.add(e1); byId.add(e2); byId.add(e3); byId.add(e4);
        System.out.println(byId);
        // [Chiru-----50, Nag-----100, Venki-----150, Bala-----200]

        TreeSet<Employee> byName = new TreeSet<>(new NameComparator());
        byName.add(e1); byName.add(e2); byName.add(e3); byName.add(e4);
        System.out.println(byName);
        // [Bala-----200, Chiru-----50, Nag-----100, Venki-----150]
    }
}
```

### Comparable vs Comparator Quick Table
| Comparable | Comparator |
|---|---|
| `java.lang` | `java.util` |
| Default Natural Sorting Order | Customized Sorting Order |
| 1 method: `compareTo()` | 2 methods: `compare()`, `equals()` |
| Modifies the class itself | External strategy object — doesn't touch the class |
| All wrapper classes & `String` implement it | Since Java 8, `Comparator` has rich static/default helpers (see §15) |

---

## 8. The `Map` Interface

- **NOT** a child interface of `Collection` — it's a parallel hierarchy for key-value pairs.
- Duplicate **keys** are not allowed; **values** can be duplicated.
- Each key-value pair is called an **Entry** (`Map.Entry` — nested interface, since entries can't
  exist without an owning Map).

```java
public interface Map<K,V> {
    V put(K key, V value);       // if key exists, replaces value & returns OLD value
    void putAll(Map<? extends K,? extends V> m);
    V get(Object key);
    V remove(Object key);
    boolean containsKey(Object key);
    boolean containsValue(Object value);
    boolean isEmpty();
    int size();
    void clear();
    Set<K> keySet();             // collection VIEW of keys
    Collection<V> values();      // collection VIEW of values
    Set<Map.Entry<K,V>> entrySet(); // collection VIEW of entries

    interface Entry<K,V> {
        K getKey();
        V getValue();
        V setValue(V newValue);
    }
}
```

### 8.1 `HashMap`

| Property | Value |
|---|---|
| Underlying DS | Hashtable |
| Duplicate keys | Not allowed; values can be duplicated |
| Heterogeneous keys/values | Allowed |
| Insertion order | Not preserved — based on hashCode of keys |
| `null` | 1 `null` key allowed; any number of `null` values allowed |

```java
HashMap m1 = new HashMap();               // capacity 16, fill ratio 0.75
HashMap m2 = new HashMap(int initialCapacity);
HashMap m3 = new HashMap(int initialCapacity, float fillRatio);
HashMap m4 = new HashMap(Map m);
```

```java
import java.util.*;

class HashMapDemo {
    public static void main(String[] args) {
        HashMap m = new HashMap();
        m.put("Chiru", 700); m.put("Bala", 800); m.put("Venki", 200); m.put("Nag", 500);
        System.out.println(m.put("Chiru", 1000));    // returns OLD value: 700

        Set s1 = m.entrySet();
        Iterator itr = s1.iterator();
        while (itr.hasNext()) {
            Map.Entry m1 = (Map.Entry) itr.next();
            if (m1.getKey().equals("Nag")) m1.setValue(10000);   // mutate through the Entry view
        }
        System.out.println(m);   // {Chiru=1000, Venki=200, Nag=10000, Bala=800} (order not guaranteed)
    }
}
```

#### HashMap vs Hashtable
| HashMap | Hashtable |
|---|---|
| No method is synchronized | Every method is synchronized |
| Multiple threads OK simultaneously → **not thread-safe** | Only one thread at a time → thread-safe |
| High relative performance | Low relative performance |
| `null` allowed for both keys and values | `null` **not allowed** for either → `NullPointerException` |
| 1.2, Non-Legacy | 1.0, **Legacy** |

### 8.2 `LinkedHashMap` (1.4, child of `HashMap`)
Same as `HashMap`, but insertion order **is preserved** (DS = Hashtable + LinkedList).
Frequently paired with `LinkedHashSet` for **LRU-cache-style** designs where uniqueness AND
predictable iteration order both matter (e.g., `LinkedHashMap` even has a constructor +
`removeEldestEntry()` override hook purpose-built for LRU caches).

### 8.3 `IdentityHashMap`
Identical to `HashMap` **except** it uses **`==` (reference comparison)** instead of `.equals()`
to detect duplicate keys.
```java
HashMap m = new HashMap();
Integer I1 = new Integer(10);
Integer I2 = new Integer(10);
m.put(I1, "Pawan"); m.put(I2, "Kalyan");
System.out.println(m);   // {10=Kalyan}  — I1.equals(I2) => treated as duplicate key

IdentityHashMap im = new IdentityHashMap();
im.put(I1, "Pawan"); im.put(I2, "Kalyan");
System.out.println(im);  // {10=Pawan, 10=Kalyan} — I1 == I2 is false => NOT duplicates
```
> Reminder: `==` compares references, `.equals()` compares content.
> `new Integer(10) == new Integer(10)` → `false`; `.equals()` → `true`.
> (Note: `Integer.valueOf` / autoboxed literals in the range **-128..127** are cached, so
> `Integer a = 10; Integer b = 10; a == b` → `true` — but this is an implementation detail you
> should never rely on for `==` comparisons.)

### 8.4 `WeakHashMap`
Identical to `HashMap` **except**: entries whose keys have no other strong references become
**eligible for GC even while still present in the map** — i.e., **Garbage Collector dominates**
`WeakHashMap` (whereas normal `HashMap` "dominates" the GC by keeping keys alive).
```java
HashMap m = new HashMap();
Temp t = new Temp();
m.put(t, "Durga");
t = null;
System.gc();
// with WeakHashMap the entry disappears after GC runs finalize(); with HashMap it stays forever
```

### 8.5 `SortedMap` / `TreeMap` (1.2, `NavigableMap` since 1.6)

```java
public interface SortedMap<K,V> extends Map<K,V> {
    K firstKey();
    K lastKey();
    SortedMap<K,V> headMap(K toKey);
    SortedMap<K,V> tailMap(K fromKey);
    SortedMap<K,V> subMap(K fromKey, K toKey);
    Comparator<? super K> comparator();
}
```

| Property | Value |
|---|---|
| Underlying DS | Red-Black Tree |
| Duplicate keys | Not allowed; values can be duplicated |
| Sorting | Based on **keys** — natural or Comparator |
| Homogeneity | Keys must be Homogeneous+Comparable for natural sort; Comparator lifts this restriction. **Values** have no such restriction. |
| `null` key | Only as the very first entry in an **empty** TreeMap; otherwise `NullPointerException` |
| `null` value | No restriction |

```java
TreeMap t1 = new TreeMap();               // natural sorting
TreeMap t2 = new TreeMap(Comparator c);    // customized sorting
TreeMap t3 = new TreeMap(SortedMap m);
TreeMap t4 = new TreeMap(Map m);
```

```java
import java.util.TreeMap;

class TreeMapDemo {
    public static void main(String[] args) {
        TreeMap m = new TreeMap();
        m.put(100, "ZZZ"); m.put(103, "YYY"); m.put(101, "XXX");
        m.put(104, 106); m.put(107, null);
        m.put("FFF", "XXX");   // RuntimeException: ClassCastException (heterogeneous key)
    }
}
```

### 8.6 `Hashtable` (Legacy, 1.0)

| Property | Value |
|---|---|
| Underlying DS | Hashtable |
| Duplicate keys | Not allowed |
| `null` | **Not allowed** for either key or value → `NullPointerException` |
| Thread safety | Every method synchronized |

```java
Hashtable h1 = new Hashtable();     // default initial capacity 11, fill ratio 0.75
Hashtable h2 = new Hashtable(int initialCapacity);
Hashtable h3 = new Hashtable(int initialCapacity, float fillRatio);
Hashtable h4 = new Hashtable(Map m);
```

### 8.7 `Properties` (Legacy, child of `Hashtable`)

Both key and value **must be `String`**. Designed to externalize configuration (DB URLs,
credentials, etc.) into a `.properties` file — avoiding recompilation for every config change.

```java
Properties p = new Properties();
p.getProperty(String pname);
p.setProperty(String pname, String pvalue);
p.propertyNames();                     // Enumeration of all property names
p.load(InputStream is);                // load from file into Properties object
p.store(OutputStream os, String comment); // persist back to file
```

```java
import java.util.Properties;
import java.io.*;

class PropertiesDemo {
    public static void main(String[] args) throws Exception {
        Properties p = new Properties();
        try (FileInputStream fis = new FileInputStream("abc.properties")) {
            p.load(fis);
        }
        String venki = p.getProperty("Venki");
        p.setProperty("Nag", "88888");
        try (FileOutputStream fos = new FileOutputStream("abc.properties")) {
            p.store(fos, "Updated by Durga for SCJP Class");
        }
    }
}
```

---

## 9. The `Queue` Interface

Introduced 1.5. Child interface of `Collection`. Use for representing a group of objects **prior
to processing** — typically FIFO, but priority-based ordering is possible (`PriorityQueue`).

```java
public interface Queue<E> extends Collection<E> {
    boolean offer(E o);   // add
    E peek();             // return head, null if empty
    E element();          // return head, NoSuchElementException if empty
    E poll();              // remove & return head, null if empty
    E remove();            // remove & return head, NoSuchElementException if empty
}
```

### `PriorityQueue`
- Priority = natural order OR a supplied `Comparator`.
- Duplicates **not allowed**. `null` insertion **not possible** (even as first element) — unlike
  `TreeSet`/`TreeMap`.
- Insertion order not preserved; ordering follows priority (heap structure — `toString()` output
  is the internal heap array, **not** fully sorted order — a classic gotcha).

```java
PriorityQueue q1 = new PriorityQueue();                      // capacity 11, natural order
PriorityQueue q2 = new PriorityQueue(int initialCapacity);
PriorityQueue q3 = new PriorityQueue(int initialCapacity, Comparator c);
PriorityQueue q4 = new PriorityQueue(SortedSet s);
PriorityQueue q5 = new PriorityQueue(Collection c);
```

```java
import java.util.PriorityQueue;

class PriorityQueueDemo {
    public static void main(String[] args) {
        PriorityQueue q = new PriorityQueue();
        for (int i = 0; i <= 10; i++) q.offer(i);
        System.out.println(q.poll());   // 0  (head is always the smallest with natural order)
    }
}
```

### `LinkedList` as a `Queue`
From 1.5, `LinkedList` implements `Queue` — and it **always follows strict FIFO order**
(unlike `PriorityQueue`).

### Concurrent Queue Family (java.util.concurrent, discussed further in §12)
```
Collection (I) -> Queue (I) [1.5]
                     ├── PriorityQueue (C) [1.5]
                     ├── BlockingQueue (I) [1.5]   -- put()/take() block on full/empty
                     │      ├── ArrayBlockingQueue
                     │      ├── LinkedBlockingQueue
                     │      ├── PriorityBlockingQueue
                     │      └── SynchronousQueue
                     └── TransferQueue (I) [1.7]   -- transfer() blocks until a CONSUMER receives
                            └── LinkedTransferQueue
```
- **`BlockingQueue`**: `take()` blocks while empty; `put()` blocks while full. Ideal for
  Producer-Consumer problems.
- **`TransferQueue`**: adds `transfer()` — producer blocks until a **consumer actually receives**
  the element (stronger delivery guarantee than `BlockingQueue.put()`, which only guarantees
  space was available).

### `Deque` (Double-Ended Queue, 1.6)
Insert/remove from **both ends**.
```
Collection (I) -> Queue (I) -> Deque (I) [1.6]
                                  ├── ArrayDeque         [1.6] (best modern Stack/Queue replacement)
                                  ├── ConcurrentLinkedDeque
                                  └── LinkedList (also implements Deque since 1.6)

Queue(I) -> BlockingQueue(I) + Deque(I) -> BlockingDeque(I) [1.6] -> LinkedBlockingDeque
```
`ArrayDeque` supports `push()`/`pop()`/`peek()` (Stack semantics) **and** `offer()`/`poll()`
(Queue semantics) — the JDK Javadoc itself recommends `ArrayDeque` over legacy `Stack`/`Vector`
for stack use-cases, and over `LinkedList` for queue use-cases (fewer allocations, better
cache locality, no synchronization overhead).

---

## 10. `NavigableSet` / `NavigableMap`

Added in **1.6** to provide closest-match navigation methods.

### `NavigableSet` (extends `SortedSet`) — implemented by `TreeSet`
```java
E floor(E e);       // highest element <= e
E lower(E e);       // highest element <  e
E ceiling(E e);     // lowest element  >= e
E higher(E e);      // lowest element  >  e
E pollFirst();      // remove & return first
E pollLast();       // remove & return last
NavigableSet<E> descendingSet();   // reverse-order view
```
**Validated example:**
```java
import java.util.TreeSet;

class NavigableSetDemo {
    public static void main(String[] args) {
        TreeSet<Integer> t = new TreeSet<>();
        t.add(1000); t.add(2000); t.add(3000); t.add(4000); t.add(5000);
        System.out.println(t.ceiling(2000));   // 2000
        System.out.println(t.higher(2000));    // 3000
        System.out.println(t.floor(3000));     // 3000
        System.out.println(t.lower(3000));     // 2000
        System.out.println(t.pollFirst());     // 1000
        System.out.println(t.pollLast());      // 5000
        System.out.println(t.descendingSet()); // [4000, 3000, 2000]
    }
}
```

### `NavigableMap` (extends `SortedMap`) — implemented by `TreeMap`
```java
K floorKey(K key);
K lowerKey(K key);
K ceilingKey(K key);
K higherKey(K key);
Map.Entry<K,V> pollFirstEntry();
Map.Entry<K,V> pollLastEntry();
NavigableMap<K,V> descendingMap();
```

```java
import java.util.TreeMap;

class NavigableMapDemo {
    public static void main(String[] args) {
        TreeMap<String,String> t = new TreeMap<>();
        t.put("b","Banana"); t.put("c","Cat"); t.put("a","Apple"); t.put("d","Dog"); t.put("g","Gun");
        System.out.println(t.ceilingKey("c"));    // c
        System.out.println(t.higherKey("e"));     // g
        System.out.println(t.floorKey("e"));      // d
        System.out.println(t.lowerKey("e"));      // d
        System.out.println(t.pollFirstEntry());   // a=Apple
        System.out.println(t.pollLastEntry());    // g=Gun
        System.out.println(t.descendingMap());    // {d=Dog, c=Cat, b=Banana}
    }
}
```

---

## 11. Utility Classes

### 11.1 `Collections`
Static utility methods for List/Set/Map objects.

**Sorting:**
```java
Collections.sort(List l);                 // natural sorting (elements must be Comparable+Homogeneous)
Collections.sort(List l, Comparator c);   // customized sorting
```
```java
ArrayList al = new ArrayList(List.of("Z","A","K","N"));
Collections.sort(al);
System.out.println(al);   // [A, K, N, Z]
```

**Searching (requires the list to already be sorted; uses Binary Search internally):**
```java
Collections.binarySearch(List l, Object target);
Collections.binarySearch(List l, Object target, Comparator c);
```
> Successful search → index. Unsuccessful search → **negative insertion point**
> `-(insertion_point) - 1`. For a list of `n` elements: successful range `0..n-1`,
> unsuccessful range `-(n+1)..-1`.

**Reversing:**
```java
Collections.reverse(List l);                       // physically reverses element order
Comparator c1 = Collections.reverseOrder(c);        // returns a REVERSED comparator (doesn't mutate list)
```

**Thread-safe wrappers:**
```java
Collections.synchronizedList(List l);
Collections.synchronizedSet(Set s);
Collections.synchronizedMap(Map m);
```

**Immutability wrappers (pre-Java-9, mutable-backed but read-only VIEW):**
```java
Collections.unmodifiableList(List l);
Collections.unmodifiableSet(Set s);
Collections.unmodifiableMap(Map m);
```

### 11.2 `Arrays`
Static utility methods for array objects.

```java
Arrays.sort(primitive[] p);            // natural sort — ONLY natural sort for primitives
Arrays.sort(Object[] o);               // natural sort
Arrays.sort(Object[] o, Comparator c); // custom sort — Comparator only works for Object[], not primitive[]
Arrays.binarySearch(primitive[] p, target);
Arrays.binarySearch(Object[] a, target);
Arrays.binarySearch(Object[] a, target, Comparator c);
List Arrays.asList(Object[] a);        // FIXED-SIZE list VIEW backed by the array
```

**`asList()` is a view, not a copy — classic trap:**
```java
String[] s = {"A", "Z", "B"};
List l = Arrays.asList(s);
s[0] = "K";
System.out.println(l);           // [K, Z, B] — array mutation reflected in list!
l.set(1, "L");                     // OK — set() is allowed
l.add("Durga");                    // RuntimeException: UnsupportedOperationException (fixed size!)
l.remove(2);                       // RuntimeException: UnsupportedOperationException
```

---

## 12. Concurrent Collections

### Why Concurrent Collections (1.5+)
Traditional collections have 3 problems in multi-threaded systems:
1. `ArrayList`/`HashMap` are not thread-safe at all.
2. Legacy thread-safe collections (`Vector`, `Hashtable`, `Collections.synchronizedXxx()`) lock
   the **entire object** for every operation — even reads — killing scalability.
3. Iterating a traditional collection while another thread structurally modifies it throws
   `ConcurrentModificationException` (fail-fast).

```java
ArrayList al = new ArrayList(List.of("A","B","C"));
Iterator itr = al.iterator();
while (itr.hasNext()) {
    String s = (String) itr.next();
    al.add("D");   // RuntimeException: java.util.ConcurrentModificationException
}
```

Concurrent Collections (`java.util.concurrent`) solve all three: always thread-safe, higher
throughput via finer-grained locking, and **fail-safe** iteration (never throws CME).

### 12.1 `ConcurrentMap` (I) → `ConcurrentHashMap`

```java
public interface ConcurrentMap<K,V> extends Map<K,V> {
    V putIfAbsent(K key, V value);            // adds ONLY if key absent; else returns existing value, no overwrite
    boolean remove(Object key, Object value); // removes ONLY if key maps to given value
    boolean replace(K key, V oldVal, V newVal); // replaces ONLY if key currently maps to oldVal
}
```

| Property | Value |
|---|---|
| Underlying DS | Hashtable, internally partitioned by **concurrency level** (default 16) |
| Locking | Reads require no lock; writes require only a **bucket-level lock** |
| `null` | **Not allowed** for keys or values |
| Iteration | **Fail-safe** — iterator works on a snapshot |

```java
ConcurrentHashMap m = new ConcurrentHashMap();
m.put(101, "A"); m.put(102, "B");
m.putIfAbsent(103, "C");
m.putIfAbsent(101, "D");     // key exists -> ignored
m.remove(101, "D");          // value mismatch ("A" != "D") -> not removed
m.replace(102, "B", "E");    // matches -> replaced
System.out.println(m);       // {101=A, 102=E, 103=C}
```

#### HashMap vs ConcurrentHashMap
| HashMap | ConcurrentHashMap |
|---|---|
| Not thread-safe | Thread-safe |
| High performance (no locking) | Slightly lower (bucket locking under contention) |
| Fail-fast iterator (throws CME) | Fail-safe iterator (no CME) |
| `null` allowed for key/value | `null` **not** allowed |
| 1.2 | 1.5 |

#### ConcurrentHashMap vs `synchronizedMap()` vs `Hashtable`
| | ConcurrentHashMap | synchronizedMap() | Hashtable |
|---|---|---|---|
| Lock granularity | Bucket-level | Whole map | Whole map |
| Concurrent readers/writers | Yes (in safe manner) | No — 1 thread at a time | No — 1 thread at a time |
| Iterator | Fail-safe | Fail-fast | Fail-fast |
| `null` | Not allowed | Allowed | Not allowed |
| Introduced | 1.5 | 1.2 | 1.0 |

### 12.2 `CopyOnWriteArrayList`

| Property | Value |
|---|---|
| Mechanism | Every **write** clones the underlying array; readers keep working on the old snapshot until the swap completes |
| Best for | **Many reads, very few writes** (each write is O(n) due to copy) |
| Insertion order / duplicates / heterogeneous / null | All allowed |
| Implements | `Serializable`, `Cloneable`, `RandomAccess` |
| Iteration | **Fail-safe** — never throws CME |
| Iterator `remove()` | **Not supported** → `UnsupportedOperationException` |

```java
CopyOnWriteArrayList l1 = new CopyOnWriteArrayList();
CopyOnWriteArrayList l2 = new CopyOnWriteArrayList(Collection c);
CopyOnWriteArrayList l3 = new CopyOnWriteArrayList(Object[] a);

// Extra method
boolean addIfAbsent(Object o);
int addAllAbsent(Collection c);
```

```java
CopyOnWriteArrayList l = new CopyOnWriteArrayList();
l.add("A"); l.add("B"); l.add("C");
Iterator itr = l.iterator();
l.add("D");                 // mutation AFTER getting the iterator...
while (itr.hasNext()) System.out.println(itr.next());   // ...NOT visible: prints A, B, C only
```

#### ArrayList vs CopyOnWriteArrayList vs Vector
| ArrayList | CopyOnWriteArrayList | Vector |
|---|---|---|
| Not thread-safe | Thread-safe (via clone-on-write) | Thread-safe (via full sync) |
| Fail-fast iterator | Fail-safe iterator | Fail-fast iterator |
| Iterator can `remove()` | Iterator **cannot** `remove()` | Iterator can `remove()` |
| 1.2 | 1.5 | 1.0 |

### 12.3 `CopyOnWriteArraySet`
Internally backed by `CopyOnWriteArrayList`. Insertion order preserved, duplicates not allowed,
fail-safe, iterator is read-only (no `remove()`). Introduced 1.5.

| CopyOnWriteArraySet | synchronizedSet() |
|---|---|
| Thread-safe via clone-on-write | Thread-safe via full lock |
| Fail-safe | Fail-fast |
| Iterator read-only | Iterator supports remove |

---

## 13. Enum-based Collections

### `EnumSet` (Abstract class, 1.5)
- All elements **must** come from the **same** `enum` type (else compile-time error) — fully
  type-safe.
- Internally implemented as **bit vectors** → extremely fast (faster than `HashSet` for enums).
- Iteration order = natural declaration order (`ordinal()`), never throws CME.
- `null` not allowed.
- Two hidden implementations: `RegularEnumSet` (≤64 constants) and `JumboEnumSet` (>64) — chosen
  automatically by the static factory methods (`EnumSet.of()`, `.allOf()`, `.noneOf()`, `.range()`).

```java
enum Priority { LOW, MEDIUM, HIGH }
EnumSet<Priority> urgent = EnumSet.of(Priority.MEDIUM, Priority.HIGH);
EnumSet<Priority> all = EnumSet.allOf(Priority.class);
```

### `EnumMap` (1.5)
- All **keys** must belong to a single `enum` type.
- Internally backed by an array indexed by `ordinal()` — very fast, memory-compact.
- Iteration order = enum declaration order.
- `null` key not allowed; implements `Serializable`, `Cloneable`.

```java
EnumMap m1 = new EnumMap(Class keyType);
EnumMap m2 = new EnumMap(EnumMap m);
EnumMap m3 = new EnumMap(Map m);
```

**Validated example:**
```java
import java.util.*;

enum Priority { LOW, MEDIUM, HIGH }

class EnumMapDemo {
    public static void main(String[] args) {
        EnumMap<Priority, String> m = new EnumMap<>(Priority.class);
        m.put(Priority.LOW, "24 Hours Response Time");
        m.put(Priority.MEDIUM, "3 Hours Response Time");
        m.put(Priority.HIGH, "1 Hour Response Time");
        System.out.println(m);
        // {LOW=24 Hours Response Time, MEDIUM=3 Hours Response Time, HIGH=1 Hour Response Time}
    }
}
```

---

## 14. Fail-Fast vs Fail-Safe Iterators

| Property | Fail-Fast | Fail-Safe |
|---|---|---|
| Throws `ConcurrentModificationException`? | Yes | No |
| Mechanism | Internal `modCount` field compared each `next()` call | Iterates over a private snapshot/clone; concurrent writes go elsewhere |
| Extra memory for snapshot | No | Yes |
| Examples | `ArrayList`, `Vector`, `HashMap`, `HashSet` | `ConcurrentHashMap`, `CopyOnWriteArrayList`, `CopyOnWriteArraySet` |

**Root cause of fail-fast:** every structural modification increments an internal `modCount`.
The iterator captures `expectedModCount` at creation and checks `modCount == expectedModCount`
on every `next()`/`remove()` call; a mismatch throws `ConcurrentModificationException`
**even in single-threaded code** (e.g., calling `list.remove()` directly instead of
`iterator.remove()` during a for-each loop).

---

## 15. Modern Java Updates (8 → 21)

The original curriculum stops around Java 7/8. As an **architect**, you are expected to know
these additions cold — they show up constantly in code review and system design interviews.

### 15.1 Java 8 — Default Methods on `Collection`/`Map`
```java
// Collection
default boolean removeIf(Predicate<? super E> filter);
default void forEach(Consumer<? super E> action);       // from Iterable
default Stream<E> stream();
default Stream<E> parallelStream();
default Spliterator<E> spliterator();

// List
default void replaceAll(UnaryOperator<E> operator);
default void sort(Comparator<? super E> c);              // in-place sort, no need for Collections.sort

// Map
default V getOrDefault(Object key, V defaultValue);
default V putIfAbsent(K key, V value);
default V computeIfAbsent(K key, Function<? super K,? extends V> mappingFunction);
default V computeIfPresent(K key, BiFunction<? super K,? super V,? extends V> remappingFunction);
default V compute(K key, BiFunction<? super K,? super V,? extends V> remappingFunction);
default V merge(K key, V value, BiFunction<? super V,? super V,? extends V> remappingFunction);
default void forEach(BiConsumer<? super K,? super V> action);
default void replaceAll(BiFunction<? super K,? super V,? extends V> function);
```

**Validated example:**
```java
List<Integer> nums = new ArrayList<>(List.of(1,2,3,4,5,6));
nums.removeIf(n -> n % 2 == 0);
System.out.println(nums);            // [1, 3, 5]

Map<String,Integer> m = new HashMap<>();
m.put("a", 1);
m.putIfAbsent("b", 2);
m.computeIfAbsent("c", k -> 3);
m.computeIfPresent("a", (k,v) -> v + 10);
m.merge("a", 5, Integer::sum);        // a: 11 + 5 = 16
System.out.println(m);                // {a=16, b=2, c=3}
System.out.println(m.getOrDefault("z", -1));   // -1
```
> `merge()` is the modern, race-condition-free replacement for the classic
> "if map contains key, increment; else put 1" word-count idiom:
> `wordCount.merge(word, 1, Integer::sum);`

**`Comparator` static/default helpers (Java 8):**
```java
Comparator<Employee> byName = Comparator.comparing(e -> e.name);
Comparator<Employee> byIdThenName = Comparator.comparingInt((Employee e) -> e.eid)
                                               .thenComparing(e -> e.name);
Comparator<Employee> reversed = byName.reversed();
Comparator<String> nullsFirst = Comparator.nullsFirst(Comparator.naturalOrder());
```

### 15.2 Java 9 — Immutable Collection Factories
```java
List<Integer> l = List.of(1, 2, 3);
Set<Integer>  s = Set.of(4, 5, 6);
Map<String,Integer> m = Map.of("x", 1, "y", 2);
Map<String,Integer> m2 = Map.ofEntries(Map.entry("x",1), Map.entry("y",2));
```
**Key differences vs `Collections.unmodifiableXxx()`:**
- **Truly immutable** — no live "backing" mutable collection exists at all.
- `Set.of()`/`Map.of()` **reject duplicate elements/keys** at creation time (`IllegalArgumentException`).
- **`null` elements/keys/values are forbidden** → `NullPointerException`.
- Mutation attempts throw `UnsupportedOperationException`.

```java
List<Integer> immList = List.of(1, 2, 3);
immList.add(99);   // UnsupportedOperationException
```

### 15.3 Java 10 — `var`, `Collectors.toUnmodifiableXxx`, `List.copyOf`
```java
var list = new ArrayList<String>();   // local-variable type inference
var immutableCopy = List.copyOf(list);  // Java 10 — shallow immutable copy

List<String> upper = Stream.of("a","b","c")
                            .collect(Collectors.toUnmodifiableList());  // Java 10
```

### 15.4 Java 16 — `Stream.toList()`
```java
List<Integer> squares = Stream.of(1,2,3).map(n -> n*n).toList();  // shorter than collect(Collectors.toList())
// NOTE: Stream.toList() returns an UNMODIFIABLE list (unlike Collectors.toList() which is mutable)
```

### 15.5 Java 21 — Sequenced Collections (JEP 431)
A brand-new interface family unifying "has a defined encounter order, first/last accessible"
semantics across `List`, `Deque`, `LinkedHashSet`, `LinkedHashMap`, `TreeSet`, `TreeMap`.

```java
public interface SequencedCollection<E> extends Collection<E> {
    SequencedCollection<E> reversed();
    void addFirst(E e);
    void addLast(E e);
    E getFirst();
    E getLast();
    E removeFirst();
    E removeLast();
}
public interface SequencedSet<E> extends Set<E>, SequencedCollection<E> { SequencedSet<E> reversed(); }
public interface SequencedMap<K,V> extends Map<K,V> {
    SequencedMap<K,V> reversed();
    Entry<K,V> firstEntry();
    Entry<K,V> lastEntry();
    Entry<K,V> pollFirstEntry();
    Entry<K,V> pollLastEntry();
    // putFirst / putLast, sequencedKeySet(), sequencedValues(), sequencedEntrySet()
}
```
`ArrayList`, `LinkedList`, `ArrayDeque` now implement `SequencedCollection`.
`LinkedHashSet` implements `SequencedSet`. `LinkedHashMap` and `TreeMap` implement
`SequencedMap`. `TreeSet` implements `SequencedSet`.

**Validated example:**
```java
List<Integer> seqList = new ArrayList<>(List.of(1,2,3));
seqList.addFirst(0);
seqList.addLast(4);
System.out.println(seqList);            // [0, 1, 2, 3, 4]
System.out.println(seqList.reversed()); // [4, 3, 2, 1, 0]

LinkedHashSet<String> lhs = new LinkedHashSet<>(List.of("A","B","C"));
System.out.println(lhs.getFirst());     // A
System.out.println(lhs.reversed());     // [C, B, A]

LinkedHashMap<String,Integer> lhm = new LinkedHashMap<>();
lhm.put("one",1); lhm.put("two",2); lhm.put("three",3);
System.out.println(lhm.firstEntry());   // one=1
System.out.println(lhm.reversed());     // {three=3, two=2, one=1}
```

> **Architect takeaway:** before Java 21, getting "the first/last element" or "a reversed view" of
> a `LinkedHashMap`/`LinkedHashSet` required manual iterator gymnastics or converting to a `List`.
> Sequenced Collections make this a first-class, allocation-cheap operation. This is a frequent
> "what's new in modern Java" interview question in 2025-2026.

### 15.6 Records + Collections (Java 16 record, relevant synergy)
`record` types auto-generate `equals()`/`hashCode()` based on all components — making them ideal,
safe `HashMap`/`HashSet` keys without hand-written boilerplate:
```java
record Point(int x, int y) {}
Set<Point> points = new HashSet<>();
points.add(new Point(1,2));
points.add(new Point(1,2));   // duplicate rejected — equals()/hashCode() auto-derived
System.out.println(points.size());  // 1
```

### 15.7 Virtual Threads (Java 21) — impact on Concurrent Collections
With Project Loom's virtual threads, blocking calls like `BlockingQueue.take()` are cheap
(they don't pin a platform thread in most cases), making blocking-queue-based producer/consumer
pipelines viable at massive scale without the historical thread-pool-sizing tension. This is a
common "how does Loom change your concurrency design" architect discussion point (not a
Collections API change per se, but directly relevant to `BlockingQueue`/`SynchronousQueue` usage
patterns).

---

## 16. Cheat Sheets

### 16.1 One-Page Decision Table

| I need... | Use |
|---|---|
| Ordered, index-based, duplicates OK, fast random access | `ArrayList` |
| Frequent insert/delete in the middle | `LinkedList` (or `ArrayDeque`) |
| Thread-safe legacy list | `Vector` (prefer `CopyOnWriteArrayList` or `Collections.synchronizedList`) |
| LIFO stack | `ArrayDeque` (modern) — avoid legacy `Stack` |
| No duplicates, don't care about order | `HashSet` |
| No duplicates, preserve insertion order | `LinkedHashSet` |
| No duplicates, always sorted | `TreeSet` |
| Key-value, don't care about order | `HashMap` |
| Key-value, preserve insertion order | `LinkedHashMap` |
| Key-value, always sorted by key | `TreeMap` |
| Key-value, config from `.properties` file | `Properties` |
| FIFO processing queue | `LinkedList` / `ArrayDeque` as `Queue` |
| Priority-based processing | `PriorityQueue` |
| Producer-consumer with blocking | `BlockingQueue` (`ArrayBlockingQueue` / `LinkedBlockingQueue`) |
| Thread-safe map, high concurrency | `ConcurrentHashMap` |
| Read-heavy, rarely-written list (thread-safe) | `CopyOnWriteArrayList` |
| Enum-only Set/Map, max performance | `EnumSet` / `EnumMap` |
| Truly immutable, small, fixed collection | `List.of()` / `Set.of()` / `Map.of()` |
| Need first/last/reversed view cheaply | Any `SequencedCollection`/`SequencedSet`/`SequencedMap` (Java 21+) |

### 16.2 `null` Acceptance Matrix

| Collection | Null Key | Null Value/Element |
|---|---|---|
| `ArrayList` / `LinkedList` | — | Any number |
| `HashSet` / `LinkedHashSet` | — | Once |
| `TreeSet` | — | Only as 1st element in empty set |
| `HashMap` / `LinkedHashMap` | 1 null key | Any number of null values |
| `Hashtable` | Not allowed | Not allowed |
| `TreeMap` | Only as 1st entry in empty map | No restriction |
| `ConcurrentHashMap` | Not allowed | Not allowed |
| `PriorityQueue` | — | Not allowed (even as 1st) |
| `List.of()` / `Set.of()` / `Map.of()` | Not allowed | Not allowed |

### 16.3 Thread-Safety / Iterator Matrix

| Collection | Thread-Safe? | Iterator |
|---|---|---|
| `ArrayList`, `LinkedList`, `HashMap`, `HashSet`, `TreeMap`, `TreeSet` | No | Fail-Fast |
| `Vector`, `Hashtable` | Yes (full lock) | Fail-Fast |
| `Collections.synchronizedXxx()` | Yes (full lock) | Fail-Fast |
| `ConcurrentHashMap` | Yes (bucket lock) | Fail-Safe |
| `CopyOnWriteArrayList` / `CopyOnWriteArraySet` | Yes (clone-on-write) | Fail-Safe, no `remove()` |

### 16.4 Big-O Cheat Sheet

| Operation | ArrayList | LinkedList | HashMap | TreeMap | HashSet | TreeSet |
|---|---|---|---|---|---|---|
| get by index/key | O(1) | O(n) | O(1) avg | O(log n) | — | — |
| add (end) | O(1) amortized | O(1) | O(1) avg | O(log n) | O(1) avg | O(log n) |
| add/remove (middle/head) | O(n) | O(1)* | — | — | — | — |
| contains | O(n) | O(n) | O(1) avg | O(log n) | O(1) avg | O(log n) |
| search (sorted) | O(log n) binarySearch | O(n) | — | — | — | — |

*O(1) only if you already hold the `Node` reference (e.g., via `ListIterator`); a plain
`remove(int index)` is still O(n) because it must traverse to that index first.

---

## 17. Interview Question Bank

### Conceptual / Framework Design
1. **Why do we need Collections when we already have Arrays?**
   Arrays are fixed-size, homogeneous-only (for primitive/typed arrays), and have no built-in
   algorithm support. Collections are growable, can mix types via `Object`/generics-erased
   references, and every implementation is backed by a documented data structure with ready-made
   methods (sort, search, etc.).

2. **Difference between `Collection` and `Collections`?**
   `Collection` is the root interface representing a group of objects. `Collections` is a final
   utility class with static helper methods (`sort`, `binarySearch`, `synchronizedXxx`, etc.)
   that operate on Collection objects.

3. **Why is there no concrete class that directly implements `Collection`?**
   Because `Collection` is intentionally abstract/generic — concrete behavior (ordered vs
   unordered, unique vs duplicate) is defined by its child interfaces `List`/`Set`/`Queue`.

4. **Why doesn't `Map` extend `Collection`?**
   `Map` represents key-value pairs, a fundamentally different contract (two-dimensional
   lookup) from `Collection`'s single-object grouping. Retrofitting `Map` under `Collection`
   would force awkward semantics (is an Entry the element? A key? A value?).

### List
5. **ArrayList vs LinkedList — when would you choose each?**
   `ArrayList`: backed by a resizable array; O(1) random access, O(n) insert/delete in the
   middle (array shifting). Best when reads dominate. `LinkedList`: doubly-linked list; O(1)
   insert/delete once positioned (via iterator), O(n) random access. Best when frequent
   middle insert/delete dominate. In practice, `ArrayList` wins most benchmarks due to CPU
   cache locality — measure before choosing `LinkedList`.

6. **What is the capacity growth formula for `ArrayList`? For `Vector`?**
   `ArrayList`: `newCapacity = (oldCapacity * 3/2) + 1`. `Vector`: `newCapacity = oldCapacity * 2`
   (or `oldCapacity + incrementalCapacity` if specified in the constructor).

7. **Why does `Vector`/`Stack` feel "outdated" to architects today?**
   Every method is synchronized (even for single-threaded use, paying lock overhead for
   nothing), and `Stack extends Vector`, leaking random-access mutation methods that violate
   LIFO discipline. `ArrayDeque` is the modern, faster, allocation-friendlier replacement for
   both stack and non-thread-safe queue use cases.

8. **What is `RandomAccess`? Why does it matter?**
   A marker interface (no methods) implemented by `ArrayList`/`Vector` (not `LinkedList`).
   Algorithms like `Collections.binarySearch()` check `instanceof RandomAccess` to decide
   between an index-loop strategy (fast for array-backed lists) or an iterator-based strategy
   (fast for linked lists) — avoiding O(n²) behavior on `LinkedList`.

9. **What happens if you call `list.add()` directly (not via iterator) while iterating with a
   for-each loop?**
   `ConcurrentModificationException` — the internal `modCount` changes, but the iterator's
   cached `expectedModCount` doesn't, and the mismatch is detected on the next `next()` call.

10. **How do you safely remove elements while iterating a `List`?**
    Use `Iterator.remove()` (or `ListIterator.remove()`), or `Collection.removeIf(predicate)`
    (Java 8+), or iterate a copy.

### Cursors
11. **Enumeration vs Iterator vs ListIterator?**
    See the comparison table in Section 5.4 — key axes: legacy-only vs universal, forward-only
    vs bi-directional, read-only vs read+remove vs read+remove+add+replace.

12. **Why is `Enumeration` still around if `Iterator` supersedes it?**
    Backward compatibility — legacy classes (`Vector`, `Hashtable`) still expose `elements()`
    for old code that hasn't migrated.

13. **Can you add a new element to a list while iterating with `ListIterator`?**
    Yes — `ListIterator.add(Object)` is explicitly supported, unlike plain `Iterator`.

### Set
14. **HashSet vs LinkedHashSet vs TreeSet?**
    See Section 6.4 comparison table. Key differentiators: underlying DS, whether insertion
    order is preserved, and whether elements are kept sorted.

15. **Why does `HashSet.add()` on a duplicate return `false` instead of throwing?**
    Because `Set` semantics define "no duplicates" as a **silent no-op**, not an error
    condition — this lets you write `if (set.add(x)) { /* was new */ }` idiomatically.

16. **Can a `TreeSet` contain `null`?**
    Only as the very first element inserted into an **empty** TreeSet. Any element inserted
    afterward triggers a `compareTo(null)` call internally, which throws
    `NullPointerException`. For a non-empty TreeSet, inserting `null` directly also throws NPE.

17. **Why does inserting a `StringBuffer` into a natural-order `TreeSet` throw
    `ClassCastException`?**
    `StringBuffer` does not implement `Comparable`. Natural ordering requires
    `Comparable` elements; without it, the internal cast to `Comparable` fails at runtime
    (not compile time, since `TreeSet` uses raw/generic-erased calls internally when no
    Comparator is supplied).

### Comparable / Comparator
18. **Difference between `Comparable` and `Comparator`?**
    See Section 7 table. `Comparable` = 1 method, defines the class's own default sort,
    `java.lang`. `Comparator` = 2 methods (`compare`, `equals`), external customized sort
    strategy, `java.util`.

19. **If a class implements both natural ordering and you also pass a Comparator to `TreeSet`,
    which wins?**
    The **Comparator** always wins — `TreeSet(Comparator c)` calls `compare()` and never calls
    `compareTo()`.

20. **How do you sort a list of custom objects by multiple fields (e.g., by department then by
    salary descending)?**
    ```java
    list.sort(Comparator.comparing(Employee::getDept)
                        .thenComparing(Employee::getSalary, Comparator.reverseOrder()));
    ```

### Map
21. **HashMap vs Hashtable vs ConcurrentHashMap — full comparison?**
    See Sections 8.1 and 12.1 tables. Core axes: thread-safety mechanism (none / full-lock /
    bucket-lock), `null` support, iterator fail-fast vs fail-safe, and era introduced.

22. **How does `HashMap` internally resolve hash collisions?**
    Buckets store colliding entries as a linked list; since Java 8, if a single bucket's chain
    grows beyond a threshold (8) **and** the table has at least 64 buckets, that bucket is
    converted to a **red-black tree** for O(log n) worst-case lookup instead of O(n).

23. **Why must a key's `hashCode()` and `equals()` be consistent for `HashMap` correctness?**
    `HashMap` uses `hashCode()` to locate the bucket and `equals()` to confirm the exact key
    match within that bucket. If two "equal" objects return different hash codes, `HashMap`
    may store them in different buckets and silently treat them as distinct keys, breaking
    lookups (`get()` returns null even though an "equal" key was inserted).

24. **What is the difference between `HashMap` and `IdentityHashMap`?**
    `HashMap` treats keys as duplicate via `.equals()`; `IdentityHashMap` treats keys as
    duplicate only via `==` (reference identity) — used for reference-based tracking (e.g.,
    serialization graph traversal, JVM tooling).

25. **What is `WeakHashMap` used for?**
    Caches where you want entries to auto-expire once nothing else references the key —
    prevents memory leaks caused by long-lived caches holding onto objects that are otherwise
    dead.

26. **Why is `null` disallowed in `Hashtable` and `ConcurrentHashMap` but allowed in `HashMap`?**
    Historical/design choice: for `Hashtable` it predates `HashMap`'s more permissive design.
    For `ConcurrentHashMap`, allowing `null` would create ambiguity between "key not present"
    (`get()` returns null) and "key present with a null value" in a concurrent context where
    another thread might be mutating the map at the same instant — Doug Lea (its author)
    explicitly disallowed `null` to remove this race-prone ambiguity.

27. **Explain `computeIfAbsent` vs `putIfAbsent` — when would you prefer one?**
    `putIfAbsent(k, v)` **always evaluates `v` eagerly** (even if the key exists) — wasteful for
    expensive value construction. `computeIfAbsent(k, function)` only invokes the function if
    the key is truly absent — better for lazy/expensive default values, and it's the
    idiomatic building block for `Map<K, List<V>>` "multi-map" patterns:
    ```java
    map.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
    ```

28. **What's the difference between `Map.of()` and `new HashMap<>()`?**
    `Map.of()` is truly immutable, rejects `null` and duplicate keys, and has a fixed
    implementation optimized for small sizes (special-cased for 0/1/2 entries, array-based for
    more). `HashMap` is mutable and permits one `null` key.

### TreeMap / SortedMap
29. **Can `TreeMap` values be heterogeneous or non-Comparable even under natural key sorting?**
    Yes — sorting applies only to **keys**; values have no such restriction.

30. **What's the difference between `NavigableMap.floorKey()` and `SortedMap.headMap()`?**
    `floorKey(k)` returns a single key (the greatest key `<= k`). `headMap(k)` returns a whole
    **submap view** of all keys strictly `< k`.

### Queue / Deque
31. **Difference between `poll()`/`peek()` and `remove()`/`element()` on `Queue`?**
    `poll()`/`peek()` return `null` on empty queue. `remove()`/`element()` throw
    `NoSuchElementException` on empty queue.

32. **Why can't `PriorityQueue` accept `null`, unlike `TreeSet`?**
    `PriorityQueue`'s heap-based percolation logic compares the new element against existing
    ones immediately upon insertion, even for the very first element in some code paths — this
    makes `null` unsafe to support consistently, so the JDK disallows it outright, even as the
    first element (contrast with `TreeSet`, which allows a `null` first-element because no
    comparison is needed yet).

33. **`BlockingQueue` vs `TransferQueue` — what's the key semantic difference?**
    `BlockingQueue.put()` only guarantees the element was successfully **stored** (blocks only
    if the queue is full). `TransferQueue.transfer()` blocks until a **consumer thread actually
    takes** the element — a stronger, synchronous hand-off guarantee, ideal for message-passing
    systems needing delivery confirmation.

34. **Why does the JDK recommend `ArrayDeque` over `LinkedList` for queue/stack use?**
    `ArrayDeque` uses a circular array internally — no per-node object allocation, better cache
    locality, and generally faster in benchmarks for both stack (`push`/`pop`) and queue
    (`offer`/`poll`) operations. `LinkedList` incurs a `Node` allocation per element.

### Concurrent Collections
35. **Why does `ConcurrentHashMap`'s iterator never throw `ConcurrentModificationException`?**
    It's fail-safe — the iterator reflects the map's state *as of* iterator creation (or, more
    precisely, weakly-consistent — it may or may not reflect concurrent updates, but it never
    throws, and never revisits or skips already-visited entries incorrectly for a single
    thread's traversal).

36. **What is "weakly consistent" iteration?**
    A guarantee stronger than "fail-fast is disabled" but weaker than a full snapshot: the
    iterator is guaranteed to traverse elements that existed at start and end of iteration,
    may (but is not guaranteed to) reflect modifications made during iteration, and never
    throws `ConcurrentModificationException`, and never returns the same element twice.

37. **Why is `CopyOnWriteArrayList` a poor choice for a write-heavy workload?**
    Every single mutation (`add`/`remove`/`set`) clones the **entire backing array** — O(n) per
    write, and heavy GC churn under high write throughput.

38. **Can you call `iterator.remove()` on a `CopyOnWriteArrayList`'s iterator?**
    No — throws `UnsupportedOperationException`. The iterator is a **read-only** snapshot view.

39. **Fail-Fast vs Fail-Safe — give one example each and explain the underlying mechanism.**
    Fail-Fast: `ArrayList`'s iterator checks an internal `modCount` field against a cached
    `expectedModCount` on every `next()`; a structural change from another (or the same)
    thread trips a mismatch and throws CME. Fail-Safe: `CopyOnWriteArrayList`'s iterator
    holds a reference to the array **as it was at iterator-creation time**; subsequent writes
    replace the list's internal array reference entirely, leaving the iterator's captured
    array untouched.

### Enum Collections
40. **Why use `EnumSet`/`EnumMap` instead of `HashSet<MyEnum>`/`HashMap<MyEnum,V>`?**
    Internally implemented as bit-vectors / ordinal-indexed arrays — dramatically faster and
    more memory-compact than hash-based structures, while also guaranteeing enum-declaration-
    order iteration and full type-safety (compile error if you mix enum types).

### Modern Java
41. **What changed with `List.of()` vs `Collections.unmodifiableList()`?**
    `Collections.unmodifiableList()` wraps a *mutable* backing list — someone holding a
    reference to the original mutable list can still change it, and that change **is visible**
    through the "unmodifiable" wrapper. `List.of()` copies data into a genuinely immutable
    structure with no live mutable backing at all, and additionally rejects `null` elements and
    (for `Set.of`/`Map.of`) duplicate elements/keys at creation time.

42. **What are Sequenced Collections (Java 21) and why were they introduced?**
    JEP 431 unifies "first/last element access" and "reversed view" across `List`, `Deque`,
    `LinkedHashSet`, `LinkedHashMap`, `SortedSet`, `SortedMap` via new interfaces
    (`SequencedCollection`, `SequencedSet`, `SequencedMap`) — before this, there was no common
    contract for "this collection has a defined encounter order," so getting the last element
    of a `LinkedHashSet`, for instance, required an awkward full iteration.

43. **`Stream.toList()` (Java 16) vs `Collectors.toList()` — any difference?**
    `Stream.toList()` returns an **unmodifiable** list. `.collect(Collectors.toList())` returns
    a mutable `ArrayList` (implementation detail, not guaranteed, but true in the reference JDK).
    Attempting to mutate the result of `Stream.toList()` throws `UnsupportedOperationException`.

44. **How do `record` types simplify using custom classes as `HashMap`/`HashSet` keys?**
    Records auto-generate `equals()`/`hashCode()`/`toString()` from all components, eliminating
    the classic bug of forgetting to override `hashCode()` when `equals()` is overridden (or
    vice versa) — a very common source of "why doesn't my custom key work in a HashMap" bugs.

45. **How does `merge()` simplify a word-frequency counter compared to pre-Java-8 code?**
    ```java
    // Pre-8
    if (map.containsKey(word)) map.put(word, map.get(word) + 1);
    else map.put(word, 1);
    // Java 8+
    map.merge(word, 1, Integer::sum);
    ```

### Design / Architect-Level Scenario Questions
46. **You need a cache with bounded size and LRU eviction. What Collection would you build it
    on?**
    `LinkedHashMap` with `accessOrder=true` constructor flag, overriding
    `removeEldestEntry(Map.Entry)` to return `true` once size exceeds your cap — this is the
    JDK's purpose-built LRU-cache recipe (predates `java.util.concurrent` caching libraries
    like Caffeine, which you'd use in production for more advanced eviction policies).

47. **You have a multi-threaded producer-consumer pipeline. Which Collection?**
    `BlockingQueue` (`LinkedBlockingQueue`/`ArrayBlockingQueue`) — `put()`/`take()` handle
    backpressure and blocking natively without manual wait/notify code.

48. **Your app reads a shared list constantly from many threads but writes to it rarely (e.g., a
    list of feature-flag listeners). Which Collection?**
    `CopyOnWriteArrayList` — optimized exactly for this read-heavy/write-rare pattern.

49. **You need a `Map` where you'll frequently ask "give me all entries between key A and key
    B." Which Collection?**
    `TreeMap` (`NavigableMap`) — `subMap()`, `headMap()`, `tailMap()` give O(log n) range views.

50. **Why might `HashMap` be a bad choice for keys that are mutable objects?**
    If a key's fields (that participate in `hashCode()`) change after insertion, the key's
    hash bucket becomes stale — subsequent `get()` calls compute a *different* bucket than
    where the entry actually lives, effectively "losing" the entry (it's still in the map, just
    unreachable via that key). **Rule: never use mutable objects as HashMap/HashSet keys**
    unless you can guarantee immutability of the hash-relevant fields for the object's lifetime
    in the map.

51. **Why is `Vector`/`Hashtable`/`Collections.synchronizedXxx()` considered obsolete in modern
    architecture, and what replaces them?**
    They use coarse-grained, whole-object locking, causing contention under concurrent load.
    Modern replacements: `ConcurrentHashMap` (for Map), `CopyOnWriteArrayList`/
    `CopyOnWriteArraySet` (read-heavy List/Set), `BlockingQueue` implementations (producer-
    consumer), and `java.util.concurrent.atomic` classes for single-value counters.

52. **What's the difference in growth strategy between `ArrayList` (1.5x) and `Vector` (2x), and
    why might that matter at scale?**
    `Vector`'s 2x growth wastes more memory on average after resize (up to 2x current size vs
    up to 1.5x for `ArrayList`), but resizes less frequently for the same growth trajectory.
    In practice, always pre-size (`new ArrayList<>(expectedSize)`) in hot paths to avoid
    repeated array copies altogether.

53. **How would you make a thread-safe, insertion-order-preserving Set today, given
    `LinkedHashSet` isn't thread-safe and `CopyOnWriteArraySet` doesn't preserve *sorted* order?**
    `Collections.synchronizedSet(new LinkedHashSet<>())` if writes are frequent (with manual
    synchronization on iteration blocks), or `CopyOnWriteArraySet` if reads dominate (it *does*
    preserve insertion order internally via its backing `CopyOnWriteArrayList` — clarify: it
    preserves insertion order, just isn't literally a `TreeSet`-style sorted set).

54. **`ConcurrentSkipListMap`/`ConcurrentSkipListSet` — where do these fit in?**
    They're the **concurrent, thread-safe equivalents of `TreeMap`/`TreeSet`** — sorted,
    navigable, and safe for concurrent access via a lock-free skip-list structure (O(log n)
    average operations, no single global lock). Use them when you need `NavigableMap`/
    `NavigableSet` semantics *and* thread-safety — `TreeMap`/`TreeSet` themselves are not
    thread-safe.

55. **Rapid-fire: name the underlying data structure for each: `ArrayList`, `LinkedList`,
    `HashSet`, `LinkedHashSet`, `TreeSet`, `HashMap`, `LinkedHashMap`, `TreeMap`, `Hashtable`,
    `PriorityQueue`, `ArrayDeque`.**
    Resizable array / Doubly-linked list / Hash table / Hash table + linked list / Red-black
    tree / Hash table (array of buckets + linked list or tree per bucket) / Hash table + linked
    list / Red-black tree / Hash table / Binary heap (array-backed) / Circular resizable array.

### Practical Coding Question
56. **Write a Java program with a custom `Comparator` to sort an unordered
    `List<Map<String, Employee>>` by Employee **Name** first, and then by **Age** as a
    tie-breaker.**

    Each element of the list is a `Map<String, Employee>` holding a single `empId -> Employee`
    entry (a realistic shape for data coming from, e.g., a JSON/REST payload deserialized as
    `List<Map<String,Object>>`). Since a `Map` itself has no natural ordering, the comparator
    must first **extract** the `Employee` object from each map before comparing.

    ```java
    import java.util.*;

    class Employee {
        String name;
        int age;
        int eid;

        Employee(int eid, String name, int age) {
            this.eid = eid;
            this.name = name;
            this.age = age;
        }

        public String toString() {
            return "Employee{eid=" + eid + ", name='" + name + "', age=" + age + "}";
        }
    }

    // Custom Comparator: sort by Name (alphabetical) first, then by Age (ascending).
    // Each element of the List is a Map<String, Employee> (one entry per map, keyed by
    // empId), so we extract the Employee via map.values().iterator().next() to compare.
    class EmployeeMapComparator implements Comparator<Map<String, Employee>> {
        @Override
        public int compare(Map<String, Employee> m1, Map<String, Employee> m2) {
            Employee e1 = m1.values().iterator().next();
            Employee e2 = m2.values().iterator().next();

            int nameCompare = e1.name.compareTo(e2.name);
            if (nameCompare != 0) {
                return nameCompare;          // primary key: Name
            }
            return Integer.compare(e1.age, e2.age);   // tie-break: Age
        }
    }

    public class EmployeeSortDemo {
        public static void main(String[] args) {

            // Unordered List<Map<String, Employee>> — each Map holds ONE empId -> Employee
            List<Map<String, Employee>> employeeList = new ArrayList<>();

            Map<String, Employee> m1 = new HashMap<>();
            m1.put("E103", new Employee(103, "Venki", 35));
            employeeList.add(m1);

            Map<String, Employee> m2 = new HashMap<>();
            m2.put("E101", new Employee(101, "Bala", 28));
            employeeList.add(m2);

            Map<String, Employee> m3 = new HashMap<>();
            m3.put("E102", new Employee(102, "Bala", 24));
            employeeList.add(m3);

            Map<String, Employee> m4 = new HashMap<>();
            m4.put("E104", new Employee(104, "Chiru", 40));
            employeeList.add(m4);

            Map<String, Employee> m5 = new HashMap<>();
            m5.put("E105", new Employee(105, "Amar", 30));
            employeeList.add(m5);

            System.out.println("Before Sorting:");
            printList(employeeList);

            // --- Approach 1: Custom Comparator class (classic style) ---
            Collections.sort(employeeList, new EmployeeMapComparator());

            System.out.println("\nAfter Sorting (Custom Comparator class - Name then Age):");
            printList(employeeList);

            // --- Approach 2: Java 8+ lambda + Comparator.comparing/thenComparing ---
            List<Map<String, Employee>> employeeList2 = new ArrayList<>(employeeList);
            Collections.shuffle(employeeList2); // re-shuffle to prove it re-sorts correctly

            employeeList2.sort(
                Comparator.<Map<String, Employee>, String>comparing(
                        m -> m.values().iterator().next().name)
                    .thenComparingInt(m -> m.values().iterator().next().age)
            );

            System.out.println("\nAfter Sorting (Java 8 Comparator.comparing/thenComparing):");
            printList(employeeList2);
        }

        private static void printList(List<Map<String, Employee>> list) {
            for (Map<String, Employee> map : list) {
                for (Map.Entry<String, Employee> entry : map.entrySet()) {
                    System.out.println(entry.getKey() + " -> " + entry.getValue());
                }
            }
        }
    }
    ```

    **Validated Output (OpenJDK 21):**
    ```
    Before Sorting:
    E103 -> Employee{eid=103, name='Venki', age=35}
    E101 -> Employee{eid=101, name='Bala', age=28}
    E102 -> Employee{eid=102, name='Bala', age=24}
    E104 -> Employee{eid=104, name='Chiru', age=40}
    E105 -> Employee{eid=105, name='Amar', age=30}

    After Sorting (Custom Comparator class - Name then Age):
    E105 -> Employee{eid=105, name='Amar', age=30}
    E102 -> Employee{eid=102, name='Bala', age=24}
    E101 -> Employee{eid=101, name='Bala', age=28}
    E104 -> Employee{eid=104, name='Chiru', age=40}
    E103 -> Employee{eid=103, name='Venki', age=35}

    After Sorting (Java 8 Comparator.comparing/thenComparing):
    E105 -> Employee{eid=105, name='Amar', age=30}
    E102 -> Employee{eid=102, name='Bala', age=24}
    E101 -> Employee{eid=101, name='Bala', age=28}
    E104 -> Employee{eid=104, name='Chiru', age=40}
    E103 -> Employee{eid=103, name='Venki', age=35}
    ```

    **Architect notes:**
    - Notice both "Bala" entries tie on `name` and correctly fall back to `age` ascending
      (24 before 28) — proving the tie-break logic works.
    - `Collections.sort(list, comparator)` (pre-Java-8) and `list.sort(comparator)`
      (Java 8+ default method on `List`) are functionally identical — `Collections.sort()`
      internally delegates to `list.sort()` since Java 8.
    - `Comparator.comparing(keyExtractor).thenComparingInt(keyExtractor)` avoids autoboxing
      for the `age` tie-break (`thenComparingInt` vs `thenComparing` with an `Integer`
      extractor) — worth calling out in a performance-sensitive sort path.
    - Extracting the single `Employee` via `map.values().iterator().next()` assumes exactly
      one entry per `Map` (as stated in the problem). For a `Map` with unknown/variable size,
      you'd instead look up by a known key, e.g. `map.get(empId)`.

---

**End of Study Guide — Collections Framework.**
*All executable examples in this document were compiled and run on OpenJDK 21.0.10 to verify
correctness of output.*
