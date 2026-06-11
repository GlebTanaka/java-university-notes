# Chapter 9 — Lambdas & Streams

## Functional Interfaces

A **functional interface** (or functional type) is an interface with exactly one abstract method.

- When a class implements it, there is no ambiguity about which method must be implemented.
- This property enables lambda expressions: the method name can be omitted.
- An abstract class with only one abstract method is **not** a functional interface.

---

## Lambda Expressions

### Normal approach

```java
// 1. Functional interface
public interface SomeInterface {
    public int method(int x);
}

// 2. A class that implements it
public class Xyz implements SomeInterface {
    public int method(int x) { ...; return value; }
}

// 3. Use it somewhere
SomeInterface var = new Xyz();
// — or as an anonymous class
```

### Lambda approach

No implementing class is needed — create and implement in one step:

```java
SomeInterface var = (int x) -> { ...; return value; };
// Parameter(s) in round brackets, body in curly braces
```

A lambda expression is a nameless method. Its type is the functional interface; the method name lives in the interface. The class of the created object is genuinely anonymous.

Lambdas are only permitted where the compiler can infer the interface type — from a variable's declared type or a parameter's declared type.

### Step-by-step derivation (Runnable example)

**Step 1 — named class**

```java
private static class Greeting implements Runnable {
    public void run() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {}
        System.out.println("Hello!");
    }
}

// usage:
Runnable r = new Greeting();
Thread t = new Thread(r);
t.start();
```

**Step 2 — anonymous class** (skip the named class, substitute the body inline)

```java
Runnable r = new Runnable() {
    public void run() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {}
        System.out.println("Hello!");
    }
};
```

**Step 3 — drop `new Runnable() { ... }`** (`new` is unnecessary; the type is already declared)

**Step 4 — drop `public void run`** (only one abstract method in `Runnable`, so the name can be omitted)

**Step 5 — add the arrow** `->` between parameter list and body

```java
Runnable r = () -> {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {}
    System.out.println("Hello!");
};
Thread t = new Thread(r);
t.start();
```

What remains: parameter parentheses, the arrow, and the method body.

---

## Lambda Shorthand Rules

| Rule | Example |
|------|---------|
| Parameter types may all be omitted | `(x) -> { ...; return v; }` |
| Single parameter — drop the parentheses | `x -> x + 1` |
| Body is only `{ return expr; }` — drop braces and `return` | `x -> x + 1` |
| `void` method with a single statement — drop the braces | `x -> System.out.println(x)` |

```java
// Full form
SomeInterface var = (int x) -> { ...; return value; };

// No type annotation
SomeInterface var = (x) -> { ...; return value; };

// Single-expression return, single parameter
SomeInterface var = x -> x + 1;
```

### Example — `Comparator`

```java
// Anonymous class
Comparator<Customer> cmp = new Comparator<Customer>() {
    @Override
    public int compare(Customer a, Customer b) {
        return a.getAddress().compareTo(b.getAddress());
    }
};

// Step 1: drop new Comparator<Customer>() { ... }
// Step 2: drop public int compare(DataType)  →  keep only parameters
// Step 3: body is a single return — drop braces
Comparator<Customer> cmp = (a, b) -> a.getAddress().compareTo(b.getAddress());
```

---

## Standard Library Functional Interfaces

### `Predicate<T>`

```java
// Interface:
public interface Predicate<T> { boolean test(T t); }

// Anonymous class:
Predicate<String> pred = new Predicate<String>() {
    @Override public boolean test(String s) { return s.contains("X"); }
};

// Lambda:
Predicate<String> pred = s -> s.contains("X");
```

### `Consumer<T>`

```java
// Interface:
public interface Consumer<T> { void accept(T t); }

// Lambda:
Consumer<String> consumer = s -> System.out.println(s);

// Method reference:
Consumer<String> consumer = System.out::println;
```

### `Function<T, R>`

```java
// Interface:
public interface Function<T, R> { R apply(T t); }

// Lambda:
Function<String, String> fn = s -> s.toLowerCase();

// Method reference:
Function<String, String> fn = String::toLowerCase;
```

### `BinaryOperator<T>`

```java
// Interface:
public interface BinaryOperator<T> { T apply(T t1, T t2); }

// Lambda:
BinaryOperator<String> concat = (a, b) -> a.concat(b);

// Method reference:
BinaryOperator<String> concat = String::concat;
```

### `Comparator<T>`

```java
// Interface:
public interface Comparator<T> { int compare(T t1, T t2); }

// Lambda:
Comparator<String> byLength = (o1, o2) -> o1.length() - o2.length();
```

### `Runnable`

```java
// Interface:
public interface Runnable { void run(); }

// Lambda:
Runnable r = () -> System.out.println("lambda!");

// Method reference (requires a separate static method):
public class App {
    private static void run() { System.out.println("lambda!"); }

    public static void main(String[] args) {
        Runnable r = App::run;
    }
}
```

---

## Method References

| Lambda | Method reference |
|--------|-----------------|
| `(int x) -> { return SomeClass.method(x); }` | `SomeClass::method` |
| `(int x) -> { return obj.method(x); }` | `obj::method` |
| `(int x) -> { return new SomeClass(x); }` | `SomeClass::new` |
| `(SomeClass o, int x) -> { return o.method(x); }` | `SomeClass::method` |

The return value of each is a call to a method on an object.

**Tip:** When in doubt, write an anonymous class first — the IDE can then generate both the lambda and the method reference automatically.

---

## Custom Functional Interface Example

```java
// A conversion interface
interface Transform {
    String convert(Customer c);
}

// A method that accepts it
public static void printList(List<Customer> list, Transform fmt) {
    for (Customer c : list) {
        System.out.println(fmt.convert(c));
    }
}

// Call it — print names only
Transform t = customer -> customer.getName();
printList(list, t);

// Method reference form:
printList(list, Customer::getName);
```

---

## Variable Access in Lambdas

A lambda expression has access to:
- All fields of the enclosing class (including `private`)
- The current object of the enclosing class via `this` (unlike anonymous classes, where it would be `OuterClass.this`)
- **Effectively final** local variables and parameters of the enclosing method — the lambda may **not** modify them

```java
int x = 0;
Runnable r = () -> System.out.println(x);  // ok: reads x
// x = 1;  // would break "effectively final" — compiler error inside lambda
```

Mutating an object *through* a variable is allowed (but conflicts with the stateless design principle):

```java
Account acc = new CheckingAccount();
Runnable r = () -> {
    try {
        System.out.println(acc.withdraw(50));  // object mutation is fine
    } catch (LockedException e) { e.printStackTrace(); }
};
```

---

## Streams

When objects are stored in a collection, common operations are:
- Filter objects by a criterion
- Apply a transformation to every object
- Aggregate objects into a summary value
- Sort objects

### Creating a Stream

```java
// From any Collection:
Stream<E> s = collection.stream();

// Parallel stream (uses multiple processor cores):
Stream<E> s = collection.parallelStream();

// Note: Map has no stream() — choose keySet or values first:
Stream<Account> s = accountMap.values().stream();
Stream<Long>    s = accountMap.keySet().stream();
```

Also available: `Stream.of(...)`, `Stream.Builder`, `LongStream.range(from, to)`.

---

## Intermediate Operations

Intermediate operations return a new `Stream<T>` and can be chained.

| Method | Purpose |
|--------|---------|
| `filter(Predicate<T> p)` | Keep only elements matching `p` |
| `distinct()` | Remove duplicates |
| `map(Function<T,R> f)` | Transform each element from `T` to `R` |
| `sorted()` | Natural order (requires `Comparable`) |
| `sorted(Comparator<T> c)` | Custom order |
| `concat(Stream<T> a, Stream<T> b)` | Merge two streams |
| `limit(int n)` | Keep only the first `n` elements |
| `mapToDouble(...)` | Map to a `DoubleStream` (enables `.sum()`, `.average()`, etc.) |

Intermediate operations are **lazy** — they are not applied until a terminal operation is called.

---

## Terminal Operations

Terminal operations close the stream; it cannot be used again afterwards (`IllegalStateException`).

| Method | Purpose |
|--------|---------|
| `forEach(Consumer<T> c)` | Apply `c` to every element |
| `reduce(T start, BinaryOperator<T> b)` | Aggregate elements into one value |
| `anyMatch(Predicate<T> p)` | `true` if at least one element matches |
| `allMatch(Predicate<T> p)` | `true` if every element matches |
| `collect(Collector<T,A,R> c)` | Materialise stream into a collection |

---

## Stream Pipeline Pattern

```
1. Create stream         collection.stream()
2. Intermediate ops      .filter(...).map(...).sorted(...)
3. Terminal op           .forEach(...)  or  .collect(...)
```

Diagram:

```java
list.stream()
    .map(x -> x.toString())      // intermediate — produces Stream<String>
    .forEach(System.out::println); // terminal
```

The terminal operation triggers all intermediate operations (lazy evaluation).

---

## Worked Examples

### Example 1 — Print customer names

```java
public static void printList(List<Customer> list) {
    list.stream()
        .map(Customer::toString)
        .forEach(System.out::println);
}
```

### Example 2 — Sum account balances

```java
Map<Long, Account> accounts = new HashMap<>();
// ...

// for-each version
double total = 0.0;
for (Account a : accounts.values())
    total += a.getBalance();

// Stream version
double total = accounts.values().stream()
    .map(Account::getBalance)
    .reduce(0.0, Double::sum);

// Shorter — mapToDouble gives a DoubleStream with .sum()
double total = accounts.values().stream()
    .mapToDouble(Account::getBalance)
    .sum();

// Parallel stream (faster on large collections)
double total = accounts.values().parallelStream()
    .reduce(0.0,
            (sum, acc) -> sum + acc.getBalance(),
            Double::sum);
// The second BinaryOperator merges results from different threads.
```

### Example 3 — Sort customers by birthday

```java
// Lambda comparator
Comparator<Customer> byBirthday =
    (a, b) -> a.getBirthday().compareTo(b.getBirthday());

// Using Comparator.comparing (more readable)
Comparator<Customer> byBirthday = Comparator.comparing(Customer::getBirthday);

// Classic sort
Collections.sort(list, byBirthday);

// Stream version — produces a new sorted List
List<Customer> sorted = list.stream()
    .sorted(byBirthday)
    .collect(Collectors.toList());
```

### Example 4 — anyMatch

```java
boolean hasAName = accounts.values().stream()
    .map(Account::getOwner)
    .anyMatch(c -> c.getName().startsWith("A"));
```

### Example 5 — Join all names into a single String

```java
String result = accounts.values().stream()
    .map(acc -> acc.getOwner().getName())
    .reduce("", (acc, name) -> acc.concat(name) + System.lineSeparator());
```

### Example 6 — Account number + customer name pairs

```java
String result = accounts.keySet().stream()
    .map(nr -> {
        String name = accounts.get(nr).getOwner().getName();
        return nr.toString() + ", " + name;
    })
    .reduce("", (prev, next) -> prev.concat(next) + System.lineSeparator());
```

---

## `collect()` and `Collectors`

`collect()` folds a stream into a container using a `Collector`.

```java
// Signature:
<R, A> R collect(Collector<? super T, A, R> collector)
```

Writing a `Collector` by hand is verbose — use the `Collectors` helper class:

| Call | Result |
|------|--------|
| `Collectors.toList()` | All elements into a `List` |
| `Collectors.toSet()` | All elements into a `Set` |
| `Collectors.toMap(keyFn, valueFn)` | All elements into a `Map` |
| `Collectors.counting()` | Count of elements as `Long` |
| `Collectors.joining()` | Concatenate `String` elements |

```java
List<Customer> list = stream.collect(Collectors.toList());
Set<Customer>  set  = stream.collect(Collectors.toSet());
```

---

## Stream Operation Rules

Operations passed to stream methods must be:
- **Non-interfering** — must not modify the underlying collection during processing
- **Stateless** — must not depend on variables outside the operation that could change during processing (the passed lambda must not modify outer variables)

---

## Exercise — Bank Class Stream Methods

### Lock all overdrawn accounts

```java
public void lockInsolventAccounts() {
    accountList.values().stream()
        .filter(acc -> acc.getBalance() < 0)
        .forEach(acc -> acc.setLocked(true));
}
```

### Get customers whose balance meets a minimum

```java
public List<Customer> getCustomersAboveBalance(double minimum) {
    return accountList.values().stream()
        .filter(acc -> acc.getBalance() >= minimum)
        .map(Account::getOwner)
        .collect(Collectors.toList());
}
```

### Get customer birthdays as a formatted String

```java
public String getCustomerBirthdays() {
    return accountList.values().stream()
        .map(Account::getOwner)
        .distinct()
        .sorted(Comparator.comparing(Customer::getBirthday))
        .map(c -> c.getName() + " (" + c.getBirthday() + ")")
        .reduce("Birthday list" + System.lineSeparator(),
                (acc, line) -> acc.concat(line) + System.lineSeparator());
}

// Alternative using Collectors.joining()
public String getCustomerBirthdays2() {
    return accountList.values().stream()
        .map(Account::getOwner)
        .distinct()
        .sorted((a, b) -> a.getBirthday().compareTo(b.getBirthday()))
        .map(c -> c.getName() + " (" + c.getBirthday() + ")" + System.lineSeparator())
        .collect(Collectors.joining());
}
```

### Find free account numbers in a range

```java
private static final long FIRST_ACCOUNT_NUMBER = 0L;
private static final long LAST_ACCOUNT_NUMBER  = 10L;

public List<Long> getFreeAccountNumbers() {
    return LongStream.range(FIRST_ACCOUNT_NUMBER, LAST_ACCOUNT_NUMBER)
        .filter(nr -> getAllAccountNumbers().contains(nr))
        .boxed()                          // long → Long (for collect)
        .collect(Collectors.toList());
}
```

`LongStream.range(from, to)` creates a stream of primitive `long` values. `.boxed()` wraps them to `Long` objects for use with `Collectors.toList()`.
