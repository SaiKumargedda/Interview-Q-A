# DevOps Interview Q&A (Azure DevOps | Kubernetes | Terraform | Git)

---

# Azure DevOps

## 1. Difference between Pipeline Variables and Variable Groups

### Pipeline Variables

* Defined inside YAML or Pipeline UI.
* Used only by a single pipeline.
* Best for build-specific values like image tag, Build ID, namespace, etc.

### Variable Groups

* Stored under Azure DevOps Library.
* Shared across multiple pipelines.
* Used for common configuration and secrets.
* Can be integrated with Azure Key Vault.

### Interview Answer

> Pipeline Variables are pipeline-specific values such as image tags or build IDs, whereas Variable Groups are centrally managed and shared across multiple pipelines. In our project, we used Variable Groups for ACR name, AKS cluster, Resource Group, Key Vault references, and environment-specific configurations.

---

## 2. How does Parallel Execution work in Azure DevOps?

* Jobs without dependencies can run simultaneously.
* Azure DevOps automatically schedules them.
* Requires:

  * Available agents
  * Parallel job capacity/license

### Interview Answer

> Azure DevOps automatically executes independent jobs in parallel. No special YAML configuration is required. If multiple agents and parallel job licenses are available, the jobs run simultaneously; otherwise, they wait in the queue.

---

## 3. Where do we configure Parallel Job Capacity?

Location:

Organization Settings
→ Parallel Jobs

For self-hosted agents:

Organization Settings
→ Agent Pools

Both the number of licensed parallel jobs and available agents determine how many jobs can run concurrently.

---

## 4. Difference between dependsOn and condition

### dependsOn

* Controls execution order.

Example:

Build
↓
Test
↓
Deploy

### condition

* Controls whether a stage/job executes.

Examples:

condition: succeeded()

condition: failed()

condition: always()

condition: eq(variables['Build.SourceBranch'],'refs/heads/main')

### Interview Answer

> dependsOn controls execution order, whereas condition controls whether a stage/job runs after evaluating an expression. In our project we used dependsOn for stage sequencing and condition to deploy only from the main branch after successful builds.

---

## 5. Governance, Security and Compliance in CI/CD

Governance

* Branch Policies
* Mandatory PRs
* Build Validation
* Code Review

Security

* Azure Key Vault
* SonarQube
* Veracode
* Prisma Cloud
* Azure RBAC
* Service Connections

Compliance

* Manual Approvals
* Environment Checks
* Audit Logs
* Deployment History

### Interview Answer

> We enforced branch policies, mandatory pull requests, build validation, SonarQube quality gates, Veracode and Prisma Cloud scanning, Azure Key Vault for secrets, RBAC for least privilege, production approvals, and Azure DevOps audit logs for compliance.

---

## 6. Pipeline remains in Waiting/Running indefinitely

Troubleshooting

* Check Agent Availability
* Check Parallel Job Capacity
* Verify Agent Pool
* Check Environment Approvals
* Review Pipeline Logs
* Verify Service Connections
* Check SonarQube/Veracode availability
* Verify AKS deployment status

### Interview Answer

> I first determine whether the pipeline is waiting for an agent, approval, or an executing task. Based on where it's stuck, I verify agents, approvals, logs, service connections, external tools, and deployment targets.

---

## 7. Self-hosted Agent Offline

Troubleshooting Steps

* Verify agent status in Azure DevOps
* Login to VM
* Check Azure Pipelines Agent service
* Restart service
* Review _diag logs
* Verify Internet/DNS
* Check PAT expiration
* Check CPU/Disk
* Verify firewall/proxy
* Re-register agent if required

### Interview Answer

> I verify the agent status, restart the service if necessary, inspect logs, verify connectivity, PAT credentials, disk space, firewall/proxy settings, and finally run a test pipeline after restoring the agent.

---

# Kubernetes

## 8. How do you expose an application?

Options

* ClusterIP
* NodePort
* LoadBalancer
* Ingress

---

## 9. Ingress vs LoadBalancer

### Ingress

* Layer 7
* Host-based routing
* Path-based routing
* TLS termination
* One Public IP
* Ideal for microservices

### LoadBalancer

* Layer 4
* Single Service
* TCP/UDP
* One Public IP per service

### Interview Answer

> We generally use Ingress with AGIC in AKS because it supports host/path-based routing and TLS termination while exposing multiple services through a single public endpoint. LoadBalancer is used mainly for standalone or non-HTTP services.

---

## 10. Sidecar Containers

A Sidecar is an additional container running inside the same Pod.

Uses

* Logging
* Monitoring
* Service Mesh
* Secret Sync
* Configuration Sync

### Interview Answer

> Sidecars share the Pod network and volumes with the main application and provide supporting capabilities such as log collection, monitoring, or service mesh without modifying the application.

---

## 11. Pod Disruption Budget (PDB)

Purpose

Prevents too many Pods from being unavailable during voluntary disruptions.

Examples

minAvailable: 4

or

maxUnavailable: 1

### Interview Answer

> A PDB ensures high availability during node maintenance, upgrades, or autoscaling by limiting the number of Pods that can be disrupted at the same time.

---

## 12. AKS Scope

### Cluster Management

* AKS upgrades
* Node Pools
* RBAC
* Namespaces
* Monitoring
* Ingress
* Scaling

### Application Deployment

* Azure DevOps Pipelines
* Docker
* ACR
* Kubernetes Manifests
* Helm
* Rollouts/Rollbacks

### Interview Answer

> I was responsible for AKS operational management and application deployments. Infrastructure provisioning was handled through Terraform modules, while I managed namespaces, node pools, RBAC, monitoring, ingress, and CI/CD deployments.

---

# Git

## 13. git revert vs git reset vs git cherry-pick

### git revert

Use when:

* Commit already pushed
* Shared branch

Creates a new commit that undoes changes.

---

### git reset

Use when:

* Local branch
* Before pushing

Removes commits from history.

Avoid using on main after pushing.

---

### git cherry-pick

Use when:

* Copying one specific commit
* Hotfixes
* Production patches

### Interview Answer

> I use git revert for shared branches because it safely preserves history. I use git reset only for local or unpublished commits. I use git cherry-pick when I need to move a specific commit, such as a hotfix, from one branch to another.

---

## 14. Branching Strategy for Frequent Deployments

Recommended

Trunk-Based Development

Flow

Feature Branch
↓
Pull Request
↓
Code Review
↓
Merge to Main
↓
CI/CD

Why

* Small branches
* Fewer conflicts
* Faster deployments
* Continuous Integration

### Interview Answer

> For frequent deployments, I recommend Trunk-Based Development because developers merge small, short-lived feature branches frequently after PR validation, reducing conflicts and enabling continuous delivery.

---

# Terraform

## 15. What is null_resource?

A special Terraform resource that creates no infrastructure.

Used for:

* local-exec
* remote-exec
* triggers

Example Uses

* Execute shell scripts
* Trigger Ansible
* Configure AKS after creation

### Interview Answer

> null_resource doesn't create infrastructure. It is used to execute external scripts or commands when required.

---

## 16. Purpose of data block

Reads existing infrastructure.

Examples

* Existing Resource Group
* Existing VNet
* Existing ACR
* Existing Key Vault

### Interview Answer

> data blocks allow Terraform to reference existing resources without creating them.

---

## 17. Provisioners

Provisioners execute scripts during Terraform execution.

### local-exec

Runs commands on the Terraform machine.

Examples

* Execute scripts
* Trigger Ansible
* Update DNS

---

### remote-exec

Runs commands inside the provisioned VM.

Uses

* Install packages
* Configure software
* Execute shell commands

Requires SSH or WinRM.

---

### Difference

local-exec

* Runs locally
* No VM connection required

remote-exec

* Runs inside VM
* Requires SSH/WinRM

### Interview Answer

> local-exec runs commands on the machine executing Terraform, while remote-exec connects to the target VM and executes commands remotely. We generally avoid provisioners unless Terraform providers cannot perform the required action.

---

## 18. What is a Dynamic Block?

A dynamic block generates repeated nested blocks inside a resource using for_each.

Use Cases

* NSG Rules
* Ingress Rules
* Firewall Rules
* Multiple Node Pools

### Interview Answer

> Dynamic blocks eliminate duplicate code by generating nested blocks dynamically. They are useful for resources that contain a variable number of repeated configurations.

---

## 19. What is toset()?

Converts a list into a set.

Uses

* Removes duplicate values
* Enables for_each

Example

List

["dev","test","dev"]

Set

["dev","test"]

### Interview Answer

> toset() converts a list into a unique unordered collection. It is commonly used with for_each to iterate over unique values.
