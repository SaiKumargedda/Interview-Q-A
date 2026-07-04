# Kubernetes & Terraform Interview Q&A (Deep Dive Version)

---

## 1) Difference between DaemonSet and Deployment

### Deployment

* Manages a ReplicaSet to ensure desired number of pods
* Supports rolling updates, rollback, scaling
* Scheduler decides node placement

### DaemonSet

* Ensures one pod per node (or subset via nodeSelector/taints)
* Automatically adds/removes pods when nodes join/leave

### Deep Insight:

* Deployment = "desired replicas"
* DaemonSet = "desired coverage across nodes"

### Real Use Case:

* Deployment → microservices
* DaemonSet → logging agents, monitoring agents, security agents

---

## 2) What happens when you run kubectl run

kubectl run nginx --image=nginx

### Internally:

1. API request sent to kube-apiserver
2. Pod object created
3. Scheduler assigns node
4. Kubelet pulls image
5. Container runtime starts container

### Important:

* Creates standalone Pod (not managed)
* No self-healing like Deployment

---

## 3) Command to login into a pod

kubectl exec -it <pod-name> -- /bin/bash

### Deep Debug Usage:

* Check environment variables
* Verify mounted volumes
* Test connectivity inside cluster

Example:

kubectl exec -it pod -- curl service-name

---

## 4) Taint and Untaint nodes

### Why Taints?

Prevent unwanted scheduling

kubectl taint nodes node1 key=value:NoSchedule

### Effects:

* NoSchedule → no new pods
* PreferNoSchedule → soft restriction
* NoExecute → evicts running pods

### Remove:

kubectl taint nodes node1 key=value:NoSchedule-

---

## 5) Difference between Ingress and Service

### Service

* Provides stable IP/DNS
* Load balances traffic to pods

### Ingress

* Works at HTTP/HTTPS layer
* Routes based on host/path

### Deep Insight:

* Service = networking abstraction
* Ingress = routing + entry point

---

## 6) Application not accessible - troubleshooting

### Structured Approach:

1. DNS

   * nslookup / dig

2. Network

   * ping / telnet

3. Kubernetes Layer

   * kubectl get pods
   * kubectl get svc
   * kubectl get endpoints

4. Ingress

   * Check rules and backend health

5. Logs

   * kubectl logs

6. Dependencies

   * DB / cache

### Key Insight:

Endpoints empty = selector mismatch

---

## 7) Intermittent timeout issue troubleshooting

### Deep Debug Steps:

1. Resource pressure

   * kubectl top pods

2. Pod restarts

   * kubectl describe pod

3. Scaling issues

   * HPA configuration

4. Network latency

   * Check service-to-service calls

5. External dependencies

   * DB slow queries

6. Load balancer health probes

### RCA Writing Format:

* Issue: Intermittent timeout
* Root Cause: High load + insufficient pods
* Fix: Enabled HPA + optimized DB queries
* Prevention: Added alerts + autoscaling thresholds

---

## 8) Example issue solved independently

### STAR Format:

Situation:
Application downtime in AKS

Task:
Identify root cause

Action:

* Checked pod logs
* Found OOMKilled
* Increased memory limits
* Enabled HPA

Result:
System stabilized and no further outages

---

## 9) Terraform module versioning

### Best Practice:

* Use semantic versioning
* Tag stable releases only

Example:

source = "git::repo-url//module?ref=v1.2.0"

### Why important?

* Reproducibility
* Controlled upgrades

---

## 10) Version operators

~> operator:

* Allows patch updates only

Example:
~> 3.2 → allows 3.2.x but not 3.3

---

## 11) Git module versioning flow

1. Feature branch
2. PR review
3. Merge
4. Tag release
5. Consumers update ref

### Important:

* Do NOT tag every commit

---

## 12) Terraform plan -out=tfplan

### Why use it?

* Ensures same execution plan is applied

Flow:

terraform plan -out=tfplan
terraform apply tfplan

### Used in CI/CD pipelines

---

## 13) Registry vs Git modules

### Registry:

* Built-in versioning
* Easy usage

### Git:

* Full control
* Enterprise usage

---

## 14) Terraform state rollback (Azure)

### Steps:

1. Open Storage Account
2. Go to container
3. Select state file
4. Open Versions tab
5. Choose previous version
6. Restore (Make current)

### After rollback:

terraform plan

---

## 15) Jenkins security

### Layers:

1. Authentication (SSO)
2. Authorization (RBAC)
3. Network security
4. Secrets management
5. Agent isolation
6. Plugin security
7. Logging

### Advanced:

* Use ephemeral agents (K8s)
* Integrate with Key Vault

---

## 16) Scalable Azure architecture

### Layers:

1. Entry → Front Door
2. Routing → App Gateway
3. Compute → AKS
4. Messaging → Service Bus
5. Data → Cosmos DB / SQL
6. Cache → Redis
7. Monitoring → Azure Monitor

### Key Principle:

Design for failure + autoscaling

---

END OF DOCUMENT
