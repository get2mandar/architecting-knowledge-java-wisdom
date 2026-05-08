# Post #28: Database Transactions - ACID in Practice

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 10, 2026  
**Topic:** Database Transactions, ACID Properties, @Transactional, Isolation Levels

---

## The Problem

Half-committed data destroys trust. ACID keeps promises.

## Code Example

### ❌ No Transaction - Data Corruption Waiting to Happen

```java
@Service
public class BankService {
    
    public void transfer(Account from, Account to, BigDecimal amount) {
        from.setBalance(from.getBalance().subtract(amount));
        accountRepository.save(from);  // Saved!
        
        // Application Crashes Here - Money Vanished!
        
        to.setBalance(to.getBalance().add(amount));
        accountRepository.save(to);  // Never Executed
    }
}
```

### ✅ With @Transactional - All or Nothing

```java
@Service
public class BankService {
    
    @Transactional
    public void transfer(Account from, Account to, BigDecimal amount) {
        from.setBalance(from.getBalance().subtract(amount));
        accountRepository.save(from);
        
        // Any Exception Here - Everything Rolls Back!
        
        to.setBalance(to.getBalance().add(amount));
        accountRepository.save(to);
        
        // Both Succeed or Both Fail - Atomicity!
    }
}
```

### ✅ Understanding ACID Properties

```java
/*
A - Atomicity
    All operations succeed or all fail - no partial commits
    
C - Consistency
    Database moves from one valid state to another
    Constraints and rules always enforced
    
I - Isolation
    Concurrent transactions don't interfere with each other
    Each transaction sees consistent data
    
D - Durability
    Once committed, changes are permanent
    Survives crashes and restarts
*/
```

### ✅ Atomicity - All or Nothing

```java
@Service
public class OrderService {
    
    @Transactional
    public void placeOrder(Order order) {
        // Step 1: Save Order
        orderRepository.save(order);
        
        // Step 2: Reduce Inventory
        inventoryService.reduceStock(order.getItems());
        
        // Step 3: Process Payment
        paymentService.charge(order.getTotal());
        
        // If ANY Step Fails - ALL Steps Rollback!
        // No Half-Placed Orders in Database
    }
}
```

### ✅ Isolation Levels - Controlling Concurrency

```java
// Default - Uses Database Default (Usually REPEATABLE_READ)
@Transactional
public void defaultIsolation() {
}

// READ_UNCOMMITTED - Fastest, Least Safe (Dirty Reads Possible)
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public void readUncommitted() {
    // Can Read Uncommitted Changes from Other Transactions
    // Risk: Dirty Reads - Reading Data That Might Rollback
}

// READ_COMMITTED - Prevents Dirty Reads
@Transactional(isolation = Isolation.READ_COMMITTED)
public void readCommitted() {
    // Only Reads Committed Data
    // Risk: Non-Repeatable Reads - Same Query, Different Results
}

// REPEATABLE_READ - Prevents Dirty and Non-Repeatable Reads
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void repeatableRead() {
    // Same Query Always Returns Same Results Within Transaction
    // Risk: Phantom Reads - New Rows Appear in Range Queries
}

// SERIALIZABLE - Complete Isolation (Slowest, Safest)
@Transactional(isolation = Isolation.SERIALIZABLE)
public void serializable() {
    // Transactions Execute as if Serial - No Concurrency Issues
    // No Dirty Reads, Non-Repeatable Reads, or Phantom Reads
}
```

### ✅ Isolation Problems Explained

```java
// Dirty Read - Transaction A Reads Uncommitted Data from Transaction B
// Transaction A
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public void dirtyRead() {
    Account account = accountRepository.findById(1L).orElseThrow();
    // Reads Balance = $500 (Modified by Transaction B, Not Committed)
    
    // Transaction B Rolls Back!
    // Now Reading Data That Never Existed - Dirty Read!
}

// Non-Repeatable Read - Same Query, Different Results
// Transaction A
@Transactional(isolation = Isolation.READ_COMMITTED)
public void nonRepeatableRead() {
    Account account = accountRepository.findById(1L).orElseThrow();
    System.out.println(account.getBalance());  // $500
    
    // Transaction B Updates and Commits Balance to $600
    
    account = accountRepository.findById(1L).orElseThrow();
    System.out.println(account.getBalance());  // $600 - Different!
    
    // Same Query, Different Result - Non-Repeatable Read
}

// Phantom Read - New Rows Appear in Range Query
// Transaction A
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void phantomRead() {
    List<Account> accounts = accountRepository.findByBalanceGreaterThan(1000);
    System.out.println(accounts.size());  // 5 Accounts
    
    // Transaction B Inserts New Account with Balance $2000 and Commits
    
    accounts = accountRepository.findByBalanceGreaterThan(1000);
    System.out.println(accounts.size());  // 6 Accounts - Phantom Row!
}
```

### ✅ Choosing the Right Isolation Level

```java
@Service
public class TransactionExamples {
    
    // High Concurrency, No Critical Data - READ_COMMITTED
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void viewActivityLogs() {
        // Read-Heavy, Performance Critical
        // Minor Inconsistencies Acceptable
    }
    
    // Financial Operations - REPEATABLE_READ
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void processPayment(Payment payment) {
        // Balance Must Stay Consistent Within Transaction
        // Prevents Non-Repeatable Reads
    }
    
    // Critical Accounting - SERIALIZABLE
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void endOfDayReconciliation() {
        // Complete Isolation Required
        // Performance Less Important Than Accuracy
    }
}
```

### ✅ Rollback Behavior - What Triggers Rollback?

```java
@Service
public class RollbackService {
    
    // Default: Rollback on RuntimeException Only
    @Transactional
    public void defaultRollback() {
        accountRepository.save(account);
        throw new RuntimeException("Rolls Back!");  // Transaction Rolled Back
    }
    
    @Transactional
    public void checkedExceptionNoRollback() throws Exception {
        accountRepository.save(account);
        throw new Exception("Does NOT Roll Back!");  // Transaction Commits!
    }
    
    // Rollback on Specific Exceptions
    @Transactional(rollbackFor = {SQLException.class, IOException.class})
    public void rollbackOnChecked() throws SQLException {
        accountRepository.save(account);
        throw new SQLException("Rolls Back!");  // Explicit Rollback
    }
    
    // Don't Rollback on Specific Exceptions
    @Transactional(noRollbackFor = {ValidationException.class})
    public void noRollbackOnValidation() {
        accountRepository.save(account);
        throw new ValidationException("Commits Anyway!");  // No Rollback
    }
}
```

### ✅ Read-Only Optimization - Performance Boost

```java
@Service
public class QueryService {
    
    @Transactional(readOnly = true)
    public List<User> getAllUsers() {
        // Hibernate Skips Dirty Checking - Performance Gain!
        // Database Can Optimize for Read-Only
        return userRepository.findAll();
    }
    
    // Wrong! Read-Only with Writes
    @Transactional(readOnly = true)
    public void saveUser(User user) {
        userRepository.save(user);  // May Fail or Be Ignored!
    }
}
```

### ✅ Transaction Timeout - Prevent Hanging

```java
@Service
public class TimeoutService {
    
    @Transactional(timeout = 30)  // 30 Seconds Max
    public void longRunningOperation() {
        // If Not Complete in 30s - Rollback!
        complexCalculation();
    }
}
```

### ✅ Programmatic Rollback - Manual Control

```java
@Service
public class ManualRollbackService {
    
    @Transactional
    public void conditionalRollback() {
        accountRepository.save(account);
        
        if (fraudDetected()) {
            TransactionAspectSupport.currentTransactionStatus()
                .setRollbackOnly();
            // Transaction Will Rollback Even Without Exception
        }
    }
}
```

### ✅ Transaction Boundary - Where It Starts and Ends

```java
@Service
public class BoundaryService {
    
    // Transaction Starts Here
    @Transactional
    public void outerMethod() {
        accountRepository.save(account1);
        
        innerMethod();  // Joins Existing Transaction!
        
        accountRepository.save(account2);
    }  // Transaction Commits Here
    
    // No New Transaction - Joins Outer
    @Transactional
    public void innerMethod() {
        accountRepository.save(account3);
        // All Three Saves in Single Transaction
    }
}
```

### ❌ Common Pitfall - Self-Invocation Doesn't Work

```java
@Service
public class SelfInvocationProblem {
    
    public void nonTransactional() {
        this.transactionalMethod();  // @Transactional IGNORED!
        // Spring Proxy Not Invoked - No Transaction!
    }
    
    @Transactional
    public void transactionalMethod() {
        accountRepository.save(account);
    }
}

// ✅ Fix: Inject Self or Use Separate Service
@Service
public class SelfInvocationFix {
    
    @Autowired
    private SelfInvocationFix self;  // Inject Spring Proxy
    
    public void nonTransactional() {
        self.transactionalMethod();  // Works! Proxy Invoked
    }
    
    @Transactional
    public void transactionalMethod() {
        accountRepository.save(account);
    }
}
```

### ✅ Multiple Databases - Separate Transaction Managers

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public PlatformTransactionManager primaryTxManager() {
        return new JpaTransactionManager(primaryEntityManagerFactory());
    }
    
    @Bean
    public PlatformTransactionManager secondaryTxManager() {
        return new JpaTransactionManager(secondaryEntityManagerFactory());
    }
}

@Service
public class MultiDbService {
    
    @Transactional(transactionManager = "primaryTxManager")
    public void saveToPrimary() {
        primaryRepository.save(entity);
    }
    
    @Transactional(transactionManager = "secondaryTxManager")
    public void saveToSecondary() {
        secondaryRepository.save(entity);
    }
}
```

### ✅ Real-World Example - E-Commerce Order

```java
@Service
public class OrderService {
    
    @Transactional(isolation = Isolation.REPEATABLE_READ, timeout = 60)
    public void processOrder(OrderRequest request) {
        // Atomicity - All or Nothing
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Consistency - Enforce Business Rules
        if (order.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidOrderException("Total must be positive");
        }
        
        // Isolation - REPEATABLE_READ Prevents Balance Changes
        Account account = accountRepository.findById(request.getUserId())
            .orElseThrow();
        
        if (account.getBalance().compareTo(order.getTotal()) < 0) {
            throw new InsufficientFundsException();
        }
        
        account.debit(order.getTotal());
        accountRepository.save(account);
        
        inventoryService.reserve(order.getItems());
        
        emailService.sendConfirmation(order);
        
        // Durability - Once Committed, Order Persists Forever
    }
}
```

## 💡 Why This Matters

ACID guarantees data integrity in concurrent environments. Atomicity ensures all-or-nothing execution - no partial commits. Consistency maintains database constraints and business rules. Isolation prevents concurrent transactions from interfering with each other. Durability guarantees committed data survives crashes. Default isolation is usually REPEATABLE_READ (MySQL) or READ_COMMITTED (PostgreSQL). Lower isolation = better performance but more anomalies. Higher isolation = safer data but slower performance. @Transactional only works on public methods called from outside the class (proxy pattern). RuntimeException triggers rollback by default, checked exceptions don't.

## 🎯 Key Takeaway

Use @Transactional for operations that must succeed or fail together. Choose isolation level based on consistency needs vs performance. READ_COMMITTED for most use cases, REPEATABLE_READ for financial operations, SERIALIZABLE for critical accounting. Set readOnly=true for queries - performance boost. Avoid self-invocation - inject the bean to call transactional methods.

---

**Tags:** `#Java` `#JavaWisdom` `#Transactions` `#ACID` `#Database` `#SpringBoot`
