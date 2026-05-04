# Post #27: Functional Interfaces - Beyond the Basics

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 6, 2026  
**Topic:** Functional Interfaces, Lambda Expressions, Java 8

---

## The Problem

Lambda expressions need homes. Functional interfaces give them one.

## Code Example

### ❌ Anonymous Classes - Verbose and Ugly

```java
// Before Java 8 - Anonymous Inner Classes
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

names.sort(new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
});

// Filter List
List<String> filtered = new ArrayList<>();
for (String name : names) {
    if (name.startsWith("A")) {
        filtered.add(name);
    }
}
```

### ✅ Functional Interfaces - Clean Lambdas

```java
// With Functional Interfaces and Lambdas
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

names.sort((a, b) -> a.length() - b.length());

List<String> filtered = names.stream()
    .filter(name -> name.startsWith("A"))
    .collect(Collectors.toList());
```

### ✅ Understanding @FunctionalInterface

```java
@FunctionalInterface
public interface Validator<T> {
    boolean validate(T value);  // Single Abstract Method (SAM)
    
    // Default Methods Are Allowed
    default Validator<T> and(Validator<T> other) {
        return value -> this.validate(value) && other.validate(value);
    }
    
    // Static Methods Are Allowed
    static <T> Validator<T> alwaysTrue() {
        return value -> true;
    }
}

// Lambda Expression Implements the SAM
Validator<String> notEmpty = str -> str != null && !str.isEmpty();
```

### ✅ The Four Core Functional Interfaces

```java
// 1. Predicate<T> - Takes Input, Returns Boolean
Predicate<String> startsWithA = str -> str.startsWith("A");
System.out.println(startsWithA.test("Apple"));  // true

// 2. Function<T, R> - Takes Input, Returns Transformed Output
Function<String, Integer> toLength = str -> str.length();
System.out.println(toLength.apply("Lambda"));  // 6

// 3. Consumer<T> - Takes Input, Returns Nothing (Side Effects)
Consumer<String> printer = str -> System.out.println(str);
printer.accept("Hello World");

// 4. Supplier<T> - No Input, Returns Output
Supplier<Double> randomValue = () -> Math.random();
System.out.println(randomValue.get());  // Random Number
```

### ✅ Predicate - Filtering and Conditions

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8);

// Simple Predicate
Predicate<Integer> isEven = n -> n % 2 == 0;
List<Integer> evens = numbers.stream()
    .filter(isEven)
    .collect(Collectors.toList());

// Combining Predicates with AND, OR, NEGATE
Predicate<Integer> greaterThanFive = n -> n > 5;
Predicate<Integer> combined = isEven.and(greaterThanFive);

numbers.stream()
    .filter(combined)  // Even AND Greater Than 5
    .forEach(System.out::println);  // Output: 6, 8

// Negate Predicate
Predicate<Integer> isOdd = isEven.negate();
```

### ✅ Function - Transformation and Mapping

```java
List<String> words = Arrays.asList("apple", "banana", "cherry");

// Simple Function
Function<String, Integer> wordLength = str -> str.length();
words.stream()
    .map(wordLength)
    .forEach(System.out::println);  // 5, 6, 6

// Chaining Functions with andThen
Function<String, String> toUpper = String::toUpperCase;
Function<String, String> addPrefix = str -> ">> " + str;
Function<String, String> pipeline = toUpper.andThen(addPrefix);

System.out.println(pipeline.apply("hello"));  // >> HELLO

// compose() - Execute Before Current Function
Function<Integer, Integer> multiplyByTwo = x -> x * 2;
Function<Integer, Integer> addTen = x -> x + 10;
Function<Integer, Integer> composed = multiplyByTwo.compose(addTen);

System.out.println(composed.apply(5));  // (5 + 10) * 2 = 30
```

### ✅ Consumer - Side Effects Without Return

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

// Simple Consumer
Consumer<String> printName = name -> System.out.println(name);
names.forEach(printName);

// BiConsumer - Two Arguments
Map<String, Integer> ages = new HashMap<>();
ages.put("Alice", 25);
ages.put("Bob", 30);

BiConsumer<String, Integer> printAge = (name, age) -> 
    System.out.println(name + " is " + age + " years old");

ages.forEach(printAge);

// Chaining Consumers with andThen
Consumer<String> logName = name -> logger.info("Processing: " + name);
Consumer<String> saveName = name -> repository.save(name);
Consumer<String> pipeline = logName.andThen(saveName);

names.forEach(pipeline);
```

### ✅ Supplier - Lazy Value Generation

```java
// Simple Supplier
Supplier<String> randomId = () -> UUID.randomUUID().toString();
System.out.println(randomId.get());

// Lazy Evaluation - Expensive Operation Only When Needed
public double squareLazy(Supplier<Double> lazyValue) {
    return Math.pow(lazyValue.get(), 2);
}

Supplier<Double> expensiveCalc = () -> {
    // Simulate Expensive Operation
    sleep(1000);
    return 9.0;
};

double result = squareLazy(expensiveCalc);  // Only Executed When Called

// Random OTP Generator
Supplier<String> otpGenerator = () -> {
    StringBuilder otp = new StringBuilder();
    for (int i = 0; i < 4; i++) {
        otp.append((int) (Math.random() * 10));
    }
    return otp.toString();
};

System.out.println(otpGenerator.get());  // 7423
System.out.println(otpGenerator.get());  // 1956
```

### ✅ Specialized Functional Interfaces - Primitives

```java
// Avoid Boxing/Unboxing Overhead
IntPredicate isEvenInt = n -> n % 2 == 0;
IntFunction<String> intToString = n -> "Number: " + n;
IntConsumer printInt = System.out::println;
IntSupplier randomInt = () -> (int) (Math.random() * 100);

// Similar for Double and Long
DoublePredicate isPositive = d -> d > 0.0;
LongFunction<String> longToHex = Long::toHexString;
```

### ✅ BiFunction - Two Arguments, One Output

```java
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
System.out.println(add.apply(5, 3));  // 8

BiFunction<String, String, String> concat = (a, b) -> a + " " + b;
System.out.println(concat.apply("Hello", "World"));  // Hello World

// andThen with BiFunction
BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
Function<Integer, String> toString = n -> "Result: " + n;
BiFunction<Integer, Integer, String> pipeline = multiply.andThen(toString);

System.out.println(pipeline.apply(3, 4));  // Result: 12
```

### ✅ UnaryOperator and BinaryOperator

```java
// UnaryOperator<T> - Function<T, T> (Same Input and Output Type)
UnaryOperator<String> toUpperCase = String::toUpperCase;
System.out.println(toUpperCase.apply("hello"));  // HELLO

List<String> names = Arrays.asList("alice", "bob", "charlie");
names.replaceAll(String::toUpperCase);  // Modifies In Place

// BinaryOperator<T> - BiFunction<T, T, T> (All Same Type)
BinaryOperator<Integer> sum = (a, b) -> a + b;
BinaryOperator<String> longerString = (s1, s2) -> 
    s1.length() > s2.length() ? s1 : s2;
```

### ✅ Creating Custom Functional Interfaces

```java
@FunctionalInterface
public interface Validator<T> {
    boolean isValid(T value);
}

@FunctionalInterface
public interface TriFunction<T, U, V, R> {
    R apply(T t, U u, V v);
}

// Usage
TriFunction<Integer, Integer, Integer, Integer> sumThree = 
    (a, b, c) -> a + b + c;

System.out.println(sumThree.apply(1, 2, 3));  // 6

Validator<String> emailValidator = email -> 
    email.contains("@") && email.contains(".");
```

### ✅ Real-World Example - Validation Pipeline

```java
public class UserValidator {
    private List<Predicate<User>> validations = new ArrayList<>();
    
    public UserValidator withAgeCheck() {
        validations.add(user -> user.getAge() >= 18);
        return this;
    }
    
    public UserValidator withEmailCheck() {
        validations.add(user -> user.getEmail().contains("@"));
        return this;
    }
    
    public UserValidator withCustomCheck(Predicate<User> predicate) {
        validations.add(predicate);
        return this;
    }
    
    public boolean validate(User user) {
        return validations.stream()
            .allMatch(predicate -> predicate.test(user));
    }
}

// Usage
UserValidator validator = new UserValidator()
    .withAgeCheck()
    .withEmailCheck()
    .withCustomCheck(user -> !user.getName().isEmpty());

boolean isValid = validator.validate(user);
```

### ✅ Composing Complex Logic

```java
// Building Reusable Predicates
Predicate<String> isNotNull = Objects::nonNull;
Predicate<String> isNotEmpty = str -> !str.isEmpty();
Predicate<String> isValid = isNotNull.and(isNotEmpty);

// Function Pipeline
Function<String, String> trim = String::trim;
Function<String, String> lowercase = String::toLowerCase;
Function<String, String> normalize = trim.andThen(lowercase);

List<String> normalized = inputs.stream()
    .filter(isValid)
    .map(normalize)
    .collect(Collectors.toList());
```

## 💡 Why This Matters

Functional interfaces have exactly one abstract method (SAM). @FunctionalInterface annotation ensures compile-time safety. Predicate<T> for boolean tests, Function<T,R> for transformations, Consumer<T> for side effects, Supplier<T> for value generation. Default methods (and, or, andThen, compose) enable composition. Primitive specializations (IntPredicate, LongFunction) avoid boxing overhead. Lambda expressions are shorthand for implementing functional interfaces. Method references (String::length) are even shorter when just calling one method.

## 🎯 Key Takeaway

Use built-in functional interfaces instead of creating custom SAM interfaces. Compose predicates with and/or/negate for complex conditions. Chain functions with andThen/compose for transformation pipelines. Leverage primitive specializations for performance. Functional interfaces enable declarative, readable code.

---

**Tags:** `#Java` `#JavaWisdom` `#FunctionalInterfaces` `#Lambda` `#Java8` `#FunctionalProgramming`
