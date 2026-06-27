# Chapter 12 — JavaFX

> **Note on version:** These notes were written in 2021 with JavaFX 11/16. Since Java 11, JavaFX
> is no longer bundled with the JDK — it lives as a separate open-source project called
> **OpenJFX** (openjfx.io), maintained by Gluon. The API is stable and largely unchanged;
> only the setup/version numbers need updating for modern projects. Current stable release is
> JavaFX 21+ (LTS). Use `org.openjfx:javafx-controls:21` and plugin version `0.0.8+`.

---

## Maven Setup

```xml
<!-- Core controls -->
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-controls</artifactId>
    <version>11.0.2</version>   <!-- use 21 for modern projects -->
</dependency>

<!-- For FXML (next chapter) -->
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-fxml</artifactId>
    <version>11.0.2</version>
</dependency>
```

Maven plugin to run the app:

```xml
<plugin>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-maven-plugin</artifactId>
    <version>0.0.4</version>   <!-- use 0.0.8+ for modern projects -->
    <configuration>
        <!-- Fully qualified name of the class with main() -->
        <mainClass>javaFxVorlesung.VolumeController1</mainClass>
    </configuration>
</plugin>
```

Run with Maven (the IntelliJ run button does not work for JavaFX):

```
mvn javafx:run
```

---

## JavaFX Structure

JavaFX is built around three layers:

| Term | What it is |
|------|-----------|
| **Stage** | The window — title bar, size, close behaviour. On mobile it can be the full touch screen. |
| **Scene** | The content area inside the window. Contains all controls and graphical elements. |
| **Node** | Everything in the scene graph inherits from `Node` (controls, shapes, containers). |

The entry point class must extend `Application` and override `start(Stage)`:

```java
public class MyController extends Application {

    @Override
    public void start(Stage primaryStage) {
        Parent root = new MyView(/* model, controller */);
        Scene scene = new Scene(root, 320, 343);
        primaryStage.setScene(scene);
        primaryStage.setTitle("My App");
        primaryStage.show();
    }

    public static void main(String[] args) {
        launch(args);   // starts the JavaFX runtime; calls start()
    }
}
```

---

## Example — Volume Controller (MVC)

A small audio configuration panel: a slider for volume, a mute checkbox, and a genre dropdown.

### Model

```java
public class VolumeModel {
    private IntegerProperty volume = new SimpleIntegerProperty(50);
    private BooleanProperty muted  = new SimpleBooleanProperty(false);

    private ObservableList<String> genres =
        FXCollections.observableArrayList(
            "Chamber", "Country", "Cowbell", "Metal", "Polka", "Rock");

    // Getters / setters / property accessors — see "JavaFX Properties" section
}
```

### Controller

```java
public class VolumeController1 extends Application {

    private VolumeModel model = new VolumeModel();
    private Stage stage;

    @Override
    public void start(Stage primaryStage) {
        stage = primaryStage;
        Parent view = new VolumeView1(model, this);   // pass model + controller
        Scene scene = new Scene(view, 320, 343);
        primaryStage.setScene(scene);
        primaryStage.setTitle("Audio Configuration");
        primaryStage.show();
    }

    public static void main(String[] args) { launch(args); }

    // Called from view when a genre is selected
    protected void changeGenre(String selected) {
        switch (selected) {
            case "Chamber": model.setVolume(80);  break;
            case "Country": model.setVolume(100); break;
            case "Cowbell": model.setVolume(150); break;
            case "Metal":   model.setVolume(140); break;
            case "Polka":   model.setVolume(120); break;
            case "Rock":    model.setVolume(130); break;
        }
    }

    public void close() { stage.close(); }
}
```

### View

```java
public class VolumeView1 extends Group {   // Group = empty container pane

    private Slider slider;
    private CheckBox muteCheckBox;
    private ChoiceBox<String> genreChoiceBox;
    private Text dbText;

    private VolumeModel model;
    private VolumeController1 controller;

    public VolumeView1(VolumeModel model, VolumeController1 controller) {
        this.model = model;
        this.controller = controller;
        this.build();
    }

    private void build() {
        // --- Title text ---
        Text title = new Text();
        title.setLayoutX(65);
        title.setLayoutY(20);
        title.setText("Audio Configuration");
        title.setFill(Color.RED);
        title.setFont(new Font("Sans Serif", 20));

        // --- Close button ---
        Button btnClose = new Button("Close");
        btnClose.setLayoutX(130);
        btnClose.setLayoutY(200);
        btnClose.setOnAction(e -> controller.close());   // lambda event handler

        // --- Genre choice box ---
        genreChoiceBox = new ChoiceBox<>();
        genreChoiceBox.setLayoutX(204);
        genreChoiceBox.setLayoutY(154);
        genreChoiceBox.setPrefWidth(93);
        genreChoiceBox.setItems(model.getGenres());      // bind list from model
        genreChoiceBox.getSelectionModel().selectFirst();

        // --- Add everything to the container ---
        this.getChildren().add(title);
        this.getChildren().add(dbText);
        this.getChildren().add(slider);
        this.getChildren().add(muteCheckBox);
        this.getChildren().add(genreChoiceBox);
        this.getChildren().add(btnClose);
    }
}
```

`this.getChildren().add(node)` is how any container puts an element on screen.
Every container has a children list accessible via `getChildren()`.

---

## Container Types

All containers inherit from `Parent` (and therefore `Node`):

| Container | Layout strategy |
|-----------|----------------|
| `Group`, `Pane`, `AnchorPane` | No automatic layout — position elements with `setLayoutX/Y` |
| `HBox` | Arrange children side by side |
| `VBox` | Arrange children top to bottom |
| `BorderPane` | Five zones: top, bottom, left, right, center |
| `GridPane` | Table layout with rows and columns |
| `StackPane` | Stack children on top of each other (later children cover earlier) |
| `TabPane` | Multiple tabs, each holding a `Tab` |
| `SplitPane` | Divide area into two or more resizable regions |
| `ScrollPane` | Add a scrollbar around any content |

---

## Event Handling

Controls and nodes raise events (mouse clicks, keyboard, drag, etc.).

### `setOnAction()` — simplest form for buttons and choice boxes

```java
// Anonymous class form:
btnClose.setOnAction(new EventHandler<ActionEvent>() {
    @Override
    public void handle(ActionEvent e) {
        controller.close();
    }
});

// Lambda form (preferred):
btnClose.setOnAction(e -> controller.close());
```

`EventHandler<ActionEvent>` is a functional interface — the lambda replaces it directly.

### `addEventHandler()` — for any event type

```java
// Attach a specific event type:
genreChoiceBox.addEventHandler(ActionEvent.ACTION,
    e -> controller.changeGenre(genreChoiceBox.getValue()));

// Attach ALL events (useful for debugging — prints every event type):
genreChoiceBox.addEventHandler(EventType.ROOT, e -> System.out.println(e));
// Output: eventType = MOUSE_MOVED, SCROLL, ROTATE, etc.
```

### General syntax

```java
// Option A — setOn...() shortcut
element.setOnAction(e -> controller.doSomething(e));

// Option B — addEventHandler() with explicit type
element.addEventHandler(EventClass.EVENT_TYPE, e -> controller.doSomething(e));
```

**Important difference from Swing:** JavaFX controls have only one `setOnAction()` handler (replaces any previous one). Use `addEventHandler()` to register multiple listeners on the same event.

---

## JavaFX Properties and Observer

JavaFX has the Observer pattern built in. Model objects expose their fields as **Properties** — observable wrappers around primitive values or objects.

### Creating properties in the Model

```java
// Declare as a Property instead of a plain primitive:
private IntegerProperty volume = new SimpleIntegerProperty(50);
private BooleanProperty muted  = new SimpleBooleanProperty(false);

// Standard getter — returns the unwrapped value
public int getVolume() { return volume.get(); }

// Standard setter — stores into the Property
public void setVolume(int v) { this.volume.set(v); }

// Property accessor — returns the Property object itself (never create a new one!)
public IntegerProperty volumeProperty() { return this.volume; }
```

Naming convention: `getX()`, `setX()`, `xProperty()`.
IntelliJ can generate all three automatically once you declare the field as a Property type.

### Property types

| Java type | Property class | Simple concrete |
|-----------|---------------|----------------|
| `int` | `IntegerProperty` | `SimpleIntegerProperty` |
| `double` | `DoubleProperty` | `SimpleDoubleProperty` |
| `boolean` | `BooleanProperty` | `SimpleBooleanProperty` |
| `String` | `StringProperty` | `SimpleStringProperty` |
| `Object` / generic | `ObjectProperty<T>` | `SimpleObjectProperty<T>` |
| `List<E>` | `ListProperty<E>` | `SimpleListProperty<E>` |

For read-only properties (externally immutable), use `ReadOnly...Wrapper` and expose via `getReadOnlyProperty()`:

```java
private ReadOnlyBooleanWrapper muted = new ReadOnlyBooleanWrapper(false);

public ReadOnlyBooleanProperty mutedProperty() {
    return muted.getReadOnlyProperty();   // callers can observe but not set
}

// Can still change internally:
private void setMuted(boolean v) { muted.set(v); }
```

### ObservableList

```java
private ObservableList<String> genres =
    FXCollections.observableArrayList("Chamber", "Country", "Rock");

public ObservableList<String> getGenres() { return genres; }
```

`ChoiceBox` / `ListView` accept an `ObservableList` directly — changes in the list automatically update the control.

---

## Binding

Binding connects two observable values so that one automatically tracks the other.

### One-way binding — `bind()`

```java
// Slider value always mirrors model volume:
slider.valueProperty().bind(model.volumeProperty());

// Text field always shows model volume as a string:
dbText.textProperty().bind(model.volumeProperty().asString().concat(" dB"));

// Slider disabled when model is muted:
slider.disableProperty().bind(model.mutedProperty());
```

**Warning:** after `bind()`, you can no longer set the target property manually — it's controlled by the binding.

### Bidirectional binding — `bindBidirectional()`

```java
// Checkbox checked ↔ model.muted; changing either updates the other:
muteCheckBox.selectedProperty().bindBidirectional(model.mutedProperty());
```

Use bidirectional when the UI control should both display and change the model value.

### Binding vs. addListener()

Both achieve the same result; binding is more concise:

```java
// Option 1 — addListener (Observer style)
model.volumeProperty().addListener(e -> slider.setValue(model.getVolume()));

// Option 2 — bind (declarative, shorter)
slider.valueProperty().bind(model.volumeProperty());
```

### InvalidationListener vs. ChangeListener

```java
// InvalidationListener — fires when the observable becomes invalid (no old/new value)
model.positiveProperty().addListener(observable -> this.reformat());

// ChangeListener — fires with old and new value
adresse.textProperty().addListener(
    (ObservableValue<? extends String> obs, String oldVal, String newVal) -> {
        model.getOwner().setAddress(newVal);
    });
```

### Where to put bindings

| What | Where |
|------|-------|
| View observes model | In the View's `build()` / constructor — using `bind()` or `addListener()` |
| Genre selection triggers controller | In the View via `setOnAction()` or `valueProperty().addListener()` |
| View updates via model change | Never directly from controller to view — go through the model |

---

## MVC Flow in JavaFX

```
User action (click genre)
    → View: genreChoiceBox.valueProperty listener fires
    → Controller: changeGenre(selected) called
    → Model: model.setVolume(130) — Property changes
    → View: slider.valueProperty bound to model.volumeProperty() — slider moves
            dbText.textProperty bound to model.volumeProperty().asString() — text updates
```

The model's Properties replace `PropertyChangeSupport` from the Swing/Observer chapter.
No `firePropertyChange()` needed — changing a `SimpleIntegerProperty` automatically notifies all bound controls.

---

## Bank Exercise — Account View

A JavaFX GUI for a single `CheckingAccount`.

### Controller

```java
public class AccountController extends Application {

    private CheckingAccount accountModel =
        new CheckingAccount(new Customer(), 5662233, 500.0);
    private AccountView view;
    private Stage stage;

    @Override
    public void start(Stage primaryStage) {
        stage = primaryStage;
        primaryStage.setTitle("Checking Account");
        view = new AccountView(accountModel, this);
        Scene scene = new Scene(view, 300, 300, false);
        primaryStage.setScene(scene);
        primaryStage.show();
    }

    public static void main(String[] args) { launch(args); }

    public void deposit(double amount) {
        try {
            accountModel.deposit(amount);
            view.showMessage("Deposit successful");
        } catch (IllegalArgumentException e) {
            view.showMessage("Amount was negative");
        }
    }

    public void withdraw(double amount) {
        try {
            if (accountModel.withdraw(amount)) {
                view.showMessage("Withdrawal successful");
            } else {
                view.showMessage("Insufficient balance");
            }
        } catch (IllegalArgumentException | LockedException e) {
            view.showMessage("Error during withdrawal");
        }
    }
}
```

### Account model — add Properties

```java
public abstract class Account implements Comparable<Account>, Serializable {

    // Replace plain double with a Property
    private ReadOnlyDoubleWrapper balance = new ReadOnlyDoubleWrapper();

    public double getBalance()                    { return balance.get(); }
    public ReadOnlyDoubleProperty balanceProperty() { return balance.getReadOnlyProperty(); }

    protected void setBalance(double b) {
        double old = balance.get();
        balance.set(b);
        // Track whether balance is positive (for colour formatting in view)
        isPositiveBalance.set(b >= 0);
        prop.firePropertyChange("Balance", old, b);   // keep for legacy observers
    }

    // Read-only boolean: positive balance?
    private ReadOnlyBooleanWrapper isPositiveBalance = new ReadOnlyBooleanWrapper(true);
    public ReadOnlyBooleanProperty positiveProperty() { return isPositiveBalance.getReadOnlyProperty(); }

    // Locked state — bidirectionally bindable
    private BooleanProperty locked = new SimpleBooleanProperty(false);
    public BooleanProperty  lockedProperty()   { return locked; }
    public boolean          isLocked()         { return locked.get(); }
    public void             setLocked(boolean v){ locked.set(v); }
}
```

```java
// In Customer: make address observable
public class Customer {
    private StringProperty address;

    public Customer(String first, String last, String address, LocalDate birthday) {
        // ...
        this.address = new SimpleStringProperty(address);
    }

    public String         getAddress()      { return address.get(); }
    public void           setAddress(String a) { address.set(a); }
    public StringProperty addressProperty() { return address; }
}
```

### View

```java
public class AccountView extends BorderPane {

    private Text header, numberLabel, balanceDisplay, lockedLabel;
    private CheckBox lockedCheckBox;
    private TextArea addressField;
    private TextField amountField;
    private Button depositBtn, withdrawBtn;
    private Text messageText;

    private Account accountModel;
    private AccountController controller;

    public AccountView(Account model, AccountController controller) {
        this.accountModel = model;
        this.controller = controller;
        this.build();
    }

    private void build() {
        // --- Header ---
        header = new Text("Edit Account");
        header.setFont(new Font("Sans Serif", 25));
        BorderPane.setAlignment(header, Pos.CENTER);
        this.setTop(header);

        // --- Center grid ---
        GridPane grid = new GridPane();
        grid.setPadding(new Insets(20));
        grid.setVgap(10);
        grid.setAlignment(Pos.CENTER);

        // Account number — static, no Property needed
        grid.add(new Text("Account number:"), 0, 0);
        Text numberText = new Text(accountModel.getAccountNumberFormatted());
        grid.add(numberText, 1, 0);

        // Balance — bind one-way: text always mirrors model balance
        grid.add(new Text("Balance:"), 0, 1);
        balanceDisplay = new Text();
        balanceDisplay.textProperty().bind(accountModel.balanceProperty().asString());
        accountModel.positiveProperty().addListener(
            obs -> formatBalance());   // update colour when sign changes
        grid.add(balanceDisplay, 1, 1);

        // Locked — bidirectional: checkbox changes model, model change updates checkbox
        grid.add(new Text("Locked:"), 0, 2);
        lockedCheckBox = new CheckBox();
        lockedCheckBox.selectedProperty().bindBidirectional(accountModel.lockedProperty());
        grid.add(lockedCheckBox, 1, 2);

        // Address — bidirectional text binding
        grid.add(new Text("Address:"), 0, 3);
        addressField = new TextArea();
        addressField.setPrefColumnCount(25);
        addressField.setPrefRowCount(2);
        // Option 1 — bind (best: also writes initial value into field)
        addressField.textProperty().bindBidirectional(
            accountModel.getOwner().addressProperty());
        // Option 2 — listener (alternative)
        // addressField.textProperty().addListener(
        //     obs -> accountModel.getOwner().setAddress(addressField.getText()));
        grid.add(addressField, 1, 3);

        this.setCenter(grid);

        // --- Bottom: deposit / withdraw ---
        HBox actions = new HBox(10);
        actions.setAlignment(Pos.CENTER);

        // Amount field — regex filter: up to 7 digits, optional 2 decimal places
        amountField = new TextField("100.00");
        amountField.textProperty().addListener(
            (obs, oldVal, newVal) -> {
                if (!newVal.matches("\\d{0,7}([.]\\d{0,2})?"))
                    amountField.setText(oldVal);
            });
        actions.getChildren().add(amountField);

        depositBtn = new Button("Deposit");
        depositBtn.setOnAction(e -> {
            try {
                controller.deposit(Double.parseDouble(amountField.getText()));
            } catch (NumberFormatException ex) {
                showMessage("Enter a number");
            }
        });
        actions.getChildren().add(depositBtn);

        withdrawBtn = new Button("Withdraw");
        withdrawBtn.setOnAction(e -> {
            try {
                controller.withdraw(Double.parseDouble(amountField.getText()));
            } catch (NumberFormatException ex) {
                showMessage("Enter a number");
            }
        });
        actions.getChildren().add(withdrawBtn);

        this.setBottom(actions);
    }

    private void formatBalance() {
        balanceDisplay.setFill(
            accountModel.positiveProperty().get() ? Color.BLACK : Color.RED);
    }

    public void showMessage(String text) {
        Alert alert = new Alert(Alert.AlertType.CONFIRMATION);
        alert.setTitle("Message");
        alert.setHeaderText(text);
        alert.showAndWait();
    }
}
```

---

## JavaFX vs. Swing — key differences

| | Swing (Chapter 11) | JavaFX |
|-|--------------------|--------|
| Observer wiring | `PropertyChangeSupport` / `PropertyChangeListener` | Built-in Properties + `bind()` / `addListener()` |
| Button events | `addActionListener(ActionListener)` — multiple listeners | `setOnAction(EventHandler)` — one handler; use `addEventHandler()` for more |
| Controller interface | Implements `ActionListener` | No special interface; methods referenced by lambda |
| Layout | Absolute or layout managers | Containers with automatic layout (`HBox`, `GridPane`, etc.) |
| CSS styling | Java-only | Full CSS support via `.css` files |

---

## Summary

In JavaFX, the Observer pattern is baked into the framework through **Properties**:

- Model fields declared as `IntegerProperty`, `BooleanProperty`, etc. are automatically observable.
- UI controls expose their state as Properties too (`slider.valueProperty()`, `text.textProperty()`).
- `bind()` / `bindBidirectional()` wires them together declaratively — no manual `firePropertyChange()` or observer lists.
- `addListener()` handles cases where a binding alone isn't enough (e.g. reformatting, calling controller methods).
- The MVC roles are the same: Model owns the data and business logic; View builds and binds controls; Controller creates both and handles user actions.
