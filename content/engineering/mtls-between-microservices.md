+++
title = "mTLS Between Microservices, or Why the Network Is Not Your Security Boundary"
date = 2025-12-28
[taxonomies]
tags = ["security", "kubernetes", "microservices"]
+++

"We don't need TLS between services - they're all inside the cluster."

I've heard this sentence at least a dozen times. And every time, I have the same reaction: the network is not a security boundary, and if you treat it like one, you're going to have a bad day.

Let me explain why mTLS between microservices matters, even - especially - inside a Kubernetes cluster.

## TLS Fundamentals for Service-to-Service

Regular TLS (the thing that puts the S in HTTPS) does two things: it encrypts the communication, and it authenticates the server. Your browser verifies that it's talking to the real `google.com`, not an impersonator. But the server doesn't verify the client's identity. Any client can connect.

Mutual TLS (mTLS) adds the second direction: the client also presents a certificate. Both sides verify each other's identity. Service A proves it's Service A. Service B proves it's Service B. Neither can impersonate the other.

The handshake:

1. Client connects and presents its certificate
2. Server validates the client's certificate against its trusted CA
3. Server presents its certificate
4. Client validates the server's certificate against its trusted CA
5. Both sides derive session keys and communication is encrypted

After the handshake, you have identity (who is calling) and confidentiality (traffic is encrypted). Without mTLS, you have neither inside the cluster.

## Why mTLS Matters Inside a Cluster

"But my pods are in a private network. Only cluster traffic can reach them."

Sure. Until:

- A compromised pod can sniff traffic from other pods on the same node (if you're not using network policies, and many clusters don't)
- A supply chain attack injects malicious code into one service, which can then call any other service pretending to be legitimate traffic
- An insider (or compromised CI/CD pipeline) deploys a rogue pod that impersonates a service
- A misconfigured network policy exposes internal services to unintended traffic

Zero trust means verifying identity at every hop, not just at the perimeter. mTLS gives you that verification.

Beyond security, mTLS gives you observability: the certificate's subject tells you exactly which service is calling. No more "who's making these unauthenticated requests to the payment service?" debugging sessions.

## cert-manager on Kubernetes

cert-manager is the standard for certificate lifecycle management in Kubernetes. It integrates with multiple CAs (Let's Encrypt, Vault, self-signed, AWS ACM, etc.) and handles issuance and renewal automatically.

First, set up an issuer. For internal mTLS, a self-signed CA works:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-keypair

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca-keypair
  namespace: cert-manager
spec:
  isCA: true
  commonName: internal-ca
  secretName: internal-ca-keypair
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
```

Then request certificates for your services:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: order-service-tls
  namespace: default
spec:
  secretName: order-service-tls
  duration: 24h
  renewBefore: 8h
  commonName: order-service
  dnsNames:
    - order-service
    - order-service.default.svc.cluster.local
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
```

cert-manager creates a Kubernetes Secret with `tls.crt`, `tls.key`, and `ca.crt`. It renews the certificate automatically before expiry. You mount this secret in your pod and configure your service to use it.

Short certificate lifetimes (24h) are important. If a certificate is compromised, the window of exposure is limited. Combine this with automatic renewal and you never think about certificate expiry again.

## Spring Boot SSL Configuration

Spring Boot has gotten significantly better at TLS configuration. In Spring Boot 3.2+, SSL bundles make it clean:

```yaml
spring:
  ssl:
    bundle:
      pem:
        server:
          keystore:
            certificate: /certs/tls.crt
            private-key: /certs/tls.key
          truststore:
            certificate: /certs/ca.crt
        client:
          keystore:
            certificate: /certs/tls.crt
            private-key: /certs/tls.key
          truststore:
            certificate: /certs/ca.crt

server:
  ssl:
    bundle: server
    client-auth: need  # require client certificates (mTLS)
```

The `client-auth: need` is what makes it mutual. Without it, you have regular TLS (server auth only). With `need`, clients must present a valid certificate or the connection is refused.

For outgoing calls with RestClient:

```java
@Bean
public RestClient restClient(RestClient.Builder builder, SslBundles sslBundles) {
    return builder
        .baseUrl("https://shipping-service:8443")
        .apply(b -> b.setSslBundle(sslBundles.getBundle("client")))
        .build();
}
```

The RestClient uses the client certificate for outgoing connections. The shipping service validates it against its trusted CA. Both directions authenticated.

## Spring Boot SSL Hot Reload

Certificates rotate. With 24-hour certificates and cert-manager, they rotate daily. Restarting the service for every certificate rotation is not acceptable.

Spring Boot 3.2+ supports file-based SSL bundle reloading:

```yaml
spring:
  ssl:
    bundle:
      pem:
        server:
          reload-on-update: true
          keystore:
            certificate: /certs/tls.crt
            private-key: /certs/tls.key
          truststore:
            certificate: /certs/ca.crt
```

When the certificate files change (cert-manager updates the secret, which updates the mounted volume), Spring Boot detects the change and reloads the SSL context. No restart required. Active connections continue with the old certificate; new connections use the new one.

This was a pain point for years. Before this feature, you had to either restart the pod on certificate rotation (ugly), use a sidecar proxy like Envoy (complex), or write custom reload logic (error-prone). Now it's a configuration property.

## The Service Mesh Alternative

If you're running Istio, Linkerd, or Cilium with mTLS, you get mTLS between services without touching your application code. The sidecar proxy (or eBPF program for Cilium) handles the TLS termination and certificate management.

Pros:
- Zero application changes
- Automatic certificate rotation
- Consistent mTLS across all services regardless of language
- Comes with observability and traffic management for free

Cons:
- Operational complexity of running a service mesh
- Resource overhead (especially sidecar-based meshes)
- Debugging is harder when network issues are in the sidecar, not your app
- Another moving part that can fail

My take: if you already have a service mesh, use its mTLS. Don't implement mTLS at the application level too. If you don't have a service mesh and mTLS is your primary motivation for adopting one, consider whether application-level mTLS with cert-manager is simpler for your situation. A service mesh is a big commitment for one feature.

## Implementation Checklist

If you're adding mTLS to existing services:

1. **Start with a CA.** Set up cert-manager with an internal CA or integrate with Vault's PKI engine.

2. **Issue certificates.** Create Certificate resources for each service. Use DNS SANs matching the Kubernetes service names.

3. **Configure TLS in your services.** Use Spring Boot SSL bundles (or equivalent in your framework). Start with `client-auth: want` (optional client auth) so you don't break existing traffic.

4. **Test with one service pair.** Get mTLS working between two services before rolling out everywhere.

5. **Switch to `client-auth: need`.** Once all clients present certificates, make it mandatory.

6. **Add authorization.** mTLS tells you *who* is calling. You still need to decide *what* they're allowed to do. Extract the client certificate's subject and use it for authorization decisions.

7. **Monitor certificate expiry.** Even with automatic renewal, monitor it. cert-manager exposes Prometheus metrics. Alert if a certificate is within 2 hours of expiry and hasn't been renewed.

mTLS isn't difficult to set up. It's just tedious. But "tedious" is not a valid reason to skip a fundamental security control. Set it up once, automate the certificate lifecycle, and forget about it. Your security team will thank you.
