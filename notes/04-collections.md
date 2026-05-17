# Chapter 4 — Collections

## The Three Main Collection Types

| Type | Order | Duplicates | Access |
|------|-------|------------|--------|
| **List** | Fixed order | Allowed | By index |
| **Set** | No guaranteed order | Not allowed | By value |
| **Map** | Depends on implementation | Keys must be unique | By key |

### Choosing the Right Structure

```
Does each element have a unique key?
  → Yes: Map

Is order important, or can elements appear more than once?
  → Yes: List
  → No:  Set
```

---

## List

### `ArrayList`

- Like an array, but without a fixed size.
- Fast direct access by index (`get(i)`).
- Slower insertion and deletion (elements must shift).

### `LinkedList`

- Elements are stored as a chain — each node points to the next.
- Easy to add or remove elements anywhere in the list.
- Great for sequential traversal from front to back.
- Slow direct access by index (must walk the chain).

**Rule of thumb:**
- Many insertions/deletions → `LinkedList`
- Many random accesses → `ArrayList`
- Both implement the same `List` interface, so they are interchangeable.

### Creating a List

Old-style array (fixed length):

```java
String[] words = new String[3];
words[0] = "Hello";
words[1] = "dear";
words[2] = "students";
```

With `LinkedList` (or `ArrayList` — same API):

```java
LinkedList<String> words = new LinkedList<>();
words.add("Hello");
words.add("dear");
words.add("students");
```

Collections are **generic classes** — the type parameter (`<String>`) is set by the caller, not fixed in the class definition. Only class types are allowed, not primitives (`int` → `Integer`, `double` → `Double`, etc.).

### Key `List<E>` Methods

| Method | Description |
|--------|-------------|
| `boolean add(E e)` | Appends the element at the end |
| `void add(int index, E e)` | Inserts the element at the given index |
| `boolean remove(Object o)` | Removes the first occurrence of the element |
| `E remove(int index)` | Removes and returns the element at the given index |
| `int size()` | Returns the number of elements |
| `E get(int index)` | Returns the element at index (throws `IndexOutOfBoundsException` if out of range) |
| `E set(int index, E value)` | Replaces the element at index |
| `T[] toArray(T[] a)` | Returns an array containing all elements |
| `int indexOf(Object o)` | Returns the index of the first occurrence, or `-1` if not found |

Reference: https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/LinkedList.html

---

## Iterating a List

### `for` loop (less efficient for `LinkedList`)

```java
for (int i = 0; i < words.size(); i++) {
    System.out.println(words.get(i));
}
```

### Enhanced `for` loop (preferred)

```java
for (String word : myList) {
    // word is set to each element in order
}
```

- Works for any class that implements `Iterable`, and for arrays.
- Internally uses an `Iterator`.
- **Do not modify the list** inside a `for-each` loop — this causes a `ConcurrentModificationException`.

### Iterator (use when modifying during traversal)

```java
Iterator<String> iter = myList.iterator();

while (iter.hasNext()) {
    String element = iter.next();   // get current element and advance — don't call twice!
    if (element.indexOf('l') > -1)  // if element contains 'l'
        iter.remove();              // safe removal during iteration
}
```

Useful `String` methods for filtering:

```java
element.indexOf('d') > -1    // contains 'd'
element.indexOf("d", 1) > -1 // contains 'd' starting from index 1
element.charAt(0) == 'd'     // starts with character 'd'
element.startsWith("A")      // starts with "A"
```

### `removeIf` with Lambda (shorter alternative)

```java
myList.removeIf(element -> element.indexOf('l') > -1);
```

---

## Set

Usage is the same as `List`. Sets inherit from `Collection`.

### `HashSet`

- Internally computes a hash value using `hashCode()` (inherited from `Object`).
- Very fast insertion and deletion.
- No guaranteed iteration order.

### `TreeSet`

- Stores elements in a sorted tree structure — elements are automatically sorted.
- Requires the stored objects to implement `Comparable<E>`.

---

## `Collections` Utility Class

`java.util.Collections` provides static helper methods for working with collections:

| Method | Description |
|--------|-------------|
| `void copy(List dest, List src)` | Copies elements from `src` into `dest` |
| `boolean disjoint(Collection a, Collection b)` | Returns `true` if `a` and `b` share no elements |
| `int frequency(Collection c, Object o)` | Counts how many times `o` appears in `c` |
| `void reverse(List l)` | Reverses the order of the list |
| `void shuffle(List l)` | Randomly shuffles the list |
| `void sort(List l)` | Sorts the list (elements must be `Comparable`) |
| `E min(Collection<E> c)` | Returns the minimum element |
| `E max(Collection<E> c)` | Returns the maximum element |

Reference: https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Collections.html

---

## Map

`Map` does **not** inherit from `Collection`.

### Implementations

| Type | Behaviour |
|------|-----------|
| `HashMap` | Hash-based key lookup — unordered, fast reads |
| `TreeMap` | Keys stored sorted in a tree — ordered iteration |
| `Properties` | Specialised map where both keys and values are `String`; used for system/config properties |

### Creating and Populating a Map

```java
// TreeMap sorted by key (account number)
// Note: primitive long must be boxed to Long
TreeMap<Long, Account> bank = new TreeMap<>();

bank.put(1234L, checkingAccount1);
bank.put(999L,  checkingAccount2);
bank.put(34567L, savingsAccount);
```

### Key `Map<K, V>` Methods

| Method | Description |
|--------|-------------|
| `V put(K key, V value)` | Inserts or overwrites the entry for the given key |
| `V remove(K key)` | Removes and returns the entry for the given key |
| `int size()` | Returns the number of entries |
| `V get(K key)` | Returns the value for the key — returns **`null`** (no exception) if the key doesn't exist |
| `boolean containsKey(K key)` | Tests whether the key is present |
| `boolean containsValue(V value)` | Tests whether the value is present |

### Safe Lookup

`get()` returns `null` silently if the key is missing. Guard against it:

```java
Account a = bank.get(26576L);
if (a != null)
    a.deposit(100);
```

Or check first:

```java
if (bank.containsKey(26576L))
    bank.get(26576L).deposit(100);
```

Use **one** of these approaches — not both at once.

### Iterating a Map

Three variants:

```java
// 1. Iterate over keys only
for (String key : myMap.keySet()) {
    // use key
}

// 2. Iterate over values only
for (BigDecimal value : myMap.values()) {
    // use value
}

// 3. Iterate over both key and value
for (Map.Entry<String, BigDecimal> entry : myMap.entrySet()) {
    entry.getKey();
    entry.getValue();
}
```

(`Map` itself is not `Iterable`, so you must go through `keySet()`, `values()`, or `entrySet()`.)

### Views — `keySet()`, `values()`, `entrySet()`

These methods return **live views** backed by the original map:

- `remove()` on the view also removes from the map.
- `add()` on the view throws `UnsupportedOperationException`.

```java
Set<Long> keys = bank.keySet();
keys.remove(999L);   // also removes from bank!
```

To get an independent copy:

```java
Set<Long> copy = new TreeSet<>(bank.keySet());   // no longer connected to bank
```

---

## Collections and Polymorphism

Assigning an implementing class to an interface variable works normally:

```java
List<Integer> list = new ArrayList<>();
Collection<Integer> col = new LinkedList<>();
```

But **type parameters are not covariant** — you cannot assign between parameterised types even if one type argument is a subtype of another:

```java
List<Number> l = new ArrayList<Integer>();   // compile error
List<Integer> l = new ArrayList<Number>();   // compile error
```

---

## Wildcards (`<?>`)

`SomeClass<?>` is effectively the supertype of all parameterised forms of `SomeClass`:

```java
List<?> x = new ArrayList<String>();   // allowed
```

Use `<?>` when:
- You only iterate and don't care about the element type.
- You are counting, copying, or working only with the outer collection — not the elements themselves.
- You need to work with **bounds** (covered in chapter 6 — Generics).

Example — `Collections.shuffle` accepts any list regardless of element type:

```java
static void shuffle(List<?> list) { ... }
```

### Wildcard in practice — mixing types via cast

```java
List<Integer> intList = new ArrayList<>();
intList.add(15);

List<?> wcList = intList;                     // List<?> is a supertype of List<Integer>
List<String> stringList = (List<String>) wcList;  // cast allowed via wildcard
stringList.add("Hello");

System.out.println(wcList);   // [15, Hello]
```

This bypasses type safety — use with care.
