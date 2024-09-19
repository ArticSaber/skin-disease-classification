3Here's a full implementation of JWT-based authentication with role-based access control (RBAC) using `Spring Boot 3`. This setup ensures that only users with the `admin` role can access the `/getallusers` endpoint, and `/authenticate` allows users to log in by validating their credentials against the bcrypt password stored in the database.

### Steps:
1. **Add dependencies**: Ensure you have the necessary dependencies in your `pom.xml`.
2. **Create the JWT utilities**: For token generation and validation.
3. **Create filters for JWT authentication**.
4. **Implement the `SecurityFilterChain`**.
5. **Add authentication logic to the `/authenticate` endpoint**.

### 1. **Dependencies in `pom.xml`**:
Make sure your `pom.xml` includes the necessary Spring Security and JWT dependencies:
```xml
<dependencies>
    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- JWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.11.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId> <!-- or jjwt-gson -->
        <version>0.11.5</version>
    </dependency>
    <!-- BCrypt Password Encoder -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-crypto</artifactId>
    </dependency>
</dependencies>
```

### 2. **JWT Utility Class**:
```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.stereotype.Component;

import java.security.Key;
import java.util.Date;
import java.util.Map;
import java.util.function.Function;

@Component
public class JwtUtil {

    private final String SECRET_KEY = "your-256-bit-secret"; // Replace with secure key
    
    private Key getSigningKey() {
        return Keys.hmacShaKeyFor(SECRET_KEY.getBytes());
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder().setSigningKey(getSigningKey()).build().parseClaimsJws(token).getBody();
    }

    public String generateToken(String username, Map<String, Object> extraClaims) {
        return Jwts.builder()
                .setClaims(extraClaims)
                .setSubject(username)
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 10)) // 10 hours
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public boolean validateToken(String token, String username) {
        final String extractedUsername = extractUsername(token);
        return (extractedUsername.equals(username) && !isTokenExpired(token));
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
}
```

### 3. **JWT Authentication Filter**:
```java
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtUtil jwtUtil, UserDetailsService userDetailsService) {
        this.jwtUtil = jwtUtil;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");
        final String jwt;
        final String username;

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        jwt = authHeader.substring(7);
        username = jwtUtil.extractUsername(jwt);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
            if (jwtUtil.validateToken(jwt, userDetails.getUsername())) {
                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

### 4. **Security Configuration**:
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter, UserDetailsService userDetailsService) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeHttpRequests()
            .requestMatchers("/authenticate").permitAll()
            .requestMatchers("/getallusers").hasAuthority("ROLE_ADMIN")
            .anyRequest().authenticated()
            .and()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### 5. **Controller for Authentication**:
```java
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@RestController
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    public AuthController(AuthenticationManager authenticationManager, JwtUtil jwtUtil, UserDetailsService userDetailsService) {
        this.authenticationManager = authenticationManager;
        this.jwtUtil = jwtUtil;
        this.userDetailsService = userDetailsService;
    }

    @PostMapping("/authenticate")
    public String authenticate(@RequestBody AuthRequest authRequest) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(authRequest.getEmail(), authRequest.getPassword())
        );

        UserDetails user = userDetailsService.loadUserByUsername(authRequest.getEmail());

        // Add roles to JWT claims
        Map<String, Object> claims = new HashMap<>();
        claims.put("role", user.getAuthorities());

        return jwtUtil.generateToken(user.getUsername(), claims);
    }

    @GetMapping("/getallusers")
    public String getAllUsers() {
        // Only admins will reach this point
        return "Admin access granted. Here are all the users.";
    }
}
```

### 6. **AuthRequest DTO**:
```java
public class AuthRequest {
    private String email;
    private String password;

    // getters and setters
}
```

This implementation:
1. Authenticates the user with their email and bcrypt password using `/authenticate`.
2. Generates a JWT token with the user’s role in the payload.
3. Protects the `/getallusers` route so that only users with `ROLE_ADMIN` can access it.
import io.jsonwebtoken.Claims;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Arrays;
import java.util.Optional;

public class JwtFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private MyUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        // Retrieve JWT from cookies
        String token = null;
        String username = null;

        // Get the cookies from the request
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            // Look for a cookie named "jwt"
            Optional<Cookie> jwtCookie = Arrays.stream(cookies)
                    .filter(cookie -> "jwt".equals(cookie.getName()))
                    .findFirst();

            // If found, extract the token
            if (jwtCookie.isPresent()) {
                token = jwtCookie.get().getValue();
            }
        }

        // Process the token if it's available
        if (token != null) {
            try {
                username = jwtUtil.extractUsername(token);
            } catch (Exception e) {
                // Handle invalid token scenario, maybe log it
            }
        }

        // If username is present and not yet authenticated, validate the token
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            if (jwtUtil.validateToken(token, username)) {
                // Create authentication and set it in the security context
                UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }

        // Proceed with the filter chain
        filterChain.doFilter(request, response);
    }
}




@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {

    String token = null;
    String username = null;

    // Get the JWT from the cookie
    Cookie[] cookies = request.getCookies();
    if (cookies != null) {
        Optional<Cookie> jwtCookie = Arrays.stream(cookies)
                .filter(cookie -> "jwt".equals(cookie.getName()))
                .findFirst();

        if (jwtCookie.isPresent()) {
            token = jwtCookie.get().getValue();
        }
    }

    if (token != null) {
        try {
            username = jwtUtil.extractUsername(token);

            // Extract the role array from the JWT claims
            Claims claims = jwtUtil.extractAllClaims(token);
            List<Map<String, String>> roles = (List<Map<String, String>>) claims.get("role");

            // Convert the roles into GrantedAuthority
            List<GrantedAuthority> authorities = roles.stream()
                    .map(role -> new SimpleGrantedAuthority(role.get("authority")))
                    .collect(Collectors.toList());

            // Set the authentication if the token is valid
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                if (jwtUtil.validateToken(token, username)) {
                    UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                            userDetails, null, authorities);
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        } catch (Exception e) {
            // Handle invalid token scenario
        }
    }

    filterChain.doFilter(request, response);
}





To retrieve the user ID from a JWT stored in a cookie in Spring Boot, you can write a helper function that extracts the cookie, decodes the JWT, and retrieves the user ID. Here's a step-by-step guide:

### Steps:
1. **Extract the cookie from the request**.
2. **Decode the JWT** stored in the cookie.
3. **Retrieve the user ID** from the decoded JWT claims.

Here’s how you can write a helper function to do this:

### 1. **JWT Utility for Parsing Token**:
First, ensure you have a utility function that can extract the user ID from the JWT.

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;

public class JwtUtil {

    private String secret = "your-secret-key"; // Replace with your actual secret key

    public String extractUserId(String token) {
        Claims claims = extractAllClaims(token);
        return claims.get("userId", String.class); // Assuming you stored the userId in the token
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .setSigningKey(secret)
                .parseClaimsJws(token)
                .getBody();
    }
}
```

### 2. **Helper Function to Retrieve User ID from Cookie**:
Now, let’s write a helper function that extracts the JWT from the cookie and retrieves the user ID.

```java
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import java.util.Arrays;

public class CookieUtil {

    private JwtUtil jwtUtil;

    public CookieUtil(JwtUtil jwtUtil) {
        this.jwtUtil = jwtUtil;
    }

    public String getUserIdFromCookie(HttpServletRequest request) {
        // Extract the JWT from the cookie
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            // Find the cookie that stores the JWT
            Cookie jwtCookie = Arrays.stream(cookies)
                    .filter(cookie -> "your-cookie-name".equals(cookie.getName())) // Replace with your actual cookie name
                    .findFirst()
                    .orElse(null);

            if (jwtCookie != null) {
                // Extract user ID from JWT
                String jwtToken = jwtCookie.getValue();
                return jwtUtil.extractUserId(jwtToken);
            }
        }

        // Return null if no JWT cookie found or user ID cannot be extracted
        return null;
    }
}
```

### 3. **Using the Helper in the Update Route**:
You can use this helper function in your controller when the update route is called.

```java
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;

@RestController
@RequestMapping("/api/user")
public class UserController {

    private final CookieUtil cookieUtil;

    public UserController(CookieUtil cookieUtil) {
        this.cookieUtil = cookieUtil;
    }

    @PutMapping("/update")
    public String updateUser(HttpServletRequest request) {
        // Retrieve user ID from cookie
        String userId = cookieUtil.getUserIdFromCookie(request);

        if (userId == null) {
            return "Unauthorized request: User ID not found";
        }

        // Proceed with the update logic using the userId from the cookie
        return "User details updated for ID: " + userId;
    }
}
```

### Explanation:
1. **`JwtUtil`**: This utility extracts the user ID from the JWT.
2. **`CookieUtil`**: This utility retrieves the JWT from the cookie in the request and then uses `JwtUtil` to get the user ID from the token.
3. **Controller**: In your controller's update route, you can now directly use the `getUserIdFromCookie` method to securely get the logged-in user’s ID and proceed with the update logic.

### Key Considerations:
- Make sure the JWT secret (`your-secret-key`) in the `JwtUtil` class matches the one you used when generating the token.
- Replace `"your-cookie-name"` with the actual name of the cookie where you store the JWT.

