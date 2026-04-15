# Post #21: Validator Pattern - Beyond Bean Validation

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** April 15, 2026  
**Topic:** Validation Patterns, Custom Validators, Spring Boot

---

## The Problem

Bean Validation is great. Until it's not enough.

## Code Example

### ❌ Bean Validation - Limited to Field-Level Rules

```java
public class UserRegistrationDto {
    @NotBlank
    @Email
    private String email;
    
    @NotBlank
    @Size(min = 8)
    private String password;
    
    @NotBlank
    private String confirmPassword;
    
    // Can't Validate: Password == ConfirmPassword Across Fields!
    // Can't Validate: Email Already Exists in Database!
    // Can't Validate: Complex Business Rules!
}
```

### ❌ Validation Logic Scattered - Service Layer Does Everything

```java
@Service
public class UserService {
    public void registerUser(UserDto dto) {
        if (!dto.getPassword().equals(dto.getConfirmPassword())) {
            throw new ValidationException("Passwords don't match");
        }
        if (userRepository.existsByEmail(dto.getEmail())) {
            throw new ValidationException("Email already exists");
        }
        if (!isValidDomain(dto.getEmail())) {
            throw new ValidationException("Invalid email domain");
        }
        // Business Logic Mixed with Validation!
    }
}
```

### ✅ Custom Annotation - Cross-Field Validation

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PasswordMatchValidator 
    implements ConstraintValidator<PasswordMatch, UserRegistrationDto> {
    
    @Override
    public boolean isValid(UserRegistrationDto dto, 
                          ConstraintValidatorContext context) {
        if (dto.getPassword() == null || dto.getConfirmPassword() == null) {
            return true;  // Let @NotBlank Handle Nulls
        }
        return dto.getPassword().equals(dto.getConfirmPassword());
    }
}

@PasswordMatch
public class UserRegistrationDto {
    private String password;
    private String confirmPassword;
    // Validation Happens at Class Level!
}
```

### ✅ Stateful Validator - Inject Dependencies

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator 
    implements ConstraintValidator<UniqueEmail, String> {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) {
            return true;
        }
        return !userRepository.existsByEmail(email);
    }
}

public class UserRegistrationDto {
    @NotBlank
    @Email
    @UniqueEmail  // Checks Database!
    private String email;
}
```

### ✅ Programmatic Validator - Complex Business Rules

```java
@Component
public class UserValidator implements Validator {
    
    @Override
    public boolean supports(Class<?> clazz) {
        return UserRegistrationDto.class.equals(clazz);
    }
    
    @Override
    public void validate(Object target, Errors errors) {
        UserRegistrationDto dto = (UserRegistrationDto) target;
        
        // Complex Cross-Field Logic
        if (dto.getAge() < 18 && dto.requiresParentalConsent() == null) {
            errors.rejectValue("parentalConsent", 
                "required.for.minors", 
                "Parental consent required for users under 18");
        }
        
        // Business Rule Validation
        if (dto.getRole() == Role.ADMIN && !hasAdminApproval(dto)) {
            errors.rejectValue("role", 
                "admin.requires.approval", 
                "Admin role requires supervisor approval");
        }
    }
}
```

### ✅ Controller Integration - Layered Validation

```java
@RestController
@RequiredArgsConstructor
public class UserController {
    private final UserValidator userValidator;
    
    @PostMapping("/register")
    public ResponseEntity<?> register(
            @Valid @RequestBody UserRegistrationDto dto,  // Bean Validation First
            BindingResult result) {
        
        // Then Custom Programmatic Validation
        userValidator.validate(dto, result);
        
        if (result.hasErrors()) {
            return ResponseEntity.badRequest()
                .body(formatErrors(result));
        }
        
        userService.register(dto);
        return ResponseEntity.ok().build();
    }
}
```

### ✅ Validation Groups - Conditional Rules

```java
public interface CreateValidation {}
public interface UpdateValidation {}

public class UserDto {
    @Null(groups = CreateValidation.class)
    @NotNull(groups = UpdateValidation.class)
    private Long id;  // Null on Create, Required on Update
    
    @NotBlank(groups = {CreateValidation.class, UpdateValidation.class})
    private String email;
}

@PostMapping("/users")
public ResponseEntity<?> create(
        @Validated(CreateValidation.class) @RequestBody UserDto dto) {
    // Only CreateValidation Rules Apply
}

@PutMapping("/users/{id}")
public ResponseEntity<?> update(
        @Validated(UpdateValidation.class) @RequestBody UserDto dto) {
    // Only UpdateValidation Rules Apply
}
```

## 💡 Why This Matters

Bean Validation annotations work great for simple field-level rules. Custom `@Constraint` annotations handle cross-field validation (passwords match, dates in order). Stateful validators can inject Spring beans - check database, call external services. `ConstraintValidator` runs automatically during `@Valid` processing. Programmatic `Validator` interface gives full control for complex business rules. Validation groups apply different rules based on context (create vs update). Keep validation separate from business logic - Single Responsibility Principle.

## 🎯 Key Takeaway

Use Bean Validation for simple rules, custom annotations for cross-field logic, programmatic validators for complex business rules. Inject dependencies into validators when needed. Separate validation concerns from service layer. Let validators fail fast before business logic runs.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringBoot` `#Validation` `#CleanCode` `#BestPractices`
