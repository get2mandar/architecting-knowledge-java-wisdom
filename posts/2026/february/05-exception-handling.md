# Post #5: Exception Handling - Fail Fast, Fail Clear

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** February 18, 2026  
**Topic:** Exception Handling, Spring ControllerAdvice

---

## The Problem

Junior devs scatter try-catch. Senior devs centralize with advice.

## Code Example

### ❌ Junior Approach - try-catch Everywhere
```java
@RestController
public class UserController {
    
    public ResponseEntity getUser(Long id) {
        try {
            return ResponseEntity.ok(userService.findById(id));
        } catch (UserNotFoundException e) {
            return ResponseEntity.status(404).body(e.getMessage());
        } catch (Exception e) {
            return ResponseEntity.status(500).body("Error occurred");
        }
    }
    
    public ResponseEntity createUser(UserDTO dto) {
        try {
            return ResponseEntity.ok(userService.create(dto));
        } catch (ValidationException e) {
            return ResponseEntity.status(400).body(e.getMessage());
        } catch (Exception e) {
            return ResponseEntity.status(500).body("Error occurred");
        }
    }
}
```

### ✅ Senior Approach - Centralized Exception Handling
```java
@RestController
public class UserController {
    
    public User getUser(Long id) {
        return userService.findById(id);  // Clean business logic
    }
    
    public User createUser(UserDTO dto) {
        return userService.create(dto);  // No noise
    }
}

@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleUserNotFound(UserNotFoundException e) {
        return new ErrorResponse("USER_NOT_FOUND", e.getMessage());
    }
    
    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(ValidationException e) {
        return new ErrorResponse("VALIDATION_ERROR", e.getMessage());
    }
}
```

## Why This Matters

Scattering try-catch blocks couples error handling to business logic. ControllerAdvice separates concerns - your controllers stay clean, exception handling stays consistent across the application.

## Key Takeaway

Let exceptions bubble up to a centralized handler. Your business logic should read like a happy path, not a paranoid checklist.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringFramework` `#ExceptionHandling` `#CleanCode` `#BestPractices`
