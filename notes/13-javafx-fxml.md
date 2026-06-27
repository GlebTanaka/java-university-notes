# Chapter 13 — JavaFX with FXML

FXML is an XML dialect for declaring JavaFX UI layouts. Instead of building the scene graph in Java code, you write an XML file; a controller class handles bindings and event handlers.

---

## XML Basics

```xml
<?xml version="1.0" ?>              <!-- XML declaration — always first -->

<rootElement xmlns:ns="...">        <!-- xmlns declares a namespace; ns is its prefix -->
    <outer>
        <ns:inner>Content</ns:inner>
    </outer>
    <selfClosing />
    <withAttribute name="value" />  <!-- attributes only in opening tags -->
</rootElement>

<!-- Comment -->
```

---

## FXML Syntax Overview

### Imports

```xml
<?import java.lang.*?>
<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>
<?import javafx.scene.text.Font?>
```

Same as Java imports — any class used in the file must be imported.

### Root element

```xml
<Group xmlns="http://javafx.com/javafx/11.0.2"
       xmlns:fx="http://javafx.com/fxml/1">
    ...
</Group>
```

- Root element must be a class that inherits from `Parent`.
- Two standard namespace declarations — do not change the prefixes.
- Use the JavaFX version you're targeting in the first `xmlns`.

### Creating objects

| FXML | Java equivalent |
|------|----------------|
| `<?import package.Class?>` | `import package.Class;` |
| `<Class attr="val" />` | `Class v = new Class(); v.setAttr(val);` |
| `<Class><attr>val</attr></Class>` | same as above but nested |
| A tag starting with **uppercase** | Constructor call |
| A tag starting with **lowercase** | Setter call (`<font>` → `setFont(...)`) |

### Children of containers

```xml
<children>
    <!-- everything inside here is added via getChildren().add() -->
    <Text layoutX="65" layoutY="20" text="Hello" />
</children>
```

`<children>` corresponds to the read-only list property — accessible via `getChildren()`.

### Example — Text with a Font sub-object

```xml
<!-- FXML -->
<Text layoutX="65" layoutY="20" text="Audio Configuration" fill="red">
    <font>
        <Font name="SansSerif" size="20" />
    </font>
</Text>
```

```java
// Equivalent Java code
Text t = new Text();
t.setLayoutX(65);  t.setLayoutY(20);
t.setText("Audio Configuration");
t.setFill(Color.RED);
t.setFont(new Font("Sans Serif", 20));
```

Primitive/String attributes go inline (`layoutX="65"`); object arguments need a nested tag.

### Read-only list properties

```xml
<Class>
    <rdonlyListAttribute>
        <Class1/>
    </rdonlyListAttribute>
</Class>
```

```java
// Equivalent
Class v = new Class();
v.getRdonlyListAttribute().add(new Class1());
```

---

## Using `fx:id`

`fx:id` gives an element a unique name in the FXML file. It serves two purposes:

1. **Connects to a controller field** with the same name (annotated `@FXML` or `public`).
2. **Makes the object referenceable** via `$name` syntax in the same file.

```xml
<Text fx:id="dbText" layoutX="18" layoutY="69" />
```

```java
// In controller:
@FXML private Text dbText;   // type and name must match
```

---

## Wiring the Controller

**Preferred: load in `start()` and set controller explicitly.**

```java
@Override
public void start(Stage primaryStage) throws IOException {
    FXMLLoader loader = new FXMLLoader(getClass().getResource("/MyView.fxml"));
    loader.setController(this);         // this controller object handles the FXML
    Parent root = loader.load();

    Scene scene = new Scene(root, 300, 275);
    primaryStage.setScene(scene);
    primaryStage.show();
}
```

**Alternative: declare controller inside FXML** (creates a separate object — usually not what you want):

```xml
<Group xmlns="..." xmlns:fx="..."
       fx:controller="mypackage.MyController">
```

---

## The FXML Controller Class

```java
public class VolumeController2 extends Application {

    private Stage stage;

    // UI elements — type and name match fx:id in the FXML
    @FXML private Slider         slider;
    @FXML private CheckBox       muteCheckBox;
    @FXML private ChoiceBox<String> genreChoiceBox;
    @FXML private Text           dbText;

    // Model — can also be created in FXML via <fx:define>
    @FXML private VolumeModel lsModel = new VolumeModel();

    @Override
    public void start(Stage primaryStage) throws IOException {
        stage = primaryStage;
        FXMLLoader loader = new FXMLLoader(
            getClass().getResource("../VolumeView2.fxml"));
        loader.setController(this);
        Parent root = loader.load();

        Scene scene = new Scene(root, 300, 275);
        primaryStage.setTitle("Audio Configuration");
        primaryStage.setScene(scene);
        primaryStage.show();
    }

    public static void main(String[] args) { launch(args); }

    // Called automatically after the FXML is loaded and @FXML fields are injected
    @FXML
    public void initialize() {
        genreChoiceBox.getSelectionModel().selectFirst();

        // These bindings can't be expressed in FXML — they go here
        dbText.textProperty().bind(
            lsModel.volumeProperty().asString().concat(" dB"));

        slider.valueProperty().bindBidirectional(lsModel.volumeProperty());

        muteCheckBox.selectedProperty().bindBidirectional(lsModel.mutedProperty());

        genreChoiceBox.valueProperty().addListener(
            (Observable e) -> changeGenre(genreChoiceBox.getValue()));
    }

    @FXML
    private void changeGenre(String selected) {
        switch (selected) {
            case "Chamber": lsModel.setVolume(80);  break;
            case "Country": lsModel.setVolume(100); break;
            case "Rock":    lsModel.setVolume(130); break;
            // ...
        }
    }

    @FXML
    private void close() { stage.close(); }
}
```

Key points:
- `@FXML` fields are populated by the `FXMLLoader` after `load()` — they are `null` before then.
- `initialize()` is called automatically right after injection; put complex bindings here.
- `@FXML` methods may be `private` — the loader can still call them.
- Field type and name must match `fx:id` exactly (e.g., `private Text dbText` ↔ `<Text fx:id="dbText" .../>`).

---

## Event Handling in FXML

In Java:
```java
btnClose.setOnAction(e -> controller.close());
```

In FXML (use `#methodName` to reference a controller method):
```xml
<Button layoutX="130" layoutY="200" text="Close" onAction="#close"/>
```

Rules for the referenced method:
- No return value
- Zero parameters **or** exactly one parameter of the matching `Event` subtype
- Must be `@FXML`-annotated if private

```java
@FXML
private void close() {         // zero params — fine
    stage.close();
}

@FXML
private void handleAction(ActionEvent e) {  // one param — fine
    // ...
}
```

General FXML event syntax:
```xml
<AnyControl onSomeEvent="#controllerMethod" />
```

---

## References and Bindings in FXML

### `$name` — use an object's value as initial value

```xml
<Slider fx:id="slider"
        min="$lsModel.minVolume"
        max="$lsModel.maxVolume" />
```

Java equivalent: `slider.setMin(lsModel.getMinVolume());`

- `$` signals a reference to an object in scope.
- `lsModel.minVolume` → calls `lsModel.getMinVolume()` (drop `get`, lowercase first letter, no parentheses).

### `${name}` — bind a property

```xml
<Slider fx:id="slider" disable="${lsModel.muted}" />
```

Java equivalent: `slider.disableProperty().bind(lsModel.mutedProperty());`

- Curly braces signal a live binding.
- `lsModel.muted` → `lsModel.mutedProperty()`.

### What FXML **cannot** express

Complex bindings that chain method calls must go in `initialize()`:

```java
// Cannot be written in FXML — type conversion + string concatenation:
dbText.textProperty().bind(lsModel.volumeProperty().asString().concat(" dB"));

// Cannot be written in FXML — bidirectional binding:
slider.valueProperty().bindBidirectional(lsModel.volumeProperty());
```

### Creating the Model in FXML

Instead of `new VolumeModel()` in Java, declare it in the FXML file — useful when the model is also data-bound inside the FXML:

```xml
<?import mypackage.VolumeModel?>

<Group ...>
    <fx:define>
        <!-- fx:define for objects that are NOT Nodes and don't go on screen -->
        <VolumeModel fx:id="lsModel" />
    </fx:define>

    <children>
        <!-- $lsModel.stile and ${lsModel.muted} now work inside FXML -->
        <ChoiceBox items="$lsModel.genres" ... />
        <Slider disable="${lsModel.muted}" ... />
    </children>
</Group>
```

The controller then only declares, not creates, the field:
```java
@FXML private VolumeModel lsModel;   // injected from FXML
```

---

## CSS Styling

JavaFX supports a subset of CSS via `.css` files with `-fx-` prefixed properties.

### Selectors

| Selector | Targets |
|----------|---------|
| `.styleClass` | All elements with `styleClass="format"` |
| `.node-class` | Most Node types have a default CSS class (e.g., `.button`, `.slider`) |
| `.node-class:pseudo-class` | Element in a specific state (e.g., `.button:hover`) |
| `#id` | Exactly one element with `id="elem"` |

### Example CSS file (`style.css`)

```css
/* Root background */
.root {
    -fx-background-color: #f4f4f4;
}

/* All elements with styleClass="anzeige" */
.anzeige {
    -fx-font: 15px "Sans Serif";
}

/* Single element by id */
#message {
    -fx-text-fill: red;
}

/* All buttons */
.button {
    -fx-background-color: #29B6F6;
    -fx-text-fill: #E1F5FE;
    -fx-font-weight: 900;
}

/* Button on hover */
.button:hover {
    -fx-background-color: #01579B;
}

/* CheckBox styling */
.check-box .box {
    -fx-background-color: white;
    -fx-border-color: grey;
    -fx-border-radius: 3px;
}

.check-box:selected .box  { -fx-background-color: red; }
.check-box:selected .mark { -fx-background-color: white; }

/* Ticks on Slider */
.slider {
    -fx-show-tick-labels: true;
}
```

### Attaching CSS

In FXML (preferred):
```xml
<Group ... stylesheets="@style.css">
```

In Java:
```java
scene.getStylesheets().add("style.css");
```

### Applying CSS classes to elements

```xml
<!-- Use styleClass for a shared style -->
<Text fx:id="dbText" styleClass="bla" ... />

<!-- Use id for a unique style (#id selector) -->
<Label fx:id="message" id="message" ... />
```

Inline style (one-off, no CSS file):
```java
stand.setStyle("-fx-text-fill: green;");   // in controller code
```

---

## Bank Exercise — Account View with FXML

### Controller (`AccountController2.java`)

```java
public class AccountController2 extends Application {

    // UI elements injected from FXML
    @FXML private Label    balance;
    @FXML private CheckBox locked;
    @FXML private TextArea address;
    @FXML private Label    message;
    @FXML private TextField amount;

    // Model injected from FXML
    @FXML private CheckingAccount checkingAccount;

    private Stage stage;

    @Override
    public void start(Stage primaryStage) throws IOException {
        stage = primaryStage;
        FXMLLoader loader = new FXMLLoader(
            getClass().getResource("/AccountView2.fxml"));
        loader.setController(this);
        Parent root = loader.load();

        Scene scene = new Scene(root, 540, 342);
        primaryStage.setScene(scene);
        primaryStage.setTitle("Account Management");
        primaryStage.show();
    }

    public static void main(String[] args) { launch(args); }

    @FXML
    public void initialize() {
        updateBalanceColor();
        checkingAccount.positiveProperty().addListener(
            obs -> updateBalanceColor());
        locked.selectedProperty().bindBidirectional(
            checkingAccount.lockedProperty());
        address.textProperty().bindBidirectional(
            checkingAccount.getOwner().addressProperty());
    }

    private void updateBalanceColor() {
        // In FXML controllers, helper methods live here (no separate View class)
        balance.setStyle(checkingAccount.isPositiveBalance()
            ? "-fx-text-fill: green;"
            : "-fx-text-fill: red;");
    }

    @FXML
    public void deposit() {
        String value = amount.getText();
        try {
            checkingAccount.deposit(Double.parseDouble(value));
            message.setText("Deposited " + value + " to account "
                + checkingAccount.getAccountNumber());
        } catch (NumberFormatException e) {
            message.setText("Please enter a valid number");
        } catch (IllegalArgumentException e) {
            message.setText(e.getMessage());
        }
    }

    @FXML
    public void withdraw() {
        String value = amount.getText();
        try {
            boolean ok = checkingAccount.withdraw(Double.parseDouble(value));
            message.setText(ok
                ? "Withdrew " + value + " from account " + checkingAccount.getAccountNumber()
                : "Enter a different amount");
        } catch (IllegalArgumentException | LockedException e) {
            message.setText(e.getMessage());
        }
    }

    @FXML
    public void close() { stage.close(); }
}
```

**Note:** `@FXML` methods here take no parameters — the value is read directly from the `amount` text field inside the method. In FXML, you can't pass parameters from the button to the handler.

### FXML File (`AccountView2.fxml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>

<?import java.lang.*?>
<?import javafx.scene.*?>
<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>
<?import javafx.scene.text.Text?>
<?import javafx.scene.text.Font?>
<?import javafx.geometry.Insets?>
<?import bankproject.model.CheckingAccount?>

<Group xmlns="http://javafx.com/javafx/11.0.2"
       xmlns:fx="http://javafx.com/fxml/1"
       stylesheets="@styleAccount.css">

    <!-- Model — not a Node, so it goes in fx:define -->
    <fx:define>
        <CheckingAccount fx:id="checkingAccount"/>
    </fx:define>

    <children>

        <BorderPane prefHeight="350.0" prefWidth="540.0" styleClass="root">

            <!-- Top: heading -->
            <top>
                <Text text="Edit Account" BorderPane.alignment="CENTER">
                    <font><Font name="SansSerif" size="25"/></font>
                </Text>
            </top>

            <!-- Center: data grid -->
            <center>
                <GridPane fx:id="grid" alignment="CENTER" vgap="10.0"
                          BorderPane.alignment="CENTER" styleClass="grid">
                    <children>

                        <!-- Labels (column 0) -->
                        <Label text="Account number:"/>
                        <Label text="Balance:"      GridPane.rowIndex="1"/>
                        <Label text="Locked:"       GridPane.rowIndex="2"/>
                        <Label text="Address:"      GridPane.rowIndex="3"/>

                        <!-- Message row -->
                        <Label fx:id="message" text="Welcome"
                               GridPane.rowIndex="4" id="message"/>

                        <!-- Values (column 1) -->
                        <!-- Account number: static read — no Property needed -->
                        <Label fx:id="number"
                               GridPane.columnIndex="1" GridPane.halignment="RIGHT"
                               text="$checkingAccount.accountNumberFormatted"/>

                        <!-- Balance: one-way property binding via ${...} -->
                        <Label fx:id="balance"
                               GridPane.columnIndex="1" GridPane.halignment="RIGHT"
                               GridPane.rowIndex="1"
                               text="${checkingAccount.balance}"/>

                        <!-- Locked: CheckBox — bidirectional binding done in initialize() -->
                        <CheckBox fx:id="locked"
                                  GridPane.columnIndex="1" GridPane.halignment="RIGHT"
                                  GridPane.rowIndex="2"/>

                        <!-- Address: TextArea — bidirectional binding done in initialize() -->
                        <TextArea fx:id="address"
                                  prefColumnCount="25" prefRowCount="2"
                                  GridPane.columnIndex="1" GridPane.halignment="RIGHT"
                                  GridPane.rowIndex="3"/>
                    </children>
                    <padding>
                        <Insets bottom="20.0" left="20.0" right="20.0" top="20.0"/>
                    </padding>
                </GridPane>
            </center>

            <!-- Bottom: deposit / withdraw actions -->
            <bottom>
                <HBox fx:id="actions" alignment="CENTER" spacing="10.0"
                      BorderPane.alignment="CENTER">
                    <children>
                        <TextField fx:id="amount" text="100.00"/>

                        <!-- onAction="#method" calls controller.method() on click -->
                        <Button text="Deposit"  onAction="#deposit"  mnemonicParsing="false"/>
                        <Button text="Withdraw" onAction="#withdraw" mnemonicParsing="false"/>
                    </children>
                    <padding>
                        <Insets bottom="20.0" left="20.0" right="20.0" top="20.0"/>
                    </padding>
                </HBox>
            </bottom>

        </BorderPane>

    </children>

</Group>
```

---

## FXML vs. Pure JavaFX — Summary

| Aspect | Pure JavaFX (ch. 12) | FXML (ch. 13) |
|--------|----------------------|---------------|
| Layout defined in | Java `build()` method | `.fxml` XML file |
| Controller | Separate from View class | Combined: one controller class handles logic + bindings |
| `@FXML` annotation | Not used | Required on fields/methods wired to FXML |
| `initialize()` | Not needed | Called automatically after loading; put bindings here |
| Event handler | `element.setOnAction(e -> ...)` in Java | `onAction="#method"` in FXML |
| Simple property binding | `slider.valueProperty().bind(...)` in Java | `value="${model.prop}"` in FXML |
| Complex bindings | Anywhere in View | Must be in `initialize()` (FXML can't express them) |
| CSS | `scene.getStylesheets().add(...)` | `stylesheets="@style.css"` on root element |
| Tooling | Pure code | Works with **SceneBuilder** (drag-and-drop UI editor) |

**When to prefer FXML:** when you want to design the UI visually (SceneBuilder), or want a clean separation of layout markup from Java logic.  
**When to prefer pure JavaFX:** when the layout is highly dynamic (generated at runtime), or for very simple UIs where the extra file is unnecessary overhead.

---

## Practical Note — Imports

Always import from `javafx.scene.control.*`, not `java.awt.*`. IntelliJ sometimes auto-imports the wrong package, which causes silent failures (methods exist but don't work as expected):

```java
import javafx.scene.control.*;   // correct
// import java.awt.*;            // wrong — causes subtle errors
```
