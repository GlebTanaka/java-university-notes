# Chapter 8 — Threads & Concurrency

## Topics

- Creating and starting threads
- Critical sections
- Synchronisation (`synchronized`, `wait`/`notify`)
- Stopping threads
- Inner and anonymous classes

---

## Motivation — The Bookstore Example

Two independent processes: a `Bookstore` that restocks books, and a `Buyer` that purchases them.

```java
public class Bookstore {
    private int bookCount = 0;
    public boolean closed = false;

    public void restock(int total) {
        int i = 0;
        while (i < total) {
            int count = this.getBookCount();
            count = count + 1;
            this.setBookCount(count);
            System.out.println("On shelf: " + count);
            i++;
        }
        System.out.println("Locking up");
    }
    // getters and setters ...
}

public class Buyer {
    Bookstore store;

    public Buyer(Bookstore store) { this.store = store; }

    public void buy() {
        int i = 0;
        while (i < 1000) {
            int count = store.getBookCount();
            count = count - 1;
            store.setBookCount(count);
            System.out.println("Buyer: " + count);
            i++;
        }
        System.out.println("Buyer leaves the store");
    }
}
```

Running them sequentially:

```java
Bookstore shop = new Bookstore();
Buyer buyer = new Buyer(shop);
shop.restock(1000);
buyer.buy();         // runs AFTER restock finishes — NOT parallel
```

---

## Step 1 — Making Code Runnable

Code that should run in parallel must live inside a `run()` method, in a class that either implements `Runnable` or extends `Thread`.

```java
// Option A — implement Runnable (preferred)
public class MyRunnable implements Runnable {
    public void run() { ... }
}

// Option B — extend Thread
public class MyThread extends Thread {
    @Override
    public void run() { ... }
}
```

Adding `Runnable` to `Buyer`:

```java
public class Buyer implements Runnable {
    // ... same fields and buy() method ...

    @Override
    public void run() {
        buy();
    }
}
```

Simply calling `buyer.run()` in `main` still runs sequentially — it is just a normal method call.

---

## Step 2 — Starting a Thread

A **`Runnable`** carries the code. A **`Thread`** is the execution context. Call `start()` to launch a new execution thread:

```java
Runnable r = new MyRunnable();
Thread t = new Thread(r);
t.start();
// code here runs concurrently with r.run()
```

After `t.start()`, the JVM switches back and forth between the main thread and the new thread.

### Bookstore example — correct start

```java
Thread buyerThread = new Thread(buyer);
buyerThread.start();
shop.restock(1000);   // ← code AFTER start() runs concurrently
```

Everything before `start()` runs on the main thread. Parallelism begins at `start()`.

### Passing parameters to `run()`

`run()` takes no parameters and returns nothing. Store inputs in the `Runnable`'s fields before calling `start()`:

```java
public class Buyer implements Runnable {
    private int total;

    public void setTotal(int total) { this.total = total; }

    @Override
    public void run() { buy(total); }
}

// In main:
buyer.setTotal(500);
Thread t = new Thread(buyer);
t.start();
```

Return values work the same way: `run()` writes results into a field that can be read after the thread finishes.

---

## Race Conditions

Introducing `Thread.sleep(10)` between the read and the write makes the problem visible:

```java
int count = store.getBookCount();   // read: 7
count = count + 1;
Thread.sleep(10);                   // another thread runs here
store.setBookCount(count);          // write: 8, but buyer already set it to 6 !
```

**What goes wrong:**

| Main thread (Bookstore) | Buyer thread |
|------------------------|--------------|
| reads count = 7 | |
| *(interrupted)* | reads count = 7 |
| | count = 7 - 1 = 6 |
| | writes count = 6 |
| count = 7 + 1 = 8 | *(interrupted)* |
| writes count = 8 | |

The buyer's write was overwritten — one book appeared from nowhere. The same problem can happen in reverse.

**Root cause:** shared data is read **and** written, and the threads can be interrupted between those two operations.

---

## Critical Sections

Lines of code that must not be interrupted are called **critical sections**. Use `synchronized` to guard them:

```java
synchronized(lockObject) {
    // only one thread at a time can be here for this lock object
}
```

### Bookstore example

Both threads must use the **same** lock object. `this` inside `Bookstore` and the `store` reference inside `Buyer` refer to the same object:

```java
// In Buyer:
int count;
synchronized (store) {
    count = store.getBookCount();
    count = count - 1;
    store.setBookCount(count);
}
System.out.println("Buyer: " + count);   // outside the block

// In Bookstore:
int count;
synchronized (this) {
    count = this.getBookCount();
    count = count + 1;
    this.setBookCount(count);
}
System.out.println("On shelf: " + count);
```

The `println` is intentionally outside the synchronized block — keep blocks as short as possible.

### Synchronized methods

As a shortcut, `synchronized` can annotate an entire method:

```java
// Instance method — locks this
public synchronized void method() { ... }

// Static method — locks the Class object
public static synchronized void method() { ... }
```

---

## `wait()` and `notify()` — Thread Communication

**Active waiting (bad):**

```java
while (conditionNotMet) {
    Thread.sleep(1000);
}
```

This wastes CPU and, inside a `synchronized` block, blocks other threads forever.

**Better — use `wait()` and `notifyAll()`:**

```java
// Buyer — waiting for books to arrive:
synchronized (store) {
    count = store.getBookCount();
    while (count <= 0) {
        try {
            store.wait();            // releases the lock and suspends this thread
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        count = store.getBookCount(); // re-read after waking up!
    }
    count = count - 1;
    // ...
}

// Bookstore — notifying after restocking:
this.setBookCount(count);
this.notifyAll();                    // wake up all threads waiting on this object
```

### How it works

1. `wait()` atomically releases the `synchronized` lock and suspends the current thread.
2. When the bookstore calls `notifyAll()`, suspended threads are woken up.
3. They re-acquire the lock and continue from after the `wait()` call.
4. Always re-check the condition in a `while` loop (not `if`) after waking up.

### Rules

| Rule | Detail |
|------|--------|
| `wait()` / `notify()` must be inside a `synchronized` block | Otherwise: `IllegalMonitorStateException` |
| `wait()` always suspends the **current** thread | You cannot put another thread to sleep from outside |
| `notifyAll()` wakes all waiters; `notify()` wakes one | Prefer `notifyAll()` unless you know exactly which thread to wake |
| `wait(ms)` variant | Waits at most `ms` milliseconds |

Both methods are defined in `Object` and are available everywhere.

**Common mistake — `wait()` on the wrong object:**

```java
// WRONG — does not suspend thread t
Thread t = new Thread(...);
t.start();
t.wait();   // suspends the CURRENT thread, not t

// RIGHT — suspend the current thread inside its own run():
public void run() {
    someObject.wait();
    // continues after notifyAll() on someObject
}
```

---

## Stopping a Thread

### Method 1 — `stop()` (avoid)

```java
threadVariable.stop();      // deprecated, dangerous
// also: suspend() and resume() — do not use
```

The thread cannot clean up (close connections, etc.) and may leave shared state inconsistent.

### Method 2 — Custom flag (cooperative)

```java
// From outside:
runnableObject.shouldStop = true;

// Inside run():
if (this.shouldStop) {
    // clean up
    return;
}
```

Works, but the thread won't notice while it's sleeping or waiting.

### Method 3 — `interrupt()` (preferred)

```java
threadVariable.interrupt();
```

Two effects:
- If the thread is **running**: sets an internal interrupt flag (check with `Thread.interrupted()`).
- If the thread is **sleeping or waiting**: throws `InterruptedException` immediately.

```java
// Inside run():
try {
    if (Thread.interrupted()) {
        // clean up
        return;
    }
    // ...
    Thread.sleep(5000);
} catch (InterruptedException e) {
    // clean up
    return;
}
```

`interrupt()` triggers; `Thread.interrupted()` tests (and clears) the flag.

### Method 4 — `System.exit()`

Terminates all threads immediately. Use only as a last resort.

---

## Bookstore — Complete Shutdown Example

A `ClosingTime` thread waits a while, then closes the store:

```java
public class ClosingTime implements Runnable {
    private Bookstore store;

    public ClosingTime(Bookstore store) { this.store = store; }

    @Override
    public void run() {
        try { Thread.sleep(200); } catch (InterruptedException e) {}
        store.closed = true;
    }
}
```

The `Bookstore.restock()` checks the flag at each iteration:

```java
while (i < total && !this.closed) { ... }
```

For the `Buyer`, use a second closing thread with `interrupt()` so it is woken even from `wait()`:

```java
public class ClosingTime2 implements Runnable {
    private Thread thread;

    public ClosingTime2(Thread thread) { this.thread = thread; }

    @Override
    public void run() {
        try { Thread.sleep(200); } catch (InterruptedException e) {}
        thread.interrupt();
    }
}

// In Buyer.buy():
while (i < total && !Thread.interrupted()) {
    // ...
    while (count <= 0) {
        try {
            store.wait();
        } catch (InterruptedException e) {
            return;   // interrupted while waiting — exit cleanly
        }
    }
    // ...
    try {
        Thread.sleep(1);
    } catch (InterruptedException e) {
        break;        // interrupted while sleeping — exit loop
    }
}
```

**Main:**

```java
Bookstore shop = new Bookstore();
Buyer buyer = new Buyer(shop);
buyer.setTotal(500);

Thread buyerThread = new Thread(buyer);
buyerThread.start();

Thread closeShop = new Thread(new ClosingTime(shop));
closeShop.start();

Thread closeBuyer = new Thread(new ClosingTime2(buyerThread));
closeBuyer.start();

shop.restock(1000);
```

---

## Thread Utility Methods

| Method | Effect |
|--------|--------|
| `Thread.sleep(ms)` | Suspends the current thread for `ms` milliseconds |
| `Thread.yield()` | Gives up the CPU but is immediately ready again |
| `threadVar.join()` | Blocks the calling thread until `threadVar` finishes |

---

## Thread-Safe Collections

Standard collections are not thread-safe. Two options:

```java
// 1. Wrap with Collections helper methods:
List<String> safeList = Collections.synchronizedList(new ArrayList<>());
Set<String>  safeSet  = Collections.synchronizedSet(new HashSet<>());
Map<K, V>    safeMap  = Collections.synchronizedMap(new HashMap<>());

// 2. Use ConcurrentHashMap (more efficient for maps):
Map<String, Choice> chosen = new ConcurrentHashMap<>();
```

---

## Inner Classes

### Member (non-static) inner class

```java
public class Outer {
    private class Inner implements Runnable {
        public void run() {
            // has direct access to all of Outer's members
        }
    }
}

// Create an instance:
outer.new Inner()
```

- All four visibility modifiers are allowed.
- Static members inside the inner class are forbidden.
- Each `Outer` object gets its own `Inner` class.
- Full qualified name: `Outer.Inner`.

### Static inner class

```java
public class Outer {
    private static class Inner implements Runnable {
        public void run() { ... }
    }
}

// Create an instance (no outer object needed):
new Outer.Inner()
```

- Only one class shared across all `Outer` objects.
- Cannot access non-static members of `Outer`.

### `this` inside inner classes

```java
// Inside Inner — refers to the Inner object
this

// Refers to the enclosing Outer object
Outer.this
```

---

## Exercise — Rock, Paper, Scissors

**Design:**

- Outer class `RockPaperScissors` — holds the shared `chosen` map and `determineWinner()`.
- Inner enum `Choice` — `ROCK`, `PAPER`, `SCISSORS`.
- Inner class `SignalGiver` — waits, then calls `notifyAll()`.
- Inner class `Player` — waits for the signal, then picks a random choice.

```java
public class RockPaperScissors {

    private Map<String, Choice> chosen = new ConcurrentHashMap<>();

    enum Choice { SCISSORS, ROCK, PAPER; }  // ordinal order matters for compareTo

    private class SignalGiver implements Runnable {
        @Override
        public void run() {
            try { Thread.sleep(500); } catch (InterruptedException e) {}
            synchronized (RockPaperScissors.this) {
                RockPaperScissors.this.notifyAll();
            }
        }
    }

    private class Player implements Runnable {
        private String name;

        public Player(String name) { this.name = name; }

        public Choice pick() {
            Random r = new Random();
            return Choice.values()[r.nextInt(3)];
        }

        @Override
        public void run() {
            try {
                synchronized (RockPaperScissors.this) {
                    RockPaperScissors.this.wait();
                }
            } catch (InterruptedException e) {}
            chosen.put(name, pick());
        }
    }

    public String determineWinner() {
        String name = "";
        Choice best = Choice.SCISSORS;
        for (Map.Entry<String, Choice> entry : chosen.entrySet()) {
            if (entry.getValue().compareTo(best) >= 0) {
                name = entry.getKey();
                best = entry.getValue();
            }
        }
        return name + " " + best;
    }

    public RockPaperScissors() {
        Player alice = new Player("Alice");
        Player bob   = new Player("Bob");
        Thread t1 = new Thread(alice);
        Thread t2 = new Thread(bob);
        t1.start();
        t2.start();

        SignalGiver signal = new SignalGiver();
        Thread s = new Thread(signal);
        s.start();

        try {
            t1.join();   // wait for both players to finish
            t2.join();
        } catch (InterruptedException e) {}

        System.out.println(determineWinner());
    }

    public static void main(String[] args) {
        new RockPaperScissors();
    }
}
```

**Key points:**
- Both `SignalGiver` and `Player` use `RockPaperScissors.this` as the lock — it's the same object, so `notify`/`wait` communicate correctly.
- `ConcurrentHashMap` is used because two `Player` threads write to `chosen` concurrently.
- `join()` ensures `determineWinner()` is not called before both players have made their choice.
- Inner classes access `chosen` directly — no getters needed, which is the main advantage over separate top-level classes.

---

## Exercise — Bouncing Ball

**Structure:**
- `Ball implements Runnable` — encapsulates its own thread; can be paused, resumed, and shut down.
- `BallGame` — manages a list of active balls.
- `BallFrame extends JFrame` — GUI with four buttons wired to `BallGame`.

### `Ball` class

```java
public class Ball implements Runnable {

    private JPanel box;
    private Object lock = new Object();   // dedicated lock object for wait/notify
    private int duration;
    private Boolean isRunning = false;
    private boolean shutdown = false;

    public Ball(JPanel panel, int duration) {
        this.box = panel;
        this.duration = duration;
        new Thread(this).start();         // Ball starts its own thread
    }

    public void bounce(int steps) {
        this.draw();
        for (int i = 1; i <= steps; i++) {
            if (shutdown) break;          // exit loop if shutdown requested

            synchronized (lock) {
                while (!isRunning) {
                    try {
                        lock.wait();      // pause here until resume() calls notifyAll()
                    } catch (InterruptedException e) {}
                }
            }

            this.move();
            try { Thread.sleep(5); } catch (InterruptedException e) {}
        }
        this.erase();
    }

    public void pause() {
        isRunning = false;                // next loop iteration enters wait()
    }

    public void resume() {
        isRunning = true;
        synchronized (lock) {
            lock.notifyAll();             // wake the thread out of wait()
        }
    }

    public void shutdown() {
        resume();                         // wake from wait() first
        shutdown = true;                  // then let the loop exit
    }

    @Override
    public void run() {
        isRunning = true;
        bounce(duration);
    }
}
```

### `BallGame` class

```java
public class BallGame {
    private BallFrame frame;
    private List<Ball> balls = new LinkedList<>();

    public BallGame(BallFrame frame) { this.frame = frame; }

    public void startBall() {
        int duration = new Random().nextInt(500) + 1000;
        Ball b = new Ball(frame.getCanvas(), duration);
        balls.add(b);
    }

    public void stopBalls()   { for (Ball b : balls) b.pause(); }
    public void resumeBalls() { for (Ball b : balls) b.resume(); }

    public void removeAll() {
        for (Ball b : balls) b.shutdown();
        balls.clear();
    }
}
```

### `BallFrame` — anonymous class example

The clock display uses an **anonymous class** (an inline implementation of `Runnable` without a named class):

```java
private void addClockThread(JTextField field) {
    SimpleDateFormat fmt = new SimpleDateFormat("HH:mm:ss");

    Runnable r = new Runnable() {
        @Override
        public void run() {
            while (true) {
                field.setText(fmt.format(new Date()));
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            }
        }
    };

    new Thread(r).start();
}
```

The four buttons call `BallGame` methods via anonymous `ActionListener` implementations:

```java
addButton(panel, "Start Ball",  e -> game.startBall());
addButton(panel, "Pause",       e -> game.stopBalls());
addButton(panel, "Resume",      e -> game.resumeBalls());
addButton(panel, "Clear All",   e -> game.removeAll());
```

---

## `while (true)` Loops

An intentional infinite loop is sometimes used when a thread should keep running until an external condition is met:

```java
while (true) {
    // try to satisfy some condition
    if (satisfied) break;
}
```

This is fine as long as the break condition is reachable and the loop does not spin-wait without sleeping.
