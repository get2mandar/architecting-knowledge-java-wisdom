# Post #15: Pattern Matching - Switch on Steroids

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 25, 2026  
**Topic:** Pattern Matching, Switch Expressions, Modern Java

---

## The Problem

Your switch statement is from 2004. Here's what you're missing.

## Code Example

### ❌ Old School - Verbose and Error-Prone
```java
public String processShape(Object obj) {
    String result;
    if (obj instanceof Circle) {
        Circle c = (Circle) obj;
        result = "Circle with area " + c.area();
    } else if (obj instanceof Rectangle) {
        Rectangle r = (Rectangle) obj;
        result = "Rectangle with area " + r.area();
    } else if (obj == null) {
        result = "Null shape";
    } else {
        result = "Unknown shape";
    }
    return result;
}
```

### ✅ Pattern Matching Switch - Clean and Safe
```java
public String processShape(Object obj) {
    return switch (obj) {
        case null -> "Null shape";
        case Circle c -> "Circle with area " + c.area();
        case Rectangle r -> "Rectangle with area " + r.area();
        default -> "Unknown shape";
    };
}
```

### ✅ Record Patterns - Destructure in Place
```java
record Point(int x, int y) {}
record Line(Point start, Point end) {}

public void processLine(Object obj) {
    switch (obj) {
        case Line(Point(var x1, var y1), Point(var x2, var y2)) ->
            System.out.println("Line from (" + x1 + "," + y1 + ") to (" + x2 + "," + y2 + ")");
        default -> System.out.println("Not a line");
    }
}
```

### ✅ Guarded Patterns - Add Conditions
```java
public String categorizeNumber(Object obj) {
    return switch (obj) {
        case Integer i when i < 0 -> "Negative integer";
        case Integer i when i > 0 -> "Positive integer";
        case Integer i -> "Zero";
        case Double d when d < 0.0 -> "Negative double";
        case Double d -> "Non-negative double";
        default -> "Not a number";
    };
}
```

### ✅ Exhaustive Sealed Types - No Default Needed!
```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double radius) implements Shape { }
record Rectangle(double width, double height) implements Shape { }
record Triangle(double base, double height) implements Shape { }

public double calculateArea(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t -> 0.5 * t.base() * t.height();
        // No Default! Compiler knows these are All Possibilities
    };
}
```

## 💡 Why This Matters

Pattern matching for switch (Java 21) eliminates the instanceof-cast-assign ceremony. Type patterns handle casting automatically. Record patterns let you destructure nested data structures inline. Guarded patterns add conditional logic with when clauses. With sealed types, the compiler verifies exhaustiveness - add a new shape, and every switch breaks until you handle it. Runtime safety becomes compile-time safety.

## 🎯 Key Takeaway

Stop writing instanceof chains and manual casts. Use pattern matching switch with type patterns, record destructuring, and guards. Combine with sealed types for compiler-verified exhaustiveness. Make illegal states unrepresentable.

---

**Tags:** `#Java` `#JavaWisdom` `#PatternMatching` `#ModernJava` `#Java21` `#TypeSafety`
