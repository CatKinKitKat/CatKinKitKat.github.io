+++
title = "JWT - How It Works and How It Breaks"
date = 2025-12-20
[taxonomies]
tags = ["security", "java", "spring-boot"]
+++

JWT (JSON Web Token) is one of those technologies that's simple enough to understand in an afternoon and complex enough to get wrong for years. I've been on both sides. Let me walk you through the mechanics, the footguns, and the eternal debate about whether you should just use sessions instead.

## The Structure

A JWT has three parts, separated by dots: `header.payload.signature`

**Header** - tells you the token type and signing algorithm:
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Payload** - the claims. Some are standard (`iss`, `sub`, `exp`, `aud`), others are custom:
```json
{
  "iss": "https://auth.example.com",
  "sub": "user-123",
  "aud": "order-service",
  "exp": 1703001600,
  "iat": 1703000700,
  "roles": ["ADMIN", "USER"],
  "email": "goncalo@example.com"
}
```

**Signature** - proves the token hasn't been tampered with. It's the header + payload, base64-encoded and signed with the algorithm specified in the header.

The whole thing is base64url-encoded and concatenated. No encryption - anyone can decode and read the payload. The signature only guarantees integrity, not confidentiality. If you put sensitive data in the payload, anyone with the token can read it.

## Signing Algorithms: RS256 vs HS256

This matters more than most people think.

**HS256** (HMAC-SHA256) - symmetric. The same secret signs and verifies. If you have three microservices that need to verify tokens, all three need the shared secret. If one is compromised, an attacker can forge tokens. Also, the authorization server and every resource server share the same secret, which means any resource server can create tokens.

**RS256** (RSA-SHA256) - asymmetric. The authorization server signs with a private key. Resource servers verify with the public key. The public key can't create tokens. If a resource server is compromised, the attacker can't forge tokens - they only have the public key.

Use RS256 for anything beyond a single-service toy project. The performance overhead is negligible (verify operations use the public key, which is fast), and the security model is dramatically better.

Spring Boot configures this automatically when you point it at an OIDC provider:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/myapp
```

Spring Security fetches the public keys from the JWKS (JSON Web Key Set) endpoint at `{issuer-uri}/.well-known/openid-configuration` -> `jwks_uri`. No shared secrets to manage.

## Token Validation

Every resource server must validate these things:

1. **Signature** - the token hasn't been tampered with
2. **Expiration** (`exp`) - the token hasn't expired
3. **Issuer** (`iss`) - the token came from your authorization server
4. **Audience** (`aud`) - the token was intended for your service
5. **Not Before** (`nbf`) - the token is already valid (if the claim is present)

Spring Security handles 1-3 out of the box. The audience check requires explicit configuration, and I've seen it skipped way too often:

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);

    OAuth2TokenValidator<Jwt> audienceValidator = new JwtClaimValidator<>(
        "aud", aud -> aud != null && ((Collection<?>) aud).contains("order-service")
    );

    OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer(issuerUri);
    OAuth2TokenValidator<Jwt> combined = new DelegatingOAuth2TokenValidator<>(
        withIssuer, audienceValidator
    );

    decoder.setJwtValidator(combined);
    return decoder;
}
```

Without audience validation, a token issued for `notification-service` can access `order-service`. In a microservices architecture with shared identity providers, this is a real attack vector.

## Refresh Tokens

Access tokens should be short-lived (5-15 minutes). But you don't want users logging in every 15 minutes. That's what refresh tokens are for.

The flow:
1. User logs in, gets an access token (15 min) and a refresh token (hours or days)
2. Access token expires
3. Client sends the refresh token to the authorization server
4. Authorization server validates the refresh token, issues a new access token (and optionally a new refresh token)

Refresh tokens should be:
- **Opaque** (not JWTs) - the authorization server stores them and can revoke them
- **Single-use** (rotation) - each refresh token can only be used once. When used, a new refresh token is issued and the old one is invalidated. If an attacker steals a refresh token and the legitimate client also uses it, one of them will present an already-used token, and the authorization server can invalidate the entire chain
- **Bound to the client** - a refresh token issued to client A can't be used by client B

```java
// On the client side, Spring Security handles refresh automatically
@Bean
public OAuth2AuthorizedClientManager clientManager(
        ClientRegistrationRepository registrations,
        OAuth2AuthorizedClientRepository clients) {

    var provider = OAuth2AuthorizedClientProviderBuilder.builder()
        .authorizationCode()
        .refreshToken()  // enables automatic refresh
        .build();

    var manager = new DefaultOAuth2AuthorizedClientManager(registrations, clients);
    manager.setAuthorizedClientProvider(provider);
    return manager;
}
```

## How JWTs Break

Here are the failures I've seen in the wild:

**The `alg: none` attack.** The JWT spec allows `"alg": "none"` - no signature. Old libraries accepted this, meaning an attacker could create a token with any claims and no signature, and the server would accept it. Modern libraries reject `none` by default, but if you're using a library from 2016, check.

**Algorithm confusion.** If the server is configured for RS256 but doesn't enforce it, an attacker can change the header to HS256 and sign with the public key (which is, by definition, public). Some libraries would then verify the HS256 signature using the public key as the HMAC secret. This is catastrophic. Always enforce the expected algorithm server-side.

**No expiration.** Tokens without `exp` that live forever. If leaked, they're valid indefinitely. Always set expiration. Always validate it.

**Sensitive data in the payload.** Social security numbers, passwords, PII. The payload is base64-encoded, not encrypted. Anyone who intercepts the token can decode it.

**Token replay.** Someone intercepts an access token and reuses it. Short expiry limits the window. For high-security operations, combine JWT with additional measures (e.g., bind the token to a client certificate with mTLS).

**Clock skew.** Service A's clock is 30 seconds ahead of the authorization server's clock. Tokens appear expired when they shouldn't be, or appear valid when they shouldn't be. Spring Security allows a default clock skew tolerance of 60 seconds. Tune it if needed, but also fix your clocks (NTP).

## The "Just Use Sessions" Argument

Every few months someone writes a blog post titled "Stop Using JWTs for Sessions" and it generates a lot of discussion. The argument: server-side sessions are simpler, revocable, and don't have the footguns of JWTs. Just use a session cookie.

They're not wrong. For a monolithic application with a single server (or sticky sessions), server-side sessions are simpler and perfectly adequate. You don't need JWTs.

But in a microservices architecture with multiple resource servers, sessions become problematic:

- **Session sharing** - if the user hits Service A and then Service B, both need access to the session. You need a shared session store (Redis, database). Now you have a centralized dependency.
- **Statelessness** - JWTs are self-contained. The resource server validates the token without calling anyone. With sessions, every request needs a session lookup.
- **Cross-service authorization** - a JWT can carry scopes and roles that multiple services can read. A session ID carries no information; every service needs to look it up.

My take: for frontend-to-backend auth in a monolith, sessions are fine. For service-to-service auth in microservices, JWTs are the right tool. For frontend-to-backend in a microservices architecture, use the Backend for Frontend pattern - the BFF keeps sessions, and uses JWTs internally.

## Practical Advice

Keep your access tokens small. Don't cram the user's entire profile, permissions tree, and organizational hierarchy into the JWT. The token goes on every request. A 4KB token on every API call adds up.

Use your authorization server's standard claims. Don't invent your own claim names for standard concepts. `sub` for subject, `iss` for issuer, `aud` for audience, `roles` or `realm_access` (Keycloak) for roles.

Log token validation failures. When a request is rejected because of a bad token, log why (expired, bad signature, wrong audience). These logs are invaluable for debugging auth issues and detecting attacks.

Don't build your own JWT library. Use `nimbus-jose-jwt` (what Spring Security uses under the hood), `auth0-java-jwt`, or `jjwt`. The edge cases in JWT parsing and validation are numerous and subtle. Let someone who's spent years on it handle it.

JWTs are a good tool when used correctly. They're a security incident when used carelessly. Know the mechanics, validate properly, keep them short-lived, and you'll be fine.
