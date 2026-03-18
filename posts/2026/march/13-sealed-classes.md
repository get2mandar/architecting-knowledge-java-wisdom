# Post #13: Sealed Classes - Controlling Your Hierarchy

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 18, 2026  
**Topic:** Sealed Classes, Pattern Matching, Type Safety

---

## The Problem

Open inheritance is technical debt. Here's how to control it.

## Code Example

### ❌ Open Inheritance - Anyone Can Extend
```java
public abstract class PaymentMethod {
    public abstract void processPayment(double amount);
}

// Problem: Unknown Implementations Can Appear Anywhere
public class BitcoinPayment extends PaymentMethod {  // Surprise!
    @Override
    public void processPayment(double amount) { }
}

// Handling Payments becomes Fragile
public void handlePayment(PaymentMethod payment) {
    if (payment instanceof CreditCard) {
        // Handle Credit Card
    } else if (payment instanceof BankTransfer) {
        // Handle Transfer
    } else {
        // What else could it be?
        throw new IllegalArgumentException("Unknown payment type");
    }
}
```

### ✅ Sealed Class - Controlled Hierarchy
```java
public sealed class PaymentMethod 
    permits CreditCard, BankTransfer, DigitalWallet {
    
    public abstract void processPayment(double amount);
}

public final class CreditCard extends PaymentMethod {
    @Override
    public void processPayment(double amount) { }
}

public final class BankTransfer extends PaymentMethod {
    @Override
    public void processPayment(double amount) { }
}

public final class DigitalWallet extends PaymentMethod {
    @Override
    public void processPayment(double amount) { }
}
```

### ✅ Exhaustive Pattern Matching - Compiler-Verified
```java
public void handlePayment(PaymentMethod payment) {
    switch (payment) {
        case CreditCard card -> processCreditCard(card);
        case BankTransfer transfer -> processBankTransfer(transfer);
        case DigitalWallet wallet -> processDigitalWallet(wallet);
        // No default needed! Compiler knows these are the only options
    }
}
```

### ✅ Sealed with Records - Perfect Combination
```java
public sealed interface Result<T> permits Success, Failure {
    
    record Success<T>(T data) implements Result<T> { }
    
    record Failure<T>(String error, Exception cause) implements Result<T> { }
}

// Pattern Matching becomes Elegant
public <T> void handleResult(Result<T> result) {
    switch (result) {
        case Success<T>(var data) -> processData(data);
        case Failure<T>(var error, var cause) -> logError(error, cause);
    }
}
```

## 💡 Why This Matters

Sealed classes restrict who can extend your hierarchy, making the set of subtypes finite and known. The compiler can verify exhaustiveness in pattern matching - no default clause needed. When you add a new permitted subclass, every switch breaks until you handle the new case. This turns runtime errors into compile-time errors.

## 🎯 Key Takeaway

Use sealed classes when you own the complete set of implementations and want compiler-verified exhaustiveness. Perfect for state machines, result types, and domain models with finite variants.

---

**Tags:** `#Java` `#JavaWisdom` `#SealedClasses` `#PatternMatching` `#ModernJava` `#TypeSafety`
