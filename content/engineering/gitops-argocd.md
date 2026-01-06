+++
title = "GitOps with ArgoCD: Declarative Deployments That Actually Work"
date = 2025-05-15
[taxonomies]
tags = ["kubernetes", "gitops", "devops"]
+++

I used to deploy by SSHing into a server and running a script. Then I deployed by clicking buttons in Jenkins. Then I deployed by running `kubectl apply` from my laptop. Each step was an improvement, and each step was still fundamentally wrong because the state of the cluster lived in someone's head (mine) and in whatever was last applied from wherever.

GitOps fixed this. The cluster state lives in Git. ArgoCD watches Git and makes the cluster match. If someone manually changes something in the cluster, ArgoCD reverts it. The Git repo is the single source of truth, and it's enforced, not aspirational.

## The Core Idea

GitOps is simple: your desired cluster state is committed to a Git repository. An agent running inside the cluster continuously compares the actual state to the desired state and reconciles any drift.

ArgoCD is that agent. It's a Kubernetes controller that watches Git repos and applies manifests. It handles Helm charts, Kustomize overlays, plain YAML, and Jsonnet. It has a web UI that shows you what's deployed, what's out of sync, and what's different.

The key insight: deployments become pull requests. You don't `kubectl apply` anything. You open a PR that changes an image tag, someone reviews it, it merges, and ArgoCD picks up the change. Every deployment is auditable, reversible, and peer-reviewed.

## The App-of-Apps Pattern

When you have more than a few services, managing individual ArgoCD Application resources gets tedious. The app-of-apps pattern solves this: you create one "root" Application that points to a directory of Application manifests.

```yaml
# root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-gitops.git
    path: apps
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

The `apps/` directory contains individual Application manifests:

```yaml
# apps/order-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-gitops.git
    path: services/order-service
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

Add a new service? Commit a new YAML file to the `apps/` directory. ArgoCD picks it up automatically. Delete the file and ArgoCD prunes the resources. The entire cluster topology is visible in one directory.

## Sealed Secrets Integration

Secrets in Git. Everyone knows you shouldn't do it. But GitOps says everything should be in Git. Sealed Secrets resolves this tension.

You install the Sealed Secrets controller in your cluster. It generates a key pair. You encrypt secrets locally using `kubeseal`, commit the encrypted SealedSecret to Git, and the controller decrypts them in-cluster.

```bash
echo -n "my-database-password" | kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format yaml > sealed-secret.yaml
```

The result:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    password: AgBy8j3k... # encrypted, safe for Git
```

ArgoCD syncs this like any other resource. The Sealed Secrets controller decrypts it and creates a regular Kubernetes Secret. Only the cluster can decrypt the sealed secrets - the encryption key never leaves the cluster.

Rotate the key periodically. Back up the key. If you lose the key and the cluster, you lose all your secrets. Ask me how close I've come to this scenario.

## Continuous Promotion Across Environments

The promotion pattern I use: one Git repo, multiple directories per environment.

```
k8s-gitops/
  base/
    order-service/
      deployment.yaml
      service.yaml
  overlays/
    dev/
      kustomization.yaml
    staging/
      kustomization.yaml
    production/
      kustomization.yaml
```

Kustomize overlays let you define a base and patch per environment. Dev gets 1 replica and debug logging. Production gets 3 replicas and warn logging. The base is shared.

Promotion is a PR that updates the image tag in the staging overlay:

```yaml
# overlays/staging/kustomization.yaml
resources:
  - ../../base/order-service
images:
  - name: myregistry.azurecr.io/order-service
    newTag: "1.4.2"
```

CI builds the image and pushes it to the registry. A bot opens a PR to update the tag in dev. After testing, another PR promotes to staging. After staging validation, another PR to production. Each promotion is a Git commit with a reviewer.

No more "what version is running in staging?" You look at Git. Done.

## Preview Environments

This is where GitOps gets genuinely cool. When a developer opens a PR on the application repo, CI builds the image and creates a temporary ArgoCD Application pointing to a preview namespace.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-pr-42
  namespace: argocd
  labels:
    preview: "true"
    pr: "42"
spec:
  source:
    repoURL: https://github.com/myorg/k8s-gitops.git
    path: services/order-service
    targetRevision: main
    helm:
      parameters:
        - name: image.tag
          value: "pr-42-abc1234"
        - name: ingress.host
          value: "pr-42.preview.myapp.dev"
  destination:
    server: https://kubernetes.default.svc
    namespace: preview-pr-42
```

The developer gets a live URL with their changes running. Reviewers can test the PR in a real environment. When the PR closes, the Application gets deleted and the namespace is cleaned up.

The setup isn't trivial - you need wildcard DNS, a way to provision preview databases (or use shared ones), and cleanup automation. But once it works, it transforms how your team reviews code.

## The Gotchas

ArgoCD's sync can be slow if you have a lot of resources. The default sync interval is 3 minutes. For faster deployments, configure webhooks from your Git provider.

`selfHeal: true` is essential but terrifying the first time. If someone manually scales a deployment, ArgoCD scales it back. This is correct behavior but will annoy people who are used to `kubectl scale` as a quick fix.

The web UI is great for visibility but don't let it become a deployment tool. If people start clicking "Sync" in the UI, you've lost the GitOps model. The sync button should be for emergencies, not the normal workflow.

## The Verdict

GitOps with ArgoCD removed an entire category of problems from my work. No more "who deployed what when." No more drift between environments. No more manual hotfixes that get lost. The deployment process is boring, repeatable, and auditable.

The initial setup is real work. The app-of-apps structure, Sealed Secrets, Kustomize overlays, CI integration - it takes a week or two to get right. But once it's running, deployments become the least stressful part of my job, which is exactly how it should be.
