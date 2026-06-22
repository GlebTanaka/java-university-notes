# Practice Project — Java Banking App

A step-by-step personal project to reinforce the topics from the university notes.
Each phase maps to one chapter and adds a layer of features on top of the previous one.
Start this after the notes migration is complete.

## Why banking?

The same `Bank` / `Account` / `Customer` domain runs through the entire course as the exercise vehicle.
Building it from scratch means mental energy goes to the Java concepts, not to understanding the domain.

## Phases

| Phase | Chapter topic | What to build |
|-------|--------------|---------------|
| 1 | Interfaces, Exceptions | `Account` interface, `Customer` class, custom exceptions (`LockedException`, `InsufficientFundsException`) |
| 2 | Classes & Enums | `AbstractAccount`, `CheckingAccount`, `SavingsAccount`, `Currency` enum, `Shutdown Hook` for cleanup |
| 3 | Maven | Set up the Maven project: `pom.xml`, folder structure, JUnit 5 dependency |
| 4 | Collections | `Bank` class with `Map<Long, Account>` account store; `Comparable` / `Comparator` for sorting |
| 5 | UML & Testing | Write JUnit tests for deposit, withdrawal, transfer; document design with a UML diagram |
| 6 | Generics | Generic `Repository<T>` or typed `AccountList<T extends Account>` container |
| 7 | Mocking | Use Mockito to test service classes in isolation (mock the `Bank`, mock accounts) |
| 8 | Concurrency | Thread-safe `deposit()` / `withdraw()` with `synchronized` or `ReentrantLock`; simulate concurrent transactions |
| 9 | Lambdas & Streams | `getCustomersAboveBalance()`, `lockInsolventAccounts()`, birthday report — all via stream pipelines |
| 10 | Serialization & Patterns | Save/load bank state to file; Template Method for transaction flow; Factory for account creation; Abstract Factory for mock vs. real accounts |
| 11 | Observer & MVC | Make `Account` observable (`PropertyChangeSupport`); console MVC front-end |
| 12 | JavaFX | Replace console view with a JavaFX GUI: account list, deposit/withdraw form |
| 13 | JavaFX + FXML | Rewrite the GUI using FXML + controller classes |
| 14 | More patterns | Singleton `Bank` instance; State pattern for account lifecycle (active → locked → closed); Decoder for parsing bank file format |

## Suggested repo structure

```
banking-app/
├── pom.xml
└── src/
    ├── main/java/
    │   ├── model/          # Account, Customer, Bank, enums
    │   ├── service/        # Business logic, factories
    │   ├── observer/       # PropertyChangeListener implementations
    │   ├── controller/     # MVC controllers
    │   └── ui/             # JavaFX views, FXML files
    └── test/java/          # JUnit + Mockito tests mirroring main structure
```

## Notes

- Each phase should leave the project in a working, tested state before moving on.
- Commit after each phase with a message like `Phase 3: add Maven setup`.
- The goal is not a production app — keep it simple enough that the *pattern* is clear.
