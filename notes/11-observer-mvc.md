# Chapter 11 — Observer & MVC Patterns

## Observer Pattern

### When to use it

An object changes its state over time. Multiple **observers** should be notified about every state change and update themselves accordingly.

---

### Motivating example — Weather Station

A `WeatherData` class holds temperature, humidity, and air pressure:

```java
public class WeatherData {
    private float temperature, humidity, airPressure;
    private float sunshine = 0;
    private float rain = 0;

    public WeatherData() {
        this.temperature = 0;
        this.humidity = 0;
        this.airPressure = 0;
    }

    public void setMeasurements(float temp, float humid, float pressure) {
        this.temperature = temp + sunshine;
        this.humidity = humid + rain;
        this.airPressure = pressure;
    }

    public float getTemperature() { return temperature; }
    public float getAirPressure() { return airPressure; }

    public void sunshineDance() { sunshine++; }
    public void rainDance() { rain++; }
}
```

A `WeatherStation` class has a `main()` method and a helper that changes values every 2 seconds:

```java
public class WeatherStation {

    public static void changeWeather(WeatherData w) {
        Random r = new Random();
        while (true) {
            w.setMeasurements(r.nextInt(35), r.nextInt(100), (float)(Math.random() + 1010));
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
        }
    }

    public static void main(String[] args) {
        WeatherData weatherData = new WeatherData();
        // ...
        // new Thread(() -> changeWeather(weatherData)).start();
    }
}
```

Two display classes must be updated whenever `WeatherData` changes:
- `CurrentConditionsDisplay` — shows current temperature and humidity
- `StatisticsDisplay` — tracks min/max/average temperature

---

### Idea 1 — Direct references (bad)

Add both displays as public fields in `WeatherData` and call them manually inside `setMeasurements()`:

```java
public class WeatherData {
    public StatisticsDisplay stat;
    public CurrentConditionsDisplay current;

    public void setMeasurements(float temp, float humid, float pressure) {
        // ...
        stat.newValues(this);
        current.update(this);   // different method names — no uniform call
    }
}
```

**Problem:** every new observer requires a new field and a new line of code inside the subject.
Adding a new `ForecastDisplay` means modifying `WeatherData` again.

---

### Idea 2 — Common Observer interface

Define a shared interface for all observers:

```java
public interface Observer {
    void update(WeatherData w);
}
```

All display classes implement it:

```java
public class CurrentConditionsDisplay implements Observer {
    @Override
    public void update(WeatherData w) {
        System.out.println("Current conditions: " + w.getTemperature()
            + "°C and " + w.getHumidity() + "% humidity");
    }
}

public class StatisticsDisplay implements Observer {
    private float maxTemp, minTemp = 200, tempSum;
    private int count;

    @Override
    public void update(WeatherData weather) {
        tempSum += weather.getTemperature();
        count++;
        if (weather.getTemperature() > maxTemp) maxTemp = weather.getTemperature();
        if (weather.getTemperature() < minTemp) minTemp = weather.getTemperature();
        System.out.println("Avg/Max/Min Temp = "
            + (tempSum / count) + "/" + maxTemp + "/" + minTemp);
    }
}

public class ForecastDisplay implements Observer {
    private float lastPressure = 29.2f;

    @Override
    public void update(WeatherData weather) {
        float current = weather.getAirPressure();
        System.out.print("Forecast: ");
        if (current > lastPressure)       System.out.println("Improving weather ahead!");
        else if (current == lastPressure) System.out.println("Same as before.");
        else                              System.out.println("Cooler, rainy weather expected.");
        lastPressure = current;   // update AFTER comparison
    }
}
```

`WeatherData` now stores observers as their interface type:

```java
public Observer current;
public Observer stat;
public Observer forecast;

public void setMeasurements(float temp, float humid, float pressure) {
    // ...
    current.update(this);
    stat.update(this);
    forecast.update(this);  // uniform call, but still one line per observer
}
```

**Problem:** adding a new observer still requires changing `WeatherData` and adding another field.

---

### Idea 3 — Observer list (correct approach)

`WeatherData` holds a list of observers and provides register/deregister/notify methods:

```java
public class WeatherData {
    private List<Observer> displayList = new LinkedList<>();

    public void register(Observer o)   { displayList.add(o); }
    public void deregister(Observer o) { displayList.remove(o); }

    protected void notifyObservers() {
        displayList.forEach(o -> o.update(this));
        // 'protected' so subclasses can also call it
    }

    public void setMeasurements(float temp, float humid, float pressure) {
        // update fields...
        this.notifyObservers();
    }
}
```

In `main()`, observers are registered dynamically without touching `WeatherData`:

```java
WeatherData weatherData = new WeatherData();
CurrentConditionsDisplay current = new CurrentConditionsDisplay();
StatisticsDisplay stats = new StatisticsDisplay();
ForecastDisplay forecast = new ForecastDisplay();

weatherData.register(forecast);
weatherData.register(current);
// weatherData.deregister(forecast);  // remove anytime
```

---

### UML — custom Observer pattern

```
+----------------------------+              +----------------------------+
|   WeatherData (Subject)    +----------->  | <<interface>>              |
+----------------------------+           *  |   Observer                 |
|  + stateChanged()          |              +----------------------------+
|  + register(Observer)      |              |  + update(WeatherData)     |
|  + deregister(Observer)    |              +----------+-----------------+
|  # notifyObservers()       |                         △
+----------------------------+              +----------+----------+
                                            |                     |
                                   +--------+-------+    +--------+------+
                                   | Observer1      |    | Observer2     |
                                   +----------------+    +---------------+
```

---

### Key design principle — Loose coupling

> **Strive for loosely coupled designs.**

The subject knows nothing about which specific observers exist or how many there are.
The only contract: they all implement the `Observer` interface and have an `update()` method.

Result: observers can be added, removed, or replaced without changing the subject's code.

---

### Java's built-in Observer infrastructure

Java provides `PropertyChangeSupport` and `PropertyChangeListener` (in `java.beans`) for exactly this purpose — use them instead of writing your own.

```
+----------------------------------------------------------+      +-----------------------------------+
|   PropertyChangeSupport                                  +----> | <<interface>>                     |
+----------------------------------------------------------+   *  | PropertyChangeListener            |
|  + addPropertyChangeListener(PropertyChangeListener)    |      +-----------------------------------+
|  + removePropertyChangeListener(PropertyChangeListener) |      | + propertyChange(PropertyChange-  |
|  + firePropertyChange(String name, Object old, Object new)|    |       Event e) {abstract}         |
|  + PropertyChangeSupport(Object source)                 |      +---------------+-------------------+
+---------------------------+------------------------------+                      △
                            △                                      +--------------+---------------+
                            |                              +--------+----------+  +-------+-------+
          +-----------------+-------------------+          | Listener1         |  | Listener2     |
          |      ConcreteSubject                |          +-------------------+  +---------------+
          +-------------------------------------+          | propertyChange()  |  | propertyChange|
          |  - support: PropertyChangeSupport   |          +-------------------+  +---------------+
          |  + stateChanged()                   |
          |  + addPropertyChangeListener(...)   |
          |  + removePropertyChangeListener(..) |
          +-------------------------------------+
```

#### Subject

```java
public class WeatherData {
    private PropertyChangeSupport support = new PropertyChangeSupport(this);

    public void setMeasurements(double temp, double humid, double pressure) {
        double old = this.temperature;
        this.temperature = temp + sunshine;
        support.firePropertyChange("Temperature", old, this.temperature);
        // firePropertyChange does NOT fire if old == new!

        old = this.humidity;
        this.humidity = humid + rain;
        support.firePropertyChange("Humidity", old, this.humidity);

        old = this.airPressure;
        this.airPressure = pressure;
        support.firePropertyChange("Pressure", old, this.airPressure);
    }

    public void register(PropertyChangeListener l)   { support.addPropertyChangeListener(l); }
    public void deregister(PropertyChangeListener l) { support.removePropertyChangeListener(l); }
}
```

Two ways to fire a change event:

```java
// Option A — pass values directly
support.firePropertyChange("Temperature", oldValue, newValue);

// Option B — build an event object
PropertyChangeEvent evt = new PropertyChangeEvent(this, "Humidity", oldValue, newValue);
support.firePropertyChange(evt);
```

#### Observer

```java
public class CurrentConditionsDisplay implements PropertyChangeListener {

    @Override
    public void propertyChange(PropertyChangeEvent e) {
        if (e.getPropertyName().equals("Temperature")) {
            WeatherData w = (WeatherData) e.getSource();  // cast source to our type
            System.out.println("Current: " + w.getTemperature() + "°C, "
                + w.getHumidity() + "% humidity");
        }
        if (e.getPropertyName().equals("Humidity")) {
            double oldH = (double) e.getOldValue();
            double newH = (double) e.getNewValue();
            System.out.println("Humidity changed from " + oldH + "% to " + newH + "%");
        }
        if (e.getPropertyName().equals("Pressure")) {
            double oldP = (double) e.getOldValue();
            double newP = (double) e.getNewValue();
            System.out.println("Pressure changed from " + oldP + " Pa to " + newP + " Pa");
        }
    }
}
```

#### Main

```java
WeatherData weatherData = new WeatherData();
CurrentConditionsDisplay display = new CurrentConditionsDisplay();
weatherData.register(display);

weatherData.setMeasurements(30, 65, 30.4);
weatherData.setMeasurements(30, 70, 29.2);   // temperature unchanged — no fire for "Temperature"
weatherData.setMeasurements(28, 90, 29.2);
```

---

## MVC Pattern

MVC builds on Observer to structure an entire application into three roles.

| Role | Weather analogy | Responsibilities |
|------|-----------------|-----------------|
| **Model** | `WeatherData` | Holds data + business logic; no user I/O; manages views via Observer |
| **View** | `WeatherDisplay`, `WeatherSurface` | Shows model data; collects user input; passes input to controller; only calls getters on model |
| **Controller** | `WeatherController` | Creates/wires model and view; handles user-triggered events; calls model setters/business methods; no user I/O |

---

### MVC with a GUI — Weather example

#### View

```java
public class WeatherSurface extends JFrame implements PropertyChangeListener {

    private JProgressBar pbHumidity;
    private JLabel lblTemperature;
    private JLabel lblAirPressure;
    private JButton btnSunshineDance;
    private JButton btnRainDance;

    private WeatherController controller;

    public WeatherSurface(WeatherController controller) {
        this.controller = controller;

        // Button wires to controller — lambda delegates to controller
        btnSunshineDance = new JButton("Sunshine Dance");
        btnSunshineDance.addActionListener(arg0 -> controller.sunshine());

        // Or: pass the controller directly if it implements ActionListener
        btnRainDance = new JButton("Rain Dance");
        btnRainDance.setActionCommand("R");
        btnRainDance.addActionListener(controller);
    }

    @Override
    public void propertyChange(PropertyChangeEvent evt) {
        WeatherData w = (WeatherData) evt.getSource();
        this.lblAirPressure.setText(w.getAirPressure() + " hP");
        this.lblTemperature.setText(w.getTemperature() + "°C");
        this.pbHumidity.setValue((int) w.getHumidity());
    }

    public void setTitle(String action) {
        this.setTitle(action);
    }
}
```

#### Controller

```java
public class WeatherController implements ActionListener {

    private WeatherData berlin;
    private WeatherSurface surface;

    public WeatherController(WeatherData data) {
        berlin = data;
        surface = new WeatherSurface(this);   // pass controller to view
        berlin.register(surface);             // register view as observer of model
        surface.setVisible(true);
    }

    // Called from the "Sunshine Dance" button via lambda
    public void sunshine() {
        berlin.sunshineDance();
        surface.setTitle("Sunshine");
    }

    // Called from the "Rain Dance" button (controller passed as ActionListener directly)
    @Override
    public void actionPerformed(ActionEvent e) {
        String cmd = e.getActionCommand();
        if (Objects.equals(cmd, "R")) {
            berlin.rainDance();
            surface.setTitle("Rain");
        }
        if (Objects.equals(cmd, "S")) {
            berlin.sunshineDance();
            surface.setTitle("Sunshine");
        }
    }
}
```

#### Main

```java
public static void main(String[] args) {
    WeatherData weatherData = new WeatherData();
    WeatherController wc = new WeatherController(weatherData);
    new Thread(() -> changeWeather(weatherData)).start();
}
```

The background thread calls `setMeasurements()` → fires property change → view's `propertyChange()` updates labels.
Button clicks go: view → controller → model → `firePropertyChange()` → view updates.

---

### How the three roles are connected

```
        +------------------+     Observer (PropertyChange)    +------------------+
        |      Model       | <------------------------------- |      View        |
        |   WeatherData    |                                  | WeatherSurface   |
        +------------------+                                  +------------------+
               △                                                      |
               |                                           ActionListener (Observer)
               |                                                      |
               +------------------+    +------------------------------>
                                  |    |
                           +------+----+------+
                           |    Controller    |
                           | WeatherController|
                           +------------------+
```

- Model → View: `PropertyChangeSupport` / `PropertyChangeListener`
- View → Controller: `ActionListener` / `actionPerformed()`
- Controller → Model: direct method calls (setters, business methods)
- Controller wires everything in its constructor

---

### Why MVC?

- **Separation of concerns** — small, focused classes; specialists can own each layer
- **Model is reusable** — can be shown by different views without changes
- **View is replaceable** — swap GUI for a console view if all views share an interface
- **Controller logic can be shared** — one action reachable via menu, keyboard shortcut, or context menu

---

## Bank Exercise — Observable Account

Making `Account` (`Konto`) observable with `PropertyChangeSupport`.

```java
public abstract class Account implements Comparable<Account>, Serializable {

    // transient: must not be serialized (PropertyChangeSupport is not Serializable)
    private transient PropertyChangeSupport prop = new PropertyChangeSupport(this);

    // Called whenever the balance changes — centralises notifications
    protected void setBalance(double balance) {
        double old = this.balance;
        this.balance = balance;
        prop.firePropertyChange("Balance", old, balance);
    }

    public final void setOwner(Customer owner) throws LockedException {
        if (owner == null)
            throw new IllegalArgumentException("Owner must not be null!");
        if (this.locked)
            throw new LockedException(this.accountNumber);

        Customer old = this.owner;
        this.owner = owner;
        prop.firePropertyChange("Owner", old, owner);
    }

    public final void changeCurrency(Currency newCurrency) {
        balance = Currency.convert(balance, this.currency, newCurrency);
        specificConversion(newCurrency);

        Currency old = this.currency;
        this.currency = newCurrency;
        prop.firePropertyChange("Currency", old, newCurrency);
    }

    // Helper so subclasses can also fire events through 'prop' (which is private)
    protected void firePropertyChange(String name, Object oldVal, Object newVal) {
        prop.firePropertyChange(name, oldVal, newVal);
    }

    public void register(PropertyChangeListener l)   { prop.addPropertyChangeListener(l); }
    public void deregister(PropertyChangeListener l) { prop.removePropertyChangeListener(l); }
}
```

Subclass `CheckingAccount` fires for its own `overdraftLimit` field:

```java
public class CheckingAccount extends Account implements Transferable {

    public void setOverdraftLimit(double limit) {
        if (limit < 0 || Double.isNaN(limit))
            throw new IllegalArgumentException("Overdraft limit is not valid!");

        // Fire BEFORE assignment so oldValue is still the current value
        this.firePropertyChange("OverdraftLimit", this.overdraftLimit, limit);
        this.overdraftLimit = limit;
    }
}
```

### Observer class

```java
public class AccountObserver implements PropertyChangeListener {

    @Override
    public void propertyChange(PropertyChangeEvent evt) {
        Account a = (Account) evt.getSource();
        String changed = evt.getPropertyName();
        Object oldVal = evt.getOldValue();
        Object newVal = evt.getNewValue();

        System.out.println("Change on account " + a.getAccountNumber()
            + ": " + changed + " changed from " + oldVal + " to " + newVal);
    }
}
```

### Testing with Mockito

```java
class CheckingAccountTest {

    CheckingAccount ca;
    Customer c;
    long number = 17;
    double overdraft = 1000;

    // Option A: mock the interface
    PropertyChangeListener observer = Mockito.mock(PropertyChangeListener.class);

    // Option B: mock the concrete class
    // AccountObserver observer = Mockito.mock(AccountObserver.class);

    @BeforeEach
    void setUp() {
        // Tell the mock to do nothing when propertyChange() is called
        Mockito.doNothing().when(observer).propertyChange(ArgumentMatchers.any());
        c = new Customer("Alice", "Smith", "home", LocalDate.parse("1976-07-13"));
    }

    @Test
    void depositTest() {
        ca = new CheckingAccount();
        ca.register(observer);       // attach observer

        ca.deposit(100);             // trigger a state change

        // ... normal assertions ...

        // Verify that propertyChange() was called with the right arguments
        Mockito.verify(observer).propertyChange(
            ArgumentMatchers.argThat(e ->
                e.getPropertyName().equals("Balance")
                && e.getNewValue().equals(ca.getBalance())
            )
        );
    }
}
```

Steps:
1. Create the account object
2. Register the (mocked) observer
3. Trigger a state-changing action
4. Verify that `propertyChange()` was called with the expected `PropertyChangeEvent`
