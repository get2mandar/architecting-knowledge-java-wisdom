# Post #1: The Optional Mindset

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** February 4, 2026  
**Topic:** Optional API, Null Safety

---

## The Problem

Defensive null-checking makes code noisy and error-prone.

## Code Example

### ❌ Defensive Programming
```java
public void processUser(User user) {
    if (user != null && user.getEmail() != null) {
        sendEmail(user.getEmail());
    }
}
```

### ✅ Explicit Optionality
```java
public void processUser(Optional<User> user) {
    user.flatMap(User::getEmail)
        .ifPresent(this::sendEmail);
}
```

## 💡 The Shift

From null-checking to optional-thinking. Your API becomes honest about what might not exist.

## 🎯 Key Takeaway

Make absence explicit in your method signatures. Let the type system work for you.

---

## Note on Optional as Method Parameters

While this example shows `Optional<User>` as a method parameter for teaching purposes, Oracle's official guidance recommends using Optional primarily as a return type. In production code, the more conventional approach is:

```java
public void processUser(User user) {
    Optional.ofNullable(user)
        .flatMap(User::getEmail)
        .ifPresent(this::sendEmail);
}
```

This follows Java's design intent while still leveraging Optional's benefits. See [Oracle's Optional documentation](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Optional.html) for official guidance.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringFramework` `#CleanCode` `#JavaDevelopment`
