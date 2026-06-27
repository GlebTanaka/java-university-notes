# Java University Notes — Migration Plan

Source: `MyNotesProg03.txt` (~14,600 lines, originally written in German)

## Goal

Migrate raw text notes into clean, English-language Markdown files organized by topic.

## Chapters

| # | Original Title | English Title | Lines | Target File |
|---|---------------|---------------|-------|-------------|
| 1 | Interface, JavaDoc, Exceptions, Comparable vs Comparator | Interfaces, JavaDoc, Exceptions, Comparable vs Comparator | 1–622 | `01-interfaces-exceptions.md` |
| 2 | Klassen, Enums, Shutdown-Hook | Classes, Enums, Shutdown Hook | 623–1263 | `02-classes-enums.md` |
| 3 | Maven | Maven | 1264–1966 | `03-maven.md` |
| 4 | Collections | Collections | 1967–2465 | `04-collections.md` |
| 5 | UML, Testen | UML & Testing | 2466–2997 | `05-uml-testing.md` |
| 6 | Generische Klassen | Generics | 2998–3734 | `06-generics.md` |
| 7 | Mocking | Mocking | 3735–4700 | `07-mocking.md` |
| 8 | Threads | Threads & Concurrency | 4701–6356 | `08-threads-concurrency.md` |
| 9 | Lambda-Ausdrücke + Streams | Lambdas & Streams | 6357–7903 | `09-lambdas-streams.md` |
| 10 | I/O Streams, Serialisierung + Template/Factory Entwurfsmuster 1 | I/O Streams, Serialization & Design Patterns 1 | 7904–10008 | `10-io-serialization.md` |
| 12 | Observer und MVC - Entwurfsmuster 2 | Observer & MVC Patterns | 10009–11319 | `11-observer-mvc.md` |
| 13 | JavaFX | JavaFX | 11320–12853 | `12-javafx.md` |
| 14 | Oberflächen mit FXML | JavaFX with FXML | 12854–13788 | `13-javafx-fxml.md` |
| 15 | Singleton, State, Decoder - Entwurfsmuster 3 | Singleton, State & Decoder Patterns | 13789–14619 | `14-design-patterns.md` |

## Folder Structure

```
java/
├── MyNotesProg03.txt          # original source (keep as reference)
├── OVERVIEW.md                # this file
└── notes/
    ├── 01-interfaces-exceptions.md
    ├── 02-classes-enums.md
    ├── 03-maven.md
    ├── 04-collections.md
    ├── 05-uml-testing.md
    ├── 06-generics.md
    ├── 07-mocking.md
    ├── 08-threads-concurrency.md
    ├── 09-lambdas-streams.md
    ├── 10-io-serialization.md
    ├── 11-observer-mvc.md
    ├── 12-javafx.md
    ├── 13-javafx-fxml.md
    └── 14-design-patterns.md
```

## Migration Guidelines

- Translate all content from German to English
- Convert ASCII box diagrams (`+---+`) to fenced code blocks
- Convert bullet symbols (`°`, `~`, `>`, `•`) to standard Markdown lists
- Preserve all code examples in fenced Java code blocks (` ```java `)
- Turn section markers (`° Section`, `> Item`) into `##` / `###` headings
- Keep external URLs as Markdown links
- Fix known typos from the source where obvious (e.g. `defualt` → `default`)

## Status

| File | Status |
|------|--------|
| `01-interfaces-exceptions.md` | done |
| `02-classes-enums.md` | done |
| `03-maven.md` | done |
| `04-collections.md` | done |
| `05-uml-testing.md` | done |
| `06-generics.md` | done |
| `07-mocking.md` | done |
| `08-threads-concurrency.md` | done |
| `09-lambdas-streams.md` | done |
| `10-io-serialization.md` | done |
| `11-observer-mvc.md` | done |
| `12-javafx.md` | done |
| `13-javafx-fxml.md` | done |
| `14-design-patterns.md` | done |
