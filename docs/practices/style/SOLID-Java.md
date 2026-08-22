# SOLID Principle in Java

## Single Responsibility (SRP)

> A class should have only one reason to change, which corresponds to one role

Bad example
```java
public class Invoice {
    public void calculateTotal() { ... } // business logic
    public void saveToDatabase() { ... } // persistence logic
    public void printInvoice() { ... }   // presentation logic
}
```

Good Example
```java
class Invoice {
    public BigDecimal calculateTotal() { ... }
}

class InvoiceRepository {
    public void save(Invoice invoice) { ... }
}

class InvoicePrinter {
    public void print(Invoice invoice) { ... }
}
```

---

## Open/Closed Principle

> You should be able to add new functionality by adding code, not by modifying existing classes

```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    public void send(String message) {
        // send email
    }
}

public class SMSNotification implements Notification {
    public void send(String message) {
        // send SMS
    }
}

public class NotificationService {
    private Notification notification;

    public NotificationService(Notification notification) {
        this.notification = notification;
    }

    public void notifyUser(String message) {
        notification.send(message);
    }
}
```
You can add new notification types without modifying NotificationService.

---

## Liskov Substitution Principle (LSP)

> A subclass should be usable anywhere its superclass is expected, without causing bugs or incorrect behavior.

Examle: `Rectangle` should not inherit `Square`, they should both inherit `Shape`. Because their area are different given same width and height.
```java
// Superclass: A generic document
class Document {
    protected String data;
    protected boolean isSaved;

    public Document(String initialData) {
        this.data = initialData;
        this.isSaved = false; // Initially not saved
        System.out.println("Document created with: \"" + initialData + "\"");
    }

    // The contract: allows updating data and then saving it.
    // A client expects that after calling save(), isSaved will be true.
    public void updateData(String newData) {
        this.data = newData;
        this.isSaved = false; // Data changed, needs saving again
        System.out.println("Document data updated to: \"" + this.data + "\"");
    }

    public void save() {
        // Simulate saving the document
        this.isSaved = true;
        System.out.println("Document with data \"" + this.data + "\" has been saved.");
    }

    public boolean isSaved() {
        return isSaved;
    }

    public String getData() {
        return data;
    }
}

// Subclass: A document that is read-only after initial creation.
// It IS-A Document, but its behavior for 'save' or 'updateData' might be problematic.
class ReadOnlyDocument extends Document {
    public ReadOnlyDocument(String initialData) {
        super(initialData);
        // Mark as "saved" immediately as it's read-only from this point
        super.save(); // Initial "save" to establish read-only state
        System.out.println("ReadOnlyDocument established. No further modifications allowed.");
    }

    @Override
    public void updateData(String newData) {
        // Violates LSP: Cannot update data in a read-only document.
        // Superclass allows updateData, subclass restricts it.
        System.err.println("Cannot update data on a ReadOnlyDocument. Operation ignored.");
        // Silently ignoring is one way to violate, throwing an exception is another.
        // The key is that the superclass contract (data can be updated) is broken.
    }

    @Override
    public void save() {
        // Violates LSP: A ReadOnlyDocument might not need a 'save' operation
        // or its 'save' might mean something different or do nothing.
        // If the superclass 'save' implies a change can be persisted,
        // this 'save' changes that contract.
        if (!this.isSaved) { // Should ideally always be true after constructor
             System.out.println("ReadOnlyDocument: Re-affirming it's already in its final saved state.");
             this.isSaved = true; // Ensure it stays saved
        } else {
            System.out.println("ReadOnlyDocument is already in its final saved state. 'Save' operation has no effect.");
        }
        // It doesn't allow the typical "update then save" cycle.
    }
}
```

---

## Interface Segregation Principle (ISP)

> No client should be forced to depend on methods it does not use

```java
// BAD: too many responsibilities
interface Worker {
    void work();
    void eat();
}

// GOOD: split interfaces
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

class HumanWorker implements Workable, Eatable {
    public void work() { ... }
    public void eat() { ... }
}

class RobotWorker implements Workable {
    public void work() { ... }
}
```

---

# Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

Bad exmaple
```java
public class MySQLDatabase {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

public class ReportService {
    private MySQLDatabase db = new MySQLDatabase(); // directly depends on concrete class
}
```

Good example
```java
public interface Database {
    void save(String data);
}

public class MySQLDatabase implements Database {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

public class InMemoryDatabase implements Database {
    public void save(String data) {
        System.out.println("Pretend saving to memory: " + data);
    }
}

public class ReportService {
    private final Database db;

    public ReportService(Database db) {
        this.db = db;
    }
}
```