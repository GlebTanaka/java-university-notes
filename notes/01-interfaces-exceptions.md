# Chapter 1 — Interfaces, JavaDoc, Exceptions, Comparable vs Comparator

## Interfaces

Reference: http://www.angelikalanger.com/Articles/EffectiveJava/72.Java8.DefaultMethods/72.Java8.DefaultMethods.html

A class implements an interface like this:

```java
public class MyClass implements InterfaceName, Interface02, ... {
    @Override
    public void interfaceMethod1() { ... }
    @Override
    public int interfaceMethod2() { ... }

    ...
}
```

### Usage

- The interface name can be used as a data type anywhere:
  - as method parameters
  - as return types
  - for variable and field declarations
- You cannot call a constructor of an interface directly — you need a class that implements it.

### `instanceof`

The `instanceof` operator checks whether an object is assignment-compatible with a given type. Use it sparingly; there are often better alternatives.

```java
if (object instanceof SomeClass) { ... }
// SomeClass can also be an interface
```

An object is assignment-compatible with:
- its own class and all superclasses
- all interfaces it implements

`null instanceof SomeClass` always returns `false`.

### Interface Structure

All members are implicitly `public` — the keyword can be omitted.

```java
public interface InterfaceName {

    DataType CONSTANT = value;   // implicitly static final

    void method();               // abstract method

    default void defaultMethod() {
        // can call other interface methods
        this.method();
        // implementing classes may, but do not have to, override default methods
    }

    static void staticMethod() {
        // works like a static method in a regular class
    }
}
```

### Default Methods

- Allow you to write algorithms directly inside the interface, using the abstract methods defined there.
- Goals: better organisation, less boilerplate in implementing classes.

### Static Methods

Work just like static methods in a regular class.

---

## JavaDoc

See the official tutorial for JavaDoc syntax and usage.

---

## Exception Handling

Reference: http://tutorials.jenkov.com/java-exception-handling/index.html

### 1. `try-catch`

Use when the exception can be resolved within the method itself.

```java
try {
    // statement that may throw
} catch (Exception e) {
    // handle the error
}
// execution continues here whether or not an exception occurred
```

### 2. `throws` in the Method Signature

Use when the method cannot handle the exception itself — it breaks out of the method and passes responsibility to the caller.

```java
public void method() throws SomeException { ... }
```

- An exception can be thrown somewhere inside the method.
- The method aborts immediately at the point of failure.
- The calling code must handle (or re-throw) the exception.

---

## Console Input with Scanner

```java
import java.util.Scanner;

public static void main(String[] args) {

    Scanner s = new Scanner(System.in);  // System.in is a static constant

    s.useDelimiter(System.lineSeparator());  // System.lineSeparator() is a static method

    String input = s.next();     // reads a string token
    int i = s.nextInt();         // reads an integer
}
```

---

## Summary

- **Interfaces** allow you to write algorithms where small implementation details can be filled in later.
- **`instanceof`** tests whether an object is assignment-compatible with a given type.
- **Default methods** in interfaces implement algorithms based on the abstract methods declared in the same interface.
- Each method should have exactly one responsibility — separate processing from input/output.
- Every class, method, and field should be documented with JavaDoc.
- **Exception handling**: use `try-catch` when the exception can be handled within the method; use `throws` when it cannot.
- Exceptions abort the method (or the enclosing `try` block) immediately.

---

## Comparable vs Comparator

Both are interfaces used to sort collections.

| Feature | `Comparable` | `Comparator` |
|---------|-------------|--------------|
| Sorting sequences | Single (one natural order) | Multiple (different orderings possible) |
| Class modification | Modifies the original class | Does not modify the original class |
| Sort method | `compareTo(other)` | `compare(o1, o2)` |
| Used with | `Collections.sort(list)` | `Collections.sort(list, comparator)` |
| Package | `java.lang` | `java.util` |

### How `compareTo()` and `compare()` Work

Both follow the same contract — compare one aspect of object1 to object2:

```
object1.compareTo(object2):
    returns  0  if object1 == object2
    returns  1  if object1 >  object2
    returns -1  if object1 <  object2
```

For numeric fields (`int`, `double`, etc.) you can use the built-in helpers:

```java
@Override
public int compareTo(Student other) {
    return Integer.compare(this.age, other.age);
}
```

When comparing two objects:

```java
object1.compareTo(object2) == 0  // expect equal
object1.compareTo(object2) > 0   // expect object1 greater
object1.compareTo(object2) < 0   // expect object1 smaller
```

### The Core Idea

**Comparable** — implement inside the class itself when there is one natural ordering:
- The class implements `Comparable<T>` and overrides `compareTo()`.
- Objects can then only be sorted by that one attribute.

**Comparator** — implement in separate helper classes when multiple orderings are needed:
- The original class stays unchanged.
- Write one `Comparator<T>` class per attribute you want to sort by, each overriding `compare()`.

### Example — `Comparable`

```java
public class Student implements Comparable<Student> {

    int studentId;
    int age;
    String name;

    public Student(int studentId, int age, String name) { ... }

    @Override
    public int compareTo(Student other) {
        return Integer.compare(this.age, other.age);
    }
}
```

Sorting in `main`:

```java
List<Student> students = new ArrayList<>();
students.add(...);
// ...

Collections.sort(students);  // uses compareTo() — sorts by age
```

### Example — `Comparator`

The `Student` class itself implements nothing:

```java
public class Student {
    int studentId;
    int age;
    String name;

    public Student(int studentId, int age, String name) { ... }
}
```

Separate comparator for age:

```java
public class AgeComparator implements Comparator<Student> {
    @Override
    public int compare(Student o1, Student o2) {
        return Integer.compare(o1.age, o2.age);
    }
}
```

Separate comparator for name (String already implements `Comparable`, so we can reuse `compareTo`):

```java
public class NameComparator implements Comparator<Student> {
    @Override
    public int compare(Student o1, Student o2) {
        return o1.name.compareTo(o2.name);
    }
}
```

Sorting in `main`:

```java
List<Student> students = new LinkedList<>();
students.add(...);

// sort by name
Collections.sort(students, new NameComparator());
students.sort(new NameComparator());  // shorter alternative

// sort by age
students.sort(new AgeComparator());
```

---

## Exercise Example — Finite State Machine

Project structure:

```
.
├── FiniteAutomaton.java
└── testfiles/
    ├── GenericAutomaton.java
    ├── Main.java
    └── VariableNameAutomaton.java
```

### `FiniteAutomaton.java` (Interface)

```java
package ueb01.endlicher_automat;

/**
 * Represents a deterministic finite automaton.
 */
public interface FiniteAutomaton {

    /**
     * Starts the automaton, i.e. resets it to the initial state.
     */
    void start();

    /**
     * Performs a state transition from the current state to the next,
     * based on the given character.
     *
     * @param character the current character of the input word
     */
    void transition(char character);

    /**
     * Checks whether the automaton is currently in an accepting state.
     *
     * @return true if the current state is accepting
     */
    boolean accepts();

    /**
     * Runs the automaton on the given string.
     *
     * @param word the input string to test
     * @return true if the word is accepted
     */
    default boolean test(String word) {
        this.start();
        for (int i = 0; i < word.length(); i++) {
            this.transition(word.charAt(i));
        }
        return this.accepts();
    }
}
```

How `test()` works:
1. Calls `start()` on the current object to reset to the initial state.
2. Iterates over the string character by character using `charAt(i)`.
3. Calls `transition(char)` for each character — the implementing class handles the state change.
4. Returns the result of `accepts()`.

### `VariableNameAutomaton.java`

Accepts valid C variable names (must start with a letter or `_`, followed by letters, digits, or `_`).

```java
package ueb01.endlicher_automat.testdateien;

import ueb01.endlicher_automat.EndlicherAutomat;

/**
 * Accepts valid variable names in C.
 */
public class VariableNameAutomaton implements FiniteAutomaton {

    private int current = 0;

    @Override
    public void start() {
        this.current = 0;
    }

    @Override
    public void transition(char ch) {
        if (current == 0) {
            if (('a' <= ch && ch <= 'z')
            || ('A' <= ch && ch <= 'Z')
            || ch == '_')
                current = 1;
            else
                current = -1;
        }
        if (current == 1) {
            if (('a' <= ch && ch <= 'z')
            || ('A' <= ch && ch <= 'Z')
            || ch == '_'
            || ('0' <= ch && ch <= '9'))
                current = 1;
            else
                current = -1;
        }
    }

    @Override
    public boolean accepts() {
        return current == 1;
    }
}
```

- `current` defaults to `0` (initial state).
- `start()` resets it to `0`.
- `accepts()` returns `true` only when `current == 1`.
- `transition()` sets `current` to `1` if the character is valid, otherwise `-1`.

Usage in `main`:

```java
FiniteAutomaton va = new VariableNameAutomaton();
System.out.println(va.test("abc"));   // true
System.out.println(va.test("1abc"));  // false
```

The `test()` default method from the interface coordinates the calls — without the implementing methods, nothing works.

### `GenericAutomaton.java`

A configurable automaton where the transition function is passed in via the constructor.

```java
package ueb01.endlicher_automat.testdateien;

import ueb01.endlicher_automat.EndlicherAutomat;
import java.util.HashMap;
import java.util.Map;

/**
 * A general finite automaton whose transition function is supplied at construction time.
 * The initial state is always 0.
 *
 * @param transitions  the transition function; transitions[i] contains the transitions
 *                     for state i, formatted as: "char1->nextState, char2->nextState, ..."
 *                     Use -1 to denote an error state with no outgoing transitions.
 * @param accepting    array of accepting state numbers
 * @throws IndexOutOfBoundsException if transition strings are malformed
 * @throws NumberFormatException     if a target state is not a valid integer
 */
public class GenericAutomaton implements FiniteAutomaton {

    private int state;
    private int[] accepting;
    private Map<Integer, Map<Character, Integer>> function = new HashMap<>();

    public GenericAutomaton(String[] transitions, int[] accepting) {
        this.accepting = accepting;
        for (int s = 0; s < transitions.length; s++) {
            Map<Character, Integer> fromState = new HashMap<>();
            String[] parts = transitions[s].split(", ");
            for (String part : parts) {
                char ch = part.charAt(0);                        // e.g. "a->1" → 'a'
                int next = Integer.parseInt(part.substring(3));  // e.g. "a->1" → 1
                fromState.put(ch, next);
            }
            function.put(s, fromState);
        }
    }
```

Constructor walkthrough:
1. Iterates over the `transitions` string array.
2. For each state, creates a `Map<Character, Integer>`.
3. Splits the transition string on `", "` and extracts the character (index 0) and target state (after `"->"`, i.e. `substring(3)`).
4. Stores `character → targetState` in the inner map.
5. Stores `state → innerMap` in the outer map.

```java
    @Override
    public void start() {
        state = 0;
    }

    @Override
    public void transition(char ch) {
        try {
            state = function.get(state).get(ch);
        } catch (NullPointerException e) {
            state = -1;  // no transition defined → error state
        }
    }
```

`transition()` looks up the current state in the outer map, then looks up the character in the inner map. If either lookup fails (no transition defined), the state becomes `-1`.

```java
    @Override
    public boolean accepts() {
        for (int i = 0; i < accepting.length; i++)
            if (state == accepting[i])
                return true;
        return false;
    }
}
```

Usage in `main`:

```java
// accepts words over {a, b} that end with 'a'
FiniteAutomaton fa = new GenericAutomaton(
    new String[] {"a->1, b->0", "a->1, b->0"},
    new int[] {1}
);
System.out.println(fa.test("ab"));   // false
System.out.println(fa.test("aba"));  // true
```
