# Chapter 6 — Generics

## Generic Methods

A generic method declares one or more **type parameters** that are resolved at the call site.

### Syntax

```java
/**
 * @param <T> meaning of T
 */
modifier <T> ReturnType methodName(T parameter, ...) { ... }
```

The type parameter `<T>` is declared **before the return type** and can be used anywhere a data type is needed within the method.

### Wildcard vs. Type Parameter

**Wildcard (`?`)** — use when you only iterate or work with the outer collection and don't need to refer back to the element type:

```java
public static void printAll(List<?> list) {
    for (Object x : list) {
        System.out.println(x);
    }
}
```

`List<Object>` would not work here — it only accepts a list whose element type is literally `Object`, not any subtype. Generics are not covariant, so polymorphism on type parameters is not allowed.

**Named type parameter** — use when the return type must match the element type, or when you need to refer to the type in multiple places:

```java
public static <E> E firstElement(List<E> list) {
    return !list.isEmpty() ? list.get(0) : null;
}
```

Without `<E>`, you would have to return `Object` and force the caller to cast.

```java
// Both call forms work:
String first = firstElement(words);                    // type inferred
String first = ClassName.<String>firstElement(words);  // explicit (rarely needed)
```

**Best practice:** if the type parameter only appears in the method signature (not used to return a value or access type-specific behaviour), prefer the wildcard form — it's simpler.

---

## Generic Classes

```java
/**
 * @param <T> meaning of T
 */
public class MyClass<T> {
    // T can be used almost everywhere a data type is needed
    private T value;
    ...
}
```

- The type parameter is declared **after the class name**.
- Multiple type parameters are separated by commas: `class Pair<K, V>`.
- When a generic class also has a generic method with a different type, use a different name to avoid confusion:

```java
class Container<T> { <V> void process(V val); }  // clear: T ≠ V
class Container<T> { <T> void process(T val); }  // confusing: shadows outer T
```

---

## Upper Bound — `extends`

Constrains a type parameter so that only types that are a subtype of a given class/interface are accepted.

### On a class

```java
public class SortedList<T extends Comparable<T>> {
    // T must implement Comparable<T>
    // without a bound, T only has access to Object methods: equals, hashCode, toString
}
```

### On a method (named type parameter form)

```java
public <SubType extends T> void addAll(List<SubType> list) {
    for (SubType item : list) {
        this.add(item);
    }
}
```

### On a method (wildcard form — preferred when type isn't returned)

```java
public void addAll(List<? extends T> list) { ... }
// ? stands for "any subtype of T"
```

Both forms allow passing a `List<CheckingAccount>` into an `addAll` expecting `List<? extends Account>`.

### Multiple bounds

```java
<T extends ClassA & InterfaceB & InterfaceC>
// Always "extends" even for interfaces
// Use & to separate bounds, , to separate multiple type parameters: <E, T>
```

### The `Comparable<? super T>` pattern

When a subclass inherits `compareTo` from its superclass rather than implementing it directly, the simple bound `T extends Comparable<T>` is too strict. Use:

```java
public class SortedList<T extends Comparable<? super T>> { ... }
```

This means: *T must be comparable, but the comparison method may be defined on a supertype of T.*

Example — `CheckingAccount extends Account implements Comparable<Account>`:

```
CheckingAccount extends(implements) Comparable<Account super CheckingAccount>
```

Without `? super T`, creating a `SortedList<CheckingAccount>` would fail even though `CheckingAccount` can be compared via its superclass `Account`.

---

## Lower Bound — `super`

Constrains a wildcard so that only types that are a **supertype** of a given type are accepted. **Only available with wildcards** — not with named type parameters.

### Use case: inserting elements into a more general collection

```java
// Works only for List<T>:
public void insertInto(List<T> target) { ... }

// Works for List<T>, List<SuperclassOfT>, List<Object>, etc.:
public void insertInto(List<? super T> target) { ... }
```

Example:

```java
SortedList<CheckingAccount> giroList = new SortedList<>();
giroList.add(g1);
giroList.add(g2);

List<Account> accountList = new LinkedList<>();
giroList.insertInto(accountList);   // OK — Account is a supertype of CheckingAccount
```

Without the lower bound, passing `List<Account>` to a method expecting `List<CheckingAccount>` would be a compile error because type parameters are not covariant.

---

## Upper vs. Lower Bound — Quick Summary

| Bound | Syntax | Meaning | Typical use case |
|-------|--------|---------|-----------------|
| Upper | `<? extends T>` | T or any subtype | Reading from a collection |
| Lower | `<? super T>` | T or any supertype | Writing/inserting into a collection |
| Both | `<T extends Comparable<? super T>>` | T is comparable, comparison may use a supertype | Sorted data structures |

---

## What Is Not Allowed

| What | Why |
|------|-----|
| `class MyException<T> extends Exception` | Generic exceptions (Throwables) are forbidden |
| `new T()` | Cannot instantiate a type parameter (workaround: reflection) |
| `if (x instanceof MyClass<Integer>)` | `instanceof` with parameterised types is forbidden (workaround: reflection) |
| `T[] arr = new T[5]` | Cannot create arrays of type parameters |
| `public static T method()` | `T` is an instance-level parameter — cannot be used in static context |

What **is** allowed:

```java
MyClass<?>[] arr = new MyClass<?>[5];   // array of wildcard type: OK
```

---

## Method Overloading Pitfall

Overloading a method for both `T` and a concrete type is legal, but causes ambiguity when `T` is that same concrete type:

```java
public class Box<T> {
    public void process(T param) { ... }
    public void process(String param) { ... }
}

Box<String> box = new Box<>();
box.process("hello");   // compile error — ambiguous: which overload?
```

---

## Exercise Examples

### `Interval<T>` class

```java
public class Interval<T extends Comparable<? super T>> {

    private T lower, upper;

    public Interval(T lower, T upper) { ... }
}
```

`T extends Comparable<? super T>` means:
- T must implement `Comparable` (so its elements can be compared).
- The comparison can be defined on a supertype of T — not just T itself.

#### Intersection of two intervals

```java
public <T1 extends T> Interval<T> intersect(Interval<T1> other) {
    T min = this.lower;
    if (min.compareTo(other.lower) < 0)
        min = other.lower;

    T max = this.upper;
    if (max.compareTo(other.upper) > 0)
        max = other.upper;

    return new Interval<>(min, max);
}
```

- `<T1 extends T>` declares that the parameter's element type must be a subtype of `T`.
- This is necessary because boundary values from `other` may end up in the returned `Interval<T>`.
- Equivalent wildcard form: `public Interval<T> intersect(Interval<? extends T> other)`

### Merging two lists of compatible types

**Goal:** merge a `List<Integer>` and a `List<Double>` into a `List<Number>`.

**Wildcard form (preferred):**

```java
public static <T> List<T> merge(List<? extends T> l1, List<? extends T> l2) {
    List<T> result = new LinkedList<>(l1);
    result.addAll(l2);
    return result;
}
```

- `<T>` is the common supertype (inferred as `Number` by the compiler).
- `? extends T` allows passing `List<Integer>` or `List<Double>` where `List<Number>` is expected.

**Named type parameter form (more explicit):**

```java
public static <T, T1 extends T, T2 extends T> List<T> merge(List<T1> l1, List<T2> l2) {
    List<T> result = new LinkedList<>();
    for (T1 o : l1) result.add(o);
    for (T2 o : l2) result.add(o);
    return result;
}
```

Usage:

```java
List<Integer> ints = new LinkedList<>(List.of(1, 2, 3));
List<Double>  doubles = new LinkedList<>(List.of(1.1, 2.2, 3.3));

List<Number> combined = merge(ints, doubles);
```

### Reading generic type parameters in a method chain (Mockito example)

```java
Konto testAccount = Mockito.when(Mockito.mock(CheckingAccount.class).withdraw(100))
    .thenThrow(AccountLockedException.class)
    .getMock();
```

Broken down with explicit type arguments:

```java
// mock<X>(Class<X>)  →  X = CheckingAccount
CheckingAccount g = Mockito.<CheckingAccount>mock(CheckingAccount.class);

// when<Y>(Y methodCall)  →  Y = Boolean (return type of withdraw)
OngoingStubbing<Boolean> stub = Mockito.<Boolean>when(g.withdraw(100));

// thenThrow(Class<? extends Throwable>)  →  ? = AccountLockedException
OngoingStubbing<Boolean> stub2 = stub.thenThrow(AccountLockedException.class);

// getMock<M>()  →  M inferred from target variable = Account
Account testAccount = stub2.<Account>getMock();
```

The compiler infers each type argument from context — explicit forms are only needed when inference fails or for clarity.
