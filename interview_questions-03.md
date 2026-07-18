# Azure DevOps / AKS / Terraform Interview Questions & Answers

## 1. Your AKS cluster is healthy but requests intermittently return HTTP 503. How would you troubleshoot?

**Answer:**

I follow a layer-by-layer approach to identify where the 503 is generated.

### Step 1: Identify where the 503 is generated

Check whether the response comes from:

* Azure Front Door
* Application Gateway
* Ingress Controller
* Kubernetes Service
* Application

Review:

* Application Gateway Backend Health
* Access Logs
* WAF Logs
* Azure Monitor Metrics

---

### Step 2: Check Pods

```bash
kubectl get pods -n app
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

Verify:

* Restarts
* CrashLoopBackOff
* OOMKilled
* CPU throttling
* Application exceptions

---

### Step 3: Check Service Endpoints

```bash
kubectl get svc
kubectl get endpoints
kubectl describe svc
```

Ensure:

* Service selectors match pod labels.
* Endpoints are healthy.

---

### Step 4: Check Readiness Probe

Readiness probe failures commonly cause intermittent 503 errors.

Check:

```bash
kubectl describe pod
```

Look for:

* Readiness probe failed
* Connection refused
* Timeout
* HTTP 500

---

### Step 5: Test Service and Individual Pods

```bash
kubectl run curl-test --image=curlimages/curl --rm -it -- sh

curl http://service-name
curl http://pod-ip
```

Identify whether only one pod is failing.

---

### Step 6: Check Application Gateway Backend Health

Verify:

* Backend health
* Health probe path
* Host headers
* Port
* HTTP/HTTPS configuration

---

### Step 7: Check Resources

```bash
kubectl top pods
kubectl top nodes
```

Review:

* CPU
* Memory
* Node utilization

Correlate using Azure Monitor and Application Insights.

---

### Step 8: Check HPA

```bash
kubectl get hpa
kubectl describe hpa
```

Verify:

* Current replicas
* Desired replicas
* Scaling events

---

### Step 9: Check Deployment

```bash
kubectl rollout history deployment/app
kubectl describe deployment app
kubectl get events
```

Verify RollingUpdate strategy and rollout events.

---

### Step 10: Check Dependencies

Look for:

* Database timeout
* Redis timeout
* Service Bus issues
* External API failures

---

## Strong Interview Answer

I first identify where the 503 originates by checking Front Door/Application Gateway, then verify pod health, readiness probes, service endpoints, Application Gateway backend health, HPA events, deployment history, Azure Monitor metrics, Application Insights traces, and application logs. This allows me to isolate whether the issue is infrastructure, Kubernetes, or application related.

---

# 2. How would you migrate a Stateful Application in Kubernetes without downtime?

### Approach

1. Deploy new application.
2. Configure database replication.
3. Validate new deployment.
4. Switch traffic.
5. Monitor.
6. Decommission old environment.

Examples:

* PostgreSQL Streaming Replication
* MySQL Replication
* MongoDB Replica Set
* Kafka MirrorMaker

---

## Strong Answer

Run old and new environments simultaneously, continuously synchronize data, validate the new deployment, then switch traffic after replication lag becomes zero.

---

# 3. How do you design a rollback strategy if deployment fails?

### Steps

* Use immutable image tags.
* Configure RollingUpdate.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Pipeline:

Deploy

↓

kubectl rollout status

↓

Smoke Test

↓

Health Check

↓

If failed

↓

kubectl rollout undo

↓

Verify rollback

---

## Strong Answer

The pipeline never finishes immediately after deployment. It waits for rollout completion, validates health, performs smoke tests, and automatically executes rollback if validation fails.

---

# 4. Is mentioning RollingUpdate in Kubernetes Manifest enough?

Yes.

Deployment strategy belongs inside the Kubernetes Deployment manifest.

Azure DevOps pipeline should only:

* Deploy manifests
* Wait for rollout
* Validate deployment
* Rollback if required

Pipeline should not define rollout strategy.

---

# 5. Multi-environment CI/CD with configuration drift

### CI

* Build once.
* Scan.
* Push one immutable Docker image.

### CD

Promote same image:

Dev

↓

QA

↓

UAT

↓

Prod

### Environment Config

Use:

* Variable Groups
* Azure Key Vault
* Helm values files

### Infrastructure

Terraform modules:

* Network
* AKS
* ACR
* Key Vault
* Monitoring

### Drift Prevention

* Terraform Plan
* Git as source of truth
* No manual infrastructure changes

---

# 6. CI/CD Pipeline takes 30 minutes. How would you optimize?

Steps:

* Identify bottleneck.
* Cache dependencies.
* Run independent jobs in parallel.
* Optimize Docker layers.
* Incremental builds.
* Path filters.
* Faster agents.
* Parallel testing.
* Promote artifacts instead of rebuilding.

---

# 7. Terraform state file is 300MB and plan takes 15 minutes.

### Solution

Split state:

* Network
* AKS
* Monitoring
* ACR
* Key Vault

Other optimizations:

* Remove unused resources.
* Minimize data sources.
* Avoid unnecessary depends_on.
* Review provider performance.
* Increase apply parallelism where appropriate.

---

# 8. Runtime Security beyond Container Image Scanning

Implement:

* Microsoft Defender for Cloud
* Prisma Cloud Defender
* Falco

Security:

* Run as Non-root
* Disable Privilege Escalation
* Drop Linux Capabilities
* Read-only Root Filesystem
* RBAC
* Network Policies
* Azure Key Vault
* Kubernetes Audit Logs
* Azure Policy / Gatekeeper

---

# 9. SLO-based Alerting using Azure Monitor

Use:

* Application Insights
* Azure Monitor
* Log Analytics
* Action Groups

Monitor:

* Availability
* HTTP 5xx
* P95 latency

Avoid alerting on transient CPU spikes.

Use:

* Sustained thresholds
* Multi-condition alerts
* Alert suppression
* Service-level alerts

---

# 10. Correlating Logs, Metrics and Traces

Scenario:

Users receive HTTP 503.

Metrics:

* Error rate increased
* Latency increased

↓

Traces:

* Inventory Service slow

↓

Logs:

* SQL Connection Pool Exhausted

Root Cause:
Database bottleneck.

---

# 11. Design a Self-Healing Platform

Architecture:

* Multi-AZ deployment
* Multiple replicas
* Pod Anti-Affinity
* Pod Disruption Budget
* Liveness Probe
* Readiness Probe
* Startup Probe
* HPA
* Cluster Autoscaler
* Azure Monitor
* Automatic Rollback
* Terraform

Goal:
Automatic detection and recovery with minimal manual intervention.

---

# 12. Azure bill increased by 40% overnight. How would you investigate?

Steps:

1. Azure Cost Management
2. Identify expensive resource
3. Azure Activity Logs
4. Deployment history
5. HPA/Cluster Autoscaler events
6. Azure Monitor
7. Orphaned resources

Cost Reduction:

* Right-sizing
* HPA
* Cluster Autoscaler
* Reserved Instances (if applicable)
* Azure Advisor
* Log retention optimization
* Cleanup unused resources

---

# 13. Automating Non-production Shutdown

Tools:

* Azure Automation
* Azure CLI
* Shell Script
* Azure DevOps Scheduled Pipeline
* Managed Identity

Example:

```bash
az aks stop --resource-group rg-dev --name aks-dev
```

or

```bash
az aks nodepool scale \
--cluster-name aks-dev \
--node-count 0
```

Morning:

```bash
az aks start
```

or scale node pool back.

---

# 14. If CPU reaches its limit, will Kubernetes automatically create new pods?

No.

CPU reaching its limit only causes CPU throttling.

Scaling happens only if:

* HPA is configured.
* HPA threshold is crossed.

If new pod cannot be scheduled:

Cluster Autoscaler creates a new node.

Flow:

High CPU

↓

HPA creates Pod

↓

Scheduler

↓

No Capacity

↓

Cluster Autoscaler

↓

New Node

↓

Pod Scheduled

---

# 15. What is CPU Throttling?

CPU throttling means Linux cgroups restrict the container from consuming more CPU than its configured limit.

Effects:

* Slower response time
* Reduced throughput
* Container is NOT killed

Memory behaves differently.

Memory limit exceeded:

→ Pod becomes OOMKilled.

---

# 16. Can HPA scale based on Memory?

Yes.

Example:

```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80
```

Important:

Memory HPA uses **memory requests**, not limits.

---

# 17. Are Requests and Limits application-level or namespace-level?

### Application Level

Defined inside Deployment.

Example:

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

These apply only to that container.

---

### Namespace Level

LimitRange

Defines:

* Default requests
* Default limits
* Minimum
* Maximum

ResourceQuota

Defines total namespace consumption:

* CPU
* Memory
* Storage
* Object counts

Important:

HPA uses Pod resource requests.

ResourceQuota may prevent HPA from creating additional pods if namespace quotas are exhausted.

---

## Key Interview Tips

* Always troubleshoot from **Load Balancer → Ingress → Service → Pod → Application → Database**.
* Build **once** and promote the **same artifact** across environments.
* Use **immutable image tags**.
* Never use the **latest** tag in production.
* Keep Terraform state files **small and modular**.
* Use **Azure Monitor + Application Insights + Log Analytics** together for observability.
* Design for **self-healing** using probes, HPA, Cluster Autoscaler, and automatic rollback.
* Focus alerts on **SLOs (availability, latency, error rate)** rather than individual CPU or memory spikes.
* Remember the scaling sequence:

  * High CPU → HPA creates pods.
  * No node capacity → Cluster Autoscaler adds nodes.
* CPU limit reached → **Throttling**.
* Memory limit reached → **OOMKilled**.
