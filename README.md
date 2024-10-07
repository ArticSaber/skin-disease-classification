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
- 



In the context of your team-based collaboration project using WebSockets, the client (typically a web app using JavaScript) establishes a connection with the server and communicates with it in real-time by sending and receiving messages. This allows for immediate updates without the need to refresh or poll the server for changes.

### 1. **Setting Up WebSocket on the Client Side (Frontend)**

For this, you would typically use something like **SockJS** or **Stomp.js** (especially in Spring WebSocket applications) to manage WebSocket connections and messaging.

Here's how you can use functions (fn) in JavaScript to handle WebSocket interactions for your todo app.

#### Step 1: **Connect to the WebSocket**
You'll need to establish a connection to the WebSocket server using **SockJS** and **Stomp.js**.

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/sockjs-client/1.5.1/sockjs.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/stomp.js/2.3.3/stomp.min.js"></script>
<script>
    var stompClient = null;

    // Connect to WebSocket
    function connect() {
        var socket = new SockJS('/ws');
        stompClient = Stomp.over(socket);
        stompClient.connect({}, function (frame) {
            console.log('Connected: ' + frame);

            // Subscribe to personal todos
            stompClient.subscribe('/topic/personalTodoUpdates', function (message) {
                displayTodo(JSON.parse(message.body)); // Handle personal todo updates
            });

            // Subscribe to team todos
            stompClient.subscribe('/topic/teamTodoUpdates', function (message) {
                displayTeamTodo(JSON.parse(message.body)); // Handle team todo updates
            });
        });
    }

    // Disconnect the WebSocket
    function disconnect() {
        if (stompClient !== null) {
            stompClient.disconnect();
        }
        console.log("Disconnected");
    }
</script>
```

#### Step 2: **Send Messages to the WebSocket (Adding/Updating Todos)**

When a user adds or updates a todo, you'll send a message to the server using WebSocket.

```html
<script>
    // Send new personal todo
    function sendPersonalTodo(todo) {
        stompClient.send("/app/personalTodo/add", {}, JSON.stringify(todo));
    }

    // Send new team todo
    function sendTeamTodo(teamTodo) {
        stompClient.send("/app/teamTodo/add", {}, JSON.stringify(teamTodo));
    }

    // Update personal todo
    function updatePersonalTodo(todo) {
        stompClient.send("/app/personalTodo/update", {}, JSON.stringify(todo));
    }

    // Update team todo
    function updateTeamTodo(teamTodo) {
        stompClient.send("/app/teamTodo/update", {}, JSON.stringify(teamTodo));
    }
</script>
```

#### Step 3: **Handle Messages Received from the WebSocket**

When the server sends a message (like a new or updated todo), your frontend will receive the message, and you can use a function (`fn`) to handle it.

```html
<script>
    // Display personal todo when a new one is added/updated
    function displayTodo(todo) {
        // Logic to add todo to the DOM
        console.log("New Personal Todo:", todo);
        // e.g., Append todo to a list in the HTML
    }

    // Display team todo when a new one is added/updated
    function displayTeamTodo(teamTodo) {
        // Logic to add team todo to the DOM
        console.log("New Team Todo:", teamTodo);
        // e.g., Append team todo to a list in the HTML
    }
</script>
```

### 2. **How It Works:**

#### **Connecting to WebSocket:**
- The `connect()` function establishes a WebSocket connection to the server using SockJS and subscribes to specific topics (`/topic/personalTodoUpdates` and `/topic/teamTodoUpdates`).
- Whenever a new todo is added or an existing todo is updated, the server broadcasts the message to the appropriate topic, and the subscribed clients receive the update in real-time.

#### **Sending Data via WebSocket:**
- Functions like `sendPersonalTodo()` and `sendTeamTodo()` send new todos to the server using WebSocket when the user adds them.
- Similarly, the `updatePersonalTodo()` and `updateTeamTodo()` functions send updates when a todo is modified.

#### **Handling Received Data:**
- When the server sends updates (new/modified todos) to the client, they are handled by the `displayTodo()` and `displayTeamTodo()` functions.
- These functions would typically update the DOM to reflect the new or updated todo in the UI without a page refresh.

### 3. **Backend Spring WebSocket Controller Integration**

On the backend (as I mentioned in the previous response), you'll have a WebSocket controller that handles the messages sent by the client and processes them (adding/updating todos), and then broadcasts the results back to subscribed clients.

```java
@Controller
public class TodoWebSocketController {

    // Add new personal todo
    @MessageMapping("/personalTodo/add")
    @SendTo("/topic/personalTodoUpdates")
    public TodoItem addPersonalTodo(TodoItem todoItem) {
        // Save the new todo to the database
        return todoItem; // Broadcast the new todo to all subscribed clients
    }

    // Add new team todo
    @MessageMapping("/teamTodo/add")
    @SendTo("/topic/teamTodoUpdates")
    public TeamTodoItem addTeamTodo(TeamTodoItem todoItem) {
        // Save the new team todo to the database
        return todoItem; // Broadcast the new team todo to all subscribed clients
    }

    // Update personal todo
    @MessageMapping("/personalTodo/update")
    @SendTo("/topic/personalTodoUpdates")
    public TodoItem updatePersonalTodo(TodoItem todoItem) {
        // Update the todo in the database
        return todoItem; // Broadcast the updated todo to all subscribed clients
    }

    // Update team todo
    @MessageMapping("/teamTodo/update")
    @SendTo("/topic/teamTodoUpdates")
    public TeamTodoItem updateTeamTodo(TeamTodoItem todoItem) {
        // Update the team todo in the database
        return todoItem; // Broadcast the updated team todo to all subscribed clients
    }
}
```

### Summary:
- WebSockets allow real-time communication for both adding and updating todos.
- The client uses functions (`fn`) to send and receive WebSocket messages, updating the UI without page reloads.
- The server (using Spring WebSocket) broadcasts updates to all subscribed clients, ensuring real-time synchronization.

This setup ensures that todos—whether personal or team-based—are immediately visible and editable in real-time by all relevant users.















Sure! Below is a complete Spring Boot WebSocket application setup for a todo list with two WebSocket endpoints: one for personal todos and one for team-based todos. This will cover the basic configuration, the WebSocket controller, and a basic service layer for handling the business logic. You can expand this setup as needed.

1. Project Structure:

Here’s what the project structure will look like:

src
└── main
    ├── java
    │   └── com
    │       └── example
    │           └── websocketdemo
    │               ├── config
    │               │   └── WebSocketConfig.java
    │               ├── controller
    │               │   └── TodoWebSocketController.java
    │               ├── model
    │               │   ├── TodoItem.java
    │               │   └── TeamTodoItem.java
    │               ├── service
    │               │   └── TodoService.java
    │               └── WebSocketDemoApplication.java
    └── resources
        └── application.properties

2. Dependencies:

In your pom.xml file, add the following dependencies:

<dependencies>
    <!-- Spring Boot Starter for WebSocket -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>

    <!-- Spring Boot Starter for Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Optional: Spring Boot DevTools for LiveReload -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
</dependencies>

3. WebSocket Configuration:

Create a configuration file WebSocketConfig.java in the config package:

package com.example.websocketdemo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic"); // Enable simple broker with /topic prefix
        config.setApplicationDestinationPrefixes("/app"); // Application destination prefix
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS(); // WebSocket endpoint for SockJS
    }
}

4. Model Classes:

Create a model package and add the following two classes for todo items:

TodoItem.java (For Personal Todos)

package com.example.websocketdemo.model;

public class TodoItem {
    private String id;
    private String title;
    private String description;
    private boolean completed;

    // Getters and Setters
    public String getId() {
        return id;
    }
    public void setId(String id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }
    public void setTitle(String title) {
        this.title = title;
    }

    public String getDescription() {
        return description;
    }
    public void setDescription(String description) {
        this.description = description;
    }

    public boolean isCompleted() {
        return completed;
    }
    public void setCompleted(boolean completed) {
        this.completed = completed;
    }
}

TeamTodoItem.java (For Team-Based Todos)

package com.example.websocketdemo.model;

public class TeamTodoItem {
    private String id;
    private String teamId;
    private String title;
    private String description;
    private boolean completed;

    // Getters and Setters
    public String getId() {
        return id;
    }
    public void setId(String id) {
        this.id = id;
    }

    public String getTeamId() {
        return teamId;
    }
    public void setTeamId(String teamId) {
        this.teamId = teamId;
    }

    public String getTitle() {
        return title;
    }
    public void setTitle(String title) {
        this.title = title;
    }

    public String getDescription() {
        return description;
    }
    public void setDescription(String description) {
        this.description = description;
    }

    public boolean isCompleted() {
        return completed;
    }
    public void setCompleted(boolean completed) {
        this.completed = completed;
    }
}

5. Service Layer:

Create a service package and a simple TodoService class to handle adding/updating todos:

package com.example.websocketdemo.service;

import com.example.websocketdemo.model.TodoItem;
import com.example.websocketdemo.model.TeamTodoItem;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

@Service
public class TodoService {

    private Map<String, TodoItem> personalTodos = new HashMap<>();
    private Map<String, TeamTodoItem> teamTodos = new HashMap<>();

    public TodoItem addPersonalTodo(TodoItem todo) {
        personalTodos.put(todo.getId(), todo);
        return todo;
    }

    public TeamTodoItem addTeamTodo(TeamTodoItem todo) {
        teamTodos.put(todo.getId(), todo);
        return todo;
    }

    public TodoItem updatePersonalTodo(TodoItem todo) {
        personalTodos.put(todo.getId(), todo);
        return todo;
    }

    public TeamTodoItem updateTeamTodo(TeamTodoItem todo) {
        teamTodos.put(todo.getId(), todo);
        return todo;
    }
}

6. WebSocket Controller:

Create a controller package and add the following TodoWebSocketController.java:

package com.example.websocketdemo.controller;

import com.example.websocketdemo.model.TodoItem;
import com.example.websocketdemo.model.TeamTodoItem;
import com.example.websocketdemo.service.TodoService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class TodoWebSocketController {

    @Autowired
    private TodoService todoService;

    // For adding personal todos
    @MessageMapping("/personalTodo/add")
    @SendTo("/topic/personalTodoUpdates")
    public TodoItem addPersonalTodo(TodoItem todoItem) {
        return todoService.addPersonalTodo(todoItem);
    }

    // For adding team todos
    @MessageMapping("/teamTodo/add")
    @SendTo("/topic/teamTodoUpdates")
    public TeamTodoItem addTeamTodo(TeamTodoItem todoItem) {
        return todoService.addTeamTodo(todoItem);
    }

    // For updating personal todos
    @MessageMapping("/personalTodo/update")
    @SendTo("/topic/personalTodoUpdates")
    public TodoItem updatePersonalTodo(TodoItem todoItem) {
        return todoService.updatePersonalTodo(todoItem);
    }

    // For updating team todos
    @MessageMapping("/teamTodo/update")
    @SendTo("/topic/teamTodoUpdates")
    public TeamTodoItem updateTeamTodo(TeamTodoItem todoItem) {
        return todoService.updateTeamTodo(todoItem);
    }
}

7. Spring Boot Main Class:

Create a main class to run the Spring Boot application:

package com.example.websocketdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class WebSocketDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebSocketDemoApplication.class, args);
    }
}

8. Frontend Integration:

You can connect to this WebSocket backend using JavaScript and the Stomp.js library, similar to the client-side code provided in the previous response.

9. Run the Application:

Run the Spring Boot application.

Ensure your WebSocket client is connected to ws://localhost:8080/ws.


This setup provides a basic WebSocket-based todo management system for both personal and team-based todos.


<dependencies>
    <!-- Spring Boot Starter for WebSocket -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>

    <!-- Spring Boot Starter for Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Optional: Spring Boot DevTools for LiveReload -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
</dependencies>






---------------
Alright, let’s simplify it by using a single table for todos, which will handle both personal and team-based todos based on whether a teamId is present or not.

Updated Project Structure

src
└── main
    ├── java
    │   └── com
    │       └── example
    │           └── websocketdemo
    │               ├── config
    │               │   └── WebSocketConfig.java
    │               ├── controller
    │               │   └── TodoWebSocketController.java
    │               ├── model
    │               │   └── TodoItem.java
    │               ├── repository
    │               │   └── TodoRepository.java
    │               ├── service
    │               │   └── TodoService.java
    │               └── WebSocketDemoApplication.java
    └── resources
        └── application.properties

1. Database Configuration in application.properties

Make sure your Oracle database is set up and configured correctly:

# Oracle Database Configuration
spring.datasource.url=jdbc:oracle:thin:@//localhost:1521/ORCLPDB1
spring.datasource.username=YOUR_DB_USERNAME
spring.datasource.password=YOUR_DB_PASSWORD
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver

# JPA Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.Oracle12cDialect

2. Model Class

Create a TodoItem model class that represents both personal and team todos based on the teamId attribute.

TodoItem.java

package com.example.websocketdemo.model;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Table(name = "TODO_ITEM")
public class TodoItem {
    @Id
    private String id;
    private String userId; // User ID of the owner
    private String teamId; // If null, it's a personal todo; if not null, it's a team todo
    private String title;
    private String description;
    private boolean completed;

    // Getters and Setters
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }

    public String getTeamId() {
        return teamId;
    }

    public void setTeamId(String teamId) {
        this.teamId = teamId;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public boolean isCompleted() {
        return completed;
    }

    public void setCompleted(boolean completed) {
        this.completed = completed;
    }
}

3. Repository Interface

Create a repository interface that extends JpaRepository for accessing the Oracle database.

TodoRepository.java

package com.example.websocketdemo.repository;

import com.example.websocketdemo.model.TodoItem;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface TodoRepository extends JpaRepository<TodoItem, String> {
    List<TodoItem> findByUserId(String userId);
    List<TodoItem> findByTeamId(String teamId);
}

4. Service Layer

Create a TodoService class to handle business logic.

TodoService.java

package com.example.websocketdemo.service;

import com.example.websocketdemo.model.TodoItem;
import com.example.websocketdemo.repository.TodoRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class TodoService {

    @Autowired
    private TodoRepository todoRepository;

    public TodoItem addOrUpdateTodo(TodoItem todoItem) {
        return todoRepository.save(todoItem);
    }

    public List<TodoItem> getUserTodos(String userId) {
        return todoRepository.findByUserId(userId);
    }

    public List<TodoItem> getTeamTodos(String teamId) {
        return todoRepository.findByTeamId(teamId);
    }
}

5. WebSocket Configuration

Configure the WebSocket using WebSocketConfig.java:

WebSocketConfig.java

package com.example.websocketdemo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS();
    }
}

6. WebSocket Controller

Create a TodoWebSocketController to handle WebSocket messages for todos.

TodoWebSocketController.java

package com.example.websocketdemo.controller;

import com.example.websocketdemo.model.TodoItem;
import com.example.websocketdemo.service.TodoService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

import java.util.List;

@Controller
public class TodoWebSocketController {

    @Autowired
    private TodoService todoService;

    // Add or update a todo item
    @MessageMapping("/todo/addOrUpdate")
    @SendTo("/topic/todoUpdates")
    public TodoItem addOrUpdateTodo(TodoItem todoItem) {
        return todoService.addOrUpdateTodo(todoItem);
    }

    // Get all personal todos for a specific user
    @MessageMapping("/todo/user")
    @SendTo("/topic/userTodos")
    public List<TodoItem> getUserTodos(String userId) {
        return todoService.getUserTodos(userId);
    }

    // Get all team todos for a specific team
    @MessageMapping("/todo/team")
    @SendTo("/topic/teamTodos")
    public List<TodoItem> getTeamTodos(String teamId) {
        return todoService.getTeamTodos(teamId);
    }
}

7. Main Application

Create a main class to run the Spring Boot application:

WebSocketDemoApplication.java

package com.example.websocketdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class WebSocketDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebSocketDemoApplication.class, args);
    }
}

8. Database Table Setup

Create a TODO_ITEM table in your Oracle database with the following schema:

CREATE TABLE TODO_ITEM (
    ID VARCHAR2(50) PRIMARY KEY,
    USER_ID VARCHAR2(50),
    TEAM_ID VARCHAR2(50),
    TITLE VARCHAR2(100),
    DESCRIPTION VARCHAR2(255),
    COMPLETED NUMBER(1)
);

Summary

Single WebSocket endpoint (/ws) to handle personal and team-based todos.

The table structure determines if a todo is personal (teamId is null) or team-based (teamId is set).

Messages are sent to topics (/topic/todoUpdates, /topic/userTodos, and /topic/teamTodos) for different updates.


To handle personal and team todos separately in a real-time collaborative environment, you need to have a structured way of sending, receiving, and reflecting updates based on the context (whether it’s a personal or team todo). This involves configuring WebSocket endpoints correctly and making use of specific channels (topics) for personal and team-based updates.

WebSocket Design for Personal and Team Todos

1. Separate WebSocket Topics for Personal and Team Todos:

For personal todos, each user can have their own unique topic (e.g., /topic/personal/{userId}).

For team todos, use a topic based on the teamId (e.g., /topic/team/{teamId}).



2. Filtering and Broadcasting Updates:

When a todo is added or updated, identify whether it’s a personal or team todo based on the teamId.

Personal Todo: If teamId is null or empty, broadcast the update to the user-specific topic.

Team Todo: If teamId is present, broadcast to the teamId topic.



3. Client Subscription Strategy:

Each client (user) should subscribe to their own personal topic and the topics for each team they are a member of.

When fetching todos from the database, clients should request personal todos (userId) and team todos (teamId) separately.

When a todo is updated or added, the server should push these changes to the correct topic based on whether the todo is personal or team-based.




Implementation Plan

1. Separate WebSocket topics for personal todos and team todos.


2. Listeners and WebSocket handlers to broadcast changes to the appropriate topics.


3. Fetching and reflecting changes based on client subscriptions.



Code Implementation in Spring Boot

Here’s how you can structure the backend WebSocket configuration, service logic, and WebSocket controller for managing personal and team todos.

1. Configure WebSocket Endpoints

Set up WebSocket configurations for the client to connect and subscribe to the relevant topics.

package com.example.websocketdemo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");  // Prefix for broadcast messages
        config.setApplicationDestinationPrefixes("/app");  // Prefix for client-to-server messages
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-todo").setAllowedOrigins("*").withSockJS();
    }
}

/topic is used for broadcasting updates (e.g., /topic/personal/{userId} and /topic/team/{teamId}).

/app is used for client-to-server communication.


2. Create the WebSocket Controller

Create a controller that maps and handles messages for adding and updating todos.

package com.example.websocketdemo.controller;

import com.example.websocketdemo.model.TodoItem;
import com.example.websocketdemo.service.TodoService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class TodoWebSocketController {

    @Autowired
    private TodoService todoService;

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    // Add a new todo item and broadcast to the respective topic
    @MessageMapping("/todo/create")
    public void createTodo(TodoItem todoItem) {
        TodoItem createdTodo = todoService.createTodo(todoItem);

        if (createdTodo.getTeamId() != null) {
            // Broadcast to team topic
            messagingTemplate.convertAndSend("/topic/team/" + createdTodo.getTeamId(), createdTodo);
        } else {
            // Broadcast to personal user topic
            messagingTemplate.convertAndSend("/topic/personal/" + createdTodo.getUserId(), createdTodo);
        }
    }

    // Patch an existing todo and broadcast updates to the respective topic
    @MessageMapping("/todo/update")
    public void updateTodo(TodoItem todoItem) {
        TodoItem updatedTodo = todoService.updateTodoFields(todoItem.getId(), todoItem);

        if (updatedTodo.getTeamId() != null) {
            // Broadcast to team topic
            messagingTemplate.convertAndSend("/topic/team/" + updatedTodo.getTeamId(), updatedTodo);
        } else {
            // Broadcast to personal user topic
            messagingTemplate.convertAndSend("/topic/personal/" + updatedTodo.getUserId(), updatedTodo);
        }
    }
}

3. Service Layer for Creating and Updating Todos

The service layer should handle creating new todos and partially updating existing todos while validating the data.

package com.example.websocketdemo.service;

import com.example.websocketdemo.model.TodoItem;
import com.example.websocketdemo.repository.CredsRepository;
import com.example.websocketdemo.repository.TeamMemberRepository;
import com.example.websocketdemo.repository.TodoRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class TodoService {

    @Autowired
    private TodoRepository todoRepository;

    @Autowired
    private CredsRepository credsRepository;

    @Autowired
    private TeamMemberRepository teamMemberRepository;

    // Create a new TodoItem
    public TodoItem createTodo(TodoItem todoItem) {
        if (validateTodoItem(todoItem)) {
            return todoRepository.save(todoItem);
        } else {
            throw new RuntimeException("Invalid Todo Data: User is not authorized.");
        }
    }

    // Patch (partial update) for TodoItem
    public TodoItem updateTodoFields(String todoId, TodoItem patchData) {
        TodoItem existingTodo = todoRepository.findById(todoId)
                .orElseThrow(() -> new RuntimeException("Todo not found"));

        // Apply partial updates
        if (patchData.getTitle() != null) {
            existingTodo.setTitle(patchData.getTitle());
        }
        if (patchData.getDescription() != null) {
            existingTodo.setDescription(patchData.getDescription());
        }
        if (patchData.getCompleted() != null) {
            existingTodo.setCompleted(patchData.getCompleted());
        }

        // Save updated entity
        return todoRepository.save(existingTodo);
    }

    private boolean validateTodoItem(TodoItem todoItem) {
        if (!credsRepository.existsByUserId(todoItem.getUserId())) {
            return false;
        }
        if (todoItem.getTeamId() != null && !todoItem.getTeamId().isEmpty()) {
            return teamMemberRepository.existsByTeamIdAndUserId(todoItem.getTeamId(), todoItem.getUserId());
        }
        return true;
    }
}

4. Client Subscription Strategy

Personal Todos: Clients should subscribe to /topic/personal/{userId} to receive updates for their personal todos.

Team Todos: Clients should subscribe to /topic/team/{teamId} for each team they belong to.


5. Fetching Todos in the Client

1. Personal Todos: Make a separate API call to get all personal todos based on the userId.


2. Team Todos: Make a separate API call for each teamId the user belongs to.



By setting up separate WebSocket topics for personal and team todos and using appropriate endpoints, you can easily differentiate and broadcast updates based on the type of todo, ensuring that the UI is updated in real-time for all members involved.

Let me know if this approach fits your requirements or if you need additional modifications!










_-__-_-_-_-_-_-__-_-_-_-_-_-_-__-_-_-_-_-_-_-_-_-_-__-_-_-_-_-_-_-_-__-_-_-_-_-_






Here’s a complete implementation for Spring Boot WebSocket configuration using header-based JWT authentication for WebSocket connections, and cookie-based JWT authentication for standard HTTP routes. The provided code will include the following components:

1. **JWT Utility Class (`JwtUtil`)**: To create and parse JWT tokens.
2. **Custom `JwtHandshakeInterceptor`**: For extracting JWT from headers during WebSocket handshakes.
3. **WebSocket Configuration (`WebSocketConfig`)**: To configure the WebSocket endpoint with the interceptor.
4. **Security Configuration (`SecurityConfig`)**: To secure HTTP and WebSocket routes.
5. **Controller and Service Examples**: To demonstrate how to implement authentication checks.
6. **Simple WebSocket Controller**: For real-time todo updates.

### 1. **`JwtUtil` Class**

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.function.Function;

public class JwtUtil {

    private static final String SECRET_KEY = "my_secret_key";

    public String generateToken(String username, List<String> roles) {
        return Jwts.builder()
                .setSubject(username)
                .claim("roles", roles)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60 * 10))  // 10 hours expiry
                .signWith(SignatureAlgorithm.HS256, SECRET_KEY)
                .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser().setSigningKey(SECRET_KEY).parseClaimsJws(token).getBody();
    }

    public Boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    public Boolean validateToken(String token, String username) {
        final String extractedUsername = extractUsername(token);
        return (extractedUsername.equals(username) && !isTokenExpired(token));
    }
}
```

### 2. **`JwtHandshakeInterceptor` Class**

```java
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

import java.util.Map;

public class JwtHandshakeInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) {
        String jwtToken = request.getHeaders().getFirst("bearertoken");

        if (jwtToken != null && !jwtToken.isEmpty()) {
            JwtUtil jwtUtil = new JwtUtil();
            String username = jwtUtil.extractUsername(jwtToken);

            if (username != null && jwtUtil.validateToken(jwtToken, username)) {
                attributes.put("username", username);
                return true;
            }
        }
        return false;  // Reject the WebSocket connection if JWT is invalid or missing
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                               WebSocketHandler wsHandler, Exception exception) {
    }
}
```

### 3. **`WebSocketConfig` Class**

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-todo")
                .addInterceptors(new JwtHandshakeInterceptor())  // Register JWT Interceptor
                .setAllowedOrigins("http://localhost:3000")  // Frontend origin
                .withSockJS();  // Enable SockJS fallback option
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");  // Broker destination prefix
        config.setApplicationDestinationPrefixes("/app");  // Application destination prefix
    }
}
```

### 4. **`SecurityConfig` Class**

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/ws-todo/**").permitAll()  // Permit WebSocket handshake
                .antMatchers(HttpMethod.GET, "/api/admin/**").hasAuthority("ADMIN")
                .antMatchers(HttpMethod.GET, "/api/**").authenticated()
                .anyRequest().permitAll()
            .and()
            .csrf().disable()
            .headers().frameOptions().disable();
    }
}
```

### 5. **Sample WebSocket Controller**

```java
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class TodoWebSocketController {

    @MessageMapping("/addTodo")  // Listen to "/app/addTodo"
    @SendTo("/topic/todos")  // Send updates to subscribers of "/topic/todos"
    public String addTodo(String todo) {
        return "New Todo Added: " + todo;
    }

    @MessageMapping("/updateTodo")
    @SendTo("/topic/todos")
    public String updateTodo(String todo) {
        return "Todo Updated: " + todo;
    }
}
```

### 6. **Testing the WebSocket Using `SockJS` (Frontend JavaScript Client)**

```javascript
// JavaScript client for testing WebSocket connection
const socket = new SockJS('http://localhost:8080/ws-todo');
const stompClient = Stomp.over(socket);

// Connect with JWT token in header
stompClient.connect({ bearertoken: 'YOUR_JWT_TOKEN' }, (frame) => {
    console.log('Connected: ' + frame);

    // Subscribe to "/topic/todos" to receive updates
    stompClient.subscribe('/topic/todos', (message) => {
        console.log('Received message: ' + message.body);
    });

    // Send a test message
    stompClient.send('/app/addTodo', {}, 'My new todo item');
});
```

### Explanation and Setup
1. **Backend Components**:
   - `JwtUtil` handles JWT creation and validation.
   - `JwtHandshakeInterceptor` processes the `bearertoken` from WebSocket headers.
   - `WebSocketConfig` sets up the WebSocket endpoint and registers the interceptor.
   - `SecurityConfig` secures the WebSocket route and HTTP API routes.
   - `TodoWebSocketController` handles incoming WebSocket messages and broadcasts them.

2. **Frontend/WebSocket Testing**:
   - Use the JavaScript client to test WebSocket connections.
   - Connect to the `ws-todo` endpoint with a valid JWT in the headers.

3. **Database Integration**: The example assumes a basic setup for JWT handling. For a complete project, integrate with repositories, services, and database entities to handle the todos.

Let me know if this configuration fits your requirements or if you need adjustments for any specific features!













_-_-_-_&#*÷*÷;#&×*<#>[+[+*×,₹;%&₹[(÷;@>+*₹[×**+₹>#&*×*@&@*#
Certainly! If you want to handle WebSocket authentication using a cookie instead of a header, you can modify your JwtHandshakeInterceptor to extract the JWT from the cookie. This approach will work if the JWT is stored as a cookie named, for example, auth-token. Below is a complete implementation of a JwtHandshakeInterceptor that extracts the JWT from the cookies:

JwtHandshakeInterceptor Using Cookies

import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

import javax.servlet.http.Cookie;
import java.util.Map;

public class JwtHandshakeInterceptor implements HandshakeInterceptor {

    private final JwtUtil jwtUtil;

    public JwtHandshakeInterceptor(JwtUtil jwtUtil) {
        this.jwtUtil = jwtUtil;
    }

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) {
        // Extract JWT token from cookies
        String jwtToken = getJwtFromCookies(request);

        if (jwtToken != null) {
            if (jwtUtil.validateToken(jwtToken)) {
                String username = jwtUtil.extractUsername(jwtToken);
                attributes.put("username", username);  // Store the username as a WebSocket session attribute
                return true;
            }
        }
        return false;  // Reject handshake if JWT is invalid or missing
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                               WebSocketHandler wsHandler, Exception exception) {
    }

    private String getJwtFromCookies(ServerHttpRequest request) {
        if (request.getHeaders().containsKey("Cookie")) {
            String cookieHeader = request.getHeaders().getFirst("Cookie");
            if (cookieHeader != null) {
                for (String cookie : cookieHeader.split(";")) {
                    String[] cookieParts = cookie.split("=");
                    if (cookieParts.length == 2) {
                        String cookieName = cookieParts[0].trim();
                        String cookieValue = cookieParts[1].trim();
                        if ("auth-token".equals(cookieName)) {  // Replace with your cookie name
                            return cookieValue;
                        }
                    }
                }
            }
        }
        return null;
    }
}

Key Points of the Implementation:

1. Extracting JWT from Cookies:

The getJwtFromCookies method extracts the auth-token cookie value from the incoming WebSocket request.

You need to specify the name of the cookie where the JWT is stored (auth-token in this example). Modify this as per your application's cookie name.



2. JWT Validation:

If a valid JWT is found in the cookie, the interceptor extracts the username and attaches it to the WebSocket session attributes.

If the JWT is invalid or missing, the handshake is rejected.



3. Storing Attributes for WebSocket Session:

The attributes.put("username", username); line stores the username in the WebSocket session, which can be retrieved in the WebSocket controller.




WebSocket Configuration with the Cookie-Based Interceptor

Now, update your WebSocketConfig to use the custom JwtHandshakeInterceptor:

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-todo")
                .addInterceptors(new JwtHandshakeInterceptor(jwtUtil))  // Register the custom cookie-based handshake interceptor
                .setAllowedOrigins("http://localhost:3000")  // Set your frontend origin
                .withSockJS();  // Enable SockJS fallback
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");  // Broker destination prefix for subscriptions
        config.setApplicationDestinationPrefixes("/app");  // Prefix for application-specific messages
    }
}

Using the Cookie-Based Authentication in Frontend/Testing

When testing this configuration with Postman or another frontend, ensure that:

1. Your WebSocket connection includes the auth-token cookie.


2. Set the auth-token cookie in the browser or Postman before connecting to the WebSocket.



Setting the Cookie in Postman:

1. In Postman, go to the Cookies tab.


2. Add a new cookie for localhost (or the appropriate domain).


3. Set the auth-token cookie to your JWT value:

auth-token=YOUR_JWT_TOKEN



4. Now initiate a WebSocket request using Postman.



Why Use a Cookie-Based Interceptor?

1. Simpler for Browser-Based Clients:

If your JWT is stored as a cookie (e.g., after a successful login), it is automatically included in every subsequent WebSocket connection, making the frontend code simpler.



2. Improved Security:

Depending on the browser and security configurations, cookie-based authentication can be more secure, especially when using HTTP-only and secure flags for the cookies.




Let me know if this implementation fits your requirements or if you need help configuring Postman or the frontend for testing!








[÷(@,,'cnxmmznzzmznznjsbfidneneeee

If you’re using Spring Boot, you can achieve this using JPA (Java Persistence API) to query your database. Below is a simple example to guide you through the process of creating the necessary repository and service layer to fetch the team details for a given user.

1. Entities:

Create JPA entities corresponding to your team and team_member tables.

// Team.java
@Entity
public class Team {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String teamName;

    // Getters and Setters
}

// TeamMember.java
@Entity
public class TeamMember {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;
    private Long teamId;

    // Getters and Setters
}

2. Repository Layer:

Create a TeamMemberRepository with a custom query to fetch team details using userId.

// TeamMemberRepository.java
@Repository
public interface TeamMemberRepository extends JpaRepository<TeamMember, Long> {
    
    @Query("SELECT new com.example.dto.TeamDto(t.id, t.teamName) " +
           "FROM TeamMember tm JOIN Team t ON tm.teamId = t.id " +
           "WHERE tm.userId = :userId")
    List<TeamDto> findTeamsByUserId(@Param("userId") Long userId);
}

3. DTO Class:

Create a Data Transfer Object (DTO) to represent the team information to be sent to the client.

// TeamDto.java
package com.example.dto;

public class TeamDto {
    private Long teamId;
    private String teamName;

    public TeamDto(Long teamId, String teamName) {
        this.teamId = teamId;
        this.teamName = teamName;
    }

    // Getters and Setters
}

4. Service Layer:

Create a service to fetch the team details for the given user.

// TeamService.java
@Service
public class TeamService {
    
    @Autowired
    private TeamMemberRepository teamMemberRepository;

    public List<TeamDto> getTeamsByUserId(Long userId) {
        return teamMemberRepository.findTeamsByUserId(userId);
    }
}

5. Controller Layer:

Create a REST controller to handle the request and send the response to the client.

// TeamController.java
@RestController
@RequestMapping("/api/teams")
public class TeamController {
    
    @Autowired
    private TeamService teamService;

    @GetMapping("/{userId}")
    public ResponseEntity<List<TeamDto>> getTeamsByUserId(@PathVariable Long userId) {
        List<TeamDto> teams = teamService.getTeamsByUserId(userId);
        return ResponseEntity.ok(teams);
    }
}

6. Running the Application:

Once everything is set up, run your Spring Boot application and make a GET request to the endpoint:

GET http://localhost:8080/api/teams/{userId}

Key Points:

The custom query in TeamMemberRepository uses a JOIN between TeamMember and Team entities to fetch the teamId and teamName.

The data is sent using a simple DTO (TeamDto) to keep your response format clean and separate from the database models.


This setup should provide you with the required team details for a given user ID in a structured format. Let me know if you need to expand it further!












odo993930393993030030303030303030030303003030303030030303030303003030
It looks like you’re using a custom JwtHandshakeInterceptor to extract and validate a JWT token from cookies during the WebSocket handshake. I see some structural and syntax issues in the provided code. Let’s refine it and ensure the setup works properly.

Corrected Implementation with Cookie-based JWT for WebSocket

1. WebSocketConfig: This configures the WebSocket endpoint and integrates the JwtHandshakeInterceptor.


2. JwtHandshakeInterceptor: Extracts the JWT token from cookies and validates it.


3. TodoWebSocketController: Handles incoming WebSocket messages.


4. Postman or WebSocket Client: A testing client to subscribe and send messages.



1. WebSocketConfig Implementation

This class configures the WebSocket with STOMP support and adds the JwtHandshakeInterceptor to handle JWT extraction.

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic"); // Enable a simple broker for "/topic" prefix
        config.setApplicationDestinationPrefixes("/app"); // Prefix for mapped methods
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // Register STOMP endpoint and add JWT Handshake Interceptor
        registry.addEndpoint("/ws-todo")
                .addInterceptors(new JwtHandshakeInterceptor(jwtUtil))
                .setAllowedOrigins("http://localhost:3000");  // Frontend origin
                // Uncomment if using SockJS fallback
                //.withSockJS();
    }
}

2. JwtHandshakeInterceptor Implementation

The JwtHandshakeInterceptor will extract the JWT token from the BearerToken cookie and validate it during the WebSocket handshake.

import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.server.HandshakeInterceptor;

import java.util.Map;

public class JwtHandshakeInterceptor implements HandshakeInterceptor {

    private final JwtUtil jwtUtil;

    public JwtHandshakeInterceptor(JwtUtil jwtUtil) {
        this.jwtUtil = jwtUtil;
    }

    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler wsHandler, Map<String, Object> attributes) {
        // Extract JWT token from cookies
        String jwtToken = getJwtFromCookies(request);
        if (jwtToken != null && !jwtToken.isEmpty()) {
            String username = jwtUtil.extractUsername(jwtToken);
            if (username != null && jwtUtil.validateToken(jwtToken, username)) {
                // Add username to WebSocket session attributes
                attributes.put("username", username);
                return true;
            }
        }
        return false; // Reject handshake if JWT is invalid or missing
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
                               WebSocketHandler wsHandler, Exception exception) {
        // Optional: Additional configurations or logging after the handshake
    }

    // Helper method to extract JWT from cookies
    private String getJwtFromCookies(ServerHttpRequest request) {
        if (request.getHeaders().containsKey("Cookie")) {
            String cookieHeader = request.getHeaders().getFirst("Cookie");
            if (cookieHeader != null) {
                for (String cookie : cookieHeader.split(";")) {
                    String[] cookieParts = cookie.split("=");
                    if (cookieParts.length == 2) {
                        String cookieName = cookieParts[0].trim();
                        String cookieValue = cookieParts[1].trim();
                        if ("BearerToken".equals(cookieName)) {
                            return cookieValue;
                        }
                    }
                }
            }
        }
        return null;  // Return null if cookie is not found
    }
}

3. TodoWebSocketController Implementation

This is a simple WebSocket controller that listens for messages on /app/addTodo and /app/updateTodo and sends responses to the subscribed /topic/todos topic.

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class TodoWebSocketController {

    private static final Logger LOG = LoggerFactory.getLogger(TodoWebSocketController.class);

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    @MessageMapping("/addTodo")  // Map to "/app/addTodo"
    @SendTo("/topic/todos")      // Send to "/topic/todos"
    public String addTodo(String todo) {
        LOG.info("Received new Todo: " + todo);
        return "New Todo Added: " + todo;
    }

    @MessageMapping("/updateTodo")
    @SendTo("/topic/todos")
    public String updateTodo(String todo) {
        LOG.info("Todo Updated: " + todo);
        return "Todo Updated: " + todo;
    }
}

4. Testing with Postman or a WebSocket Client

When using Postman to test the WebSocket:

1. Establish a WebSocket Connection:

URL: ws://localhost:8080/ws-todo

Headers: Include a Cookie header with the JWT:

Cookie: BearerToken=YOUR_JWT_TOKEN



2. Subscribe to a Topic: Use the SUBSCRIBE command to listen to /topic/todos:

{
  "command": "SUBSCRIBE",
  "headers": {
    "id": "sub-0",
    "destination": "/topic/todos"
  }
}


3. Send a Message: Send a SEND frame to /app/addTodo:

{
  "command": "SEND",
  "headers": {
    "destination": "/app/addTodo"
  },
  "body": "Buy groceries"
}



Summary

With the provided setup:

WebSocket connections are validated using JWTs stored in cookies.

The JwtHandshakeInterceptor ensures only valid users can establish WebSocket connections.

You can use Postman or any WebSocket client to connect, send, and receive messages by setting the correct Cookie header.


If you encounter any issues or need further modifications, let me know!



<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Demo</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1.4.0/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
</head>
<body>
    <h1>WebSocket Messages</h1>
    <ul id="messages"></ul>

    <script type="text/javascript">
        var socket = new SockJS('/ws');
        var stompClient = Stomp.over(socket);

        stompClient.connect({}, function (frame) {
            console.log('Connected: ' + frame);

            // Subscribe to /topic/messages to receive broadcasts
            stompClient.subscribe('/topic/messages', function (message) {
                var messageList = document.getElementById('messages');
                var newMessage = document.createElement('li');
                newMessage.appendChild(document.createTextNode(message.body));
                messageList.appendChild(newMessage);
            });
        });
    </script>
</body>
</html>

