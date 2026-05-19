# Chapter 5 — UML & Testing

## UML Class Diagrams

Class diagrams show classes and their **static relationships** (not runtime behaviour).

Used for:
- **Planning phase** — domain model (represents reality) and structural model (plans the classes)
- **Documentation**
- Showing one specific aspect of a program at a time — leaving things out is fine and encouraged

---

## Class Notation

```
+---------------------------------------+
|              ClassName                |  ← bold, centred
+---------------------------------------+
|  -field1: DataType                    |  ← attributes
|  -field2: DataType = defaultValue     |
+---------------------------------------+
|  +ClassName()                         |  ← constructors and methods
|  +ClassName(param: Type)              |
|  +methodName(param: Type): ReturnType |
|  #setField(field: Type)               |
+---------------------------------------+
```

### Visibility Symbols

| Symbol | Java keyword | Meaning |
|--------|-------------|---------|
| `+` | `public` | Visible everywhere |
| `#` | `protected` | Visible in package and subclasses |
| `~` | *(none)* | Package-visible |
| `-` | `private` | Visible only within the class |

### Attribute Syntax

```
[visibility] [/] name [: DataType] [multiplicity] [= defaultValue] [{properties}]
```

| Part | Meaning |
|------|---------|
| `/` | Derived attribute — computed from other data, not stored directly |
| `[1..*]` | Multiplicity — how many values this attribute can hold |
| `= 0` | Default value |
| `{readOnly}` | Maps to `final` in Java — cannot be reassigned |
| `{unique}` | Value must be unique across all instances (enforced in logic) |
| `{ordered}` | Order matters (≠ sorted) |

Examples:

```
-balance: double = 0
-accountNumber: long {readOnly, unique}
-firstName: String[1..*] {ordered}
+/fullName: String          ← derived: computed from lastName + firstName
```

### Method Syntax

```
[visibility] methodName(paramName: DataType [multiplicity] [= default] [{props}]) [:ReturnType] [{properties}]
```

- Omitting the return type means `void`.

### Method Properties

| Property | Meaning |
|----------|---------|
| `{abstract}` | No implementation; or write the signature in *italics* |
| `{query}` | No side effects — does not change any fields of `this` (like a pure getter) |
| `{leaf}` | Cannot be overridden in subclasses (`final` in Java) |
| `{raised-Exception = ExceptionName}` | The method may throw this exception |

Example:

```
+deposit(amount: double) {raised-Exception=IllegalArgumentException}
+getBalance(): double {query}
+getAccountNumber(): long {leaf}
```

### Static Members

Static members are **underlined** in UML:

```
+SAMPLE_CUSTOMER: Customer       ← underlined = static
```

### Abstract Classes and Methods

Mark with `{abstract}` after the name, or write the name in *italics*:

```
+--------------------+
|  Account {abstract}|
+--------------------+
|+withdraw(amount: double): boolean|
+--------------------+
```

### Class Stereotypes

Written before the class name in `<< >>`:

| Stereotype | Meaning |
|------------|---------|
| `<<focus>>` | Core domain class (e.g. `Account`, `Customer`, `Bank`) |
| `<<auxiliary>>` | Helper class |
| `<<utility>>` | Only static methods/fields (e.g. `Math`, `Collections`) |
| `<<interface>>` | Java interface |
| `<<enumeration>>` | Java enum |

---

## Relationships

Relationships are the most valuable part of a class diagram — prefer showing relationships over listing attributes.

### Directed Association (solid line with open arrowhead →)

Shows that one class holds a reference to another (as a field). The arrow points in the direction of navigation.

```
+------+  -accountList       +-----------------+
| Bank |-------------------→ | Account{abstract}|
+------+  *                  +-----------------+
                                      |
                              -owner  |
                                      ↓ 1
                              +--------+
                              |Customer|
                              +--------+
```

- From `Bank` you can reach its `Account` objects (stored in a `Set<Account>`).
- From `Account` you can reach its `Customer` (stored in a `customer` field).
- Navigation is **not** possible in the reverse direction.
- `*` = zero or more; `1` = exactly one.

### Multiplicity

Placed at the **target** end of the arrow (how many target objects belong to one source object):

| Notation | Meaning |
|----------|---------|
| `1` | Exactly one |
| `0..1` | Zero or one (optional) |
| `*` or `0..*` | Zero or more |
| `1..*` | One or more |
| `3` | Exactly three |

```
+---------+  -relationshipName    +---------+
| Class1  |─────────────────────→ | Class2  |
|         | 0..1              1..* |         |
+---------+                       +---------+
```

Arrow direction: given a Class1 object, you can reach the associated Class2 objects — but not the other way around.

Bidirectional navigation (no arrowhead) requires attributes in **both** classes and keeping them in sync — often a sign of poor design.

### Inheritance (solid line with hollow arrowhead △)

Arrow points from **subclass → superclass**. Only list overridden methods in the subclass box — never duplicate inherited members.

```
         +-----------------+
         | Account{abstract}|
         +-----------------+
               △
               |
    +----------+----------+
    |                     |
+---+--------+    +-------+------+
|CheckingAcct|    |SavingsAccount|
+------------+    +--------------+
```

### Interface Implementation (dashed line with hollow arrowhead, or lollipop symbol)

```
+--------+
|Comparable|  <<interface>>
+--------+
     △
  - - -
     |   <<bind<T→Customer>>>
     |
+----------+
| Customer |
+----------+
```

### Dependency (dashed line with open arrowhead, labelled `<<use>>` or `<<create>>`)

Short-lived relationship — e.g. a method calls another class or throws its exception. Use sparingly; clutters diagrams.

### Generic Classes

Type parameters are shown in a dashed box in the top-right corner:

```
          +---+
+---------|T  |
|<<interface>> +---+
| Comparable       |
+------------------+
|+compareTo(o:T):int|
+------------------+
         △
         | <<bind<T→Customer>>>
+----------+
| Customer |
+----------+
```

### Enums

Use the `<<enumeration>>` stereotype; list constants in the attribute section:

```
+----------------+
| <<enumeration>>|
|   Currency     |
+----------------+
|  EUR           |
|  USD           |
|  GBP           |
+----------------+
        △
        | -currency
    +-------+
    |Account|
    +-------+
```

---

## Testing

> "Testing is the process of executing a program with the intent of finding errors."

A **successful** test case is one that uncovers a previously unknown bug. The mindset: *"How do I best break this?"*

### Unit Tests

- A **unit** = one functional component, usually one class in OOP.
- Has a clearly defined interface (its public methods).
- Relatively few test cases should be enough to cover it fully.

> "In principle, testing cannot prove the absence of errors."

### Integration Tests

- **Unit test** — tests a unit in isolation (other dependencies are simulated/mocked).
- **Integration test** — assumes each unit is correct individually, and tests how they work together.
- Unit tests are **whitebox tests** — the tester knows the code.
  - Often written in the context of **test-driven development (TDD)**: write the test first, then write the code that makes it pass.
- The focus is still on externally visible behaviour:
  - Return values
  - State changes in the object (fields)
  - Output / side effects

> "Good documentation is the foundation."

---

## Testing Guidelines

**Coverage**
- Call every method and constructor at least once.
- Focus on methods where bugs are likely — don't waste effort on trivial getters/setters.
- Multiple methods can share a single test case.

**Calculating expected results**
- Never copy formulas from the code under test — calculate the expected result yourself.
- If possible, use a different formula that should produce the same result.

**Input values**
- Normal values within the expected range.
- Boundary values at the edges of the data type.
- Special cases: `null`, leap day (29.02.2000), empty collections.
- Invalid values: negative numbers where only positive are allowed, etc.
- Push outside your "comfort zone" — explore the full range of the parameter type.

**Method interactions**
- Start by testing individual method calls.
- Then combine multiple method calls in sequence:
  - Flows that will commonly occur in practice.
  - Flows that occur in edge-case situations.
  - Flows that should never happen but are technically possible.

**What to check after each call**
- Return value.
- All fields of the object that should have changed.
- Fields of parameter objects that should have changed.
- External side effects (were other resources notified?).
- Also check the inverse: did the method change anything it should **not** have?

**Scope**
- Do not test more than what the documentation says the method does.

> "Testing is an extremely creative and intellectually challenging activity. The goal is to find bugs."
