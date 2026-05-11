# Post #29: Enums - More Than Just Constants

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 13, 2026  
**Topic:** Java Enums, Advanced Enum Features, Best Practices

---

## The Problem

Enums aren't glorified integers. They're full-featured classes in disguise.

## Code Example

### ❌ Old Way - Magic Numbers Everywhere

```java
public class OrderStatus {
    public static final int PENDING = 0;
    public static final int CONFIRMED = 1;
    public static final int SHIPPED = 2;
    public static final int DELIVERED = 3;
}

public class Order {
    private int status;
    
    public void setStatus(int status) {
        this.status = status;  // Any Int Allowed!
    }
}

// Problems
order.setStatus(999);  // Compiles! Invalid Status
order.setStatus(OrderStatus.SHIPPED);  // What Does 2 Mean Again?
```

### ✅ Basic Enum - Type Safety Built-In

```java
public enum OrderStatus {
    PENDING,
    CONFIRMED,
    SHIPPED,
    DELIVERED
}

public class Order {
    private OrderStatus status;
    
    public void setStatus(OrderStatus status) {
        this.status = status;  // Only Valid Statuses Allowed!
    }
}

// Type Safe
order.setStatus(OrderStatus.SHIPPED);  // Clear and Explicit
order.setStatus(999);  // Compile Error!
```

### ✅ Enums with Fields - Store Additional Data

```java
public enum ShippingMethod {
    STANDARD(5.99, 7),
    EXPRESS(12.99, 3),
    OVERNIGHT(24.99, 1);
    
    private final double cost;
    private final int deliveryDays;
    
    // Constructor Must Be Private
    ShippingMethod(double cost, int deliveryDays) {
        this.cost = cost;
        this.deliveryDays = deliveryDays;
    }
    
    public double getCost() {
        return cost;
    }
    
    public int getDeliveryDays() {
        return deliveryDays;
    }
}

// Usage
ShippingMethod method = ShippingMethod.EXPRESS;
System.out.println("Cost: $" + method.getCost());  // $12.99
System.out.println("Delivery: " + method.getDeliveryDays() + " days");  // 3 days
```

### ✅ Enums with Methods - Business Logic Inside

```java
public enum PaymentStatus {
    PENDING {
        @Override
        public boolean canProcess() {
            return true;
        }
    },
    PROCESSING {
        @Override
        public boolean canProcess() {
            return false;
        }
    },
    COMPLETED {
        @Override
        public boolean canProcess() {
            return false;
        }
    },
    FAILED {
        @Override
        public boolean canProcess() {
            return true;  // Can Retry Failed Payments
        }
    };
    
    // Abstract Method - Each Constant Must Implement
    public abstract boolean canProcess();
}

// Usage
if (payment.getStatus().canProcess()) {
    paymentProcessor.process(payment);
}
```

### ✅ Enums with Common Methods

```java
public enum Operation {
    ADD {
        @Override
        public double apply(double x, double y) {
            return x + y;
        }
    },
    SUBTRACT {
        @Override
        public double apply(double x, double y) {
            return x - y;
        }
    },
    MULTIPLY {
        @Override
        public double apply(double x, double y) {
            return x * y;
        }
    },
    DIVIDE {
        @Override
        public double apply(double x, double y) {
            if (y == 0) throw new ArithmeticException("Division by zero");
            return x / y;
        }
    };
    
    public abstract double apply(double x, double y);
}

// Usage
double result = Operation.ADD.apply(5, 3);  // 8.0
double product = Operation.MULTIPLY.apply(4, 7);  // 28.0
```

### ✅ Switch with Enums - Compile-Time Exhaustiveness

```java
public class OrderProcessor {
    
    public void processOrder(Order order) {
        switch (order.getStatus()) {
            case PENDING -> validateOrder(order);
            case CONFIRMED -> reserveInventory(order);
            case SHIPPED -> notifyCustomer(order);
            case DELIVERED -> closeOrder(order);
            // Compiler Warns If Any Status Is Missing!
        }
    }
}
```

### ✅ Built-In Enum Methods

```java
// values() - Get All Constants
for (OrderStatus status : OrderStatus.values()) {
    System.out.println(status);
}
// Output: PENDING, CONFIRMED, SHIPPED, DELIVERED

// valueOf() - String to Enum
OrderStatus status = OrderStatus.valueOf("SHIPPED");

// name() - Enum to String
String name = OrderStatus.SHIPPED.name();  // "SHIPPED"

// ordinal() - Position in Declaration (Avoid Using!)
int position = OrderStatus.SHIPPED.ordinal();  // 2
```

### ❌ Never Use ordinal() for Logic

```java
// Wrong! Fragile and Breaks on Reordering
public enum Priority {
    LOW, MEDIUM, HIGH, CRITICAL
}

// Bad Code
if (priority.ordinal() > 2) {
    escalate();  // Breaks If Enum Order Changes!
}

// ✅ Right! Use Fields Instead
public enum Priority {
    LOW(1), MEDIUM(2), HIGH(3), CRITICAL(4);
    
    private final int level;
    
    Priority(int level) {
        this.level = level;
    }
    
    public int getLevel() {
        return level;
    }
}

// Good Code
if (priority.getLevel() > 2) {
    escalate();  // Safe and Explicit
}
```

### ✅ EnumSet - Efficient Enum Collections

```java
public enum Permission {
    READ, WRITE, EXECUTE, DELETE
}

// EnumSet - Optimized for Enums (Bit Vector Implementation)
EnumSet<Permission> adminPermissions = EnumSet.of(
    Permission.READ,
    Permission.WRITE,
    Permission.DELETE
);

EnumSet<Permission> allPermissions = EnumSet.allOf(Permission.class);
EnumSet<Permission> readOnly = EnumSet.of(Permission.READ);

// Fast Operations
if (adminPermissions.contains(Permission.WRITE)) {
    allowModification();
}
```

### ✅ EnumMap - Type-Safe Enum-Keyed Maps

```java
public enum ErrorCode {
    NOT_FOUND, UNAUTHORIZED, SERVER_ERROR
}

// EnumMap - Faster and More Memory Efficient Than HashMap
EnumMap<ErrorCode, String> errorMessages = new EnumMap<>(ErrorCode.class);
errorMessages.put(ErrorCode.NOT_FOUND, "Resource not found");
errorMessages.put(ErrorCode.UNAUTHORIZED, "Access denied");
errorMessages.put(ErrorCode.SERVER_ERROR, "Internal server error");

String message = errorMessages.get(ErrorCode.NOT_FOUND);
```

### ✅ Implementing Interfaces - Polymorphic Enums

```java
public interface Discountable {
    double applyDiscount(double price);
}

public enum MembershipLevel implements Discountable {
    BRONZE {
        @Override
        public double applyDiscount(double price) {
            return price * 0.95;  // 5% Discount
        }
    },
    SILVER {
        @Override
        public double applyDiscount(double price) {
            return price * 0.90;  // 10% Discount
        }
    },
    GOLD {
        @Override
        public double applyDiscount(double price) {
            return price * 0.80;  // 20% Discount
        }
    };
}

// Usage
MembershipLevel level = user.getMembershipLevel();
double finalPrice = level.applyDiscount(100.00);
```

### ✅ Enum Singleton Pattern - Thread-Safe Laziness

```java
public enum DatabaseConnection {
    INSTANCE;
    
    private Connection connection;
    
    DatabaseConnection() {
        // Initialization Guaranteed Thread-Safe by JVM
        this.connection = createConnection();
    }
    
    public Connection getConnection() {
        return connection;
    }
    
    private Connection createConnection() {
        // Heavy Initialization Logic
        return DriverManager.getConnection("jdbc:mysql://...");
    }
}

// Usage - Thread-Safe Singleton
Connection conn = DatabaseConnection.INSTANCE.getConnection();
```

### ✅ Real-World Example - HTTP Status Codes

```java
public enum HttpStatus {
    OK(200, "OK"),
    CREATED(201, "Created"),
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    NOT_FOUND(404, "Not Found"),
    SERVER_ERROR(500, "Internal Server Error");
    
    private final int code;
    private final String message;
    
    HttpStatus(int code, String message) {
        this.code = code;
        this.message = message;
    }
    
    public int getCode() {
        return code;
    }
    
    public String getMessage() {
        return message;
    }
    
    public boolean isSuccess() {
        return code >= 200 && code < 300;
    }
    
    public boolean isClientError() {
        return code >= 400 && code < 500;
    }
    
    public boolean isServerError() {
        return code >= 500 && code < 600;
    }
    
    // Find by Code
    public static HttpStatus fromCode(int code) {
        for (HttpStatus status : values()) {
            if (status.code == code) {
                return status;
            }
        }
        throw new IllegalArgumentException("Invalid status code: " + code);
    }
}

// Usage
HttpStatus status = HttpStatus.NOT_FOUND;
if (status.isClientError()) {
    logClientError(status.getCode(), status.getMessage());
}
```

## 💡 Why This Matters

Enums are type-safe alternatives to constants - compiler prevents invalid values. Enum constructors are always private - you cannot instantiate enums with new. Each enum constant is a singleton instance created at class loading. Enums can have fields, methods, and constructors just like regular classes. Use EnumSet and EnumMap for better performance than regular collections with enums. Avoid ordinal() for business logic - it breaks when enum order changes. Enums implement Comparable and Serializable automatically. Switch statements on enums provide compile-time exhaustiveness checking.

## 🎯 Key Takeaway

Use enums for fixed sets of related constants. Add fields and methods to enums when constants need behavior or data. EnumSet and EnumMap are faster than HashSet and HashMap for enum types. Never use ordinal() in production code - use explicit fields instead. Enum singleton pattern is the safest singleton implementation in Java.

---

**Tags:** `#Java` `#JavaWisdom` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#Enums` `#TypeSafety` `#BestPractices` `#DesignPatterns`
