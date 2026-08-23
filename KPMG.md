# Azure DevOps + Terraform + AKS Interview Preparation

A detailed README of questions and answers covering dynamic blocks, variable types, module design, centralized pipeline templates vs Terraform modules, state management, pipeline architecture, and AKS + Application Gateway/AGIC integration.

---

## Table of Contents

1. [Dynamic Blocks](#1-dynamic-blocks--what-are-they-and-how-are-they-implemented)
2. [Terraform Variable Types](#2-different-types-of-terraform-variables)
3. [Building Modules From Scratch](#3-have-you-created-terraform-modules-from-scratch)
4. [Thought Process for an AKS Module](#4-thought-process-while-creating-an-aks-module)
5. [AKS Module Example](#5-aks-module-example)
6. [How Root Module Calls Child Module](#6-how-root-module-calls-main-module)
7. [Modules in a Different Centralized Location (Same Repo)](#7-what-if-modules-are-in-a-different-centralized-location-in-azure-repos)
8. [Centralized Modules in a Different Repo](#8-centralized-terraform-modules-in-a-different-azure-repos-repository)
9. [Centralized Pipeline Templates in a Different Repo](#9-centralized-pipeline-templates-in-a-different-azure-repos-repository)
10. [Pipeline Templates vs Terraform Modules](#10-difference-between-centralized-pipeline-templates-and-terraform-modules)
11. [Conditional Resource Creation with count](#11-non-prod-resource-but-not-production-using-count)
12. [count vs for_each — Detailed](#12-count-vs-for_each)
13. [AKS Additional Node Pools Using for_each](#13-aks-additional-node-pools-using-for_each)
14. [One Pipeline vs Separate Pipelines](#14-one-pipeline-or-separate-non-prod-and-prod-pipelines)
15. [Selecting Subscriptions Across Environments](#15-different-azure-subscriptions--where-is-the-subscription-selected)
16. [Service Connection Selection in One Pipeline](#16-one-pipeline-for-prod-and-non-prod--how-is-service-connection-selected)
17. [State Files Per Environment](#17-terraform-state-files-for-different-environments)
18. [When Does State Lock Happen?](#18-terraform-state-lock--when-does-it-happen)
19. [What Happens If State Is Locked?](#19-what-happens-if-terraform-state-is-locked)
20. [Why State Corruption Happens](#20-why-terraform-state-corruption-happens)
21. [Apply Failed Halfway — Why?](#21-terraform-apply-failed-halfway--why)
22. [Recovering From a Half-Failed Apply](#22-what-do-you-do-after-a-half-failed-terraform-apply)
23. [Pipeline: Plan, Approval, Apply](#23-terraform-pipeline--plan-approval-apply)
24. [Why Save the Terraform Plan?](#24-why-save-the-terraform-plan)
25. [Pipeline Example With Centralized Templates](#25-azure-devops-terraform-pipeline-example-with-centralized-templates)
26. [Inside the Centralized Plan Template](#26-what-is-inside-the-centralized-terraform-plan-template)
27. [Inside the Apply Template](#27-what-is-inside-the-apply-template)
28. [How Azure DevOps Finds the Centralized Template](#28-how-does-azure-devops-know-where-the-centralized-template-is)
29. [Full Flow: Pipeline Template vs Terraform Module](#29-centralized-pipeline-template-vs-centralized-terraform-module--complete-flow)
30. [Centralized Module in a Different Repo (Recap)](#30-if-the-centralized-module-is-in-a-different-azure-repos-repository)
31. [Checking Out Multiple Repos](#31-if-multiple-azure-repos-are-checked-out-by-the-pipeline)
32. [Enforcing Approval Between Plan and Apply](#32-how-do-you-enforce-approval-after-plan-and-before-apply)
33. [Why Deployment Jobs for Environment Approval](#33-why-use-deployment-job-for-environment-approval)
34. [Apply Must Use the Reviewed Plan](#34-terraform-apply-should-use-the-reviewed-plan)
35. [Cloud Resources Created via Terraform](#35-cloud-resources-created-using-terraform)
36. [AKS + App Gateway + AGIC Flow](#36-aks--application-gateway--agic-working)
37. [Explaining Your Overall Architecture](#37-how-to-explain-your-overall-terraform-architecture)
38. [Short Interview Cheat Sheet](#38-short-interview-cheat-sheet)
39. [Best Overall Interview Response](#39-best-overall-interview-response)

---

## 1. Dynamic Blocks — What are they and how are they implemented?

### What is a dynamic block?

A Terraform **dynamic block** is used when a resource contains a repeatable nested block and we want Terraform to generate those nested blocks from a variable instead of hardcoding them.

Think of it as:

> "For every item in my input collection, generate one nested block."

Typical input types:
- `list(object(...))`
- `map(object(...))`
- sometimes `set(object(...))`

**Example:**

```hcl
variable "security_rules" {
  type = list(object({
    name     = string
    priority = number
    port     = string
  }))
}
```

```hcl
resource "azurerm_network_security_group" "nsg" {

  name                = "aks-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name

  dynamic "security_rule" {

    for_each = var.security_rules

    content {

      name     = security_rule.value.name
      priority = security_rule.value.priority

      direction = "Inbound"
      access    = "Allow"
      protocol  = "Tcp"

      source_port_range      = "*"
      destination_port_range = security_rule.value.port

      source_address_prefix      = "*"
      destination_address_prefix = "*"
    }
  }
}
```

If the variable contains three rules, Terraform generates three `security_rule` blocks.

### Important distinction: `for_each` vs `dynamic`

`for_each` and `dynamic` are **not** the same.

**`for_each`** — used to create multiple resource *instances*:

```hcl
resource "azurerm_storage_account" "sa" {
  for_each = var.storage_accounts
}
```

**`dynamic`** — used to generate multiple *nested blocks* inside one resource:

```hcl
dynamic "security_rule" {
  for_each = var.security_rules

  content {
    ...
  }
}
```

### AKS example — important correction

The AzureRM provider does **not** use a dynamic block inside `azurerm_kubernetes_cluster` to create additional AKS node pools. The default node pool is a nested block, but additional node pools are separate `azurerm_kubernetes_cluster_node_pool` resources.

Therefore, for additional AKS node pools, the production approach is normally **`for_each`**, not a dynamic block.

---

## 2. Different Types of Terraform Variables

**String**

```hcl
variable "location" {
  type = string
}
```
```hcl
location = "Central India"
```

**Number**

```hcl
variable "node_count" {
  type = number
}
```

**Boolean**

```hcl
variable "private_cluster" {
  type = bool
}
```

**List**

```hcl
variable "locations" {
  type = list(string)
}
```

**Set** (removes duplicate values)

```hcl
variable "zones" {
  type = set(string)
}
```

**Map**

```hcl
variable "tags" {
  type = map(string)
}
```
```hcl
tags = {
  environment = "dev"
  owner       = "platform"
}
```

**Object**

```hcl
variable "node_pool" {
  type = object({
    vm_size    = string
    node_count = number
  })
}
```

**List of objects** — very useful for reusable modules:

```hcl
variable "node_pools" {
  type = list(object({
    name       = string
    vm_size    = string
    node_count = number
  }))
}
```

For enterprise Terraform modules, `object`, `map(object)`, and `list(object)` are frequently useful because infrastructure resources usually require several related attributes.

---

## 3. Have you created Terraform modules from scratch?

**Yes.**

### Interview Answer

> "Yes. I have created reusable Terraform modules for resources such as AKS, Virtual Network, NSGs, ACR, Key Vault, Application Gateway, and monitoring resources. Before creating a module, I first understand the requirements, identify mandatory and optional inputs, identify dependencies, decide appropriate variable types, define outputs, and make sure the module is environment-agnostic and reusable."

### Typical Module Structure

```
modules/
└── aks/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── versions.tf
```

The module should **not** contain environment-specific values such as:

```hcl
cluster_name = "dev-aks"
```

Instead:

```hcl
cluster_name = var.cluster_name
```

The environment supplies the value.

---

## 4. Thought Process While Creating an AKS Module

The interviewer is usually testing your **design approach**, not your ability to type Terraform syntax.

### Step 1 — Understand requirements

For AKS, identify:

- Private or public cluster?
- Azure CNI or Kubenet?
- Azure RBAC?
- Managed Identity?
- System node pool?
- Additional user node pools?
- Cluster autoscaler?
- Availability Zones?
- Network policy?
- Monitoring?
- ACR integration?
- Key Vault integration?
- Kubernetes version?
- Tags?
- Maintenance windows?

### Step 2 — Separate mandatory and optional inputs

**Mandatory examples:** cluster name, resource group, location, subnet ID, Kubernetes version, system node pool.

**Optional examples:** private cluster, autoscaling, additional node pools, availability zones, network policy, monitoring, tags.

### Step 3 — Choose variable types

Simple values:
```hcl
cluster_name    = string
location        = string
private_cluster = bool
```

Tags: `map(string)`

Multiple node pools:
```hcl
map(object({
  vm_size             = string
  node_count          = number
  mode                = string
  enable_auto_scaling = bool
  min_count           = number
  max_count           = number
}))
```

### Step 4 — Identify dependencies

```
Resource Group
      |
      v
VNet
      |
      v
AKS Subnet
      |
      v
Managed Identity
      |
      v
Log Analytics
      |
      v
AKS
```

The AKS module should not necessarily create the VNet itself — it can receive the subnet ID from the network module:

```hcl
subnet_id = module.network.aks_subnet_id
```

### Step 5 — Define outputs

```hcl
output "aks_id" {
  value = azurerm_kubernetes_cluster.aks.id
}

output "node_resource_group" {
  value = azurerm_kubernetes_cluster.aks.node_resource_group
}

output "principal_id" {
  value = azurerm_kubernetes_cluster.aks.identity[0].principal_id
}
```

### Step 6 — Keep the module reusable

The same module should work for Dev, QA, UAT, and Prod without changing the module code — only the input values should change.

---

## 5. AKS Module Example

### Root module

```hcl
module "aks" {

  source = "../../modules/aks"

  cluster_name         = var.cluster_name
  location             = var.location
  resource_group_name  = module.resource_group.name
  subnet_id            = module.network.aks_subnet_id

  kubernetes_version   = var.kubernetes_version
  vm_size              = var.vm_size
  node_count           = var.node_count
}
```

### Module `variables.tf`

```hcl
variable "cluster_name" {
  type = string
}

variable "location" {
  type = string
}

variable "resource_group_name" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "kubernetes_version" {
  type = string
}

variable "vm_size" {
  type = string
}

variable "node_count" {
  type = number
}
```

### Module `main.tf`

```hcl
resource "azurerm_kubernetes_cluster" "aks" {

  name                = var.cluster_name
  location            = var.location
  resource_group_name = var.resource_group_name

  dns_prefix         = var.cluster_name
  kubernetes_version = var.kubernetes_version

  default_node_pool {
    name       = "system"
    vm_size    = var.vm_size
    node_count = var.node_count
  }

  identity {
    type = "SystemAssigned"
  }
}
```

---

## 6. How Root Module Calls Main Module

This is a key concept.

Terraform starts from the directory where we execute `terraform init` / `terraform plan`. That directory is the **Root Module**.

```
terraform-infra/
│
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── backend.tf
│   │   ├── providers.tf
│   │   └── dev.tfvars
│   │
│   └── prod/
│       ├── main.tf
│       ├── backend.tf
│       ├── providers.tf
│       └── prod.tfvars
│
└── modules/
    ├── aks/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── vnet/
    └── acr/
```

Pipeline:

```bash
cd environments/dev
terraform init
terraform plan
```

Terraform starts with `environments/dev/main.tf`. The root module contains:

```hcl
module "aks" {
  source = "../../modules/aks"

  cluster_name         = var.cluster_name
  location             = var.location
  resource_group_name  = module.resource_group.name
  subnet_id            = module.network.aks_subnet_id
}
```

Terraform follows:

```
environments/dev/main.tf
        |
        | source = ../../modules/aks
        v
modules/aks/
        |
        +-- main.tf
        +-- variables.tf
        +-- outputs.tf
```

Terraform then uses the resources defined in `modules/aks/main.tf`.

**Important:** Azure DevOps does **not** call the Terraform module. Terraform resolves the module source. Azure DevOps only runs the Terraform commands.

---

## 7. What if Modules Are in a Different Centralized Location in Azure Repos?

Two important cases.

### Case A — Same Azure Repos repository, different folder

```
Terraform-Infra/
│
├── environments/
│   └── dev/
│       └── main.tf
│
└── modules/
    └── aks/
        └── main.tf
```

Use a relative path:

```hcl
module "aks" {
  source = "../../modules/aks"
}
```

Terraform resolves the path relative to the root module.

---

## 8. Centralized Terraform Modules in a Different Azure Repos Repository

```
Azure DevOps Project
│
├── Application-Infrastructure Repo
│   └── environments/dev/main.tf
│
├── Shared-Terraform-Modules Repo
│   ├── aks/
│   ├── vnet/
│   └── keyvault/
│
└── Shared-Pipeline-Templates Repo
```

Reference the module using the Azure Repos Git source:

```hcl
module "aks" {

  source = "git::https://dev.azure.com/Company/Platform/_git/Shared-Terraform-Modules//aks?ref=v2.0.0"

  cluster_name = var.cluster_name
  location     = var.location
  subnet_id    = module.network.aks_subnet_id
}
```

Terraform downloads the module during `terraform init`. This is different from a centralized pipeline template.

---

## 9. Centralized Pipeline Templates in a Different Azure Repos Repository

```
Application Repo
└── azure-pipelines.yml

Shared-DevOps Repo
└── templates/
    ├── terraform-plan.yml
    └── terraform-apply.yml
```

In the application pipeline:

```yaml
resources:
  repositories:
  - repository: sharedTemplates
    type: git
    name: Platform/Shared-DevOps
    ref: refs/heads/main
```

Then:

```yaml
stages:

- stage: TerraformPlan

  jobs:

  - template: templates/terraform-plan.yml@sharedTemplates
```

Azure DevOps sees `@sharedTemplates` and knows which repository to use. The path `templates/terraform-plan.yml` tells it which YAML file to load.

The pipeline can pass parameters:

```yaml
- template: templates/terraform-plan.yml@sharedTemplates

  parameters:
    environment: dev
    tfvarsFile: environments/dev.tfvars
```

### Important distinction

```
Azure DevOps
    |
    +-- calls centralized YAML templates
    |      using template: ...@alias
    |
    +-- runs terraform init
             |
             v
          Terraform
             |
             +-- calls centralized Terraform modules
                    using module source = ...
```

---

## 10. Difference Between Centralized Pipeline Templates and Terraform Modules

| Item | Pipeline Template | Terraform Module |
|---|---|---|
| Used by | Azure DevOps | Terraform |
| Purpose | Reuse pipeline logic | Reuse infrastructure code |
| Called using | `template:` | `module` + `source` |
| Example | `terraform-plan.yml` | `aks/main.tf` |
| Repository alias | `@sharedTemplates` | Git source or local path |
| Executes | Pipeline tasks | Azure resource definitions |

### Strong interview statement

> "Azure DevOps imports pipeline templates, whereas Terraform resolves Terraform modules. The pipeline does not directly call a Terraform module."

---

## 11. Non-Prod Resource but Not Production Using count

Suppose a resource is required in Dev and QA but not Prod.

```hcl
variable "environment" {
  type = string
}
```

```hcl
resource "azurerm_storage_account" "logs" {

  count = var.environment == "prod" ? 0 : 1

  name                      = var.storage_name
  resource_group_name       = var.resource_group_name
  location                  = var.location

  account_tier              = "Standard"
  account_replication_type  = "LRS"
}
```

Terraform conditional syntax: `condition ? value_if_true : value_if_false`

| Environment | Condition | Result |
|---|---|---|
| Dev | `dev == prod` → false | `count = 1` → created |
| QA | `qa == prod` → false | `count = 1` → created |
| Prod | `prod == prod` → true | `count = 0` → skipped |

The pipeline does not fail. Terraform considers the resource intentionally absent.

---

## 12. count vs for_each

### `count`

```hcl
count = 3
```

Terraform creates `resource[0]`, `resource[1]`, `resource[2]`. `count.index` is available.

**Good for:**
- Optional resources
- Fixed number of similar resources
- Simple enable/disable logic

```hcl
count = var.environment == "prod" ? 0 : 1
```

### `for_each`

```hcl
for_each = var.node_pools
```

Terraform creates instances based on keys.

```hcl
node_pools = {
  app = {
    vm_size    = "Standard_D4s_v5"
    node_count = 2
  }

  batch = {
    vm_size    = "Standard_D8s_v5"
    node_count = 1
  }
}
```

Access: `each.key`, `each.value`

**Advantages:**
- Key-based identity
- Stable resource addressing
- Better for collections
- Adding/removing one item usually does not affect unrelated instances

**Rule of thumb:** For optional single resources, `count` is usually simpler. For collections such as node pools, `for_each` is usually preferred.

---

## 13. AKS Additional Node Pools Using for_each

```hcl
variable "node_pools" {
  type = map(object({
    vm_size             = string
    node_count          = number
    mode                = string
    enable_auto_scaling = bool
    min_count           = number
    max_count           = number
  }))
}
```

```hcl
node_pools = {
  app = {
    vm_size             = "Standard_D4s_v5"
    node_count          = 2
    mode                = "User"
    enable_auto_scaling = true
    min_count           = 2
    max_count           = 5
  }

  batch = {
    vm_size             = "Standard_D8s_v5"
    node_count          = 1
    mode                = "User"
    enable_auto_scaling = true
    min_count           = 1
    max_count           = 3
  }
}
```

```hcl
resource "azurerm_kubernetes_cluster_node_pool" "userpool" {

  for_each = var.node_pools

  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id

  name       = each.key
  vm_size    = each.value.vm_size
  node_count = each.value.node_count
  mode       = each.value.mode

  enable_auto_scaling = each.value.enable_auto_scaling
  min_count            = each.value.min_count
  max_count            = each.value.max_count
}
```

---

## 14. One Pipeline or Separate Non-Prod and Prod Pipelines?

A common enterprise approach is to use **one reusable pipeline** for all environments.

```
azure-pipelines.yml
      |
      +-- Dev
      +-- QA
      +-- UAT
      +-- Prod
```

The same pipeline selects: environment, tfvars file, state file, service connection, approval behavior.

```
Dev  -> dev.tfvars  -> dev.tfstate
QA   -> qa.tfvars   -> qa.tfstate
Prod -> prod.tfvars -> prod.tfstate
```

Some organizations use separate Prod and Non-Prod pipelines because of stronger production governance. Both are valid.

### Interview Answer

> "We preferred a single reusable Terraform pipeline with environment-specific parameters and centralized templates. Production had additional approval and governance checks through Azure DevOps Environments."

---

## 15. Different Azure Subscriptions — Where is the Subscription Selected?

Suppose:
- Dev Subscription
- QA Subscription
- Prod Subscription

Normally each subscription has an Azure DevOps Service Connection: `SC-DEV`, `SC-QA`, `SC-PROD`.

The Service Connection provides Azure authentication and targets the appropriate subscription. The `.tfvars` file normally contains infrastructure configuration such as:

```hcl
location     = "Central India"
cluster_name = "dev-aks"
node_count   = 2
```

The subscription is generally selected by the **Azure DevOps Service Connection** rather than treating the `.tfvars` file as the authentication mechanism.

It's technically possible to pass a subscription ID as a Terraform variable, but enterprise pipelines commonly use service connections for authentication and subscription targeting.

---

## 16. One Pipeline for Prod and Non-Prod — How is Service Connection Selected?

Suppose: `SC-DEV`, `SC-QA`, `SC-PROD`.

Pipeline parameter:

```yaml
parameters:
- name: environment
  type: string
  values:
  - dev
  - qa
  - prod
```

Conditional mapping:

```yaml
variables:

  ${{ if eq(parameters.environment, 'dev') }}:
    serviceConnection: 'SC-DEV'

  ${{ if eq(parameters.environment, 'qa') }}:
    serviceConnection: 'SC-QA'

  ${{ if eq(parameters.environment, 'prod') }}:
    serviceConnection: 'SC-PROD'
```

Then the selected connection is passed to the Azure/Terraform task.

```
environment = dev
      |
      v
SC-DEV
      |
      v
Dev Subscription
```

```
environment = prod
      |
      v
SC-PROD
      |
      v
Prod Subscription
```

**Important:** The pipeline should only allow authorized/predefined service connections. The service connection name should not be treated as an arbitrary user-provided string without proper pipeline permissions.

---

## 17. Terraform State Files for Different Environments

Each environment should have independent state.

```
Azure Storage Account
        |
        +-- tfstate container
              |
              +-- dev.tfstate
              +-- qa.tfstate
              +-- prod.tfstate
```

Backend configuration:

```hcl
terraform {
  backend "azurerm" {}
}
```

Pipeline supplies the backend key:

```bash
terraform init \
  -backend-config="resource_group_name=tfstate-rg" \
  -backend-config="storage_account_name=tfstate123" \
  -backend-config="container_name=tfstate" \
  -backend-config="key=dev.tfstate"
```

For production:

```bash
terraform init \
  -backend-config="key=prod.tfstate"
```

The `key` identifies the state file. The same Storage Account/container can be used, but each environment must have a separate state key.

---

## 18. Terraform State Lock — When Does It Happen?

Terraform locks state whenever it needs to **modify** state, not only during simultaneous `terraform apply`.

Examples:
- `terraform apply`
- `terraform destroy`
- `terraform import`
- `terraform state mv`
- `terraform state rm`
- `terraform state replace-provider`
- other state-changing operations

With an AzureRM backend, state locking is implemented using an **Azure Blob lease**.

```
Terraform operation
       |
       v
Acquire Blob lease
       |
       v
Modify state
       |
       v
Release lease
```

If another Terraform process tries to modify the same state while the lock is held, it cannot safely proceed.

---

## 19. What Happens If Terraform State Is Locked?

First determine whether another Terraform operation is actually running.

Check:
- Azure DevOps pipeline runs
- Another engineer's Terraform operation
- Scheduled pipeline
- Long-running AKS/resource operation

**Do not immediately force-unlock.**

If the lock is stale because a process crashed and no Terraform operation is running, `terraform force-unlock` may be used with the lock ID after proper verification:

```bash
terraform force-unlock <LOCK_ID>
```

**Never** force-unlock while another Terraform operation is actively running.

---

## 20. Why Terraform State Corruption Happens

State corruption is uncommon with a properly configured remote backend, but possible causes include:

- Manual editing of `.tfstate`
- Interrupted or failed state writes
- Backend/storage connectivity issues
- Incorrect backend configuration
- Using different backends unintentionally
- Improper state manipulation commands
- Operational mistakes while recovering state

**Manual state editing should be avoided.**

### If state is damaged:

1. Check the actual Azure resources.
2. Check the remote state.
3. Check Azure Storage blob versions if enabled.
4. Restore a known-good previous state version if appropriate.
5. Use `terraform import` for resources that exist but are missing from state.
6. Run `terraform plan`.
7. Never blindly overwrite state.

---

## 21. Terraform Apply Failed Halfway — Why?

Terraform is not necessarily atomic across all Azure resources.

```
Resource Group     -> created
VNet                -> created
Subnet              -> created
AKS                 -> failed
```

### Possible reasons

- **Permission issue** — Terraform identity can create networking but does not have the required AKS permissions.
- **Azure API failure/timeout** — Azure resource provisioning can fail or time out.
- **Quota** — VM quota, public IP quota, regional capacity.
- **Dependency failure** — a resource required by another resource failed.
- **Resource already exists** — a resource was manually created in Azure and is not represented correctly in Terraform state.
- **Network interruption** — the pipeline agent loses connectivity.
- **Provider/resource error** — invalid configuration or unsupported resource settings.
- **Resource provider registration** — required Azure Resource Provider may not be registered.

---

## 22. What Do You Do After a Half-Failed Terraform Apply?

**Do not immediately run `terraform destroy`.** First investigate.

**Step 1 — Read pipeline logs.** Identify the exact resource and error.

**Step 2 — Check Terraform state.**
```bash
terraform state list
```

**Step 3 — Check Azure.** Verify which resources actually exist.

**Step 4 — Compare state and Azure.**

Possible situation: Azure resource exists but Terraform state does not contain it. If the resource should be Terraform-managed, import it:

```bash
terraform import <resource_address> <azure_resource_id>
```

**Step 5 — Fix root cause** (permission, quota, invalid configuration, network, dependency).

**Step 6 — Run plan.**
```bash
terraform plan
```
Review what Terraform wants to change.

**Step 7 — Apply.**
```bash
terraform apply
```

Never manually edit the `.tfstate` file as the normal recovery approach.

---

## 23. Terraform Pipeline — Plan, Approval, Apply

A production Terraform pipeline should normally separate planning and applying.

```
Code
 |
 v
Terraform fmt
 |
 v
Terraform validate
 |
 v
Terraform init
 |
 v
Terraform plan -out=tfplan
 |
 v
Publish tfplan artifact
 |
 v
Manual approval
 |
 v
Download tfplan
 |
 v
terraform apply tfplan
```

**The important principle:** Review the plan first, then apply the exact plan that was reviewed.

---

## 24. Why Save the Terraform Plan?

```bash
terraform plan -out=tfplan
```

Then publish `tfplan`. After approval:

```bash
terraform apply tfplan
```

This is preferable to generating a completely new plan after approval because the saved plan represents the proposed changes that were reviewed.

In a real production pipeline, also ensure the configuration, backend, provider versions, credentials, and workspace/state context used for apply are consistent with the plan stage.

---

## 25. Azure DevOps Terraform Pipeline Example With Centralized Templates

### Application repository

```
Application-Infrastructure
│
├── azure-pipelines.yml
├── environments/
│   ├── dev.tfvars
│   ├── qa.tfvars
│   └── prod.tfvars
│
└── terraform/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

### Shared pipeline repository

```
Shared-DevOps
│
└── templates/
    ├── terraform-plan.yml
    └── terraform-apply.yml
```

### Main pipeline

```yaml
resources:
  repositories:
  - repository: sharedTemplates
    type: git
    name: Platform/Shared-DevOps
    ref: refs/heads/main

parameters:
- name: environment
  type: string
  values:
  - dev
  - qa
  - prod

stages:

- stage: TerraformPlan

  jobs:

  - template: templates/terraform-plan.yml@sharedTemplates

    parameters:
      environment: ${{ parameters.environment }}
      tfvarsFile: environments/${{ parameters.environment }}.tfvars


- stage: TerraformApply

  dependsOn: TerraformPlan

  jobs:

  - deployment: TerraformApply

    environment: ${{ parameters.environment }}

    strategy:
      runOnce:
        deploy:

          steps:

          - template: templates/terraform-apply.yml@sharedTemplates
```

> The exact Azure DevOps task syntax can vary depending on whether the organization uses Terraform CLI scripts, TerraformTask extensions, or another approved implementation. The architecture remains the same.

---

## 26. What Is Inside the Centralized Terraform Plan Template?

```yaml
parameters:

- name: environment
  type: string

- name: tfvarsFile
  type: string

jobs:

- job: TerraformPlan

  steps:

  - script: terraform fmt -check

  - script: terraform init

  - script: terraform validate

  - script: |
      terraform plan \
        -var-file=${{ parameters.tfvarsFile }} \
        -out=tfplan

  - publish: tfplan
    artifact: terraformPlan
```

The main pipeline doesn't repeat these steps — it simply calls:

```yaml
- template: templates/terraform-plan.yml@sharedTemplates
```

---

## 27. What Is Inside the Apply Template?

```yaml
parameters:

- name: environment
  type: string

steps:

- download: current
  artifact: terraformPlan

- script: terraform init

- script: terraform apply tfplan
```

The actual production template should also ensure that the correct working directory, backend configuration, provider authentication, and plan artifact are used consistently.

---

## 28. How Does Azure DevOps Know Where the Centralized Template Is?

This is the key concept.

**First:**

```yaml
resources:
  repositories:
  - repository: sharedTemplates
    type: git
    name: Platform/Shared-DevOps
```

This registers the external Azure Repos repository and gives it the alias `sharedTemplates`.

**Then:**

```yaml
- template: templates/terraform-plan.yml@sharedTemplates
```

means:

```
Go to repository alias sharedTemplates
        |
        v
Find templates/
        |
        v
Find terraform-plan.yml
        |
        v
Load it into this pipeline
```

Azure DevOps retrieves the referenced template. You do not manually copy the YAML into the application repository.

---

## 29. Centralized Pipeline Template vs Centralized Terraform Module — Complete Flow

This is very important for interviews.

```
Developer
   |
   v
azure-pipelines.yml
   |
   +------------------------------+
   |                              |
   v                              |
Azure DevOps                      |
   |                              |
   | template: ...@sharedTemplates
   v                              |
Shared Pipeline Template          |
   |                              |
   v                              |
terraform init -------------------+
   |
   v
Terraform reads main.tf
   |
   v
module "aks"
source = ...
   |
   v
Centralized Terraform Module
   |
   v
AKS Resource
```

The two calls are made by different tools:

- **Azure DevOps** → pipeline template
- **Terraform** → Terraform module

---

## 30. If the Centralized Module Is in a Different Azure Repos Repository

```
Application Repo
    |
    +-- environments/dev/main.tf

Shared Modules Repo
    |
    +-- aks/
        +-- main.tf
        +-- variables.tf
        +-- outputs.tf
```

Root module:

```hcl
module "aks" {

  source = "git::https://dev.azure.com/Company/Platform/_git/Shared-Terraform-Modules//aks?ref=v2.0.0"

  cluster_name = var.cluster_name
  location     = var.location
  subnet_id    = module.network.aks_subnet_id
}
```

Terraform downloads it during `terraform init`. This is different from the pipeline template mechanism.

---

## 31. If Multiple Azure Repos Are Checked Out by the Pipeline

Another possible enterprise design is to check out multiple repositories.

```yaml
resources:
  repositories:
  - repository: modules
    type: git
    name: Platform/Shared-Terraform-Modules
```

Then:

```yaml
steps:
- checkout: self
- checkout: modules
```

The build agent has both repositories available. Terraform can then reference the local checked-out module path, provided the exact workspace path is known and stable.

However, when the module repository is intended to be independently versioned and reused, using Terraform's Git module source with a pinned ref/tag is often cleaner.

---

## 32. How Do You Enforce Approval After Plan and Before Apply?

Use **Azure DevOps Environments**.

```
Terraform Plan
      |
      v
Publish plan
      |
      v
Prod Environment
      |
      v
Approval and Checks
      |
      v
Terraform Apply
```

The Apply stage is a deployment job:

```yaml
- deployment: TerraformApply

  environment: Prod

  strategy:
    runOnce:
      deploy:
        steps:
        - ...
```

In Azure DevOps:

```
Pipelines
   |
   v
Environments
   |
   v
Prod
   |
   v
Approvals and Checks
   |
   v
Manual Approval
```

The approver can review the Terraform plan before approving. If rejected, Apply does not execute.

---

## 33. Why Use Deployment Job for Environment Approval?

Azure DevOps Environments are used with **deployment jobs**.

```yaml
- deployment: DeployInfra
  environment: Prod
```

The environment provides:

- Approvals
- Checks
- Deployment history
- Audit trail
- Environment-level governance

A normal build job is not the same as a deployment job and does not provide the same environment deployment mechanism.

---

## 34. Terraform Apply Should Use the Reviewed Plan

**Plan:**

```bash
terraform plan -out=tfplan
```

**Publish:** `terraformPlan` artifact

**After approval:**

```bash
terraform apply tfplan
```

This provides a strong plan-review/apply workflow.

---

## 35. Cloud Resources Created Using Terraform

**Resource Management**
- Resource Groups

**Networking**
- VNet
- Subnets
- NSG
- Route Tables
- NAT Gateway
- Azure Firewall
- VNet Peering
- Private Endpoints
- Private DNS Zones

**AKS**
- AKS Cluster
- System Node Pool
- User Node Pools
- Managed Identity
- Azure RBAC configuration
- Autoscaling configuration
- Monitoring integration

**Container**
- Azure Container Registry

**Security**
- Key Vault
- Managed Identities
- Role Assignments

**Ingress**
- Application Gateway
- WAF configuration
- AGIC-related Azure infrastructure

**Monitoring**
- Log Analytics Workspace
- Application Insights
- Diagnostic Settings
- Azure Monitor-related resources

**Storage**
- Storage Account used for Terraform remote state

**DNS/Traffic** (project-dependent)
- Azure DNS
- Front Door
- Traffic Manager

---

## 36. AKS + Application Gateway + AGIC Working

### Typical flow

```
Internet
   |
   v
Azure Front Door
   |
   v
Application Gateway / WAF
   |
   v
AGIC
   |
   v
Kubernetes Ingress
   |
   v
Kubernetes Service
   |
   v
Pod
```

**AGIC** = Application Gateway Ingress Controller. It watches Kubernetes Ingress resources.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

AGIC reads the Kubernetes Ingress configuration and configures the Application Gateway accordingly.

**Application Gateway can provide:**
- Layer 7 routing
- Host-based routing
- Path-based routing
- TLS termination
- WAF
- Health probes

---

## 37. How to Explain Your Overall Terraform Architecture

### Strong Interview Answer

> "We followed a modular Terraform architecture. Environment folders acted as root modules, and reusable infrastructure components such as networking, AKS, ACR, Key Vault, Application Gateway, and monitoring were implemented as child modules. The root module passed environment-specific inputs into those modules. We maintained separate state files for each environment in an Azure Storage backend. The Azure DevOps pipeline was standardized using centralized YAML templates. The same pipeline could be used for Dev, QA, and Prod, while the environment selected the appropriate tfvars file, backend state key, and Azure service connection. Production deployments were protected with Azure DevOps Environment approvals."

---

## 38. Short Interview Cheat Sheet

| Term | Definition |
|---|---|
| **Dynamic block** | Generates repeated nested blocks from a collection |
| **count** | Creates zero, one, or multiple resource instances based on a number |
| **for_each** | Creates resource instances based on collection keys |
| **Terraform module** | Reusable Terraform infrastructure code |
| **Root module** | Directory from which Terraform is executed |
| **Child module** | Module called by the root module |
| **Module source** | `source = "../../modules/aks"` or a centralized repository source |
| **Pipeline template** | `- template: templates/terraform-plan.yml@sharedTemplates` |
| **State key** | `dev.tfstate`, `prod.tfstate` |
| **State lock** | Prevents concurrent state modification |
| **Half apply** | Investigate logs, compare Azure and state, fix root cause, import missing resources if required, then plan/apply again |
| **Approval** | Azure DevOps Environment + Approvals and Checks |
| **Production** | Same reusable pipeline can be used, but Prod has stricter service connection permissions and mandatory approvals |
| **Subscription** | Normally selected through the Azure DevOps service connection rather than relying on tfvars for authentication |

---

## 39. Best Overall Interview Response

If the interviewer asks you to explain your complete Terraform/Azure DevOps approach, use this:

> "We followed a reusable and environment-independent Terraform architecture. Our environment folders acted as root modules, and reusable components such as AKS, networking, ACR, Key Vault, Application Gateway, and monitoring were maintained as child modules. The root module called these modules using the source attribute and passed environment-specific variables.
>
> For shared CI/CD logic, we maintained centralized Azure DevOps YAML templates in a shared Azure Repos repository. The application pipeline registered that repository under resources.repositories and called templates using template: path@repositoryAlias. So Azure DevOps handled pipeline template reuse, while Terraform handled Terraform module reuse.
>
> We used a single parameterized pipeline across environments where appropriate. The selected environment determined the tfvars file, Terraform backend state key, and Azure service connection. Each environment had an independent state file, such as dev.tfstate and prod.tfstate.
>
> The Terraform pipeline was separated into Plan and Apply stages. The Plan stage performed init, validate, and plan and published the plan artifact. Production Apply was implemented as a deployment job targeting the Prod Azure DevOps Environment, where manual approvals and checks were configured. After approval, the pipeline downloaded and applied the reviewed Terraform plan.
>
> This gave us reusable infrastructure, environment isolation, standardized pipelines, production governance, and a controlled plan-review-apply process."

---

*Document prepared for interview preparation — Azure DevOps + Terraform + AKS architecture, centralized modules/templates, state management, and pipeline governance.*
