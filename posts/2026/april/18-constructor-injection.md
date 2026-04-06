# Post #18: Constructor Injection vs Field Injection

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** April 5, 2026  
**Topic:** Dependency Injection, Spring Framework, Best Practices

---

## The Problem

Field injection is convenient. And terrible. Here's why.

## Code Example

### ❌ Field Injection - Looks Simple, Hides Problems
```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private NotificationService notificationService;
    
    // How Many Dependencies? Can't Tell at a Glance
}

// Testing? Good Luck!
@Test
void testOrderService() {
    OrderService service = new OrderService();
    service.processOrder(order);  // NullPointerException!
    // Can't Instantiate Without Spring Context
}
```

### ❌ Setter Injection - Mutable Dependencies
```java
@Service
public class UserService {
    private UserRepository userRepository;
    
    @Autowired
    public void setUserRepository(UserRepository repo) {
        this.userRepository = repo;
    }
    
    // Dependency Can Be Reassigned Later - Not Immutable!
}
```

### ✅ Constructor Injection - Explicit and Immutable
```java
@Service
@RequiredArgsConstructor  // Lombok Generates Constructor
public class OrderService {
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final EmailService emailService;
    
    // Dependencies Are Clear, Immutable, and Required
}

// Without Lombok - Still Clean
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    
    public OrderService(PaymentService paymentService, 
                       InventoryService inventoryService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }
}
```

### ✅ Testing - Simple!
```java
@Test
void testOrderService() {
    PaymentService mockPayment = mock(PaymentService.class);
    InventoryService mockInventory = mock(InventoryService.class);
    
    OrderService service = new OrderService(mockPayment, mockInventory);
    service.processOrder(order);  // Works Without Spring!
}
```

### ✅ Too Many Dependencies - Smell Test!
```java
@Service
public class OrderService {
    // If This Constructor Has 7+ Parameters, Your Class Does Too Much
    public OrderService(
        PaymentService payment,
        InventoryService inventory,
        EmailService email,
        NotificationService notification,
        AuditService audit,
        TaxService tax,
        ShippingService shipping
    ) {
        // IDE Warning: Too Many Parameters!
        // Time to Refactor Into Smaller Services
    }
}
```

## 💡 Why This Matters

Field injection hides dependencies - you can't tell what a class needs without reading the entire file. Dependencies are mutable (non-final), risking reassignment bugs. Testing requires Spring context or reflection. Constructor injection makes dependencies explicit in one place. Final fields guarantee immutability. Testing is trivial - just pass mocks to the constructor. Bonus: when your constructor grows past 5-7 parameters, it screams "refactor me!" Field injection silently lets classes violate SRP.

## 🎯 Key Takeaway

Use constructor injection for mandatory dependencies. Mark fields final. Let your constructor size be your code smell detector. Field injection is convenient today, painful tomorrow.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringFramework` `#DependencyInjection` `#CleanCode` `#BestPractices`
