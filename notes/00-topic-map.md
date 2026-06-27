# Topic Map — How the Chapters Connect

A guide to the dependencies and relationships between the 14 chapters,
useful as a review reference and as a design guide for the banking project.

---

## Dependency Tree

```
01 Interfaces & Exceptions
   └─ foundation for: everything below

02 Classes & Enums
   └─ OOP building blocks used in: all domain models

03 Maven
   └─ project setup: dependencies, plugins, test runner

04 Collections  ←── uses: 01 (Comparable/Comparator), 02 (Enum keys)
   └─ used by: 06 Generics, 08 Threads, 09 Streams

05 UML & Testing ←── applies to: all chapters
   └─ used by: 07 Mocking

06 Generics ←── extends: 04 Collections
   └─ used by: 07, 09, 11

07 Mocking ←── requires: 01 (interfaces to mock), 05 (JUnit)
   └─ used in: 11 (mock Observer), 10 (mock factories)

08 Threads & Concurrency ←── uses: 04 (thread-safe collections)
   └─ appears in: 11 (background thread → Observer), 12 (JavaFX thread)

09 Lambdas & Streams ←── uses: 01 (functional interfaces), 04 (Collections)
   └─ simplifies: event handlers in 12-13, stream queries in banking project

10 I/O, Serialization & Patterns 1
   ├─ I/O Decorator ←── structural preview of: 14 Decorator
   ├─ Serialization ←── transient keyword reappears in: 11 (PropertyChangeSupport)
   ├─ Template Method ←── uses: 01 (abstract classes/interfaces)
   └─ Factory / Abstract Factory ←── uses: 01 (interfaces for products)

11 Observer & MVC ←── uses: 01 (interface), 08 (threads), 09 (lambdas for listeners)
   └─ directly extended by: 12-13 JavaFX (Properties replace PropertyChangeSupport)

12 JavaFX ←── applies: 11 (MVC + Observer via Properties), 09 (lambdas for handlers)
   └─ extended by: 13 (FXML layout)

13 JavaFX with FXML ←── extends: 12 (same patterns, XML layout + CSS)

14 Singleton, State & Decorator
   ├─ Singleton ←── uses: private constructor (02)
   ├─ State ←── uses: 01 (interface per state)
   └─ Decorator ←── mirrors: I/O stream wrapping (10); uses: 01 (shared base type)
```

---

## Central Themes

### Interfaces are the backbone
Almost every pattern depends on interfaces:
- **Lambdas** need functional interfaces (single abstract method)
- **Observer/MVC** — `PropertyChangeListener`, `ActionListener`
- **Template Method** — abstract method contracts
- **Factory** — product interface decouples creation from use
- **State** — one interface per machine; all states implement it
- **Decorator** — decorator and base share the same type
- **Mocking** — Mockito mocks interfaces and abstract classes

### Design patterns across chapters
Patterns appear in multiple chapters; the chocolate examples in ch14 are deliberate callbacks:

| Pattern | First appearance | Revisited |
|---------|-----------------|-----------|
| Decorator | Ch 10 (I/O stream wrapping) | Ch 14 (Chocolate toppings) |
| Observer | Ch 11 (custom `Beobachter`) → `PropertyChangeSupport` | Ch 12 (JavaFX Properties — built-in) |
| Factory | Ch 10 (BeverageFactory) | Bank exercise (AccountFactory/MockFactory) |
| Template Method | Ch 10 (`withdraw()`, `changeCurrency()`) | — |
| MVC | Ch 11 (Swing-style) | Ch 12-13 (JavaFX) |
| Singleton | Ch 14 (ChocolateMaker) | Useful for Bank instance in project |
| State | Ch 14 (ChocolateMaker states) | Account lifecycle in project |

### Observer evolution across chapters
Each chapter refines the same idea:

```
Ch 11 (custom)     → own Beobachter interface + List<Beobachter>
Ch 11 (Java built-in) → PropertyChangeSupport + PropertyChangeListener
Ch 12 (JavaFX)     → Properties + bind() / addListener() — no boilerplate needed
Ch 13 (FXML)       → same Properties, declared in XML with ${binding} syntax
```

### Lambdas make everything shorter
Lambdas (ch9) appear as simplifications throughout:
- Ch 11: `forEach(b -> b.update(this))` in `notifyObservers()`
- Ch 11: `arg0 -> controller.sunshine()` as ActionListener
- Ch 12: `e -> controller.close()` as EventHandler
- Ch 13: `(Observable e) -> changeGenre(...)` as InvalidationListener
- Banking project: all stream queries

---

## How Chapters Map to Banking Project Phases

| Project phase | Chapters it applies |
|--------------|---------------------|
| Core model (Account, Customer, exceptions) | 01, 02 |
| Maven setup | 03 |
| Collections + sorting | 04 |
| Unit tests | 05 |
| Generic repository | 06 |
| Mockito tests | 07 |
| Thread-safe transactions | 08 |
| Account queries and reports | 09 |
| File persistence + account factory | 10 |
| Observable accounts + console MVC | 11 |
| JavaFX GUI | 12 |
| FXML-based GUI | 13 |
| Singleton Bank + State for account lifecycle | 14 |

---

## Quick Concept Lookup

| Concept | Chapter |
|---------|---------|
| `Comparable` / `Comparator` | 01, 04 |
| `enum` | 02 |
| Maven `pom.xml`, `mvn test` | 03 |
| `HashMap`, `ArrayList`, `LinkedList` | 04 |
| JUnit 5 (`@Test`, `@BeforeEach`) | 05 |
| Generics (`<T extends ...>`) | 06 |
| Mockito (`mock`, `verify`, `when`) | 07 |
| `synchronized`, `ReentrantLock`, `volatile` | 08 |
| Lambda expressions, method references | 09 |
| Stream pipeline, `Collectors` | 09 |
| `BufferedReader`, `try-with-resources` | 10 |
| `Serializable`, `transient`, `serialVersionUID` | 10 |
| Template Method, Factory, Abstract Factory | 10 |
| Observer / `PropertyChangeSupport` | 11 |
| MVC (Model / View / Controller) | 11, 12, 13 |
| JavaFX `Stage`, `Scene`, `Application` | 12 |
| JavaFX `Property`, `bind()`, `bindBidirectional()` | 12 |
| FXML, `@FXML`, `initialize()`, `FXMLLoader` | 13 |
| JavaFX CSS | 13 |
| Singleton | 14 |
| State pattern | 14 |
| Decorator pattern | 14 |
