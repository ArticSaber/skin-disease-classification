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



