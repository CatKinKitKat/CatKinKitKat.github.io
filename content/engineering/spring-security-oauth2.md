+++
title = "Spring Security and OAuth2 in Microservices: The Full Picture"
date = 2026-02-19
[taxonomies]
tags = ["spring-boot", "security", "oauth2"]
+++

Security in microservices is one of those topics where the gap between "I read the tutorial" and "this works in production" is enormous. The tutorial shows you a happy path with a single service and a test user. Production has service-to-service auth, token propagation, CORS nightmares, CSRF decisions, and that one legacy SAML2 identity provider that refuses to die.

Let me walk through how I actually set this up.

## OAuth2 and OIDC: The Quick Primer

OAuth2 is an authorization framework. OpenID Connect (OIDC) is an authentication layer built on top of it. In practice, you use OIDC for "who is this user?" and OAuth2 for "what are they allowed to do?"

The flow for a typical microservices setup:

1. User authenticates with the Identity Provider (Keycloak, Auth0, Okta, etc.)
2. Identity Provider issues a JWT (JSON Web Token) containing claims (user ID, roles, permissions)
3. Client includes the JWT in the `Authorization` header for API requests
4. API gateway or individual services validate the JWT and extract claims
5. Authorization decisions are made based on claims

The JWT is self-contained. Services don't need to call back to the IdP to validate it - they just verify the signature using the IdP's public key. This is important for performance and availability.

## Keycloak Integration

Keycloak is my go-to IdP for projects where we control the identity infrastructure. It's open source, battle-tested, and handles OAuth2/OIDC, SAML2, social login, and user management in one package.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/my-realm
          jwk-set-uri: https://keycloak.example.com/realms/my-realm/protocol/openid-connect/certs
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            )
            .build();
    }

    private JwtAuthenticationConverter jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        // Keycloak puts roles in realm_access.roles
        // We need a custom converter to extract them
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            Collection<GrantedAuthority> authorities = new ArrayList<>();

            // Extract realm roles
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            if (realmAccess != null) {
                List<String> roles = (List<String>) realmAccess.get("roles");
                if (roles != null) {
                    roles.forEach(role ->
                        authorities.add(new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    );
                }
            }

            return authorities;
        });
        return converter;
    }
}
```

The Keycloak role extraction is the part that trips people up. Keycloak nests roles inside `realm_access.roles` in the JWT, but Spring Security's default converter expects them in the `scope` claim. The custom converter bridges this gap.

## JWT: What to Put In, What to Leave Out

JWTs are not encrypted (by default). They're base64-encoded and signed. Anyone with the token can read the claims. Keep this in mind.

**Good claims**: User ID, username, roles, tenant ID, token expiry.

**Bad claims**: Email (PII), full name (PII), permissions list (too large), anything sensitive.

```java
// Extracting claims in a service
@RestController
public class OrderController {

    @GetMapping("/api/orders")
    public List<Order> getOrders(@AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();
        String tenantId = jwt.getClaimAsString("tenant_id");

        return orderService.getOrdersForUser(userId, tenantId);
    }
}
```

Token size matters. Every request carries the JWT. A token with 50 roles and 200 permissions bloats every HTTP request. I've seen JWTs over 4KB, which is larger than some response bodies. If you need fine-grained permissions, consider a claims-based approach where the token contains coarse roles and the service looks up specific permissions locally.

## Service-to-Service Authentication

User-to-service auth is straightforward. Service-to-service is where it gets interesting.

**Option 1: Client Credentials Grant.** Services authenticate with the IdP using a client ID and secret to get their own JWT.

```java
@Configuration
public class ServiceAuthConfig {

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clients,
            OAuth2AuthorizedClientRepository authorizedClients) {

        OAuth2AuthorizedClientProvider provider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .clientCredentials()
                .build();

        DefaultOAuth2AuthorizedClientManager manager =
            new DefaultOAuth2AuthorizedClientManager(clients, authorizedClients);
        manager.setAuthorizedClientProvider(provider);
        return manager;
    }
}
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          inventory-service:
            client-id: order-service
            client-secret: ${ORDER_SERVICE_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: inventory:read,inventory:write
        provider:
          inventory-service:
            token-uri: https://keycloak.example.com/realms/my-realm/protocol/openid-connect/token
```

**Option 2: Token Propagation.** Forward the user's JWT to downstream services. The downstream service then knows both the calling service and the original user.

```java
@Bean
public RestClient inventoryClient(OAuth2AuthorizedClientManager clientManager) {
    return RestClient.builder()
        .baseUrl("http://inventory-service:8080")
        .requestInterceptor((request, body, execution) -> {
            // Propagate the current user's token
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth instanceof JwtAuthenticationToken jwtAuth) {
                request.getHeaders().setBearerAuth(jwtAuth.getToken().getTokenValue());
            }
            return execution.execute(request, body);
        })
        .build();
}
```

I prefer token propagation for requests that act on behalf of a user, and client credentials for autonomous service operations (batch jobs, scheduled tasks).

## SAML2: Because Legacy Exists

Some enterprises still use SAML2 identity providers. If you need to support SAML2 alongside OAuth2 (because the corporate IdP speaks SAML and your APIs speak JWT), Keycloak can act as a protocol bridge.

```
User -> SAML2 IdP -> Keycloak (SAML2 -> OIDC bridge) -> JWT -> Your Services
```

This way your services only deal with JWTs. The SAML2 complexity is contained in Keycloak. If you need to support both protocols simultaneously, Keycloak's identity brokering feature handles this natively.

If you must handle SAML2 directly in Spring:

```java
@Configuration
@EnableWebSecurity
public class Saml2SecurityConfig {

    @Bean
    public SecurityFilterChain samlFilterChain(HttpSecurity http) throws Exception {
        return http
            .saml2Login(saml2 -> saml2
                .relyingPartyRegistrationRepository(relyingPartyRegistrations())
            )
            .saml2Logout(Customizer.withDefaults())
            .build();
    }
}
```

But seriously, use Keycloak as a bridge. Life is too short for debugging SAML assertions directly.

## Vault for Secrets Management

Client secrets, signing keys, database passwords - these shouldn't be in your application.yml. HashiCorp Vault is the proper solution.

```yaml
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      authentication: kubernetes
      kubernetes:
        role: order-service
      kv:
        enabled: true
        backend: secret
        default-context: order-service
```

Vault provides:
- **Dynamic secrets**: Database credentials generated on-demand with TTLs. No more shared passwords.
- **Secret rotation**: Credentials rotate automatically. Your application picks up new credentials without restarts.
- **Audit logging**: Every secret access is logged. When security asks "who accessed the production database credentials?", you have an answer.

## CSRF and CORS

Two topics that cause more confusion than they should.

**CSRF (Cross-Site Request Forgery)**: If your API is consumed by browsers with session-based auth, you need CSRF protection. If your API uses Bearer tokens (JWTs), CSRF protection is unnecessary because the browser doesn't automatically attach Bearer tokens to cross-origin requests.

```java
// For JWT-based APIs: disable CSRF
http.csrf(csrf -> csrf.disable());

// For session-based web apps: keep CSRF enabled (it's on by default)
```

**CORS (Cross-Origin Resource Sharing)**: If your frontend and backend are on different origins (which they almost always are in microservices), you need CORS configuration.

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of(
        "https://app.example.com",
        "https://admin.example.com"
    ));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

Never use `allowedOrigins("*")` in production. It defeats the purpose of CORS. If you need to support multiple origins, list them explicitly or resolve them from configuration.

## The Architecture

Putting it all together, here's how security flows in a typical microservices setup:

1. **API Gateway**: Validates JWTs, applies CORS, rate limits. Forwards valid requests with the JWT attached.
2. **Backend Services**: Extract claims from the JWT, make authorization decisions. Trust the gateway for authentication.
3. **Service-to-Service**: Client credentials for autonomous calls, token propagation for user-context calls.
4. **Secrets**: Vault manages all credentials. Services request secrets at startup or on-demand.
5. **Identity Provider**: Keycloak (or equivalent) handles user authentication, token issuance, and protocol bridging.

Security is not something you bolt on at the end. It's infrastructure that needs to be in place from day one. Getting it right early saves you from the far more painful experience of retrofitting security into a running system.
