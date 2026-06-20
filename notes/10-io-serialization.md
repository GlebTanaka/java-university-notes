# Chapter 10 — I/O Streams, Serialization & Design Patterns 1

> **Note:** Java's built-in serialization mechanism is stable but dated (introduced in Java 1.1). Modern practice often prefers JSON/XML libraries (Jackson, Gson) or Protocol Buffers over Java object serialization, especially across service boundaries. The mechanics here are still valid and appear in legacy code.

---

## Part 1 — I/O Streams

### Stream Concept

A stream is a serial interface between a program and the outside world.

- Data is read in the same order it was written.
- Data remains in the stream until it is read.
- Each read operation removes the consumed data from the stream.

Two categories:

| Category | Direction | Base type |
|----------|-----------|-----------|
| Byte-by-byte | read | `InputStream` |
| Byte-by-byte | write | `OutputStream` |
| Character-by-character (Unicode, e.g. text files) | read | `Reader` |
| Character-by-character | write | `Writer` |

Converting between byte and character streams:
- `InputStream` → `Reader`: wrap in `InputStreamReader`
- `OutputStream` → `Writer`: wrap in `OutputStreamWriter`

### Step 1 — Choose a Source

First select a stream that points at the data source:

| Class | Source |
|-------|--------|
| `FileReader` / `FileInputStream` | File on disk |
| `CharArrayReader` | `char[]` array |
| `StringReader` | `String` in memory |
| `ByteArrayInputStream` | `byte[]` array |
| `PipedReader` / `PipedInputStream` | Another stream |
| `AudioInputStream` | Audio device |

### Step 2 — Wrap With a Decorator Stream

Layer a stream on top to add capabilities (Decorator pattern):

| Wrapper | Capability added |
|---------|-----------------|
| `BufferedReader` | Buffered reading (whole lines) |
| `LineNumberReader` | Buffered reading with line numbers |
| `ObjectInputStream` | Reading complete serialized objects |
| `DataInputStream` | Reading primitive values |
| `CheckedInputStream` | Checksum verification |
| `InflaterInputStream` | Decompression |

### Step 3 — Use the Methods

### Step 4 — Close

Always close the outermost stream; this also closes the underlying streams.

---

### Example 1 — Read From Console

```java
// 1. System.in is a raw InputStream (byte-by-byte, reads keyboard)
// 2. Wrap in InputStreamReader to get character-by-character access
// 3. Wrap in BufferedReader to read whole lines at once

BufferedReader console =
    new BufferedReader(
        new InputStreamReader(System.in));

String line = console.readLine();
```

---

### Example 2 — Read a Text File

The user enters a file path, the file content is printed to the console.

```java
// Setup
String path = console.readLine();          // e.g. "target/classes/example.txt"
FileReader fr = new FileReader(path);      // source
BufferedReader br = new BufferedReader(fr); // decorator

// Read
while (br.ready()) {
    String line = br.readLine();           // reads and removes one line from stream
    System.out.println(line);
}
br.close();
```

- `readLine()` reads one line and removes it from the stream. Random access to line indices is not possible.
- Closing the `BufferedReader` also closes the underlying `FileReader`.

> **Maven note:** `.txt` files placed in `src/main/resources/` are copied to `target/classes/` during the build, making them accessible by path at runtime.

---

### Example 3 — Write a Text File

```java
FileWriter fw = new FileWriter("target/classes/output.txt");
BufferedWriter bw = new BufferedWriter(fw);

bw.write("First line of text");
bw.write("Second line");

bw.flush();  // empty the buffer — write any remaining bytes to disk
bw.close();
```

> **Warning:** If the file does not exist it is created; if it already exists it is **overwritten** silently.

---

### Exception Handling

**Option 1 — declare `throws IOException` on `main`**

```java
public static void main(String[] args) throws IOException {
    // stream code here — compiler stops complaining
}
```

**Option 2 — try-with-resources (preferred)**

Objects opened inside `try(...)` must implement `AutoCloseable`. They are closed automatically whether the block exits normally, via return, or via exception.

```java
try (FileReader fr = new FileReader(path);
     BufferedReader br = new BufferedReader(fr)) {

    while (br.ready()) {
        String line = br.readLine();
        if (line.equals("stop")) return;  // close() still called automatically
        System.out.println(line);
    }

} catch (IOException e) {
    System.out.println("Could not read file");
}
```

Multiple resources are separated by semicolons inside the parentheses.

```java
// Write with try-with-resources
try (FileWriter fw = new FileWriter("output.txt");
     BufferedWriter bw = new BufferedWriter(fw)) {

    bw.write("new content");
    bw.flush();
    // no explicit close() needed

} catch (IOException e) {
    System.out.println("Could not write file");
}
```

---

### Redirect `System.out` to a File

```java
PrintStream ps = new PrintStream(
    new FileOutputStream("target/classes/output.txt"));
System.setOut(ps);

// All subsequent println() calls go to the file, not the terminal
System.out.println("This goes into the file");
```

Tip: keep processing and I/O separate — avoid scattering `System.out.println()` throughout business logic.

---

## Part 2 — Serialization

### What Is Serialization?

Serialization converts structured data (objects) into a sequential byte representation so it can be:
- Written to a file or database
- Sent over a network
- Deserialized later — in the same or another program, or after a restart

### What Gets Saved?

Standard serialization saves all **instance fields** of an object:
- Its own fields (all visibility levels)
- Inherited fields — but only if the superclass is also `Serializable`

**Static fields are not saved.**

### Making a Class Serializable

Requirements — every instance field must be one of:
- A primitive type (all primitives are serializable by default)
- A type whose class implements `Serializable`
- A `Collection` or array of serializable types

Once all fields satisfy this, add `implements Serializable` to the class:

```java
public abstract class Account implements Comparable<Account>, Serializable { ... }
public class Customer implements Comparable<Customer>, Serializable { ... }
public enum Currency implements Serializable { ... }
```

> **Note:** If you mark a class `Serializable` but one of its fields is not, the compiler will not catch this. A `NotSerializableException` is thrown at runtime when you actually try to serialize an instance.

---

### Writing Objects (Serialization)

```java
// 1. Choose a sink (destination) stream
FileOutputStream fo = new FileOutputStream("target/classes/account.dat");

// 2. Wrap with ObjectOutputStream to gain writeObject()
ObjectOutputStream oos = new ObjectOutputStream(fo);

// 3. Write objects
oos.writeObject(account);

// 4. Flush and close
oos.flush();
oos.close();
```

With try-with-resources:

```java
try (FileOutputStream fo = new FileOutputStream("target/classes/account.dat");
     ObjectOutputStream oos = new ObjectOutputStream(fo)) {

    oos.writeObject(account1);
    oos.writeObject(account2);  // multiple objects are allowed

} catch (IOException e) {
    System.out.println("Could not write file");
}
```

The resulting `.dat` file is binary (not human-readable), though variable names may be faintly visible inside it.

---

### Reading Objects (Deserialization)

```java
try (FileInputStream fi = new FileInputStream("target/classes/account.dat");
     ObjectInputStream ois = new ObjectInputStream(fi)) {

    try {
        Object raw = ois.readObject();       // returns Object — cast required
        Account account = (Account) raw;
        System.out.println(account);

    } catch (ClassNotFoundException e) {     // class might not exist in this JVM
        e.printStackTrace();
    }

} catch (IOException e) {
    System.out.println("Could not read file");
}
```

Objects must be read back in the same order they were written. `readObject()` throws `ClassNotFoundException` because the class definition might not be present in the loading JVM.

---

### General Pattern

**Serialize (write):**

```java
OutputStream sink = ...;            // e.g. FileOutputStream, network socket
ObjectOutputStream oos = new ObjectOutputStream(sink);
oos.writeObject(obj1);
oos.writeObject(obj2);
oos.flush();
oos.close();
```

**Deserialize (read):**

```java
InputStream source = ...;           // e.g. FileInputStream, network socket
ObjectInputStream ois = new ObjectInputStream(source);
Object o1 = ois.readObject();       // cast before using
Object o2 = ois.readObject();
ois.close();
```

---

### `transient` — Excluding Fields

If a field cannot be serialized (e.g. a network `Socket` connection), mark it `transient`:

```java
private transient TransferManager manager;
```

The field is excluded from serialization and deserialization. After deserialization its value is `null`.

To reinitialize a `transient` field on deserialization, implement `readObject()`:

```java
private synchronized void readObject(ObjectInputStream s)
        throws IOException, ClassNotFoundException {
    s.defaultReadObject();                     // deserialize all non-transient fields
    this.manager = new TransferManager();      // reinitialize the transient field
}
```

---

### Inheritance

| Scenario | Behaviour |
|----------|-----------|
| Superclass is `Serializable` | Its fields are saved and restored too |
| Superclass is **not** `Serializable` | Its fields are **not** saved; on deserialization the superclass **no-arg constructor** is called |

A non-serializable superclass must have a non-private no-arg constructor — otherwise `InvalidClassException` is thrown during deserialization.

---

### Version Numbers

When a class changes (e.g. a field is added or renamed), previously serialized instances become incompatible. An `InvalidClassException` is thrown on deserialization.

Prevent this by declaring an explicit version UID:

```java
private static final long serialVersionUID = 1L;  // choose any number
```

Change the UID only when saved objects should genuinely be invalidated (e.g. if a field's type changes). Adding or removing a field is generally safe to keep compatible.

---

### Criticism of Java Serialization

- The mechanism is very old (Java 1.1). Today annotations would be preferred over a marker interface and a `transient` keyword.
- Marking a class `Serializable` also makes all its subclasses serializable — which may not be intended.
- The compiler cannot verify that a class actually satisfies the serialization requirements at compile time.
- `serialVersionUID` is special only because of its name — fragile convention.
- **Modern alternative:** prefer JSON (Jackson, Gson), XML, or Protocol Buffers for persistence and network transfer.

---

### Exercise — Deep-Copy a Bank via Serialization

Implement `Bank.clone()` using serialization to a `ByteArrayOutputStream` and back — this performs a true deep copy of every `Account` in the bank.

```java
public Bank clone() {

    // Serialize each account to a byte array
    List<byte[]> serializedAccounts = new LinkedList<>();
    for (Account acc : this.accountList.values()) {
        try (ByteArrayOutputStream sink = new ByteArrayOutputStream();
             ObjectOutputStream oos = new ObjectOutputStream(sink)) {
            oos.writeObject(acc);
            serializedAccounts.add(sink.toByteArray());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // Deserialize each byte array back into a fresh Account
    TreeMap<Long, Account> cloneList = new TreeMap<>();
    for (byte[] data : serializedAccounts) {
        try (ByteArrayInputStream source = new ByteArrayInputStream(data);
             ObjectInputStream ois = new ObjectInputStream(source)) {
            Account acc = (Account) ois.readObject();
            cloneList.put(acc.getAccountNumber(), acc);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }

    return new Bank(this.routingNumber, cloneList);   // private helper constructor
}

// Private helper constructor used only by clone()
private Bank(long routingNumber, TreeMap<Long, Account> accounts) {
    this.routingNumber = routingNumber;
    this.accountList = accounts;
}
```

---

## Part 3 — Design Patterns 1: Template Method & Factory

---

### Template Method Pattern

#### When to Use

- The overall algorithm is fixed and unlikely to change.
- Some steps inside the algorithm vary by subclass.
- You can foresee that more variants will be added in the future.

#### Starting Point (Problem)

```java
public class Coffee {
    public void makeCoffee() {
        System.out.println("Boil water");
        this.brewCoffee();
        this.pourIntoCup();
        this.addSugarAndMilk();
    }
    private void addSugarAndMilk() { System.out.println("Add milk and sugar"); }
    private void pourIntoCup()     { System.out.println("Pour into cup"); }
    private void brewCoffee()      { System.out.println("Run water through filter"); }
}

public class Tea {
    public void prepareTea() {
        System.out.println("Boil water");
        this.steep();
        this.pourIntoCup();
        this.addLemon();
    }
    private void addLemon()    { System.out.println("Add lemon"); }
    private void pourIntoCup() { System.out.println("Pour into cup"); }
    private void steep()       { System.out.println("Steep the tea"); }
}
```

Problems: the two `make` methods have different names, the caller must know the concrete type, `pourIntoCup()` is duplicated.

#### Step 1 — Extract a common superclass

```java
public abstract class VendingDrink {
    public abstract void brew();
    protected void pourIntoCup() { System.out.println("Pour into cup"); }
}
```

`Coffee` and `Tea` both extend `VendingDrink` and implement `brew()`. The caller can now use `VendingDrink g = ...; g.brew();` regardless of type.

#### Step 2 — Apply the Template Method

The overall algorithm is still duplicated across `Coffee.brew()` and `Tea.brew()`. Extract it into a `final` method in the superclass:

```java
public abstract class VendingDrink {

    // Template method — defines the fixed algorithm; cannot be overridden
    public final void brew() {
        System.out.println("Boil water");
        this.infuse();           // varies → abstract
        this.pourIntoCup();      // same for all → concrete here
        this.addIngredients();   // varies → hook (may be empty)
    }

    protected abstract void infuse();

    protected void addIngredients() {
        // hook — empty default; subclasses may override
    }

    protected void pourIntoCup() {
        System.out.println("Pour into cup");
    }
}
```

```java
public class Coffee extends VendingDrink {
    @Override protected void infuse() {
        System.out.println("Run water through coffee filter");
    }
    @Override protected void addIngredients() {
        System.out.println("Add milk and sugar");
    }
}

public class Tea extends VendingDrink {
    @Override protected void infuse() {
        System.out.println("Steep the tea");
    }
    @Override protected void addIngredients() {
        System.out.println("Add lemon");
    }
}

public class Cocoa extends VendingDrink {
    @Override protected void infuse() {
        System.out.println("Stir in cocoa powder");
    }
    // addIngredients() not overridden — hook default (empty) is used
}
```

The template method defines the skeleton; abstract methods fill in the varying details; hook methods are optional overrides.

#### Template Method in the Bank — `withdraw()`

`withdraw()` in the abstract `Account` class is a real-world template method:

```java
// In abstract class Account:
public final boolean withdraw(double amount)
        throws LockedException, IllegalArgumentException {

    if (amount < 0)
        throw new IllegalArgumentException("Amount must be positive");
    if (this.isLocked())
        throw new LockedException(this.getAccountNumber());
    if (!this.checkWithdrawalCondition(amount))
        return false;

    this.setBalance(this.getBalance() - amount);
    afterWithdrawal(amount);     // hook — default is empty
    return true;
}

protected abstract boolean checkWithdrawalCondition(double amount); // abstract step
protected void afterWithdrawal(double amount) {}                    // hook
```

```java
// CheckingAccount — checks against overdraft limit
@Override
public boolean checkWithdrawalCondition(double amount) {
    return getBalance() - amount >= -overdraft;
}

// SavingsAccount — checks monthly limit and minimum balance
@Override
public boolean checkWithdrawalCondition(double amount) {
    LocalDate today = LocalDate.now();
    double alreadyWithdrawn = this.withdrawnThisMonth;
    if (today.getMonth() != lastWithdrawalDate.getMonth()
            || today.getYear() != lastWithdrawalDate.getYear()) {
        alreadyWithdrawn = 0;
    }
    return getBalance() - amount >= 0.50
        && alreadyWithdrawn + amount <= Currency.EUR.convert(SavingsAccount.MAX_MONTHLY);
}

@Override
protected void afterWithdrawal(double amount) {
    // Update the monthly tracking fields
    LocalDate today = LocalDate.now();
    if (today.getMonth() != lastWithdrawalDate.getMonth()
            || today.getYear() != lastWithdrawalDate.getYear()) {
        this.withdrawnThisMonth = 0;
    }
    withdrawnThisMonth += amount;
    this.lastWithdrawalDate = LocalDate.now();
}
```

---

### Factory Method Pattern

#### When to Use

- Several related classes exist.
- Which one to instantiate is only known at runtime (depends on input or configuration).
- You want to cleanly separate object creation from object use.

#### Starting Point (Problem)

```java
// In VendingMachine — creation mixed with use
public void dispenseDrink() {
    int choice = console.nextInt();
    VendingDrink drink = null;
    switch (choice) {
        case 1: drink = new Coffee(); break;
        case 2: drink = new Tea();    break;
        case 3: drink = new Cocoa();  break;
    }
    drink.brew();  // use
}
```

Adding a new drink type requires changing the switch in `dispenseDrink()`.

#### Step 1 — Extract a Factory Method

Move creation into its own method:

```java
public VendingDrink create(int choice) {
    VendingDrink drink = null;
    switch (choice) {
        case 1: drink = new Coffee(); break;
        case 2: drink = new Tea();    break;
        case 3: drink = new Cocoa();  break;
    }
    return drink;
}

public void dispenseDrink() {
    int choice = console.nextInt();
    VendingDrink drink = create(choice);  // creation
    drink.brew();                         // use — now separate
}
```

#### Step 2 — Make the Factory Abstract

To allow completely different beverage families without changing the base class:

```java
public abstract class BeverageMachine {
    public abstract VendingDrink create(int choice);  // factory method

    public void dispenseDrink() {                     // stays here
        int choice = console.nextInt();
        VendingDrink drink = create(choice);
        drink.brew();
    }
}
```

```java
public class UniversityMachine extends BeverageMachine {
    @Override
    public VendingDrink create(int choice) {
        switch (choice) {
            case 1: return new Coffee();
            case 2: return new Tea();
            case 3: return new Cocoa();
            default: return null;
        }
    }
}

public class LuxuryMachine extends BeverageMachine {
    @Override
    public VendingDrink create(int choice) {
        if (choice == 1) return new LuxuryCoffee();
        if (choice == 2) return new LuxuryTea();
        if (choice == 3) return new LuxuryCocoa();
        return null;
    }
}
```

```java
// main — only creation changes when switching machines
BeverageMachine machine = new LuxuryMachine();
machine.dispenseDrink();
```

Structure summary:
- **Abstract factory**: `BeverageMachine` — holds `create()` (abstract) + `dispenseDrink()` (concrete)
- **Concrete factories**: `UniversityMachine`, `LuxuryMachine`
- **Abstract product**: `VendingDrink`
- **Concrete products**: `Coffee`, `Tea`, `Cocoa`, `LuxuryCoffee`, …

---

### Abstract Factory Pattern

An extension of Factory Method that:
- Fully separates object **creation** (factory) from object **use** (client)
- Supports creating families of related objects

#### Difference From Factory Method

In Factory Method, the factory class still does the work (`dispenseDrink()` lives in `BeverageMachine`). In Abstract Factory, a separate **client** class handles the work and receives the factory as a parameter.

```java
// Abstract factory — only creation
public abstract class BeverageMachine {
    public abstract VendingDrink create(int choice);
    public abstract String menuText();      // second "product line": the menu string
}

// Client — only work; factory injected as parameter
public class DrinkDispenser {
    public static void dispense(BeverageMachine machine) {
        System.out.println(machine.menuText());   // machine generates its own menu
        int choice = new Scanner(System.in).nextInt();
        VendingDrink drink = machine.create(choice);
        drink.brew();
    }

    public static void main(String[] args) {
        BeverageMachine machine = new LuxuryMachine();
        dispense(machine);                         // pass factory to client
    }
}
```

```java
public class UniversityMachine extends BeverageMachine {
    @Override public VendingDrink create(int choice) { ... }
    @Override public String menuText() { return "1: Coffee, 2: Tea, 3: Cocoa"; }
}

public class LuxuryMachine extends BeverageMachine {
    @Override public VendingDrink create(int choice) { ... }
    @Override public String menuText() { return "1: Arabica Coffee, 2: Earl Grey, 3: Dark Cocoa"; }
}
```

Each concrete machine produces a consistent family: its drinks *and* its menu text match.

#### When to Use Which

| Pattern | Use when |
|---------|----------|
| Template Method | The algorithm's skeleton is fixed; only some steps vary |
| Factory Method | Creation must be separated from use; one family of products |
| Abstract Factory | Multiple families of related products; creation and use in separate classes |

Abstract Factory is more widely used in practice.

---

### Design Principles

| Principle | Summary |
|-----------|---------|
| No code duplication | Extract into a shared superclass or method |
| Encapsulate what varies | Isolate the parts that change; keep the stable parts unchanged |
| Hollywood Principle | "Don't call us — we'll call you." High-level components call low-level ones, not vice versa. |
| Open/Closed Principle | Open for extension (add new classes), closed for modification (don't change existing code) |
| Depend on abstractions | Variable types should be interfaces/abstract classes, not concrete types |

---

### Exercise — Abstract Factory in the Bank

The `Bank` originally mixed account creation with registration:

```java
// Before: creation mixed into Bank methods
public long createCheckingAccount(Customer owner) {
    long nr = getNewAccountNumber();
    Checking acc = new Checking(owner, nr, 1000);
    accountList.put(nr, acc);
    return nr;
}

public long createSavingsAccount(Customer owner) {
    long nr = getNewAccountNumber();
    Savings acc = new Savings(owner, nr);
    accountList.put(nr, acc);
    return nr;
}
```

After applying Abstract Factory:

```java
// Bank is the Client — takes a factory, delegates creation
public long createAccount(AccountFactory factory, Customer owner) {
    long nr = getNewAccountNumber();
    Account acc = factory.create(owner, nr);
    accountList.put(nr, acc);
    return nr;
}
```

```java
// Abstract factory
public abstract class AccountFactory {
    public abstract Account create(Customer owner, long accountNumber);
}

// Concrete factories
public class CheckingFactory extends AccountFactory {
    @Override
    public Account create(Customer owner, long accountNumber) {
        return new CheckingAccount(owner, accountNumber, 500.0);
    }
}

public class SavingsFactory extends AccountFactory {
    @Override
    public Account create(Customer owner, long accountNumber) {
        return new SavingsAccount(owner, accountNumber);
    }
}

// Mock factory (for testing — no real Mockito.mock() wiring needed in Bank)
public class MockFactory extends AccountFactory {
    private Account lastCreated;
    public AccountType type = AccountType.CHECKING;

    @Override
    public Account create(Customer owner, long accountNumber) {
        switch (type) {
            case CHECKING: lastCreated = Mockito.mock(CheckingAccount.class); break;
            case SAVINGS:  lastCreated = Mockito.mock(SavingsAccount.class);  break;
            default: throw new IllegalArgumentException();
        }
        Mockito.when(lastCreated.getOwner()).thenReturn(owner);
        Mockito.when(lastCreated.getAccountNumber()).thenReturn(accountNumber);
        return lastCreated;
    }

    public Account getLastCreated() { return lastCreated; }
}
```

Usage in `main` or tests:

```java
Bank bank = new Bank(16050000);
bank.createAccount(new CheckingFactory(), customer);
bank.createAccount(new SavingsFactory(), customer);

// In tests:
MockFactory mock = new MockFactory();
mock.type = AccountType.CHECKING;
long nr = bank.createAccount(mock, customer);
Mockito.when(mock.getLastCreated().getBalance()).thenReturn(1.23);
```

---

### Exercise — Template Method for Currency Conversion

Before refactoring:

```java
// Account.changeCurrency() — partially duplicated in each subclass
public void changeCurrency(Currency newCur) {
    balance = Currency.convert(balance, this.currentCurrency, newCur);
    this.currentCurrency = newCur;
}

// CheckingAccount — overrides and calls super
@Override public void changeCurrency(Currency newCur) {
    overdraft = Currency.convert(overdraft, this.currentCurrency, newCur);
    super.changeCurrency(newCur);
}

// SavingsAccount — overrides and calls super
@Override public void changeCurrency(Currency newCur) {
    withdrawnThisMonth = Currency.convert(withdrawnThisMonth, this.currentCurrency, newCur);
    super.changeCurrency(newCur);
}
```

After applying Template Method — the algorithm lives in one place:

```java
// In abstract Account — final template method
public final void changeCurrency(Currency newCur) {
    balance = Currency.convert(balance, this.currentCurrency, newCur);
    convertSpecificFields(newCur);      // abstract — subclass fills in
    this.currentCurrency = newCur;
}

protected abstract void convertSpecificFields(Currency newCur);
```

```java
// CheckingAccount
@Override
protected void convertSpecificFields(Currency newCur) {
    overdraft = Currency.convert(overdraft, this.currentCurrency, newCur);
}

// SavingsAccount
@Override
protected void convertSpecificFields(Currency newCur) {
    withdrawnThisMonth = Currency.convert(withdrawnThisMonth, this.currentCurrency, newCur);
}
```
