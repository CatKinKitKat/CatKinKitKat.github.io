+++
title = "Bicep Curls: Azure IaC That Doesn't Suck"
date = 2026-03-01
[taxonomies]
tags = ["azure", "bicep", "iac", "devops"]
+++

I spent over a year doing infrastructure as code on Azure with Bicep, and I have opinions. Shocking, I know.

## Why Bicep Over Terraform

Look, Terraform is great. It's mature, it's multi-cloud, it has a massive ecosystem. But if you're **all-in on Azure** - and I mean actually committed, not just "we have one storage account" - Bicep has some real advantages:

- **Zero state file management.** Bicep deployments are declarative against Azure Resource Manager directly. No state file to lose, corrupt, or argue about where to store. If you've ever had a Terraform state lock at 5 PM on a Friday, you know why this matters.
- **Day-zero support.** New Azure resource? Bicep supports it immediately because it compiles to ARM templates. Terraform waits for the provider to be updated. Sometimes that's days. Sometimes it's weeks.
- **It's just Azure's language.** The tooling, the portal integration, the `what-if` deployment previews - it all works natively. No shims, no plugins.

The trade-off is obvious: you're locked to Azure. If multi-cloud is a real requirement (not a hypothetical one that someone in a meeting mentioned once), use Terraform. But if you're already married to Azure, Bicep is the better tool for the job.

## Patterns That Worked

### Module Everything

Bicep modules are the unit of reuse. One module per resource type, parametrized for environment. A virtual network module. A key vault module. An app service module. Each one takes the minimum parameters needed and has sensible defaults for everything else.

```bicep
module appService 'modules/app-service.bicep' = {
  name: 'app-${environment}'
  params: {
    location: location
    environment: environment
    appName: 'my-api'
    sku: environment == 'prod' ? 'P1v3' : 'B1'
  }
}
```

The ternary for SKU selection is a pattern I use everywhere. Dev gets the cheap tier, prod gets the real one, and you don't need a separate parameter file for it.

### Environment-Based Composition

One main Bicep file per environment is a trap. Instead, one main file that takes an environment parameter and conditionally includes what's needed. Prod needs a WAF? Conditional deployment. Dev needs a debugging container? Conditional deployment. The infrastructure definition stays in one place.

### Tagging as a First-Class Concern

Every resource gets tagged with `environment`, `project`, `owner`, and `managed-by: bicep`. This isn't optional. When someone inevitably creates a resource manually through the portal (and they will), the absence of the `managed-by` tag is how you find it.

## The CI/CD Side

Bicep deployments live in the same Jenkins pipeline as the application code. The flow:

1. `bicep build` to validate syntax and catch errors early.
2. `az deployment group what-if` to preview changes. This is non-negotiable - you review infra changes the same way you review code changes.
3. Deploy to staging, run smoke tests, deploy to prod.

The `what-if` step has saved me from destroying resources more than once. ARM sometimes interprets "update this property" as "delete and recreate the whole thing." Catching that before it hits production is... important.

## Things I Got Wrong

**Not parametrizing early enough.** My first Bicep modules had hardcoded values because "it's just dev, I'll fix it later." Later came faster than expected, and I spent an afternoon extracting parameters that should have been there from the start.

**Ignoring deployment ordering.** Bicep handles most dependencies automatically through resource references. But when you have cross-resource-group deployments or dependencies on resources that aren't in the same template, you need explicit `dependsOn`. I learned this the hard way when a Key Vault reference failed because the vault hadn't finished provisioning.

Nothing groundbreaking here. Just hard-earned common sense, written down so I don't make the same mistakes twice. Or at least not three times.
