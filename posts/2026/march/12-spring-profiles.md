# Post #12: Spring Profiles - Beyond Dev and Prod

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 15, 2026  
**Topic:** Spring Profiles, Configuration Management

---

## The Problem

You're using profiles wrong. Here's what most developers miss.

## Code Example

### ❌ Profile Explosion - Unmaintainable
```java
@Configuration
@Profile("dev-with-mock-api-and-h2-db")
public class DevConfig { }

@Configuration
@Profile("dev-with-real-api-and-postgres")
public class DevConfig2 { }

@Configuration
@Profile("qa-with-test-data")
public class QAConfig { }

// Activating: spring.profiles.active=dev-with-mock-api-and-h2-db
```

### ❌ Hardcoding Environment Logic
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    public DataSource dataSource() {
        String env = System.getProperty("spring.profiles.active");
        if (env.equals("dev")) {
            return createH2DataSource();
        } else if (env.equals("prod")) {
            return createPostgresDataSource();
        }
        return null;  // What Could Go Wrong?
    }
}
```

### ✅ Composable Profiles with Profile Groups
```properties
# application.properties
spring.profiles.group.local=localdb,mockapi,debug
spring.profiles.group.dev=devdb,realapi,logging
spring.profiles.group.prod=proddb,realapi,monitoring

# Now activate with clean names
spring.profiles.active=local
```

### ✅ Declarative profile-specific Beans
```java
@Configuration
@Profile("localdb")
public class LocalDatabaseConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@Profile("proddb")
public class ProductionDatabaseConfig {
    @Bean
    public DataSource dataSource() {
        // Read from Environment Variables
        return DataSourceBuilder.create()
            .url(System.getenv("DB_URL"))
            .username(System.getenv("DB_USER"))
            .password(System.getenv("DB_PASSWORD"))
            .build();
    }
}

@Configuration
@Profile("mockapi")
public class MockApiConfig {
    @Bean
    public PaymentService paymentService() {
        return new MockPaymentService();
    }
}

@Configuration
@Profile("realapi")
public class RealApiConfig {
    @Bean
    public PaymentService paymentService() {
        return new StripePaymentService();
    }
}
```

## 💡 Why This Matters

Profile groups (Spring Boot 2.4+) let you compose multiple orthogonal concerns into logical environments. Instead of creating dev-local-h2-mock-ssl-off nightmares, you combine small, focused profiles like localdb + mockapi + debug. Each profile does one thing. The group composes them. This scales as your configuration grows and makes testing combinations trivial.

## 🎯 Key Takeaway

Stop creating mega-profiles for every environment permutation. Use profile groups to compose small, focused profiles. Think "database strategy" + "API mode" + "observability level" instead of "dev-scenario-17".

---

**Tags:** `#Java` `#JavaWisdom` `#SpringBoot` `#SpringProfiles` `#Configuration` `#BestPractices`
