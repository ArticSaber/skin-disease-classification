To secure your WebSocket subscriptions and ensure users can only subscribe to their own topics, you can extend your current setup by:

1. JWT Validation in Handshake Interceptor: If you already have a HandshakeInterceptor that validates the cookie, you can enhance it to validate the JWT token and attach the user information to the WebSocket session.


2. Topic Validation at Subscription: Ensure that during the subscription phase (after the WebSocket handshake), the server checks that the user is only subscribing to a topic that belongs to them, based on their validated JWT token.



Here’s how you can approach this:

1. Modify Handshake Interceptor to Validate JWT

You likely have a HandshakeInterceptor that looks like this (for cookie validation). You can extend it to validate the JWT and store the user’s details in the WebSocket session.

public class JwtHandshakeInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(
        ServerHttpRequest request,
        ServerHttpResponse response,
        WebSocketHandler wsHandler,
        Map<String, Object> attributes) throws Exception {
        
        // Extract JWT token from cookies or headers
        List<String> cookies = request.getHeaders().get("Cookie");
        String jwtToken = extractJwtFromCookies(cookies);
        
        // Validate JWT
        if (isTokenValid(jwtToken)) {
            // Decode JWT and get user details (e.g., userId)
            String userId = decodeJwt(jwtToken);
            
            // Store user info in session attributes
            attributes.put("userId", userId);
            return true;
        } else {
            return false; // Reject the connection if JWT is invalid
        }
    }

    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception exception) {
        // Optional: additional logic after the handshake
    }
}

2. Validate Topic Subscription

When a user tries to subscribe to a topic, the server should validate that the user’s userId (stored in the WebSocket session) matches the topic they are subscribing to.

Assuming you are using Spring WebSockets, you can use a ChannelInterceptor to intercept subscription requests.

@Component
public class TopicSubscriptionInterceptor extends ChannelInterceptor {

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {

        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        
        // Check if the message is a subscription
        if (StompCommand.SUBSCRIBE.equals(accessor.getCommand())) {
            // Extract the topic and userId from the session
            String destination = accessor.getDestination();
            String sessionId = accessor.getSessionId();
            String userId = (String) accessor.getSessionAttributes().get("userId");
            
            // Extract the expected userId from the topic, e.g. /topic/<userId>
            String expectedUserId = getUserIdFromDestination(destination);

            // Ensure the user is only subscribing to their own topic
            if (!expectedUserId.equals(userId)) {
                throw new IllegalArgumentException("Unauthorized subscription attempt");
            }
        }

        return message;
    }
    
    private String getUserIdFromDestination(String destination) {
        // Parse the topic to get the userId, e.g. /topic/{userId}
        return destination.split("/")[2];
    }
}

3. WebSocket Configuration

Make sure that your HandshakeInterceptor and ChannelInterceptor are registered in your WebSocket configuration.

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
        registry.addEndpoint("/ws")
                .addInterceptors(new JwtHandshakeInterceptor()) // Register JWT Interceptor
                .setAllowedOriginPatterns("*")
                .withSockJS(); // Or without SockJS if not using it
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new TopicSubscriptionInterceptor()); // Register Subscription Interceptor
    }
}

Summary of the Flow:

1. The JwtHandshakeInterceptor validates the JWT during the WebSocket handshake and attaches the userId to the WebSocket session.


2. When the user attempts to subscribe to a topic, the TopicSubscriptionInterceptor checks that the topic matches the userId stored in the session.


3. If the user tries to subscribe to another user’s topic, the server throws an error and rejects the subscription.



This setup ensures that users cannot subscribe to topics that don’t belong to them, effectively securing your WebSocket connection based on JWT tokens.

