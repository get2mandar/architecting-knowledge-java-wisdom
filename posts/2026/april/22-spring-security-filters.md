# Post #22: Spring Security Filters - The Chain Explained

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** April 19, 2026  
**Topic:** Spring Security, Filter Chain, SecurityFilterChain

---

## The Problem

Authentication isn't magic. It's 15 filters in perfect order.

## Code Example

### ❌ Common Misconception - Security Is One Filter

```java
// "Spring Security Just Checks Authentication, Right?"
// Wrong! It's a Chain of Filters Working Together
```

### ❌ Custom Filter in Wrong Position - Breaks Everything

```java
@Component
public class MyCustomFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain chain) {
        // Try to Get Authentication - But It Doesn't Exist Yet!
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        // Auth is Null! Filter Runs Before Authentication Filters
    }
}
```

### ✅ Understanding the Filter Chain Order

```java
/*
1. SecurityContextPersistenceFilter - Loads SecurityContext from Session
2. CsrfFilter - Validates CSRF Token
3. LogoutFilter - Handles Logout Requests
4. UsernamePasswordAuthenticationFilter - Processes Login Form
5. BasicAuthenticationFilter - Processes HTTP Basic Auth
6. BearerTokenAuthenticationFilter - Processes JWT Tokens
7. SessionManagementFilter - Manages Sessions
8. ExceptionTranslationFilter - Handles Auth Exceptions
9. AuthorizationFilter - Checks Access Permissions
*/
```

### ✅ SecurityFilterChain Configuration - Modern Approach

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/api/**").hasRole("USER")
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        
        return http.build();
    }
}
```

### ✅ Multiple Filter Chains - Different Security Per URL

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    // API Security - Stateless with JWT
    @Bean
    @Order(1)
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> 
                auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()));
        
        return http.build();
    }
    
    // Web Security - Session-Based with Form Login
    @Bean
    @Order(2)
    public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
            );
        
        return http.build();
    }
}
```

### ✅ Custom Filter - Positioned Correctly

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain chain) 
            throws ServletException, IOException {
        
        logger.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        
        chain.doFilter(request, response);  // Continue Chain!
        
        logger.info("Response Status: {}", response.getStatus());
    }
}

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
                                          RequestLoggingFilter loggingFilter) 
            throws Exception {
        http
            .addFilterBefore(loggingFilter, 
                UsernamePasswordAuthenticationFilter.class)
            // OR
            .addFilterAfter(loggingFilter, 
                BasicAuthenticationFilter.class)
            // OR
            .addFilterAt(loggingFilter, 
                UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}
```

### ✅ Debugging Filter Chain - See What's Running

```java
@EnableWebSecurity(debug = true)  // Enable Only in Development!
public class SecurityConfig {
    // Logs All Filter Invocations
}

// application.properties
logging.level.org.springframework.security.web.FilterChainProxy=DEBUG
```

### ✅ Exception Handling - AuthenticationEntryPoint

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint((request, response, authException) -> {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("Authentication Required");
            })
            .accessDeniedHandler((request, response, accessDeniedException) -> {
                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                response.getWriter().write("Access Denied");
            })
        );
    
    return http.build();
}
```

## 💡 Why This Matters

FilterChainProxy delegates to SecurityFilterChain beans. Filters execute in strict order - position matters! First matching SecurityFilterChain wins (use @Order). Each filter has a specific job - context loading, authentication, authorization, exception handling. Custom filters must be positioned correctly using addFilterBefore/After/At. OncePerRequestFilter ensures execution happens once per request. @EnableWebSecurity(debug=true) shows filter execution flow. Multiple SecurityFilterChain beans allow different security per URL pattern.

## 🎯 Key Takeaway

Don't fight the filter chain - understand it. Position custom filters correctly. Use multiple SecurityFilterChain beans for different security requirements. Always call chain.doFilter() to continue. Debug mode is your friend during development.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringSecurity` `#Security` `#WebSecurity` `#BestPractices`