+++
title = "Kubernetes Secrets Management: base64 Is Not Encryption"
date = 2025-05-25
[taxonomies]
tags = ["kubernetes", "security", "devops"]
+++

I need to say this clearly because I keep seeing it in production clusters: Kubernetes Secrets are not secret. They're base64-encoded, stored in etcd, and anyone with the right RBAC can read them in plain text. Base64 is an encoding scheme, not encryption. Your "secret" database password is one `kubectl get secret -o json | base64 -d` away from being readable.

The default Kubernetes secret mechanism is fine for separating config from secrets conceptually. It's not fine for actual security. Let's talk about what is.

## Option 1: Sealed Secrets

Bitnami's Sealed Secrets is the simplest approach. A controller runs in your cluster with a private key. You encrypt secrets locally using `kubeseal`, commit the encrypted manifest to Git, and the controller decrypts them in-cluster.

```bash
kubectl create secret generic db-creds \
  --from-literal=password=hunter2 \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-db-creds.yaml
```

Pros: dead simple, GitOps-friendly, no external dependencies.

Cons: the encryption key is in the cluster. If someone compromises the cluster, they get the key. If you lose the cluster and didn't back up the key, you lose all your secrets. It's cluster-scoped - you can't share sealed secrets between clusters without re-encrypting.

I use Sealed Secrets for development and staging environments where the threat model is "keep secrets out of Git" rather than "defend against nation-state actors."

## Option 2: SOPS (Secrets OPerationS)

Mozilla SOPS encrypts specific values in YAML/JSON files while leaving the keys in plaintext. This means you can see the structure of your secrets file without being able to read the values.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
data:
  password: ENC[AES256_GCM,data:abc123...,iv:xyz...]
```

SOPS can use AWS KMS, Azure Key Vault, GCP KMS, or PGP for encryption. The nice thing is that it integrates with your cloud provider's key management - you don't have a separate key to manage.

The workflow with ArgoCD: install the `ksops` kustomize plugin, and ArgoCD decrypts SOPS-encrypted files during sync. Or use the ArgoCD Vault Plugin.

```yaml
# .sops.yaml
creation_rules:
  - azure_keyvault: https://myvault.vault.azure.net/keys/sops-key/abc123
    encrypted_regex: "^(data|stringData)$"
```

Pros: encrypts in-place, works with any KMS, the encrypted files are still readable YAML.

Cons: requires access to the KMS for encryption (developers need cloud credentials), the setup is more complex than Sealed Secrets, and you need the plugin integration for ArgoCD.

## Option 3: External Secrets Operator

The External Secrets Operator (ESO) takes a different approach. Instead of encrypting secrets and storing them in Git, you store secrets in an external provider (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) and ESO syncs them into Kubernetes Secrets.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-key-vault
    kind: ClusterSecretStore
  target:
    name: db-creds
  data:
    - secretKey: password
      remoteRef:
        key: production-db-password
```

You commit the ExternalSecret manifest to Git (which contains no sensitive data - just a reference), and ESO creates the actual Kubernetes Secret by fetching from the external store.

Pros: secrets never touch Git, rotation is handled by the external provider, works across clusters, centralized management.

Cons: dependency on the external provider (if Key Vault is down, secret refresh fails), requires the operator and a SecretStore configuration, adds operational complexity.

This is my preferred approach for production. Azure Key Vault as the source of truth, ESO syncing into the cluster, ArgoCD managing the ExternalSecret manifests.

## Option 4: HashiCorp Vault CSI Driver

Vault is the heavyweight option. The CSI Secrets Store Driver mounts Vault secrets as volumes in your pods. The secrets never exist as Kubernetes Secrets - they're injected directly into the pod filesystem.

```yaml
apiVersion: secrets-store.csi.x-secrets-store.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.mycompany.com"
    roleName: "order-service"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/production/db"
        secretKey: "password"
```

```yaml
volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "vault-db-creds"
```

Your pod reads secrets from a mounted file. Vault handles rotation, access policies, audit logging, dynamic secrets (generating temporary database credentials on the fly), and lease management.

Pros: enterprise-grade, dynamic secrets, fine-grained access policies, audit trail.

Cons: you're running Vault, which is itself a complex distributed system. It needs its own HA setup, storage backend, unsealing process, and operational expertise. If Vault goes down, new pods can't start because they can't fetch secrets.

I've seen Vault run beautifully in organizations with dedicated platform teams. I've also seen it become the single point of failure that takes down everything. Choose based on your team's capacity to operate it.

## What About etcd Encryption?

Kubernetes supports encrypting secrets at rest in etcd. This is a good baseline:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

This encrypts secrets before they're written to etcd. If someone steals the etcd data files, they can't read your secrets. But it doesn't help if they have Kubernetes API access - the API server decrypts before serving.

On managed Kubernetes (AKS, EKS, GKE), etcd encryption at rest is usually available as a checkbox. Enable it. It's free security.

## My Decision Framework

**Development**: Sealed Secrets. Simple, good enough, GitOps-friendly.

**Staging**: SOPS with Azure Key Vault KMS. Same GitOps workflow, better key management.

**Production**: External Secrets Operator with Azure Key Vault. Secrets never in Git, centralized rotation, cross-cluster support.

**Enterprise with compliance requirements**: Vault. Dynamic secrets, audit trails, policy engine. But only if you can afford to operate it.

The wrong answer is doing nothing and leaving base64-encoded secrets in your manifests. I've seen database passwords in public GitHub repos because someone committed a Kubernetes Secret manifest thinking it was encrypted. It wasn't. It never was.
