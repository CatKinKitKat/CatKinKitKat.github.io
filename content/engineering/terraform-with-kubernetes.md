+++
title = "Terraform with Kubernetes: Managing Infrastructure That Manages Infrastructure"
date = 2025-07-15
[taxonomies]
tags = ["terraform", "kubernetes", "infrastructure"]
+++

There's a certain irony in using Terraform to provision a Kubernetes cluster that runs ArgoCD that deploys your applications. You're managing infrastructure that manages infrastructure that manages applications. It's turtles all the way down, and at some point you have to decide where one tool's responsibility ends and another begins.

I've settled on a clear boundary: Terraform manages the cluster and cloud resources. ArgoCD manages what runs inside the cluster. They meet at the cluster creation point and a few shared resources. This separation has saved me from the worst IaC headaches.

## Terraform for Cluster Management

Here's what Terraform is good at: provisioning cloud resources. An AKS cluster:

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  name                = "prod-aks-cluster"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "prod"
  kubernetes_version  = "1.29"

  default_node_pool {
    name           = "system"
    node_count     = 3
    vm_size        = "Standard_D4s_v5"
    vnet_subnet_id = azurerm_subnet.aks.id

    upgrade_settings {
      max_surge = "33%"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }
}

resource "azurerm_kubernetes_cluster_node_pool" "workload" {
  name                  = "workload"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D8s_v5"
  node_count            = 3
  max_count             = 10
  min_count             = 3
  enable_auto_scaling   = true

  node_labels = {
    "workload-type" = "application"
  }
}
```

The cluster itself, node pools, networking (VNet, subnets, NSGs), container registry, Key Vault, Log Analytics workspace, managed identities - all Terraform. These are cloud resources with cloud provider APIs. Terraform is purpose-built for this.

## The Terraform + ArgoCD Handoff

After Terraform creates the cluster, ArgoCD needs to be bootstrapped. I use Terraform to install ArgoCD via Helm, then ArgoCD takes over:

```hcl
resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  version          = "5.51.0"
  namespace        = "argocd"
  create_namespace = true

  values = [file("${path.module}/argocd-values.yaml")]

  depends_on = [azurerm_kubernetes_cluster.main]
}

resource "kubectl_manifest" "root_app" {
  yaml_body = templatefile("${path.module}/templates/root-app.yaml", {
    repo_url = var.gitops_repo_url
  })

  depends_on = [helm_release.argocd]
}
```

Terraform installs ArgoCD and creates the root Application that points to the GitOps repo. From there, ArgoCD manages all application deployments. Terraform doesn't touch workload manifests.

The boundary is clean: Terraform owns the platform layer (cluster, networking, identity, observability). ArgoCD owns the application layer (deployments, services, config).

## State Management

Terraform state is the source of truth for what Terraform manages. It must be stored remotely and locked during operations.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state"
    storage_account_name = "tfstateprod"
    container_name       = "tfstate"
    key                  = "prod/aks.tfstate"
  }
}
```

Rules I've learned painfully:

**One state file per environment per component.** Don't put dev and prod in the same state. Don't put networking and Kubernetes in the same state. A bad `terraform destroy` on the wrong workspace shouldn't be able to take down production networking AND the cluster.

**Lock state.** Azure Storage provides locking via blob leases. AWS uses DynamoDB. Without locking, two simultaneous `terraform apply` runs will corrupt your state.

**Never edit state manually.** `terraform state mv` and `terraform import` exist for a reason. If you find yourself in a text editor modifying terraform.tfstate, something has gone very wrong.

**Plan before apply. Always.** `terraform plan -out=plan.tfplan` followed by `terraform apply plan.tfplan`. In CI/CD, the plan runs on PR creation, a human reviews it, and the apply runs on merge. Never auto-apply without review.

## Modules for Reusability

When you have multiple clusters (dev, staging, prod), use modules:

```hcl
module "aks_dev" {
  source = "./modules/aks-cluster"

  environment    = "dev"
  node_count     = 2
  vm_size        = "Standard_D2s_v5"
  k8s_version    = "1.29"
  vnet_subnet_id = module.networking_dev.aks_subnet_id
}

module "aks_prod" {
  source = "./modules/aks-cluster"

  environment    = "prod"
  node_count     = 3
  vm_size        = "Standard_D8s_v5"
  k8s_version    = "1.29"
  vnet_subnet_id = module.networking_prod.aks_subnet_id
}
```

The module encapsulates the cluster definition, node pools, identity configuration, and monitoring setup. The calling code just passes environment-specific values. Same pattern for every environment, different parameters.

## Terraform vs Bicep vs Pulumi vs CDK

This is the "which IaC tool" debate, and opinions are strong. Here's mine.

**Terraform** is the industry standard. Multi-cloud, massive provider ecosystem, HCL is readable (if ugly), huge community. The state management model is annoying but well-understood. If you're not sure what to pick, pick Terraform.

**Bicep** is Azure-specific. If you're all-in on Azure and will never touch another cloud, Bicep is cleaner than Terraform for Azure resources. It compiles to ARM templates, has direct Azure API integration, and doesn't need state management (Azure is the state). The downside: Azure only. The moment you need to manage something outside Azure (a Datadog monitor, a GitHub repo, a Cloudflare DNS record), you need another tool.

**Pulumi** lets you write infrastructure in real programming languages (TypeScript, Python, Go, Java). This is great if you hate HCL and want loops, conditionals, and type checking from a real language. The state model is similar to Terraform. The community is smaller. If your team is full of developers who resist learning HCL, Pulumi reduces friction.

**CDK (AWS)** is AWS-specific Pulumi. Same idea: real programming languages for infrastructure. Compiles to CloudFormation. If you're AWS-only, it's a reasonable choice. Multi-cloud? No.

My take: Terraform for multi-cloud or mixed tooling. Bicep if you're Azure-only and the team prefers it. Pulumi if your team wants a real programming language and is willing to accept the smaller ecosystem. CDK only if you're AWS-only.

We use Terraform at work because our clients are on Azure, AWS, and occasionally GCP. One tool across all of them is worth the HCL tax.

## The Anti-Patterns

**Managing Kubernetes manifests with Terraform.** The Kubernetes Terraform provider exists, but using it to manage Deployments and Services is a mistake. Terraform's reconciliation model (plan/apply) doesn't match Kubernetes' reconciliation model (desired state controllers). Use Terraform for the cluster, ArgoCD for the workloads.

**One giant Terraform configuration.** If your `terraform plan` takes 10 minutes and touches 200 resources, you need to split it up. By environment, by component, by team. Smaller state files mean faster plans, smaller blast radii, and easier debugging.

**Not using workspaces or separate state for environments.** If `terraform destroy` in dev could possibly affect production, your isolation is wrong.

**Ignoring drift.** Run `terraform plan` on a schedule (nightly in CI). If someone changed something manually, you want to know about it before it causes a problem, not when you're trying to apply the next change.

## The Practical Stack

Terraform provisions the AKS cluster, networking, Key Vault, container registry, and managed identities. It installs ArgoCD via Helm and creates the root Application. ArgoCD takes over from there, managing all workload deployments from a GitOps repo. State is in Azure Storage, one file per environment per component. CI runs plan on PRs, apply on merge.

It's not simple, but it's clear. Every piece has a defined scope and a defined owner. That clarity is worth more than any individual tool choice.
