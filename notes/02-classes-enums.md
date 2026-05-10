# Chapter 2 — Classes, Enums, Shutdown Hook

## Class Structure

```java
public class ClassName {           // class names are capitalized

    private DataTypeA field1;      // fields are lowercase; private, protected, or package-private

    public ReturnType method(DataTypeB p, ...) {
        ...;
        return value;
    }

    public ClassName(DataTypeA p, ...) {
        this.field1 = p;           // use this to refer to the current object
    }
}
```

---

## `final`

| Context | Meaning |
|---------|---------|
| Field | Constant — cannot be reassigned after initialization |
| Method | Cannot be overridden in subclasses |
| Class | Cannot be subclassed |

`String` is a well-known example of a `final` class.

---

## Static Members

```java
public class ClassName {

    public static Type name;

    static {
        // static initializer block — runs once when the class is loaded
        // used for initialization logic that can't be done inline
    }
}
```

- Static members exist once per class, not once per object.
- No object (no constructor call) is needed to access them.
- Access via the class name: `ClassName.name`

### Static Initializer Block

```java
private static String GREETING;

static {
    if (Locale.getDefault().getCountry().equals("DE"))
        GREETING = "Hallo Benutzer!";
    else
        GREETING = "Dear Customer!";
}
```

- Acts like a constructor for static fields.
- Note: if you use a static initializer block, the field cannot be `final`.
- Avoid using this for regular output — the initialization order is not guaranteed.

A directly initialized static field looks like this:

```java
public static final Customer SAMPLE = new Customer("Max", "Mustermann", "home", LocalDate.now());
```

---

## Variables of Class Types

```java
// Declaration and instantiation
import some.package.*;

ClassName variableName;
variableName = new ClassName(...);

// Access (subject to visibility)
variableName.field1;
variableName.method(...);
```

---

## Inheritance

```java
public class Sub extends Super implements Interface {

    // extends always comes before implements

    public Sub(...) {
        super(...);             // call the superclass constructor
        // additional initialization
    }
    // additional fields and methods
}
```

---

## Abstract Classes

- Declared with the `abstract` keyword — no objects can be instantiated from them.
- The constructor can only be called via `super(...)` from a subclass.
- Abstract methods have no implementation and must be overridden in subclasses.

---

## Exceptions: Checked vs Unchecked

| Type | Behavior |
|------|----------|
| `Exception` (checked) | Must be declared with `throws` and handled with `try-catch` |
| `RuntimeException` (unchecked) | Does not need to be declared or handled |

---

## Polymorphism

```java
SuperClass variable = new SubClass(...);
// subclass objects are assignment-compatible with superclass variables

variable.overriddenMethod();
// the method of the actual object is called (SubClass), not the declared type
```

The runtime type of the object determines which method runs — not the type of the variable.

### Example with `toString()`

```java
Account k = new CheckingAccount();
System.out.println(k.toString());
// calls toString() from CheckingAccount, even though k is declared as Account
```

Even if `Account` has a method like:

```java
public void print() {
    System.out.println(this.toString());
}
```

Calling `k.print()` still calls `toString()` from `CheckingAccount` — the actual object type always wins.

---

## Memory and References

Primitive types are copied — two independent values:

```java
int a = 100;
int b = a;
a += 50;
System.out.println(b); // still 100
```

Object variables are references — they point to the same object:

```java
Account ka = new CheckingAccount();
Account kb = ka;          // only one object exists; kb is an alias for ka
ka.deposit(50);
System.out.println(kb.getBalance()); // 50 — same object
```

---

## Enums

Enums are closed collections of named constants. The constants are not primitives — they are objects with their own fields and methods.

When to use enums: the set of values is fixed and unlikely to change (e.g. days of the week, months, German federal states).

### Basic Enum

```java
public enum Color {
    RED, GREEN, BLUE;
}
```

This is conceptually equivalent to:

```java
public class Color extends Enum {
    public static final Color RED   = new Color();
    public static final Color GREEN = new Color();
    public static final Color BLUE  = new Color();
    private Color() {}
}
```

### Usage

```java
Color c = Color.RED;                   // direct access
Color c = Color.valueOf("RED");        // from a string

String name   = c.name();             // "RED"
int    index  = c.ordinal();          // 0 (position in declaration order)
```

`name()` and `ordinal()` are inherited from `Enum` and are `final` — they cannot be overridden.

### `values()`

Returns an array of all constants in declaration order:

```java
for (Color c : Color.values()) {
    System.out.println(c);
}

Color[] all = Color.values();
Color second = all[1];    // GREEN
```

### Comparison

Use `==` directly — `equals()` is not needed:

```java
if (variable == Color.RED) { ... }
```

### Enum with Fields

```java
public enum AccountType {

    CHECKING("Great overdraft"),
    SAVINGS("High interest"),
    FIXED("Fixed rate");

    private String tagline;

    private AccountType(String tagline) {   // constructor is always private
        this.tagline = tagline;
    }
}
```

- Each constant is a constructor call: `CHECKING("Great overdraft")` compiles to `new AccountType("Great overdraft")`.
- The constructor is implicitly `private` — external code cannot create new constants.
- This ensures the enum remains a closed set.
- Naming convention: constants are written in ALL_CAPS (static + final).

### Enum with Methods

```java
public enum AccountType {

    CHECKING("Great overdraft"), SAVINGS("High interest"), FIXED("Fixed rate");

    private String tagline;

    private AccountType(String tagline) {
        this.tagline = tagline;
    }

    public String getTagline() {
        return this.tagline;    // use this to access the current constant's field
    }
}
```

Calling a method on a constant:

```java
AccountType.CHECKING.getTagline();   // "Great overdraft"
```

Prefer the field-based approach over a `switch(this)` inside the method — it is simpler and less error-prone.

### Overriding `toString()`

```java
@Override
public String toString() {
    return this.name() + ": " + this.tagline;
}
```

### Iterating and Looking Up

```java
// iterate with index
AccountType[] all = AccountType.values();
for (int i = 0; i < all.length; i++) {
    System.out.println(all[i].ordinal() + ": " + all[i]);
}

// look up by name string
String input = "CHECKING";
AccountType chosen = AccountType.valueOf(input);
System.out.println(chosen);
```

---

## Object Cleanup — Shutdown Hook

### The Problem with `finalize()`

`finalize()` is called by the garbage collector before freeing an object's memory, but the timing is not guaranteed:

```java
@Override
protected void finalize() {
    try {
        // cleanup work (close files, release DB connections, etc.)
    } catch (Throwable t) {
        // exceptions are ignored
    } finally {
        super.finalize();   // let the superclass clean up too
    }
}
```

Because the timing is undefined, `finalize()` is unreliable for critical cleanup.

### Preferred Alternative: Shutdown Hook

Register a `Runnable` as a JVM shutdown hook — it runs when the JVM exits normally:

```java
Runnable r = new CleanupClass();
Thread t = new Thread(r);
Runtime.getRuntime().addShutdownHook(t);
```

#### Example — registering inside the constructor

```java
public Customer(...) {
    ...
    Runnable cleanupTask = new Destroyer();
    Thread t = new Thread(cleanupTask);
    Runtime.getRuntime().addShutdownHook(t);
}
```

#### The cleanup class (inner class)

```java
private class Destroyer implements Runnable {
    @Override
    public void run() {
        System.out.println("Customer " + getName() + " destroyed");
        // close files, database connections, etc.
    }
}
```

---

## Summary

- **Enums** are closed collections of static constants that are objects with fields and methods.
- The enum constructor is always `private`. It is called implicitly in the constant definitions.
- Inside enum methods, use `this` to access the current constant's fields.
- Built-in enum methods (non-overridable): `name()`, `ordinal()`, `valueOf()`, `values()`.
- **Static members** exist once per class. Rule of thumb: prefer static methods and constants over static variables.
- **All object variables are references** — assigning one to another does not copy the object.
- **Polymorphism**: the runtime type of the object determines which method runs, not the declared type of the variable.
