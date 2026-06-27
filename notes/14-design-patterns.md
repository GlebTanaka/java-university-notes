# Chapter 14 — Singleton, State & Decorator Patterns

## Singleton Pattern

### When to use it

> Only one instance of a class may exist in the entire program, and it must be globally accessible.

**Example:** A chocolate-making machine (`ChocolateMaker`) controls real hardware. Creating a second instance would mean the second object doesn't know the first has already been filled — the machine could overflow.

---

### Evolution of the solution

**Step 1 — no constructor:** The Java compiler then generates a default public constructor, so anyone can still create instances. Not a real solution.

**Step 2 — private constructor:** Prevents external instantiation *and* prevents subclassing.

```java
public class ChocolateMaker {
    private boolean empty;
    private boolean cooked;

    private ChocolateMaker() {      // private — no one outside can call new
        empty = true;
        cooked = false;
    }
}
```

But now nothing can create an instance at all — we need the class to create one itself.

**Step 3 — static instance + static accessor:**

```java
public class ChocolateMaker {

    private static ChocolateMaker instance = new ChocolateMaker();

    private ChocolateMaker() {
        empty = true;
        cooked = false;
    }

    public static ChocolateMaker getInstance() {
        return instance;
    }
}
```

Usage:
```java
ChocolateMaker one = ChocolateMaker.getInstance();
ChocolateMaker two = ChocolateMaker.getInstance();  // same object as 'one'
```

**Lazy initialization** (create on first request, useful when setup is expensive):

```java
public static ChocolateMaker getInstance() {
    if (instance == null) {
        // complex setup here if needed
        instance = new ChocolateMaker();
    }
    return instance;
}
```

### UML

```
+------------------------------------+
|          ChocolateMaker            |
+------------------------------------+
|  -instance: ChocolateMaker {static}|
+------------------------------------+
|  -ChocolateMaker()                 |
|  +getInstance(): ChocolateMaker    |
|             {static}               |
+------------------------------------+
```

### Caution

- Private constructor **prevents inheritance**.
- Lazy initialization is **not thread-safe** without synchronization — in multithreaded code, two threads can both see `instance == null` simultaneously and create two instances. Add `synchronized` or use an enum-based singleton for thread safety.

---

## State Pattern

### When to use it

> The same method call on an object should behave differently depending on the object's current state. The method calls themselves trigger state transitions.

---

### Problem — if/else state management

The `ChocolateMaker` has two boolean fields (`empty`, `cooked`) and both buttons check them with if/else chains:

```java
public void button1() {
    if (empty) {
        System.out.println("Add milk-chocolate mixture!");
        empty = false;
    } else if (!empty && !cooked) {
        System.out.println("Start cooking!");
    } else if (cooked) {
        System.out.println("Already done!");
    }
}

public void button2() {
    if (!empty && cooked) {
        System.out.println("Enjoy the chocolate!");
        empty = true;
    } else if (empty) {
        System.out.println("Nothing inside!");
    } else {
        System.out.println("Cook it first!");
    }
}
```

State transitions:
```
empty ---(button1)---> filled ---(button1)---> cooked
  ↑                                               |
  +-------------------(button2)------------------+
```

**Problems:**
- Adding a new state means editing every method.
- "Encapsulate what varies" — the behaviour per state varies and should be extracted.

---

### Solution — State objects

Each state gets its own class. The machine stores the current state as an object and delegates method calls to it.

**Step 1 — State interface:**

```java
public interface State {
    void button1(ChocolateMaker maker);
    void button2(ChocolateMaker maker);
    boolean isEmpty(ChocolateMaker maker);
}
```

All state-dependent methods go in the interface. Each method receives the machine (`maker`) so it can trigger a state change.

**Step 2 — Context class (the machine):**

```java
public class ChocolateMaker {

    private State current;

    private ChocolateMaker() {
        current = new EmptyState();   // initial state
    }

    // Delegation — no more if/else
    public void button1()      { current.button1(this); }
    public void button2()      { current.button2(this); }
    public boolean isEmpty()   { return current.isEmpty(this); }

    // Package-visible so state classes can switch the state
    void setState(State s) {
        if (s != null) this.current = s;
    }

    // Singleton
    private static ChocolateMaker instance = new ChocolateMaker();
    public static ChocolateMaker getInstance() { return instance; }
}
```

**Step 3 — Concrete state classes:**

```java
public class EmptyState implements State {

    @Override
    public void button1(ChocolateMaker maker) {
        System.out.println("Add milk-chocolate mixture!");
        maker.setState(new FilledState());   // transition
    }

    @Override
    public void button2(ChocolateMaker maker) {
        System.out.println("Nothing inside!");
    }

    @Override
    public boolean isEmpty(ChocolateMaker maker) { return true; }
}
```

```java
public class FilledState implements State {

    @Override
    public void button1(ChocolateMaker maker) {
        System.out.println("Cooking... stage one started.");
        maker.setState(new CookedState());
    }

    @Override
    public void button2(ChocolateMaker maker) {
        System.out.println("Not ready yet!");
    }

    @Override
    public boolean isEmpty(ChocolateMaker maker) { return false; }
}
```

```java
public class CookedState implements State {

    @Override
    public void button1(ChocolateMaker maker) {
        System.out.println("Already done!");
    }

    @Override
    public void button2(ChocolateMaker maker) {
        System.out.println("Enjoy the chocolate!");
        maker.setState(new EmptyState());   // transition back
    }

    @Override
    public boolean isEmpty(ChocolateMaker maker) { return false; }
}
```

**Adding a new state** (e.g., a two-stage cooking process) is easy — just add `HalfCookedState` and change `FilledState` to transition there instead of `CookedState`. The machine itself doesn't change.

```java
public class HalfCookedState implements State {

    @Override
    public void button1(ChocolateMaker maker) {
        System.out.println("Switching to stage two.");
        maker.setState(new CookedState());
    }

    @Override
    public void button2(ChocolateMaker maker) {
        System.out.println("Not ready — switch to stage two first!");
    }

    @Override
    public boolean isEmpty(ChocolateMaker maker) { return false; }
}
```

Usage in main:
```java
ChocolateMaker maker = ChocolateMaker.getInstance();
maker.button1();   // fill
maker.button1();   // half-cook
maker.button1();   // finish cooking
maker.button2();   // drain and enjoy
```

### Advantages and trade-offs

**Advantages:**
- Adding a new state requires no changes to the machine class. *(Open/Closed principle)*
- Each state class is short, focused, and easy to test.

**Trade-offs:**
- State classes need access to the machine's internals (to call `setState()`). Solutions:
  - Package-level visibility for `setState()` (state classes in the same package).
  - Inner classes (state classes defined inside the machine class).
- An abstract class instead of an interface can hold default behaviour — state subclasses only override methods where their behaviour differs.

---

## Decorator Pattern

### When to use it

> Add new functionality to an object at runtime — particularly by extending (decorating) existing method behaviour — without creating a combinatorial explosion of subclasses.

---

### Problem — base + toppings

A `Chocolate` hierarchy:

```
         Chocolate {abstract}
         + getPrice(): double
         + toString(): String
               △
        +------+------+
        |             |
   PremiumChocolate  BudgetChocolate
```

Now we want toppings: nuts, puffed rice, white chocolate coating, gift wrapping... and any combination of toppings with any base.

**Bad approach 1:** One class per combination (`PremiumWithNuts`, `PremiumWithNutsAndRice`, ...) — combinatorial explosion.

**Bad approach 2:** Boolean fields on `Chocolate` (`withNuts`, `withRice`, ...) — adding a new topping requires changing the base class, violating Open/Closed.

---

### Solution — Decorator

The key insight: a **decorator** wraps an existing `Chocolate` object, holds a reference to it, and delegates to it while adding its own contribution.

```
         Chocolate {abstract}
         + getPrice(): double
         + toString(): String
               △
        +------+----------+-------------------+
        |                 |                   |
   PremiumChocolate  BudgetChocolate  ChocolateWithTopping {abstract}
                                      # beingRefined: Chocolate
                                            △
                                    +-------+--------+
                                    |                |
                              WithNuts           WithRice    ...
```

**Abstract base:**
```java
public abstract class Chocolate {
    public abstract double getPrice();
    public abstract String toString();
}
```

**Concrete bases:**
```java
public class PremiumChocolate extends Chocolate {
    @Override public double getPrice()   { return 2.49; }
    @Override public String toString()   { return "Premium chocolate"; }
}

public class BudgetChocolate extends Chocolate {
    @Override public double getPrice()   { return 0.49; }
    @Override public String toString()   { return "Budget chocolate with lots of sugar"; }
}
```

**Abstract decorator:**
```java
public abstract class ChocolateWithTopping extends Chocolate {
    protected Chocolate beingRefined;   // the wrapped object

    public ChocolateWithTopping(Chocolate base) {
        this.beingRefined = base;
    }
}
```

**Concrete decorators:**
```java
public class WithNuts extends ChocolateWithTopping {

    public WithNuts(Chocolate base) { super(base); }

    @Override
    public double getPrice() {
        return 0.50 + beingRefined.getPrice();   // nuts + base price
    }

    @Override
    public String toString() {
        return beingRefined.toString() + " with a handful of nuts";
    }
}

public class WithPuffedRice extends ChocolateWithTopping {

    public WithPuffedRice(Chocolate base) { super(base); }

    @Override
    public double getPrice() { return 0.20 + beingRefined.getPrice(); }

    @Override
    public String toString() { return beingRefined.toString() + " with puffed rice"; }
}

public class WhiteCoating extends ChocolateWithTopping {

    public WhiteCoating(Chocolate base) { super(base); }

    @Override
    public double getPrice() { return 0.30 + beingRefined.getPrice(); }

    @Override
    public String toString() { return "White " + beingRefined.toString(); }
}

public class GiftWrapped extends ChocolateWithTopping {

    public GiftWrapped(Chocolate base) { super(base); }

    @Override
    public double getPrice() { return 1.00 + beingRefined.getPrice(); }

    @Override
    public String toString() {
        return "Bow on top — " + beingRefined.toString() + " — in gold foil";
    }
}
```

**Usage — wrap (decorate) at runtime:**
```java
// Long form:
Chocolate mine = new PremiumChocolate();
mine = new WithNuts(mine);
mine = new WithPuffedRice(mine);

System.out.println(mine);
System.out.printf("Price: %.2f%n", mine.getPrice());
```

Output:
```
Premium chocolate with a handful of nuts with puffed rice
Price: 3.19
```

Price breakdown: 2.49 (premium) + 0.50 (nuts) + 0.20 (rice) = 3.19.

```java
// Compact nesting:
Chocolate gift = new GiftWrapped(
                    new WhiteCoating(
                        new PremiumChocolate()));
```

### How the call chain works

Each decorator calls the same method on the wrapped object before/after adding its own contribution:

```
mine.getPrice()
  → WithPuffedRice.getPrice()
      → 0.20 + WithNuts.getPrice()
              → 0.50 + PremiumChocolate.getPrice()
                         → 2.49
```

The same wrapping technique is already familiar from **I/O streams** (Chapter 10):
```java
BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
//              ↑ decorator            ↑ source stream
```

### Rules

- Decorators must work with **any** `Chocolate` object (base or already-decorated) — never use `instanceof` to check the exact type.
- Decorator behaviour extensions should be **independent** of each other.
- A decorator class may also add **new methods** not in the base interface.

---

## Pattern Summary

| Pattern | Intent | Key mechanism |
|---------|--------|---------------|
| **Singleton** | Ensure at most one instance exists globally | Private constructor + static instance + static `getInstance()` |
| **State** | Same method behaves differently per state; methods trigger transitions | Context holds a `State` object; delegates calls; state classes call `setState()` |
| **Decorator** | Add/extend behaviour at runtime by wrapping objects | Abstract decorator extends the same base and holds a reference to a wrapped instance |
