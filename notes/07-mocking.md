# Chapter 7 — Mocking

## Overview

Mocking frameworks (e.g. Mockito) and testing frameworks (e.g. JUnit, TestNG) can be freely combined with each other.

Reference: https://site.mockito.org/

---

## Why Mock?

- Classes often depend heavily on each other, making it hard to isolate which class contains a bug.
- Sometimes the dependency class isn't finished yet when you want to test your own class.

**Example — Bank project:**

- `Account` depends on `Customer` — almost no `Account` methods rely on `Customer` working correctly → not problematic.
- `Bank` depends on `Account` — most `Bank` methods only produce correct results if `Account` methods work correctly → problematic without mocking.

---

## Maven Dependency

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>3.4.6</version>
    <scope>test</scope>
</dependency>
```

---

## Structure of a Test Method

```java
@Test
public void describesSituationTest() {

    // SetUp    — initialise test objects and configure mocks

    // Exercise — call the method(s) under test

    // Verify   — check results and side effects

    // TearDown — clean up (usually not needed with JUnit 5)
}
```

Keep test methods simple: no unnecessary loops or branches.

| Phase | Purpose |
|-------|---------|
| **SetUp** | Construct the object under test; configure dependent objects to match the test scenario |
| **Exercise** | Call the method(s) being tested |
| **Verify** | Check return values and side effects; requires knowing the expected results in advance |
| **TearDown** | Release resources (rarely needed) |

---

## Mock Objects

- Placeholders for real objects (*to mock* = to simulate).
- Have the same interface as the real object.
- Return predefined test values instead of real data.
- Record which methods were called on them.

---

## Creating a Mock Object

```java
// Mockito.mock signature:
public static <T> T mock(Class<T> classToMock)

// Usage:
SomeClass mockObject = Mockito.mock(SomeClass.class);
```

Although not a real instance of the class, the mock has the class as its declared type — it can be used anywhere the real object would be used.

---

## Configuring a Mock Object

```java
// OngoingStubbing interface (returned by Mockito.when):
OngoingStubbing<T> thenReturn(T value)
OngoingStubbing<T> thenThrow(Throwable... throwables)
OngoingStubbing<T> then(Answer<?> answer)

// Set up what a method should return:
Mockito.when(mockObject.method(...)).thenReturn(value);
```

Methods that are not configured return `0`, `false`, `null`, or an empty collection by default.

---

## Worked Example — Vending Machine

The class under test:

```java
public class VendingMachine {

    private CashModule cashBox = null;
    private Compartment[] compartments = null;

    public VendingMachine(CashModule cash, Compartment[] compartments) { ... }

    /**
     * Selects a compartment number the customer wants to purchase from,
     * deducts the price from the cash module, and opens the compartment.
     *
     * @param slot compartment number
     * @return false if there is not enough money, the slot is empty, or the slot number is invalid
     * @throws JamException if the selected compartment is jammed
     */
    public boolean select(int slot) { ... }
}
```

Its dependencies (interfaces):

```java
public interface Compartment {
    /** Returns whether the compartment is full. */
    boolean isFull();

    /** Returns the price of the product in the compartment. */
    int getPrice();

    /**
     * Ejects the product from the compartment.
     * @throws JamException if the compartment is jammed
     */
    void eject();
}

public interface CashModule {
    /** Returns the current balance — the amount inserted. */
    int getBalance();

    /** Deducts the given amount from the current balance. */
    void deduct(int amount);
}
```

### SetUp

```java
@Test
void firstFullSlotWithSufficientFundsTest() {

    CashModule cash = Mockito.mock(CashModule.class);
    Mockito.when(cash.getBalance()).thenReturn(100);

    Compartment[] compartments = new Compartment[1];
    compartments[0] = Mockito.mock(Compartment.class);
    Mockito.when(compartments[0].isFull()).thenReturn(true);
    Mockito.when(compartments[0].getPrice()).thenReturn(42);

    VendingMachine machine = new VendingMachine(cash, compartments);
```

### Exercise

```java
    boolean succeeded = machine.select(0);
    assertTrue(succeeded);
```

### Verify

Testing that side effects occurred — not just the return value:

```java
    // Was deduct() called with the correct amount?
    Mockito.verify(cash).deduct(42);

    // Was eject() called on the selected compartment?
    Mockito.verify(compartments[0]).eject();
}
```

Note: `cash` is a mock — it has no real balance. Calling `cash.getBalance()` always returns `100` as configured. Verifying with `verify` checks that the method was *called*, not that the balance actually changed.

---

## `verify` — Checking Method Calls

```java
// Signature:
public static <T> T verify(T mock, VerificationMode mode)

// Check that method was called exactly once (default):
Mockito.verify(mockObject).method(...);
```

### VerificationMode Options

| Mode | Meaning |
|------|---------|
| `Mockito.times(x)` | Called exactly `x` times |
| `Mockito.atLeast(x)` | Called at least `x` times |
| `Mockito.atMost(x)` | Called at most `x` times |
| `Mockito.only()` | This was the only method called on the mock |
| `Mockito.verifyNoInteractions(mock)` | No methods were called on the mock at all |
| `Mockito.verifyNoMoreInteractions(mock)` | No methods were called beyond those already verified |

### Example — Empty Slot Test

```java
@Test
void firstEmptySlotTest() {
    // ... (same setup as above, except:)
    Mockito.when(compartments[0].isFull()).thenReturn(false);

    // ...
    assertFalse(succeeded);
    Mockito.verify(cash, Mockito.times(0)).deduct(42);
    Mockito.verify(compartments[0], Mockito.times(0)).eject();
}
```

Checking that a second compartment had no interaction at all:

```java
Compartment[] compartments = new Compartment[2];
// ...
compartments[1] = Mockito.mock(Compartment.class);
// ...
Mockito.verifyNoInteractions(compartments[1]);
```

---

## Testing Exceptions

**For methods with a return type:**

```java
Mockito.when(mockObject.method(...)).thenThrow(new MyException());
```

**For `void` methods:**

```java
Mockito.doThrow(new MyException()).when(mockObject).method(...);
```

**`doReturn` variant (use `thenReturn` for non-void methods instead):**

```java
Mockito.doReturn(value).when(mockObject).method(...);
```

### Example — Jammed Compartment

```java
// SetUp:
Mockito.doThrow(new JamException()).when(compartments[0]).eject();

// Exercise — modern form (preferred):
assertThrows(JamException.class, () -> machine.select(0));

// Exercise — classic form:
try {
    boolean succeeded = machine.select(0);
    fail("expected JamException");
} catch (JamException e) { }
```

### Bug Found via Testing

Initial implementation called `deduct()` before `eject()` — so even when `eject()` threw a `JamException`, money had already been deducted:

```java
// Buggy order:
this.cashBox.deduct(price);
box.eject();            // throws JamException — but money was already taken!
return true;

// Fixed order:
box.eject();            // throw before touching money
this.cashBox.deduct(price);
return true;
```

---

## Argument Matchers

When you cannot predict the exact parameter value at setup or verify time, use `ArgumentMatchers`:

```java
// During setup:
Mockito.when(mockObject.method(ArgumentMatchers.anyInt())).thenReturn(value);

// During verification:
Mockito.verify(mockObject).method(ArgumentMatchers.anyInt());
```

### Available Matchers

| Category | Methods |
|----------|---------|
| Primitives | `anyBoolean()`, `anyByte()`, `anyChar()`, `anyInt()`, `anyLong()`, `anyDouble()`, … |
| Collections | `anyCollection()`, `anyList()`, `anyMap()`, … |
| Strings | `anyString()`, `contains("sub")`, `endsWith("end")`, `startsWith("start")`, `matches("regex")` |
| Objects | `any()`, `isNotNull()`, `isNull()`, `isA(SomeClass.class)` |
| Equality | `eq(value)` |
| Custom | `argThat(matcher)`, `booleanThat(m)`, `intThat(m)`, … — `matcher` is a `ArgumentMatcher` with `boolean matches(Object)` |

### Example

```java
// Verify that deduct() was never called, regardless of argument:
Mockito.verify(cash, Mockito.times(0)).deduct(ArgumentMatchers.anyInt());
```

---

## Verifying Call Order

Use `InOrder` when the sequence of method calls matters:

```java
// SetUp — declare which mocks to watch:
InOrder order = Mockito.inOrder(mockObject1, mockObject2, ...);

// Verify — assert the expected order:
order.verify(mockObject1).method1();
order.verify(mockObject2).method2();
```

If the methods are called in a different order, the test fails.

**Note:** The required call order should be documented — avoid testing order that is an implementation detail not stated in the specification.

---

## Spy Objects

A spy wraps a real object and only replaces selected methods. Unspecified methods delegate to the real implementation.

Use a spy when you already have a working object but one or two methods are inconvenient to test (e.g. methods returning random values):

```java
// Create spy from a real object:
SomeClass spyObject = Mockito.spy(realObject);

// Override specific methods (use doReturn, not when, to avoid calling the real method):
Mockito.doReturn(value).when(spyObject).method(...);
```

---

## Additional Features

**Multiple sequential return values:**

```java
Mockito.when(mockObject.method(...)).thenReturn(value1, value2, value3);
// Successive calls return value1, then value2, then value3
```

**Custom answer (execute code on call):**

```java
Mockito.when(mockObject.method(...)).thenAnswer(answerObject);
// Calls answerObject.answer() each time method() is invoked
```

**Mock with extra interface:**

```java
Mockito.mock(ClassA.class, withSettings().extraInterfaces(InterfaceB.class));
// Creates a mock of ClassA that also implements InterfaceB
```

---

## Design Principles

- **Favour loose coupling.** Even interacting objects should have interchangeable partners.
- **Program to an interface, not a concrete class.** This is also what makes mocking possible.
- **A class should have only one reason to change.** Swapping out an internal dependency should not be that reason.

---

## Exercise — Testing the Bank Class with Mockito

The goal: test `Bank` methods correctly even when the `Account` implementations are replaced with empty stubs.

### Helper Method in `Bank`

Insert a pre-built mock into the bank (mirrors the structure of `createCheckingAccount`):

```java
public long insertMock(Account account) {
    long newAccountNumber = getNewAccountNumber();
    accounts.put(newAccountNumber, account);
    return newAccountNumber;
}
```

### Test Class Setup

```java
Bank bank;
Customer customer;
long checking1, checking2, savings1, savings2, nonExistent;
CheckingAccount ck1, ck2;
SavingsAccount sa1;

@BeforeEach
void setUp() throws Exception {
    bank = new Bank(16050000);
    customer = new Customer("John", "Doe", "Main St 1", LocalDate.parse("1983-06-01"));

    ck1 = Mockito.mock(CheckingAccount.class);
    Mockito.when(ck1.withdraw(ArgumentMatchers.anyDouble())).thenReturn(true);
    Mockito.when(ck1.getOwner()).thenReturn(customer);
    Mockito.when(ck1.getAccountNumber()).thenReturn(1L);
    Mockito.when(ck1.getBalance()).thenReturn(1.23);
    Mockito.when(ck1.isLocked()).thenReturn(false);
    Mockito.when(ck1.toString()).thenReturn("CheckingAccount 1");
    Mockito.when(ck1.transferOut(
            ArgumentMatchers.anyDouble(),
            ArgumentMatchers.anyString(),
            ArgumentMatchers.anyLong(),
            ArgumentMatchers.anyLong(),
            ArgumentMatchers.anyString())).thenReturn(true);

    // repeat for ck2 with different values...

    sa1 = Mockito.mock(SavingsAccount.class);
    Mockito.when(sa1.withdraw(ArgumentMatchers.anyDouble())).thenReturn(true);
    Mockito.when(sa1.getOwner()).thenReturn(customer);
    Mockito.when(sa1.getFormattedAccountNumber()).thenReturn("00 0000 0002");
    Mockito.when(sa1.isLocked()).thenReturn(false);
    Mockito.when(sa1.toString()).thenReturn("SavingsAccount 1");
}
```

`getOwner()` (and similar methods) must **not** be `final` in the `Account` base class — Mockito uses inheritance internally to override methods.

### Helper: Create Accounts

```java
private void createAccounts() {
    checking1 = bank.insertMock(ck1);
    Mockito.when(ck1.getAccountNumber()).thenReturn(checking1);

    checking2 = bank.insertMock(ck2);
    Mockito.when(ck2.getAccountNumber()).thenReturn(checking2);

    savings1 = bank.insertMock(sa1);
    Mockito.when(sa1.getAccountNumber()).thenReturn(savings1);

    savings2 = bank.createSavingsAccount(customer);   // real object

    // A number that cannot collide with any generated account number:
    nonExistent = Math.max(Math.max(Math.max(checking1, checking2), savings1), savings2) + 5;
}
```

After `insertMock()` returns the assigned number, wire it back into the mock with `thenReturn` — the mock doesn't know its own number unless told.

### Constructor Test (no mocks needed)

```java
@Test
void testConstructor() {
    List<Long> numbers = bank.getAllAccountNumbers();
    assertTrue(numbers.isEmpty());
    assertTrue(bank.getAllAccounts().equals(""));
    assertEquals(16050000, bank.getRoutingNumber());
    assertThrows(AccountNotFoundException.class, () -> bank.depositMoney(123, 100));
}
```

### Deposit Test

```java
private void deposit() throws AccountNotFoundException {
    createAccounts();
    bank.depositMoney(checking1, 100);
    bank.depositMoney(checking2, 200);
    bank.depositMoney(savings1, 300);
    bank.depositMoney(savings2, 400);   // real object — not a mock
}

@Test
void testDepositMoney() throws AccountNotFoundException {
    deposit();
    Mockito.verify(ck1).deposit(100);
    Mockito.verify(ck2).deposit(200);
    Mockito.verify(sa1).deposit(300);
    assertEquals(400, bank.getBalance(savings2));   // real object — check actual state
}
```

Mock objects have no real balance — verifying them with `verify` checks the method was called with the correct argument. The real `savings2` has an actual balance that can be asserted directly.

### Negative Deposit Test

```java
@Test
void testDepositNegative() throws AccountNotFoundException {
    createAccounts();
    Mockito.doThrow(new IllegalArgumentException()).when(ck1).deposit(ArgumentMatchers.anyDouble());
    assertThrows(IllegalArgumentException.class, () -> bank.depositMoney(checking1, 100));
}
```

### Withdraw — Succeeds

```java
@Test
void testWithdrawSucceeds() throws AccountNotFoundException, AccountLockedException {
    createAccounts();
    boolean succeeded = bank.withdrawMoney(checking1, 60);
    assertTrue(succeeded);
    Mockito.verify(ck1).withdraw(60);
    // Returns true because ck1.withdraw() was configured to always return true
}
```

### Withdraw — Insufficient Funds

```java
@Test
void testWithdrawTooMuch() throws AccountNotFoundException, AccountLockedException {
    createAccounts();
    Mockito.when(sa1.withdraw(ArgumentMatchers.anyDouble())).thenReturn(false);
    boolean succeeded = bank.withdrawMoney(savings1, 500);
    Mockito.verify(sa1).withdraw(500);
    assertFalse(succeeded);
}
```

### Withdraw from Locked Account

Without mocking it would be impossible to test this scenario — `Bank` has no method to lock an account. With a mock it's trivial:

```java
@Test
void testWithdrawFromLockedAccount() throws AccountLockedException {
    createAccounts();
    Mockito.when(ck1.withdraw(ArgumentMatchers.anyDouble()))
            .thenThrow(new AccountLockedException(checking1));
    Mockito.when(ck1.isLocked()).thenReturn(true);
    assertThrows(AccountLockedException.class, () -> bank.withdrawMoney(checking1, 100));
}
```
