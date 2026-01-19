+++
title = "Secrets Management with Vault, or Stop Putting Passwords in Git"
date = 2025-12-25
[taxonomies]
tags = ["security", "kubernetes", "devops"]
+++

Merry Christmas. Let's talk about secrets management, because nothing says "holiday spirit" like remembering that your database password has been hardcoded in `application.properties` and committed to Git since 2019.

I'm exaggerating. Slightly. But I've seen environment variables passed through Jenkins pipelines in plain text. I've seen `.env` files checked into repositories (the `.gitignore` entry was added *after* the commit - thanks, that's not how Git works). I've seen Kubernetes Secrets used as if base64 encoding is encryption (it absolutely is not).

Secrets management is one of those things that seems simple until you actually think about it. Let me walk through the options.

## HashiCorp Vault

Vault is the heavyweight champion. It does dynamic secrets, secret rotation, encryption as a service, PKI certificate management, and approximately forty-seven other things. It's also operationally complex, which is the trade-off.

The basics: Vault stores secrets and controls access to them through policies. You authenticate to Vault (using tokens, Kubernetes service accounts, LDAP, etc.), and Vault returns the secrets you're authorized to read.

### Vault on Kubernetes

In Kubernetes, the Vault Agent Injector is the cleanest integration. You annotate your pods, and the injector sidecar fetches secrets and writes them to a shared volume:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "order-service"
        vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/order-service"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "database/creds/order-service" -}}
          spring.datasource.username={{ .Data.username }}
          spring.datasource.password={{ .Data.password }}
          {{- end -}}
    spec:
      serviceAccountName: order-service
      containers:
        - name: order-service
          image: order-service:latest
          volumeMounts:
            - name: vault-secrets
              mountPath: /vault/secrets
              readOnly: true
```

The Vault agent handles authentication, secret retrieval, and token renewal. Your application just reads a file. It doesn't need to know Vault exists.

The real power: dynamic secrets. Instead of a static database password, Vault generates temporary credentials for each pod. When the pod dies, the credentials are revoked. No shared passwords. No credential rotation headaches.

### Spring Cloud Vault

If you want tighter integration, Spring Cloud Vault pulls secrets directly into Spring's `Environment` at startup:

```yaml
spring:
  cloud:
    vault:
      uri: https://vault.internal:8200
      authentication: kubernetes
      kubernetes:
        role: order-service
        kubernetes-path: kubernetes
      kv:
        backend: secret
        default-context: order-service
  config:
    import: vault://
```

Now `@Value("${db.password}")` pulls from Vault instead of a properties file. The application code doesn't change, only the configuration source.

The downside: your application now has a hard dependency on Vault at startup. If Vault is down, your service doesn't start. Use the agent injector approach if you want Vault to be a soft dependency (secrets are fetched before the app starts, and the app just reads files).

## Sealed Secrets

For teams that find Vault overkill (it is, for small setups), Bitnami Sealed Secrets is a simpler alternative for Kubernetes.

The idea: you encrypt your secrets client-side using the cluster's public key. The encrypted `SealedSecret` resource is safe to commit to Git. The controller in the cluster decrypts it and creates a standard Kubernetes Secret.

```bash
# Encrypt a secret
kubeseal --format=yaml < my-secret.yaml > my-sealed-secret.yaml
```

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: order-service-secrets
  namespace: default
spec:
  encryptedData:
    DB_PASSWORD: AgBy3i4OJSWK+PiTyS...
    API_KEY: AgBxz8FKTI2PaQ...
```

This goes in Git. Only the cluster's controller can decrypt it. If someone gets the repository, they get encrypted blobs. If someone gets the running Kubernetes Secret, they only have access if they already have cluster access.

Sealed Secrets doesn't do dynamic secrets, rotation, or access policies. It's just "encrypt secrets so they can live in Git." For many teams, that's enough.

## SOPS (Secrets OPerationS)

Mozilla SOPS encrypts files (YAML, JSON, INI, ENV, binary) using cloud KMS keys, PGP, or age. It's like Sealed Secrets but not tied to Kubernetes.

```bash
# Encrypt with AWS KMS
sops --encrypt --kms "arn:aws:kms:..." secrets.yaml > secrets.enc.yaml
```

SOPS encrypts values but leaves keys in plaintext, so diffs are meaningful:

```yaml
db:
    password: ENC[AES256_GCM,data:abc123...,iv:def456...,tag:ghi789...]
    username: ENC[AES256_GCM,data:xyz...,iv:...,tag:...]
```

You can see that `db.password` changed in a PR without seeing the value. This is a surprisingly useful property for code review.

SOPS integrates with Argo CD and Flux for GitOps workflows. Encrypt your secrets, commit them, and the CD tool decrypts them during deployment.

## mTLS and Certificate Management

Secrets aren't just passwords. TLS certificates are secrets too. And managing them is its own circle of hell.

Vault has a PKI secrets engine that acts as a certificate authority. It issues certificates with short lifetimes and handles rotation automatically:

```bash
vault write pki/issue/order-service \
    common_name="order-service.default.svc.cluster.local" \
    ttl="24h"
```

Short-lived certificates (24h) with automatic renewal mean you never have that "the certificate expired on a Saturday night" incident.

For Spring Boot services that need to reload certificates without restart:

```yaml
spring:
  ssl:
    bundle:
      pem:
        server:
          keystore:
            certificate: /vault/secrets/tls.crt
            private-key: /vault/secrets/tls.key
          truststore:
            certificate: /vault/secrets/ca.crt
      watch:
        file:
          quiet-period: 10s
server:
  ssl:
    bundle: server
```

Spring Boot 3.2+ supports SSL bundle reloading. When the Vault agent rotates the certificate files, Spring Boot picks up the new ones without a restart.

## Securing Spring Cloud Config

If you're using Spring Cloud Config Server, secrets in your config repo are a concern. Options:

**Encrypt values in the config repo.** Spring Cloud Config supports property encryption:

```yaml
spring:
  datasource:
    password: '{cipher}AQBqY8...'
```

The config server decrypts at serve-time using a symmetric key or RSA key pair. Better than plaintext, but the config server now holds the decryption key.

**Use Vault as a config backend.** Spring Cloud Config Server can read from Vault directly:

```yaml
spring:
  cloud:
    config:
      server:
        vault:
          host: vault.internal
          port: 8200
          kvVersion: 2
```

This is the better approach. Secrets live in Vault, application config lives in Git, and the config server composites them together. The config server doesn't store or cache secrets.

## The Hierarchy

Here's how I think about secrets management, from simplest to most comprehensive:

1. **Environment variables**: fine for local development, terrible for production
2. **Kubernetes Secrets**: better than env vars, but not encrypted at rest by default
3. **Sealed Secrets / SOPS**: encrypted at rest, safe for Git, no dynamic secrets
4. **Vault**: the full solution: dynamic secrets, rotation, audit logging, policies

Start at the level that matches your threat model and operational capacity. A startup with three developers doesn't need Vault. A company handling financial data probably does.

The one thing everyone should do, regardless of scale: stop committing secrets to Git. Use `.gitignore`, use pre-commit hooks that scan for secrets (like `gitleaks`), and assume that anything committed to Git is compromised. Because it probably is.
.
