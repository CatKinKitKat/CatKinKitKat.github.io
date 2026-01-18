+++
title = "OAuth2 and OIDC, or Why Your Auth Is Probably Wrong"
date = 2025-12-15
[taxonomies]
tags = ["security", "oauth2", "microservices"]
+++

I've lost count of how many times I've seen OAuth2 implemented incorrectly. Not "kind of wrong" - fundamentally wrong, in ways that either break functionality or create security holes. The spec is long, the terminology is confusing, and most developers (myself included, at one point) cargo-cult their way through it by copying Stack Overflow snippets until the login works.

Let me try to make this clearer than the RFC did.

## The Core Idea

OAuth2 is an authorization framework. It answers: "should this application be allowed to access this resource on behalf of this user?" It does NOT answer: "who is this user?" That's OIDC (OpenID Connect), which is a layer on top of OAuth2.

Four roles:
- **Resource Owner** - the user
- **Client** - the application requesting access
- **Authorization Server** - issues tokens (Keycloak, Auth0, Okta, etc.)
- **Resource Server** - the API that holds the protected resources

The flow: the client asks the authorization server for permission. If the user agrees, the authorization server issues a token. The client sends the token to the resource server. The resource server validates the token and grants access.

That's it. Everything else is details about how to get the token safely.

## Authorization Code Flow

This is the flow you should use for web applications and mobile apps. The flow that involves a redirect to the authorization server, user login, and a redirect back with a code.

Step by step:

1. Your app redirects the user to the authorization server: `https://auth.example.com/authorize?response_type=code&client_id=myapp&redirect_uri=https://myapp.com/callback&scope=openid profile`

2. The user logs in and consents.

3. The authorization server redirects back: `https://myapp.com/callback?code=abc123`

4. Your backend exchanges the code for tokens:

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=abc123
&redirect_uri=https://myapp.com/callback
&client_id=myapp
&client_secret=supersecret
```

5. You get back an access token, refresh token, and (with OIDC) an ID token.

The critical detail: the code exchange (step 4) happens server-side. The authorization code is useless without the client secret, which never leaves your backend. This is what makes the authorization code flow secure.

**PKCE** (Proof Key for Code Exchange) adds another layer: the client generates a random `code_verifier`, sends a hash of it (`code_challenge`) in step 1, and proves possession of the original verifier in step 4. This prevents interception attacks even for public clients (like SPAs) that can't keep a client secret.

Use PKCE. Always. Even for confidential clients. It's a strict improvement with no downside.

## Client Credentials for Service-to-Service

When Service A needs to call Service B and there's no user involved (background jobs, batch processing, service mesh communication), use the client credentials flow.

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=order-service
&client_secret=service-secret
&scope=shipping:read shipping:write
```

No user, no redirect, no browser. The service authenticates directly and gets an access token.

In Spring Boot:

```java
@Bean
public OAuth2AuthorizedClientManager authorizedClientManager(
        ClientRegistrationRepository clientRegistrations,
        OAuth2AuthorizedClientRepository authorizedClients) {

    var provider = OAuth2AuthorizedClientProviderBuilder.builder()
        .clientCredentials()
        .build();

    var manager = new DefaultOAuth2AuthorizedClientManager(
        clientRegistrations, authorizedClients);
    manager.setAuthorizedClientProvider(provider);
    return manager;
}
```

Configure the client in `application.yml`:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          shipping-service:
            provider: keycloak
            client-id: order-service
            client-secret: ${SHIPPING_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: shipping:read,shipping:write
        provider:
          keycloak:
            token-uri: https://auth.example.com/realms/services/protocol/openid-connect/token
```

The token gets cached and refreshed automatically. You don't manage token lifecycle manually.

## Token Introspection

There are two ways a resource server can validate an access token:

**Local validation** - for JWTs. The resource server verifies the token's signature using the authorization server's public key. No network call needed. Fast, but you can't revoke individual tokens (they're valid until they expire).

**Token introspection** - the resource server asks the authorization server "is this token valid?" on every request. Slower (network call), but the authorization server can say "no" even for non-expired tokens. Useful for opaque tokens and when you need immediate revocation.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        # For JWT validation (local)
        jwt:
          issuer-uri: https://auth.example.com/realms/myapp

        # OR for introspection (remote)
        opaquetoken:
          introspection-uri: https://auth.example.com/realms/myapp/protocol/openid-connect/token/introspect
          client-id: my-resource-server
          client-secret: ${INTROSPECTION_SECRET}
```

Pick one. For most microservices, JWT with short expiry (5-15 minutes) and refresh tokens is the right choice. Introspection is for when you need revocation and can tolerate the latency.

## Common OAuth2 Mistakes in Microservices

I've seen all of these in production:

**1. Storing tokens in localStorage.** Accessible to any JavaScript on the page. One XSS vulnerability and every token is compromised. Use httpOnly cookies for web apps.

**2. Not validating the token audience.** Your order service receives a token. Is that token meant for the order service or for a completely different API? If you don't check the `aud` claim, a token issued for Service A can be used to access Service B.

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer(issuerUri),
        new JwtClaimValidator<List<String>>("aud",
            aud -> aud != null && aud.contains("order-service"))
    ));
    return decoder;
}
```

**3. Using the implicit flow.** It was deprecated for good reason. The access token is returned in the URL fragment, which is visible in browser history and server logs. Use authorization code + PKCE instead.

**4. Long-lived access tokens.** If your access token expires in 24 hours, that's a 24-hour window where a stolen token works. Use short-lived access tokens (5-15 minutes) and refresh tokens for new access tokens.

**5. Not using scopes.** Every service gets a token with full access because nobody configured scopes. If the notification service only needs to read user profiles, its token should only have `profile:read`. Least privilege applies to service tokens too.

**6. Sharing client credentials.** Every service should have its own client ID and secret. If three services share one client credential, you can't revoke access for one without breaking the others, and you can't audit which service made which call.

**7. Skipping the state parameter.** The `state` parameter prevents CSRF attacks during the authorization flow. If you're not generating a random state, verifying it on the callback, and rejecting mismatches, you're vulnerable.

## OIDC: The Identity Layer

OIDC adds identity to OAuth2. When you include `openid` in the scope, you get an ID token - a JWT that contains claims about the user (sub, name, email, etc.).

The ID token is for the client. It tells the client who the user is. The access token is for the resource server. It tells the resource server what the client is allowed to do.

Don't send the ID token to APIs. Don't use the access token to determine who the user is. They serve different purposes.

Spring Security handles OIDC login elegantly:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: my-web-app
            client-secret: ${KEYCLOAK_SECRET}
            scope: openid, profile, email
        provider:
          keycloak:
            issuer-uri: https://auth.example.com/realms/myapp
```

That's enough to get login, logout, and user info working. Spring Security handles the authorization code flow, token exchange, and session management.

Auth is one of those things where "good enough" is never good enough. Get the fundamentals right, use a battle-tested authorization server, and don't implement your own token validation logic unless you really know what you're doing.
