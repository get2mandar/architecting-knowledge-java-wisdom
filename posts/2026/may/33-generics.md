# Post #33: Generics - Type Safety Without the Headache

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 27, 2026  
**Topic:** Java Generics, Type Safety, Wildcards, PECS Principle

---

## The Problem

Raw types compile but crash at runtime. Generics catch errors at compile time.

## Code Example

### ❌ Before Generics - Casting Hell

```java
// Raw List - No Type Safety
List items = new ArrayList();
items.add("Hello");
items.add(42);  // Mixed Types Allowed!
items.add(new User());  // Anything Goes

// Runtime Disaster
String text = (String) items.get(1);  // ClassCastException!
```

### ✅ With Generics - Compile-Time Safety

```java
List<String> items = new ArrayList<>();
items.add("Hello");
items.add(42);  // Compile Error! Type Mismatch
items.add(new User());  // Compile Error!

String text = items.get(0);  // No Cast Needed!
```

### ✅ Generic Classes - Reusable Type-Safe Containers

```java
public class Box<T> {
    private T content;
    
    public void set(T content) {
        this.content = content;
    }
    
    public T get() {
        return content;
    }
}

// Usage
Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String value = stringBox.get();  // No Cast!

Box<Integer> intBox = new Box<>();
intBox.set(42);
Integer number = intBox.get();
```

### ✅ Generic Methods - Type-Safe Utilities

```java
public class Utils {
    
    // Generic Method - Works with Any Type
    public static <T> T getFirst(List<T> list) {
        if (list.isEmpty()) {
            return null;
        }
        return list.get(0);
    }
    
    // Multiple Type Parameters
    public static <K, V> Map<K, V> createMap(K key, V value) {
        Map<K, V> map = new HashMap<>();
        map.put(key, value);
        return map;
    }
}

// Usage
String first = Utils.getFirst(Arrays.asList("a", "b", "c"));
Map<String, Integer> map = Utils.createMap("age", 25);
```

### ✅ Bounded Type Parameters - Restrict Type Range

```java
// Upper Bound - T Must Extend Number
public class Calculator<T extends Number> {
    
    public double sum(List<T> numbers) {
        return numbers.stream()
            .mapToDouble(Number::doubleValue)
            .sum();
    }
}

// Usage
Calculator<Integer> intCalc = new Calculator<>();
Calculator<Double> doubleCalc = new Calculator<>();
Calculator<String> stringCalc = new Calculator<>();  // Compile Error!

// Multiple Bounds - T Must Extend Both
public class Cache<T extends Serializable & Comparable<T>> {
    private T data;
    
    // T Can Be Serialized AND Compared
}
```

### ✅ Wildcards - Unknown Type Flexibility

```java
// Unbounded Wildcard - Any Type
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

// Works with Any List
printList(Arrays.asList(1, 2, 3));
printList(Arrays.asList("a", "b", "c"));
printList(Arrays.asList(new User(), new User()));
```

### ✅ Upper Bounded Wildcard - Producer Extends

```java
// ? extends Number - Read Only, Any Subtype of Number
public double sumNumbers(List<? extends Number> numbers) {
    double sum = 0;
    for (Number num : numbers) {
        sum += num.doubleValue();  // Can Read!
    }
    // numbers.add(10);  // Compile Error! Can't Write
    return sum;
}

// Works with All Number Subtypes
sumNumbers(Arrays.asList(1, 2, 3));  // List<Integer>
sumNumbers(Arrays.asList(1.5, 2.5));  // List<Double>
sumNumbers(Arrays.asList(1L, 2L));  // List<Long>
```

### ✅ Lower Bounded Wildcard - Consumer Super

```java
// ? super Integer - Write Only, Any Supertype of Integer
public void addIntegers(List<? super Integer> list) {
    list.add(1);  // Can Write Integers!
    list.add(2);
    list.add(3);
    
    // Object item = list.get(0);  // Can Only Read as Object
}

// Works with Integer and Supertypes
List<Integer> integers = new ArrayList<>();
List<Number> numbers = new ArrayList<>();
List<Object> objects = new ArrayList<>();

addIntegers(integers);  // Works
addIntegers(numbers);   // Works
addIntegers(objects);   // Works
```

### ✅ PECS Principle - Producer Extends, Consumer Super

```java
// PECS in Action - Collections.copy()
public static <T> void copy(
    List<? super T> dest,      // Consumer - Super
    List<? extends T> src      // Producer - Extends
) {
    for (int i = 0; i < src.size(); i++) {
        dest.set(i, src.get(i));
    }
}

// Producer Extends - Read from Source
List<Integer> integers = Arrays.asList(1, 2, 3);
List<Number> numbers = new ArrayList<>(Arrays.asList(0.0, 0.0, 0.0));

// Integers Produce Numbers (Read)
// Numbers Consume Numbers (Write)
Collections.copy(numbers, integers);  // Works!
```

### ✅ Real-World Example - Generic Repository

```java
public interface Repository<T, ID> {
    
    Optional<T> findById(ID id);
    
    List<T> findAll();
    
    T save(T entity);
    
    void delete(T entity);
}

public class UserRepository implements Repository<User, Long> {
    
    @Override
    public Optional<User> findById(Long id) {
        // Implementation
    }
    
    @Override
    public List<User> findAll() {
        // Implementation
    }
    
    @Override
    public User save(User entity) {
        // Implementation
    }
    
    @Override
    public void delete(User entity) {
        // Implementation
    }
}
```

### ✅ Generic Builder Pattern

```java
public class QueryBuilder<T> {
    private final Class<T> entityClass;
    private List<Predicate> predicates = new ArrayList<>();
    
    public QueryBuilder(Class<T> entityClass) {
        this.entityClass = entityClass;
    }
    
    public QueryBuilder<T> where(String field, Object value) {
        predicates.add(new Predicate(field, value));
        return this;
    }
    
    public QueryBuilder<T> and(String field, Object value) {
        predicates.add(new Predicate(field, value));
        return this;
    }
    
    public List<T> execute() {
        // Build and Execute Query
        return new ArrayList<>();
    }
}

// Usage - Type Safe!
List<User> users = new QueryBuilder<>(User.class)
    .where("age", 25)
    .and("active", true)
    .execute();
```

### ❌ Type Erasure - What Gets Compiled

```java
// Source Code
public class Box<T> {
    private T content;
    public void set(T value) { content = value; }
    public T get() { return content; }
}

// After Type Erasure (What JVM Sees)
public class Box {
    private Object content;  // T Becomes Object
    public void set(Object value) { content = value; }
    public Object get() { return content; }
}

// With Bounded Type
public class NumberBox<T extends Number> {
    private T value;
}

// After Erasure
public class NumberBox {
    private Number value;  // T Becomes Number (Bound)
}
```

### ❌ Generic Array Creation - Not Allowed

```java
// ❌ Compile Error!
List<String>[] arrayOfLists = new ArrayList<String>[10];

// ✅ Workaround - Unchecked Cast
@SuppressWarnings("unchecked")
List<String>[] arrayOfLists = (List<String>[]) new ArrayList[10];

// ✅ Better - Use List of Lists
List<List<String>> listOfLists = new ArrayList<>();
```

### ✅ Recursive Type Bounds - Comparable Example

```java
// Classic Recursive Bound
public static <T extends Comparable<T>> T max(List<T> list) {
    if (list.isEmpty()) {
        return null;
    }
    
    T max = list.get(0);
    for (T item : list) {
        if (item.compareTo(max) > 0) {
            max = item;
        }
    }
    return max;
}

// Usage
Integer maxInt = max(Arrays.asList(1, 5, 3, 9, 2));  // 9
String maxStr = max(Arrays.asList("apple", "zebra", "banana"));  // zebra
```

### ✅ PECS Examples - When to Use Each

```java
// Producer Extends - Reading from Collection
public void processNumbers(List<? extends Number> numbers) {
    for (Number num : numbers) {
        System.out.println(num.doubleValue());  // Read Only
    }
}

// Consumer Super - Writing to Collection
public void addToCollection(List<? super Integer> list) {
    list.add(10);
    list.add(20);  // Write Only
}

// Both - No Wildcard Needed
public void transformList(List<String> list) {
    for (int i = 0; i < list.size(); i++) {
        String value = list.get(i);  // Read
        list.set(i, value.toUpperCase());  // Write
    }
}
```

### ✅ Real-World PECS - Stack Operations

```java
public class Stack<E> {
    private List<E> elements = new ArrayList<>();
    
    public void push(E e) {
        elements.add(e);
    }
    
    public E pop() {
        return elements.remove(elements.size() - 1);
    }
    
    // Producer - Pushes Elements from Source onto This Stack
    public void pushAll(Iterable<? extends E> src) {
        for (E e : src) {
            push(e);  // Reading from src
        }
    }
    
    // Consumer - Pops Elements from This Stack into Destination
    public void popAll(Collection<? super E> dst) {
        while (!elements.isEmpty()) {
            dst.add(pop());  // Writing to dst
        }
    }
}

// Usage
Stack<Number> numberStack = new Stack<>();
List<Integer> integers = Arrays.asList(1, 2, 3);
List<Object> objects = new ArrayList<>();

numberStack.pushAll(integers);  // Producer Extends
numberStack.popAll(objects);    // Consumer Super
```

## 💡 Why This Matters

Generics provide compile-time type safety - catch errors before runtime. Type erasure removes generic type info at runtime - all become Object or bound type. Bounded types restrict acceptable types - `<T extends Number>` only allows Number subtypes. Wildcards (`?`) represent unknown types - more flexible than concrete type parameters. PECS principle guides wildcard choice - Producer Extends, Consumer Super. Upper bound (`? extends T`) allows reading but not writing - covariance. Lower bound (`? super T`) allows writing but not reading - contravariance. Generic methods infer types from arguments - no explicit type needed.

## 🎯 Key Takeaway

Use generics for type-safe collections and APIs. Apply PECS - `? extends T` for reading, `? super T` for writing. Bounded types enforce constraints at compile time. Avoid raw types - they bypass type checking. Remember type erasure - no runtime type info available.

---

**Tags:** `#Java` `#JavaWisdom` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#Generics` `#TypeSafety` `#PECS` `#Wildcards`
