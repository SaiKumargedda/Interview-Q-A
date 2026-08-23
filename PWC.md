# Azure / AKS / Terraform Interview Questions & Answers — Complete README

A comprehensive reference covering Azure observability tools, Terraform failure recovery, AKS cost optimization, Kubernetes deployment strategies, production incident troubleshooting, and High Availability / Reliability / Cost design for AKS.

---

## Table of Contents

1. [Which Tool for Traces in Azure?](#q1-which-tool-do-we-use-in-azure-for-traces)
2. [Terraform Apply Failing on Existing Infrastructure](#q2-if-you-want-to-do-changes-in-existing-infrastructure-and-terraform-apply-is-failing-what-do-you-do)
3. [Do We Use Terraform Taint?](#q3-do-we-use-terraform-taint-here)
4. [Is HPA/CA a Cost Reduction Strategy?](#q4-is-hpa-and-ca-not-a-cost-reduction-strategy)
5. [AKS/Azure Cost Optimization Approaches](#q5-what-are-the-other-approachestools-that-can-be-used-for-aksazure-cost-optimization)
6. [Kubernetes Deployment Strategies Overview](#q6-what-are-the-different-deployment-strategies-in-kubernetes-what-are-proscons-and-what-strategy-do-you-suggest-in-real-time)
7. [Rolling Deployment Deep Dive](#q7-explain-rolling-deployment-in-detail-why-are-coexistence-backward-compatibility-database-compatibility-longer-deployment-and-troubleshooting-considered-cons)
8. [Main Kubernetes Deployment Strategies Compared](#q8-what-are-the-main-kubernetes-deployment-strategies)
9. [Production Incident Walkthrough — Monitoring Tools](#q9-give-one-production-issue-where-azure-monitoring-tools-were-used-for-logs-metrics-traces-and-queries-explain-in-real-time)
10. [Designing HA, Reliability & Cost-Effective AKS](#q10-deep-explanation-of-designing-high-availability-reliability-and-cost-effective-aks)
11. [Budget-Constrained HA — RTO/RPO/Log Retention](#q11-the-interviewer-asked-every-project-does-not-have-budget-for-this-within-allocated-budget-how-well-can-you-implement-it-explain-rtorpo-log-retention-old-log-removal-storage-and-backup)
12. [HA vs Reliability vs Cost Optimization](#q12-what-is-the-difference-between-high-availability-reliability-and-cost-optimization)
13. [Best Production AKS Design](#q13-what-is-the-best-production-aks-design-considering-ha-reliability-and-cost)

[Final Interview Cheat Sheet](#final-interview-cheat-sheet)

---

## Q1. Which tool do we use in Azure for traces?

In Azure, for application traces, the main tool used is **Application Insights**.

### Azure Observability Stack

| Purpose | Azure Tool |
|---|---|
| Traces / distributed tracing | Application Insights |
| Application logs | Application Insights |
| Container/AKS logs | Log Analytics + Container Insights |
| Metrics | Azure Monitor |
| Infrastructure alerts | Azure Monitor |
| Query logs/traces | Log Analytics (KQL) |

### Interview Answer

> "We use Azure Application Insights for application-level telemetry and distributed tracing. It captures requests, dependencies, exceptions, and trace information, allowing us to follow a request across different microservices. The telemetry is stored/queryable through the Azure Monitor ecosystem, and we use KQL in Log Analytics for troubleshooting."

### Important Distinction: Logs ≠ Traces ≠ Metrics

- **Logs:** What happened? → application/container log messages
- **Traces:** How did a request travel through multiple services? → Application Insights / distributed tracing
- **Metrics:** How much/how often? → CPU, memory, request rate, latency, etc.

If using modern Azure instrumentation, **OpenTelemetry** can also be used to generate traces and send them to Application Insights/Azure Monitor.

---

## Q2. If you want to do changes in existing infrastructure and Terraform apply is failing, what do you do?

Answer this as a **controlled troubleshooting and recovery process**, not by immediately changing infrastructure manually.

### Interview Answer

> "If Terraform apply fails while making changes to existing infrastructure, first I don't retry blindly. I check the Terraform plan and the exact error from the apply output. I identify whether the failure is due to configuration, permissions, dependency, provider/API issue, or an existing resource drift."
>
> "I then check the Terraform state and Azure resource state to make sure Terraform's state is consistent with the actual infrastructure. If there is drift, I handle it through Terraform — for example, refresh/plan or import if the resource is missing from state. I fix the Terraform code or dependency issue, run terraform validate and terraform plan, review the proposed changes, and then run terraform apply again."
>
> "If the apply partially succeeded, I don't manually recreate or delete resources. I first determine what Terraform successfully created or changed and what failed. Since Terraform state is updated as resources are successfully managed, I use the subsequent plan to understand the remaining changes. If necessary, I restore from the previous known-good configuration or use Terraform lifecycle controls rather than making uncontrolled Azure Portal changes."

### Scenario Example

Suppose you change an AKS node pool:

```
Terraform plan
      ↓
terraform apply
      ↓
AKS change succeeds
      ↓
Later resource fails
      ↓
Check apply error
      ↓
Check terraform state + Azure
      ↓
Fix root cause
      ↓
terraform plan
      ↓
Review remaining changes
      ↓
terraform apply
```

### If the interviewer asks: "What if production is impacted?"

> "First I assess the impact and stop further changes. If the change is causing an outage and we have a known-good Terraform configuration, I revert the Terraform code through the normal change-management process and apply it. I avoid making manual Portal changes because that can introduce state drift."

**Key interview point:** Don't say "I delete the resource and recreate it." For existing enterprise infrastructure, the preferred approach is: identify the failure → verify state vs Azure → fix root cause → plan → controlled apply.

---

## Q3. Do we use Terraform taint here?

**Yes**, `terraform taint` can be used, but not as the first action when `terraform apply` fails.

### Interview Answer

> "If Terraform apply fails, I first troubleshoot the actual failure. If a specific resource is in a bad or inconsistent state and I need Terraform to destroy and recreate that resource, I can use terraform taint. However, I use it carefully, especially in production, because taint forces recreation."

```bash
terraform taint azurerm_linux_virtual_machine.app
terraform plan
terraform apply
```

Terraform will show that the resource is tainted and needs replacement.

### Important: Modern Terraform

In newer Terraform versions, `terraform taint` is **deprecated**. The preferred approach:

```bash
terraform apply -replace="azurerm_linux_virtual_machine.app"
```

> "Earlier we used terraform taint to mark a resource for recreation, but in current Terraform versions I prefer terraform apply -replace, because taint is deprecated."

### In the Failed-Apply Scenario

**Don't** automatically do `terraform taint`. Instead:

```
Apply failed
   ↓
Check error
   ↓
Check Terraform state + Azure resource
   ↓
Identify problematic resource
   ↓
If recreation is genuinely required
   ↓
terraform plan -replace="resource"
   ↓
Review impact
   ↓
terraform apply
```

This is the safer enterprise answer, especially for production AKS/Azure infrastructure.

---

## Q4. Is HPA and CA not a cost reduction strategy?

**Yes** — HPA and Cluster Autoscaler (CA) can contribute to cost optimization, but they are not purely "cost-reduction tools."

### How They Reduce Cost

**HPA — Horizontal Pod Autoscaler**
- Scales pods based on CPU, memory, or custom metrics.
- During low traffic → fewer pods → less workload resource consumption.
- During high traffic → more pods to handle demand.

**Cluster Autoscaler (CA)**
- Scales the AKS node pool based on pending/unused pod capacity.
- If pods don't need the nodes → nodes can be removed.
- If pods cannot be scheduled → nodes are added.

### Together

```
Traffic increases
      ↓
HPA increases pods
      ↓
Not enough node capacity?
      ↓
Cluster Autoscaler adds nodes
```

And when traffic decreases:

```
Traffic decreases
      ↓
HPA reduces pods
      ↓
Nodes become underutilized
      ↓
Cluster Autoscaler removes unnecessary nodes
      ↓
Cost optimization
```

### Interview Answer

> "Yes, HPA and Cluster Autoscaler are important cost-optimization mechanisms in AKS. HPA optimizes the number of application pods based on demand, while Cluster Autoscaler optimizes the underlying node capacity. Together they help avoid over-provisioning and ensure we pay for compute capacity according to workload demand. However, their primary purpose is scalability and resource optimization; cost reduction is an important benefit."

### Important Distinction

HPA does not directly reduce the Azure bill if the node count stays the same. **CA is what can actually reduce the underlying VM/node capacity** and therefore directly reduce compute cost.

---

## Q5. What are the other approaches/tools that can be used for AKS/Azure cost optimization?

Divide cost optimization into: **AKS compute, Azure resources, storage, and governance.**

### 1. AKS Compute Optimization

| Approach | What it does | Cost impact |
|---|---|---|
| HPA | Scales pods based on demand | Reduces over-provisioned pods |
| Cluster Autoscaler | Adds/removes nodes | Direct VM cost reduction |
| Right-sizing requests/limits | Avoids unnecessarily large CPU/memory allocation | High |
| Spot node pools | Uses discounted Azure Spot VMs for interruptible workloads | High |
| Reserved Instances / Savings Plan | Discounts predictable compute usage | High |
| Separate system & user node pools | Keeps workloads on appropriately sized nodes | Medium |
| Node affinity/taints | Places workloads on suitable nodes | Medium |
| KEDA | Scales workloads based on events/queue length | High for bursty workloads |

```
Normal workload
     ↓
HPA → Pod scaling
     ↓
CA → Node scaling
     ↓
Spot pool → cheaper compute for suitable workloads
```

### 2. Azure Cost Management

Use Azure Cost Management + Billing to:
- Track spending by subscription/resource group
- Identify expensive resources
- Set budgets
- Configure cost alerts
- Analyze cost trends

This is more of a **visibility and governance tool** than an automatic optimization mechanism.

### 3. Azure Advisor

Provides recommendations such as:
- Underutilized VMs
- Right-sizing
- Idle resources
- Reserved instance recommendations
- Security/reliability recommendations

> "We use Azure Advisor and Cost Management to identify optimization opportunities and monitor cloud expenditure."

### 4. Storage Optimization

For Storage Accounts:
- Lifecycle management → Hot → Cool → Archive
- Delete old/unnecessary data
- Use appropriate redundancy: LRS/ZRS/GRS based on requirement
- Control retention periods

```
New logs → Hot
       ↓
Older logs → Cool
       ↓
Long-term retention → Archive
       ↓
Expired → Delete
```

### 5. Log Analytics Optimization

Particularly relevant to an AKS monitoring setup. Don't retain everything indefinitely.

- Set appropriate retention periods
- Avoid collecting unnecessary logs
- Use appropriate data collection rules
- Filter noisy container logs
- Use lower-cost log tiers where appropriate

Otherwise, AKS + Container Insights + Application Insights can generate significant ingestion costs.

### 6. Environment Optimization

For Dev/Test environments:
- Automatically stop/deallocate resources when not required
- Use smaller node pools
- Use fewer replicas
- Use Spot VMs where appropriate
- Avoid unnecessary high availability in non-production environments

```
Production → HA + multiple zones + larger capacity
UAT         → Moderate capacity
Dev         → Minimal capacity
```

### 7. Terraform Helps Enforce Cost Optimization

```hcl
default_node_pool {
  vm_size = "Standard_D4s_v5"

  auto_scaling_enabled = true
  min_count            = 2
  max_count            = 10
}
```

Define separate node pools:

```
AKS
├── System pool
├── Production user pool
└── Spot user pool
```

### Strong Interview Answer

> "In AKS, we use HPA for pod-level scaling and Cluster Autoscaler for node-level scaling. We right-size CPU and memory requests and limits to avoid over-provisioning, and where the workload permits, we can use Spot node pools. For predictable workloads, Azure Savings Plans or reservations can reduce compute costs. We also use Azure Cost Management and Azure Advisor to identify expensive and underutilized resources. For monitoring, we control Log Analytics and Application Insights data collection and retention because excessive telemetry can increase cost. Finally, we use Terraform to standardize these configurations across environments."

---

## Q6. What are the different deployment strategies in Kubernetes? What are pros/cons and what strategy do you suggest in real time?

### 1. Rolling Deployment — Default choice for most applications

Kubernetes gradually replaces old pods with new pods.

```
Old:  [v1][v1][v1][v1]

        ↓ rollout

       [v1][v1][v2][v2]

        ↓

       [v1][v2][v2][v2]

        ↓

New:  [v2][v2][v2][v2]
```

**Pros**
- Zero/low downtime when configured correctly
- No need to maintain a complete duplicate environment
- Lower infrastructure cost than blue-green
- Kubernetes natively supports it
- Easy rollback using Deployment revision history
- Good fit for AKS microservices

**Cons**
- Two versions run simultaneously during deployment
- Requires backward compatibility between application versions
- Database/schema changes need careful planning
- Rollout can take longer
- Troubleshooting can be more difficult because different users may temporarily hit different versions

**Real-time use:** For an AKS microservices project, this is generally the default strategy.

---

## Q7. Explain rolling deployment in detail. Why are coexistence, backward compatibility, database compatibility, longer deployment and troubleshooting considered cons?

### 1. v1 and v2 Coexist During Deployment

Suppose your application currently has 4 pods:

```
Before deployment:

Pod 1 → v1
Pod 2 → v1
Pod 3 → v1
Pod 4 → v1
```

Now you deploy v2 using RollingUpdate. Kubernetes doesn't immediately remove all v1 pods:

```
During deployment:

Pod 1 → v1
Pod 2 → v1
Pod 3 → v2
Pod 4 → v2
```

Eventually:

```
Pod 1 → v2
Pod 2 → v2
Pod 3 → v2
Pod 4 → v2
```

So for some period, both versions are serving traffic.

**Why is this important?** Imagine your frontend/service calls your backend.

v1 expects:
```json
GET /api/customer
Response:
{
  "name": "Sai",
  "age": 25
}
```

But v2 changes the response:
```json
{
  "customerName": "Sai",
  "age": 25
}
```

If Frontend v1 → Backend v2, the frontend expects `name`, but backend v2 sends `customerName`. That can break the application.

> During a rolling deployment, old and new application versions must generally be compatible with each other.

### 2. Backward Compatibility

Suppose you're deploying v1 → v2. During rollout, the client should be able to work with both v1 and v2.

**Good approach:** Suppose v1 uses `GET /api/customer`. Instead of suddenly removing it in v2, v2 can continue supporting it:

```
v2 supports:

GET /api/customer        ← old API
GET /api/v2/customer     ← new API
```

Then old application → v2 → works, and new application → v2 → works. After all consumers migrate to v2, you can eventually remove the old API. This is called **backward-compatible deployment**.

### 3. Database/Schema Changes Need Careful Planning

Suppose v1 expects a `customer` table with `id, name, age`. Now v2 requires `id, customer_name, age`.

You might think "I'll rename `name` to `customer_name`." But during rolling deployment, v1 is still running. If you immediately rename the column, v1 may execute `SELECT name FROM customer;` and fail because `name` no longer exists.

**Better approach: backward-compatible database migration**

**Step 1 — Add the new column:**
```
id
name
age
customer_name
```
Don't remove `name` yet.

**Step 2 — Deploy application v2.** v2 can use `customer_name` while v1 continues using `name`. Both can coexist.

**Step 3 — Migrate/copy data.** Populate the new column: `customer_name = name`.

**Step 4 — Move all traffic to v2.** `v1 → removed`, `v2 → 100%`.

**Step 5 — Remove the old column later.** Now you can safely remove `name`.

This is often called the **expand-and-contract migration pattern**.

### 4. Rolling Deployment Can Take Longer

Suppose you have 10 pods and configure `maxUnavailable: 0`, `maxSurge: 2`. Kubernetes waits for new pods to become Ready before continuing. If your application takes 2 minutes to start and you have many pods, the entire rollout can take considerable time.

That's actually a good thing from an availability perspective because you don't want Kubernetes to replace everything immediately.

### 5. Troubleshooting Becomes More Difficult

Imagine 10 pods: 8 → v1, 2 → v2. A user reports "The application sometimes gives an error." You discover requests → v1 → SUCCESS, requests → v2 → ERROR. Different requests may reach different versions.

```
              Service
                 |
       ┌─────────┼─────────┐
       ↓         ↓         ↓
      v1        v1        v2
       ↓         ↓         ↓
    SUCCESS   SUCCESS    ERROR
```

So the application can appear to work sometimes and fail sometimes.

### How Do We Reduce Rolling Deployment Problems?

**Readiness Probe** — Before Kubernetes sends traffic to the new pod:

```
New Pod
   ↓
Startup
   ↓
Readiness Probe
   ↓
Ready?
   ↓ YES
Service sends traffic
```

If not ready, Kubernetes doesn't send normal Service traffic to it.

**maxUnavailable** — Controls how many existing pods can be unavailable during rollout. `maxUnavailable: 0` means don't intentionally make existing capacity unavailable while deploying.

**maxSurge** — Controls how many additional pods can temporarily be created. `maxSurge: 25%` with 4 replicas → maximum temporarily = 5 pods.

### Recommended AKS Production Configuration

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 25%
```

Combined with: Deployment + Readiness Probe + Liveness Probe + PDB + HPA + Monitoring + Backward-compatible APIs + Backward-compatible DB migrations.

### Interview-Ready Rolling Deployment Answer

> "The main challenge with rolling deployment is that old and new versions coexist temporarily. Therefore, the application and APIs need to maintain backward compatibility. Database schema changes also need to be backward compatible because old and new pods may access the database simultaneously. The rollout can take longer because Kubernetes waits for new pods to become ready, and troubleshooting can be slightly more difficult because different requests may temporarily reach different application versions. We address these issues using readiness probes, controlled maxSurge/maxUnavailable settings, PDBs, backward-compatible API and database changes, and proper monitoring."

---

## Q8. What are the main Kubernetes deployment strategies?

### 1. Rolling Deployment

```
Old: [v1][v1][v1][v1]
      ↓
     [v1][v1][v2][v2]
      ↓
     [v1][v2][v2][v2]
      ↓
New: [v2][v2][v2][v2]
```

**Pros:** Low/zero downtime; no complete duplicate environment; lower infrastructure cost than blue-green; native Kubernetes support; easy rollback; good fit for AKS microservices.

**Cons:** v1 and v2 coexist; backward compatibility required; database migrations require planning; rollout can be slower; troubleshooting can be harder.

### 2. Blue-Green Deployment

Two complete environments are maintained:

```
             Load Balancer / Ingress
                     |
               ┌─────┴─────┐
               ↓           ↓
            Blue          Green
             v1             v2
          CURRENT          NEW
```

Initially traffic goes to Blue. Deploy and test Green. Once validated, traffic → Green (v2). Blue remains available for quick rollback.

**Pros:** Very fast rollback; old and new environments isolated; good for major releases; easy to test the new version before exposing users.

**Cons:** Requires almost 2× infrastructure capacity; higher cost; database migrations can still be complicated; traffic switching needs careful configuration; more operational complexity.

**When to use:** For high-risk releases or applications where very fast rollback is critical.

### 3. Canary Deployment

Only a small percentage of traffic initially goes to the new version.

```
             Ingress
               |
      ┌────────┴────────┐
      ↓                 ↓
   v1 - 95%          v2 - 5%
                      Canary
```

If metrics are good: 5% → 25% → 50% → 100%.

**Pros:** Very low blast radius; real production traffic validates the new version; easy to monitor application behavior before full rollout; excellent for high-risk changes; can automatically promote/rollback with progressive delivery tools.

**Cons:** More complicated traffic management; requires strong monitoring; requires a way to control traffic percentages; session/state handling can complicate testing; higher operational complexity than standard rolling deployment.

**When to use:** For critical production applications and high-risk releases, especially when strong monitoring is available.

### 4. Recreate Deployment

All old pods are terminated first, then new pods are created.

```
v1 → DELETE ALL → v2
```

**Pros:** Very simple; no possibility of v1/v2 running simultaneously; useful when two versions cannot coexist.

**Cons:** Downtime during deployment; poor user experience; not suitable for most production applications; rollback also causes downtime.

**When to use:** Mostly for non-production environments or applications where downtime is acceptable.

### Deployment Strategy Comparison

| Strategy | Downtime | Cost | Rollback | Complexity | Production |
|---|---|---|---|---|---|
| Rolling | Low/zero | Low | Good | Low | Most common |
| Blue-Green | Near zero | High | Excellent | Medium | High-risk releases |
| Canary | Near zero | Medium | Excellent | High | Critical/high-risk |
| Recreate | Yes | Low | Slow | Very low | Usually avoid |

### What Would I Recommend in Real Time?

> "I would use Rolling Deployment as the default strategy because Kubernetes supports it natively, it provides zero or minimal downtime, requires less infrastructure than blue-green, and supports controlled rollout and rollback. I would configure readiness probes, maxUnavailable and maxSurge appropriately and use PDBs to maintain availability."

> "For high-risk or business-critical releases, I would consider Canary deployment so that we initially expose the new version to a small percentage of traffic and monitor Application Insights, Azure Monitor, error rate and latency before increasing traffic. For releases requiring extremely fast rollback, Blue-Green is another option, although it has higher infrastructure cost."

### Senior-Level Point

Don't say: "Canary is always better than Rolling."

**Correct:** Strategy depends on risk, cost, application compatibility, and rollback requirements.

- Normal AKS microservice → Rolling
- High-risk release → Canary
- Instant rollback / major release → Blue-Green
- Non-production/simple incompatible deployment → Recreate

Regardless of strategy, ensure: **readiness probes + monitoring + automated/controlled rollback + proper database migration strategy**.

---

## Q9. Give one production issue where Azure monitoring tools were used for logs, metrics, traces and queries. Explain in real time.

### Production Incident: API Latency and 5xx Errors After Deployment

**Architecture**

```
User
  ↓
Azure Front Door
  ↓
Application Gateway / AGIC
  ↓
AKS
  ├── service-a
  ├── service-b
  └── service-c
        ↓
     Database/API
```

**Monitoring:**

```
AKS
 ├── Container Insights ──→ Log Analytics Workspace
 ├── Azure Monitor ───────→ Metrics + Alerts
 └── Application Insights ─→ Requests + Dependencies
                              + Exceptions + Traces
```

### Step 1 — First Indication: Azure Monitor Metrics

An alert fires: "AKS application HTTP 5xx rate exceeded threshold." The application team reports: "The API is taking 8–10 seconds instead of the normal 500 ms."

```
Normal latency:       400–600 ms
Current latency:      8–10 seconds
CPU:                   45%
Memory:                55%
```

**Don't immediately conclude that AKS has a resource problem** — CPU and memory are not necessarily the issue.

**Why metrics first?** Metrics quickly tell you: Is there actually a problem? When did it start? Which resource is affected? Is the problem CPU/memory or application behavior?

```
10:00 → deployment started
10:10 → deployment completed
10:15 → latency starts increasing
```

The deployment becomes a strong suspect.

### Step 2 — Container Insights / Log Analytics Logs

Question: What is the application actually reporting?

**Example KQL:**

```kql
ContainerLogV2
| where TimeGenerated > ago(30m)
| where PodName contains "payment"
| where LogMessage contains "error"
| project TimeGenerated, PodName, ContainerName, LogMessage
| order by TimeGenerated desc
```

> The exact table/fields can vary depending on the AKS monitoring configuration, so in a real environment first confirm the available schema.

Results show:
```
payment-service
Timeout connecting to customer-service
Request timeout after 5000ms
```

Now: AKS → payment-service → customer-service → TIMEOUT. But we still don't know why the request is timing out.

### Step 3 — Application Insights Distributed Tracing

User calls `POST /api/payment`:

```
POST /api/payment                    8.7 sec
       |
       ├── payment-service           5.1 sec
       |
       ├── customer-service          3.4 sec
       |       |
       |       └── SQL query         3.2 sec
       |
       └── response
```

Now we know: customer-service → database dependency → 3.2 seconds.

**What is a trace?** A trace represents the journey of one request through the application.

```
Trace ID: abc123

Frontend
   ↓
API Gateway
   ↓
payment-service
   ↓
customer-service
   ↓
SQL Database
```

Individual operations are spans/operations. This is extremely useful for microservices because a request can cross multiple services.

### Step 4 — Application Insights Dependencies

```
Dependency             Avg duration
-----------------------------------
SQL Database             3.2 sec
Customer API              3.5 sec
Redis                     20 ms
External API              50 ms
```

The SQL/database dependency is suspicious.

```
Metric   → High latency
Logs     → Customer-service timeout
Trace    → Customer-service is slow
Dependency telemetry → SQL query is slow
```

### Step 5 — Query Slow Requests with KQL

```kql
requests
| where timestamp > ago(30m)
| where duration > 2000
| project timestamp, name, duration, resultCode, operation_Id
| order by duration desc
```

```
/api/payment       8700 ms
/api/payment       8100 ms
/api/payment       9200 ms
```

Use `operation_Id` to correlate related telemetry:

```kql
dependencies
| where timestamp > ago(30m)
| where operation_Id == "abc123"
| project timestamp, name, target, duration, success
```

Possible result:
```
SQL Database
duration = 3200 ms
success = true
```

**Important:** A dependency can technically succeed but still cause unacceptable latency.

### Step 6 — Find Exceptions

```kql
exceptions
| where timestamp > ago(30m)
| project timestamp, type, outerMessage, operation_Id
| order by timestamp desc
```

```
SqlException
Execution Timeout Expired
```

### Step 7 — Correlate with Deployment

```
09:45 → v1
10:00 → v2 deployment
10:12 → latency starts
10:15 → SQL timeout exceptions
```

The development team says: "v2 introduced a new customer lookup query."

```sql
SELECT *
FROM customer
WHERE email = ?
```

The production customer table has millions of records and the required index is missing.

```
New deployment
      ↓
New DB query
      ↓
Large table scan
      ↓
Database response time increases
      ↓
customer-service becomes slow
      ↓
payment-service waits
      ↓
API latency increases
      ↓
Some requests timeout
      ↓
5xx errors
```

**Root cause:** The new application query caused a slow database operation due to a missing/inefficient index.

### Step 8 — Remediation

**Immediate mitigation:** Roll back to the previous stable version.

```bash
helm rollback payment-service <revision>
```

Then monitor: Latency ↓, 5xx ↓, SQL dependency duration ↓.

**Permanent fix:** Development/DB team optimizes the query, adds the required index, tests it, and deploys the corrected version through normal CI/CD.

### Step 9 — Verify Resolution

**Metrics:**
```
Latency: 9 sec → 500 ms
5xx:     8% → <0.1%
CPU:     normal
```

**Logs:** Timeout messages disappear.

**Traces:**
```
Before: payment-service → customer-service → SQL = 3.2 sec
After:  payment-service → customer-service → SQL = 50 ms
```

**Application Insights:** Request success rate returns to normal. This proves the incident is actually resolved.

### Where Each Azure Monitoring Tool Fits

| Requirement | Tool | What I investigate |
|---|---|---|
| Metrics | Azure Monitor | CPU, memory, latency, request rate, failures |
| Container logs | Container Insights + Log Analytics | Pod/container errors, restarts, application logs |
| Application traces | Application Insights | End-to-end request flow |
| Dependencies | Application Insights | SQL, APIs, external services |
| Exceptions | Application Insights | Application exceptions |
| Queries | Log Analytics / Application Insights + KQL | Correlation and root-cause analysis |
| Alerts | Azure Monitor | Notify on thresholds/anomalies |

### Complete Real-Time Troubleshooting Flow

```
              Production Alert
                    ↓
             Azure Monitor
                    ↓
       ┌────────────┴────────────┐
       ↓                         ↓
    Metrics                    Alert
       ↓
Identify affected service
       ↓
Container Insights
       ↓
Log Analytics / KQL
       ↓
Find errors/timeouts
       ↓
Application Insights
       ↓
Distributed Trace
       ↓
Check dependencies
       ↓
Database / API / external service
       ↓
Find root cause
       ↓
Mitigate / rollback
       ↓
Deploy permanent fix
       ↓
Verify metrics + logs + traces
```

### Interview-Ready Production Incident Answer

> "One production issue we faced was increased API latency and intermittent 5xx errors after an application deployment. We first received an Azure Monitor alert and checked metrics such as request latency, failure rate, CPU and memory. CPU and memory were normal, but application latency and 5xx errors had increased shortly after the deployment."
>
> "We then used AKS Container Insights and Log Analytics to investigate container logs using KQL. We found timeout messages from the payment service when it was calling the customer service. To understand the complete request path, we moved to Application Insights and checked distributed traces and dependency telemetry. The trace showed that the customer-service call was taking several seconds, and the dependency telemetry identified a slow SQL database operation."
>
> "We then queried Application Insights using KQL to identify slow requests, dependencies and exceptions and correlated them using the operation ID. We found SQL timeout exceptions and confirmed that the issue started after the latest application release, which had introduced a new database query without an appropriate index."
>
> "As an immediate mitigation, we rolled back the application to the previous stable version. After rollback, we verified through Azure Monitor that latency and 5xx errors returned to normal. The permanent fix was to optimize the database query and add the required index, after which we deployed the corrected version through our CI/CD pipeline."
>
> "So my troubleshooting approach is metrics first to identify the problem and timeframe, logs to understand what is happening inside the containers, and distributed traces and dependency telemetry to identify where the request is actually spending time. I then use KQL to correlate the telemetry and determine the root cause."

**One sentence to remember:**

> "Metrics tell me that there is a problem, logs tell me what is happening, traces tell me where the request is failing or becoming slow, and KQL helps me correlate all the telemetry to find the root cause."

---

## Q10. Deep explanation of designing High Availability, Reliability and Cost-Effective AKS

These are three separate design objectives:

- **High Availability (HA)** → How do I prevent downtime?
- **Reliability** → How do I handle failures and recover safely?
- **Cost Optimization** → How do I provide required capacity without wasting money?

They overlap, but the design decisions are different.

---

### Part 1 — High Availability

**Objective:** If a node, VM, availability zone, pod, or infrastructure component fails, the application should continue serving users with minimal or zero downtime.

**Example architecture:**

```
                         INTERNET
                            |
                            v
                    Azure Front Door
                            |
                     WAF / Routing
                            |
                            v
                  Application Gateway
                       /          \
                      /            \
                     v              v
                AKS Cluster
        ┌───────────────────────────────┐
        │                               │
        │ Zone 1     Zone 2     Zone 3 │
        │                               │
        │  Node       Node       Node  │
        │   ↓          ↓          ↓    │
        │ Pod-A      Pod-B      Pod-C  │
        │                               │
        └───────────────────────────────┘
```

For a critical application:

```
Region 1 AKS
      |
      | Front Door
      |
Region 2 AKS
```

This protects against regional failure, not just node/zone failure.

#### A. Multiple Availability Zones

**Single zone:**
```
Zone 1
 ├── Node 1
 ├── Node 2
 └── Node 3
```
If the zone fails, all nodes and pods fail → application down.

**Multi-zone:**
```
Zone 1       Zone 2       Zone 3
-------      -------      -------
Node 1       Node 2       Node 3
Pod A        Pod B        Pod C
```
If Zone 1 goes down: Zone 2 → Pod B serving, Zone 3 → Pod C serving.

> "I'll distribute the node pools across multiple Availability Zones and ensure application replicas are also distributed across zones."

#### B. Multiple Replicas

**Bad:** All pods on Node 1 — node failure removes all replicas.

**Better:**
```
Node 1 / Zone 1 → Pod A
Node 2 / Zone 2 → Pod B
Node 3 / Zone 3 → Pod C
```
One node/zone failure doesn't remove all replicas.

#### C. Topology Spread Constraints / Pod Anti-Affinity

Use `topologySpreadConstraints` and `podAntiAffinity`. Purpose: don't concentrate all replicas in one failure domain.

```
Pod A → Zone 1
Pod B → Zone 2
Pod C → Zone 3
```

#### D. PodDisruptionBudget

```
replicas: 5
minAvailable: 4
```

Meaning: Kubernetes should maintain at least 4 available pods during a voluntary disruption.

**Important limitation:** PDB does not protect against every failure. A complete Availability Zone failure can still remove multiple pods. PDB mainly helps with voluntary disruptions such as maintenance and node drain.

#### E. Readiness, Liveness and Startup Probes

**Readiness** answers: "Can this pod receive traffic?"
```
Pod started → Application initialization → Database connection → Cache initialization → Ready
```
Until readiness succeeds, the Service doesn't send normal traffic to it.

**Liveness** answers: "Is this application still functioning?"
```
Pod running → Application hung → Liveness fails → Container restarted
```

**Startup** — useful for slow-starting applications so Kubernetes doesn't incorrectly restart them during initialization.

#### F. Rolling Deployment

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 25%
```

For very high-risk applications, consider Canary or Blue-Green.

#### G. Ingress HA

```
Internet
   ↓
Azure Front Door
   ↓
Application Gateway
   ↓
AKS
   ↓
Services
   ↓
Pods
```

For multiple regions, Front Door can route users toward healthy origins.

#### H. Multi-Region HA

Availability Zones protect against zone-level failures. **Multi-region protects against regional-level failures.**

```
                 Azure Front Door
                  /            \
                 /              \
        Region 1              Region 2
           ↓                     ↓
          AKS                   AKS
```

If Region 1 is unavailable, Front Door health check routes to Region 2, and users continue receiving traffic.

But multi-region adds significant cost and complexity.

> Don't automatically use multi-region just because it provides more availability. Business RTO/RPO requirements should justify it.

---

### Part 2 — Reliability

Reliability is broader than simply having multiple instances.

**Reliability question:** "What happens when something fails?"

```
Failure
   ↓
Detect
   ↓
Isolate
   ↓
Recover
   ↓
Alert
   ↓
Analyze
   ↓
Prevent recurrence
```

#### A. Kubernetes Self-Healing

Deployment controller sees `Desired = 3, Actual = 2` and creates a replacement pod. Eventually: 3 healthy replicas.

#### B. Node Failure

Pods become unavailable. Scheduler can place replacement pods on healthy nodes if capacity exists. This requires sufficient capacity.

#### C. Cluster Autoscaler

If nodes are full and a new pod is Pending, Cluster Autoscaler can add another node. CA improves scalability and can contribute to reliability by ensuring capacity during demand increases.

#### D. Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "1"
    memory: "1Gi"
```

**Requests** — used by scheduler: "Where can I place this pod?"
**Limits** — restrict maximum consumption.

Proper sizing helps prevent: CPU starvation, memory exhaustion, OOMKilled containers, poor scheduling, noisy-neighbor problems.

#### E. Namespace-Level Controls

**ResourceQuota** — controls total resources consumed by a namespace (CPU limit, memory limit, pod count limit).

**LimitRange** — defines default/minimum/maximum resource requests and limits.

#### F. Network Reliability

```
Internet
   ↓
Front Door
   ↓
Application Gateway / WAF
   ↓
AKS
   ↓
Services
```

Use where appropriate: NSGs, Azure Firewall, Network Policies, Private AKS, Private Endpoints, Private DNS, controlled outbound connectivity.

#### G. Observability

- **Azure Monitor:** CPU, memory, node health, request metrics, availability, alerts
- **Container Insights:** pod status, container logs, node metrics, Kubernetes events, restart information
- **Log Analytics:** centralized KQL querying
- **Application Insights:** request telemetry, dependencies, exceptions, application performance, distributed tracing

#### H. Alerting

Examples: pod restart rate high, node CPU/memory > threshold, pod pending, HTTP 5xx > threshold, API latency > threshold, AKS node unhealthy, disk pressure, OOMKilled.

Alerts should integrate with incident-management processes.

#### I. Backup and Disaster Recovery

Consider: AKS configuration, application configuration, secrets, databases, persistent volumes, container images, Terraform code/state.

For stateful applications, **the database and persistent data often matter more than the Kubernetes cluster itself**.

Define:
- **RPO** — how much data can we afford to lose?
- **RTO** — how quickly must we recover?

Example: `RPO = 15 minutes`, `RTO = 1 hour`. This determines backup, replication and DR requirements.

---

### Part 3 — Cost Optimization

**Objective:** Meet business performance and availability requirements without over-provisioning.

#### A. Right-Size Node Pools

Don't use very large VMs simply because it is production. Analyze CPU utilization, memory utilization, pod density, network usage, and application requirements. Then choose appropriate VM SKUs.

#### B. HPA — Pod-Level Optimization

```
Normal: 3 pods
Peak:  10 pods

HPA: 3 → 5 → 8 → 10
When traffic decreases: 10 → 7 → 4 → 3
```

#### C. Cluster Autoscaler — Node-Level Optimization

```
Normal: 3 pods fit on 2 nodes
Peak:  10 pods require 5 nodes

CA: 2 nodes → 5 nodes
When demand drops: 5 nodes → 2 nodes
```

This can directly reduce compute cost.

#### D. HPA + CA Together

```
Traffic
   ↓
HPA
   ↓
More pods
   ↓
Insufficient capacity?
   ↓
Cluster Autoscaler
   ↓
More nodes
```

When demand decreases: Traffic decreases → HPA reduces pods → Nodes become underutilized → Cluster Autoscaler removes unnecessary nodes → Cost decreases.

**HPA optimizes pod count; CA optimizes node count.**

#### E. Spot Node Pools

**Good candidates:** Batch processing, CI/CD workloads, fault-tolerant workers, stateless workloads, interruptible workloads.

**Not ideal for:** Critical system components, workloads that cannot tolerate eviction, stateful workloads without appropriate architecture.

```
AKS
 ├── System Node Pool
 │     └── Critical system components
 │
 ├── Regular User Pool
 │     └── Production workloads
 │
 └── Spot Pool
       └── Batch / fault-tolerant workloads
```

#### F. Separate Node Pools

```
System Pool      → CoreDNS / Kubernetes system workloads
Application Pool → Normal production applications
Spot Pool        → Batch workloads
```

Use taints, tolerations, node affinity, node selectors to control placement.

#### G. Non-Production Optimization

```
Production → 3 AZ, multiple replicas, larger nodes, HA
UAT        → smaller nodes, fewer replicas
Dev        → minimal nodes, minimal replicas
```

#### H. Logging and Monitoring Cost

If everything is collected indefinitely, telemetry volume grows and Log Analytics ingestion increases the bill.

- Don't collect unnecessary logs
- Filter noisy logs
- Set appropriate retention
- Optimize Application Insights telemetry
- Use appropriate data collection rules/tables where applicable

**Monitoring itself has a cost.**

#### I. Storage Cost

```
Frequently accessed      → Hot
Less frequently accessed → Cool
Long-term archival       → Archive
```

Use lifecycle policies. Select redundancy based on requirement (LRS/ZRS/GRS/GZRS). Don't automatically choose the most expensive redundancy.

#### J. Reservations / Savings Plans

For predictable 24/7 production workloads, evaluate Azure Reservations and Azure Savings Plans — can reduce compute costs when usage patterns justify the commitment.

#### K. Azure Cost Management + Advisor

**Cost Management:** Understand which subscription/resource group/service/AKS node pool, and how much is being spent.

**Advisor:** Identify underutilized resources, right-sizing, cost recommendations.

---

## Q11. The interviewer asked: "Every project does not have budget for this. Within allocated budget, how well can you implement it?" Explain RTO/RPO, log retention, old log removal, storage and backup.

This is a realistic production architecture question. The interviewer is testing whether you can design based on business requirements and budget, rather than simply saying "Use 3 AZ + multi-region + everything redundant."

Key concepts: **RTO, RPO, Backup, Retention, Archival, Cost-based HA decisions.**

### 1. What is RTO?

**RTO = Recovery Time Objective.** It answers: "If my system goes down, how quickly must I restore it?"

Example: Application goes down at 10:00 AM. Business says "We can tolerate maximum 1 hour downtime." Therefore RTO = 1 hour, recovery must complete by 11:00 AM.

| Application | Possible RTO |
|---|---|
| Internal development application | 24 hours |
| Internal business application | 4–8 hours |
| Important production application | 1 hour |
| Critical banking/payment application | Minutes |
| Extremely critical system | Near-zero |

Actual values must come from the business.

### 2. What is RPO?

**RPO = Recovery Point Objective.** It answers: "If something goes wrong, how much data can we afford to lose?"

Suppose last backup = 10:00 AM, failure = 10:30 AM. If RPO is 30 minutes, losing data generated between 10:00 and 10:30 may be acceptable. If RPO is 5 minutes, backup/replication needs to be much more frequent.

### 3. RTO vs RPO

```
              FAILURE
                 |
                 ↓
10:00 ───────────X──────────── 11:00
       ↑                      ↑
    Last backup             Recovery
                              complete
```

If RPO = 30 minutes → up to 30 minutes of data loss may be acceptable.
If RTO = 1 hour → service must be restored within 1 hour.

**Easy memory trick:**
- **RTO → Time:** How long can the application be DOWN?
- **RPO → Data:** How much data can I LOSE?

### 4. Budget-Based AKS Architecture

Suppose management says: "We have a moderate budget. We need production availability, but we cannot afford two complete AKS clusters."

Don't immediately propose Region 1 AKS + Region 2 AKS + Front Door + 3 AZ. Instead consider:

**Option A — Single region + multiple AZs**

```
                  Application Gateway
                         |
                         ↓
                  AKS - Region 1
              ┌──────────┼──────────┐
              ↓          ↓          ↓
            AZ-1       AZ-2       AZ-3
            Nodes      Nodes      Nodes
              ↓          ↓          ↓
            Pods       Pods       Pods
```

This protects against pod failure, node failure, and zone failure — without paying for a complete second production environment. This may be appropriate when RTO/RPO requirements are moderate, depending on application/data architecture.

### 5. When Would You Choose Multi-Region?

If the business says "If the entire Azure region goes down, the application must continue running" — single-region is insufficient.

```
                    Azure Front Door
                    /              \
                   /                \
             Region 1              Region 2
                ↓                     ↓
               AKS                   AKS
                ↓                     ↓
             Database              Database
```

This provides regional DR/availability but adds: two AKS clusters, duplicate infrastructure, additional networking, additional monitoring, additional database replication, additional operational overhead.

> "I wouldn't choose multi-region by default. I would first understand the application's RTO and RPO and the business SLA. If the requirement is only to tolerate node or zone failures, a multi-zone AKS cluster may be sufficient and much more cost-effective. If the RTO requires recovery from a regional outage within a very short period, then I would consider multi-region AKS and appropriate data replication."

### 6. Backup

**AKS cluster backup ≠ application data backup.**

```
AKS                      Database
 ├── Deployments          ├── Customer data
 ├── Services             ├── Transactions
 ├── ConfigMaps           └── Orders
 ├── Secrets
 └── Application pods
```

The database data is usually much more important to recover than recreating Kubernetes objects. Infrastructure should already be defined through Terraform/Helm/manifests.

If the AKS cluster is lost:

```
Terraform
     ↓
Recreate infrastructure
     ↓
AKS
     ↓
Deploy applications
     ↓
Restore/connect to data
```

Business data requires separate backup/DR.

### 7. How Backup Works

Suppose application uses an Azure database:

```
Production Database
       |
       ├── Automated backups
       |
       ├── Point-in-time restore
       |
       └── Long-term backup if required
```

If something goes wrong: Production DB → corruption/deletion → select recovery point → restore → database recovered.

For stateful Kubernetes workloads, persistent volumes also need an appropriate backup strategy.

### 8. What About AKS Configuration?

Keep infrastructure and application definitions in source control:

```
Azure Repos
   |
   ├── Kubernetes manifests
   ├── Helm charts
   ├── Terraform
   └── Pipeline YAML
```

Recovery: Git repository → Terraform → Infrastructure → Helm → Applications. This is **Infrastructure as Code-based recovery**.

### 9. Log Retention

Logs can become expensive. AKS may generate application logs, container logs, ingress logs, Kubernetes logs, security logs, audit logs, traces.

If everything is kept forever:
```
Day 1       → 10 GB
Day 30      → 300 GB
Day 180     → 1.8 TB
Day 365     → 3.6 TB
```

Therefore define a retention strategy.

### 10. Don't Treat Every Log Equally

| Data | Example retention |
|---|---|
| Debug logs | Short |
| Normal application logs | Moderate |
| Security logs | Longer |
| Audit logs | Based on compliance |
| Critical business logs | Longer |
| Archived logs | Potentially years |

Actual retention should be based on business, compliance and security requirements.

### 11. What Happens When Logs Become Old?

```
Application
    ↓
Log Analytics
    ↓
Recent logs
    ↓
Retention period
    ↓
Archive / lower-cost storage
    ↓
Delete after required retention
```

Example:
```
0–30 days   → Log Analytics → Fast querying
30–180 days → Lower-cost/archive tier where appropriate
180+ days   → Long-term storage if required
Retention expired → Delete
```

Exact implementation depends on Azure Monitor/Log Analytics data type and available retention/archival features.

### 12. Why Move Old Logs Elsewhere?

Suppose security says "We need audit logs for 1 year." Keeping everything in a highly queryable analytics tier may be unnecessarily expensive.

```
Application → Azure Monitor → Log Analytics → Archive / Storage
```

Keep recent logs easy to search and move older logs to cheaper storage if they don't need frequent querying.

### 13. Don't Blindly Move All Logs to Storage

**Better interview answer:**

> "We define retention based on log type and compliance requirements. Frequently queried operational logs remain in Log Analytics for the required period. Older data that needs long-term retention but doesn't require frequent querying can be archived using appropriate Azure Monitor/Log Analytics archival capabilities or exported to lower-cost storage, depending on the data type and requirements."

### 14. Example Production Log Strategy

- **Operational logs:** Recent → Log Analytics → Fast KQL queries
- **Older operational logs:** Older → Archive
- **Compliance/audit logs:** Long-term retention → Dedicated retention/archive strategy
- **Debug logs:** Short retention → Delete

This controls cost without sacrificing required visibility.

### 15. Backup vs Log Retention

Don't mix them up.

**Backup:** Used to recover data/system state.
**Log retention/archive:** Used to retain historical telemetry for troubleshooting, auditing and compliance.

### 16. Complete Cost-Effective Reliability Architecture

For a moderate budget:

```
                         USERS
                           |
                           ↓
                  Application Gateway
                           |
                           ↓
                    AKS - Region 1
                           |
              ┌────────────┼────────────┐
              ↓            ↓            ↓
             AZ1          AZ2          AZ3
              |            |            |
            Nodes        Nodes        Nodes
              |            |            |
            Pods         Pods         Pods
              └────────────┼────────────┘
                           |
                  Azure Monitor
                  /     |      \
                 /      |       \
                ↓       ↓        ↓
           Metrics    Logs     Traces
                      |          |
                Log Analytics  App Insights
```

**Cost:** HPA → Pod scaling → Cluster Autoscaler → Node scaling

**DR:** Terraform + Git → Recreate AKS; Database → Backup / PITR / replication

**Logs:** Recent logs → Log Analytics → Retention policy → Archive where required → Delete after retention expires

### 17. Best Interviewer Answer for Budget-Constrained HA/Reliability

> "I wouldn't start by assuming that every project needs multi-region AKS. I would first understand the business SLA, RTO, RPO, criticality and available budget. RTO tells us how quickly the application needs to be recovered after a failure, while RPO tells us how much data loss is acceptable."
>
> "For example, if the requirement is to tolerate node and zone failures but the budget doesn't allow two AKS clusters, I would use a single-region AKS cluster distributed across Availability Zones, multiple application replicas, topology spread, PDBs, probes and appropriate autoscaling. That gives good availability without the cost of multi-region."
>
> "If the business requires protection from a complete regional outage and has a very low RTO, then I would consider a multi-region architecture with Front Door and replicated data. The architecture should be driven by the required RTO/RPO rather than simply maximizing redundancy."
>
> "For data protection, I would separately define backup and disaster recovery for databases and persistent data. Since our infrastructure is managed through Terraform and application deployment through Helm or pipelines, the AKS infrastructure and application configuration can be recreated from source control, while the actual business data requires dedicated backup and recovery mechanisms."
>
> "For logs, I wouldn't keep everything in Log Analytics indefinitely because ingestion and retention can increase cost. I would classify logs based on operational, security and compliance requirements. Recent operational logs remain readily queryable in Log Analytics, while older logs that need long-term retention can be archived using appropriate Azure Monitor/Log Analytics capabilities or exported to lower-cost storage, and data is deleted once the required retention period expires."
>
> "So my approach is to balance availability, recoverability and cost based on business requirements rather than deploying the most expensive architecture by default."

**Key line:**

> "I design to the SLA, RTO, RPO and budget — not to maximum possible availability."

---

## Q12. What is the difference between High Availability, Reliability and Cost Optimization?

| Concept | Main question | Key mechanisms |
|---|---|---|
| High Availability | How do I avoid downtime? | AZs, replicas, PDB, topology spread, multi-region |
| Reliability | How do I survive and recover from failures? | Probes, self-healing, monitoring, alerts, rollback, DR |
| Cost Optimization | How do I avoid unnecessary spending? | Right-sizing, HPA, CA, Spot, Savings Plans, log optimization |

### Important Distinction

Don't say: "HPA and CA provide high availability."

**Better:**

> "HPA and Cluster Autoscaler primarily provide scalability and resource optimization. They can contribute to availability by ensuring sufficient capacity, but HA is primarily achieved through redundancy, multi-zone distribution, replicas and elimination of single points of failure."

---

## Q13. What is the best production AKS design considering HA, reliability and cost?

### A Practical Production Architecture

```
                         USERS
                           |
                           v
                  Azure Front Door
                    WAF + Health
                           |
              ┌────────────┴────────────┐
              |                         |
           Region 1                  Region 2
           AKS                       AKS
              |                         |
        ┌─────┼─────┐             ┌─────┼─────┐
      AZ-1   AZ-2   AZ-3         AZ-1  AZ-2  AZ-3
        |      |      |             |     |     |
       Nodes  Nodes  Nodes         Nodes Nodes Nodes
        |      |      |             |     |     |
       Pods   Pods   Pods          Pods  Pods  Pods
```

### Inside Each Cluster

```
AKS
│
├── System Node Pool
│
├── Production User Node Pool
│
├── Spot Node Pool
│
├── HPA
│
├── Cluster Autoscaler
│
├── PDB
│
├── Topology Spread
│
├── Readiness/Liveness/Startup Probes
│
├── Network Policies
│
└── Azure Monitor
      ├── Container Insights
      ├── Log Analytics
      └── Application Insights
```

### Infrastructure

```
Azure Repos
     ↓
Azure Pipeline
     ↓
Terraform
     ↓
AKS / Networking / Monitoring / Security
```

### But Don't Over-Engineer

**If interviewer asks:** "Would you always deploy AKS across three zones and two regions?"

**Answer:**

> "No. I would first understand the business SLA, RTO, RPO, traffic pattern and budget. For a business-critical application requiring high availability, I would use multi-zone AKS and potentially multi-region deployment. For a lower-criticality internal application, a single-region multi-zone cluster may be sufficient. Similarly, I wouldn't use Spot nodes for workloads that cannot tolerate eviction just to reduce cost."

This demonstrates engineering trade-offs.

---

## Final Interview Cheat Sheet

### Terraform Apply Failure

```
Error → State check → Azure check → Root cause → Fix code → Validate → Plan → Controlled apply
```

If recreation is required:
```bash
terraform apply -replace="resource.name"
```
Prefer this over deprecated `terraform taint`.

### Azure Traces

**Application Insights** — use it for: Requests, Dependencies, Exceptions, Distributed tracing.

### AKS Cost Optimization

- **HPA** → Pod scaling
- **CA** → Node scaling
- Right-size nodes
- Spot pools where appropriate
- Savings Plans/Reservations
- Smaller non-prod
- Log/telemetry optimization
- Azure Cost Management
- Azure Advisor

### Kubernetes Deployment Strategies

- **Rolling** → Default for normal production microservices
- **Canary** → High-risk releases; gradual production traffic
- **Blue-Green** → Fast rollback; higher infrastructure cost
- **Recreate** → Downtime; mostly special/non-production cases

### Azure Monitoring Incident Flow

```
Azure Monitor metrics
        ↓
Container Insights
        ↓
Log Analytics / KQL
        ↓
Application Insights
        ↓
Distributed trace
        ↓
Dependency
        ↓
Root cause
        ↓
Rollback/fix
        ↓
Verify metrics + logs + traces
```

### HA vs Reliability vs Cost

- **HA:** Prevent downtime.
- **Reliability:** Detect, tolerate, recover from failures.
- **Cost Optimization:** Meet requirements without paying for unnecessary capacity.

### RTO vs RPO

- **RTO:** How quickly must I recover?
- **RPO:** How much data can I afford to lose?

### Budget-Based Architecture

Don't automatically choose maximum HA. Use:

```
SLA
 ↓
RTO/RPO
 ↓
Business criticality
 ↓
Budget
 ↓
Architecture
```

**For moderate budget:** Single region + multi-AZ + replicas + PDB + probes + autoscaling + monitoring + strong backup/DR

**For strict regional DR:** Multi-region + Front Door + replicated data + stronger DR architecture

### Log Retention Strategy

```
Recent operational logs
        ↓
Log Analytics
        ↓
Retention period
        ↓
Archive/lower-cost storage if required
        ↓
Delete after required retention
```

Classify logs based on: Operational need, Security, Compliance, Audit requirements, Cost. Don't keep every log forever.

### Master Interview Principle

> "I don't design infrastructure by simply adding more Azure services. I first understand the business SLA, RTO, RPO, criticality and budget. Then I select the minimum architecture that satisfies those requirements while providing appropriate availability, reliability, security, observability and cost efficiency. I manage the infrastructure through Terraform so that the design is repeatable and recoverable."

---

*Document prepared for interview preparation — Azure observability, Terraform failure recovery, AKS cost optimization, Kubernetes deployment strategies, production incident troubleshooting, and HA/Reliability/Cost architecture design.*
