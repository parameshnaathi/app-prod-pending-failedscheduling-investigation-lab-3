# LAB-003: Kubernetes Pod Pending / FailedScheduling Investigation Lab

Hands-on runbook for repeatedly practicing common Kubernetes scheduling failures that leave Pods in `Pending` with `FailedScheduling`.

**Shell:** Windows PowerShell  
**Namespace:** `sre-lab-003`  
**Deployment:** `pending-demo`  
**Service:** `pending-demo`  
**Base image:** `nginx:alpine`

---

## 1. Project structure

```text
app-prod-pending-failedscheduling-investigation-lab-3/
|-- scenarios/
|   |-- 1-insufficient-cpu.yaml
|   |-- 2-node-selector-mismatch.yaml
|   |-- 3-node-affinity-mismatch.yaml
|   |-- 4-untolerated-taint.yaml
|   |-- 5-cordoned-node.yaml
|   |-- 6-pvc-scheduling.yaml
|   `-- 6-pvc.yaml
|-- pending-deployment.yaml
|-- pending-service.yaml
`-- README.md
```

### Code reference

All Kubernetes manifests and scenario patches already exist in the repository.

Use the files above as the source of truth instead of duplicating YAML in this document. This README focuses on execution order, scheduler evidence, troubleshooting, recovery, production lessons, and interview preparation.

---

## 2. What this lab teaches

A Pod can remain `Pending` even when nodes are `Ready` and actual CPU/memory usage is low. The scheduler must find at least one node that satisfies every hard scheduling requirement.

| # | Scenario | Primary scheduler problem |
|---|---|---|
| 1 | Insufficient CPU | Pod resource request does not fit |
| 2 | nodeSelector mismatch | Required node label does not exist |
| 3 | Required node affinity mismatch | Hard affinity rule cannot be satisfied |
| 4 | Untolerated taint | Nodes reject Pod because toleration is missing |
| 5 | Cordoned nodes | Workers are healthy but administratively unschedulable |
| 6 | PVC / StorageClass | Required persistent storage cannot be bound/provisioned |

---

## 3. Core Pending / FailedScheduling mental model

```text
Pod Pending
    |
Is NODE assigned?
    |
    +-- YES --> investigate image/container/startup/runtime path
    |
    `-- NO
         |
         v
kubectl describe pod
         |
FailedScheduling Events
         |
Why did scheduler reject candidate nodes?
         |
Resources? Taints? Selectors? Affinity? Cordon? PVC?
         |
Root Cause -> Mitigate -> Verify
```

Useful mnemonic:

```text
R T A S V

R = Resources
T = Taints / Tolerations
A = Affinity / nodeSelector
S = Schedulability / cordon
V = Volumes / PVC
```

---

## 4. Healthy baseline

Refer to:

```text
pending-deployment.yaml
pending-service.yaml
```

Deploy:

```powershell
kubectl apply -f .\pending-deployment.yaml
kubectl apply -f .\pending-service.yaml
kubectl rollout status deployment/pending-demo -n sre-lab-003
```

Verify:

```powershell
kubectl get deployment pending-demo -n sre-lab-003
kubectl get pods -n sre-lab-003 -o wide
kubectl get svc pending-demo -n sre-lab-003
```

Check EndpointSlice:

```powershell
kubectl get endpointslice `
  -n sre-lab-003 `
  -l kubernetes.io/service-name=pending-demo `
  -o wide
```

Test the Service:

```powershell
kubectl exec -n sre-lab-003 deploy/pending-demo -- wget -qO- http://pending-demo
```

Healthy target:

```text
Deployment: 2/2 Ready
Pods:       1/1 Running
Restarts:   0
HTTP:       Successful
```

---

## 5. Standard troubleshooting workflow

When a scenario is injected, do not immediately reopen the scenario YAML. Treat it like an unknown production incident.

### Step 1 - Observe Pod state

```powershell
kubectl get pods -n sre-lab-003 -o wide
```

If the Pod shows:

```text
STATUS = Pending
NODE   = <none>
```

then the scheduler has not successfully bound it to a node.

### Step 2 - Describe the Pending Pod

```powershell
kubectl describe pod <pending-pod> -n sre-lab-003
```

Focus on:

```text
Node
Status
PodScheduled
Node-Selectors
Tolerations
Requests / Limits
Volumes
Events
FailedScheduling
```

The `Events` section is normally the highest-value evidence.

### Step 3 - Inspect Pod spec after reading Events

```powershell
kubectl get pod <pending-pod> -n sre-lab-003 -o yaml
```

### Step 4 - Inspect candidate nodes

```powershell
kubectl get nodes
kubectl get nodes --show-labels
```

Compact taint / cordon view:

```powershell
kubectl get nodes `
  -o custom-columns="NAME:.metadata.name,TAINTS:.spec.taints,UNSCHEDULABLE:.spec.unschedulable"
```

Deep inspection:

```powershell
kubectl describe node <node-name>
```

### Step 5 - Follow the scheduler reason

```text
Insufficient cpu
-> compare Pod request with Node Allocatable and existing requests

didn't match Pod's node affinity/selector
-> compare Pod placement rule with node labels

untolerated taint
-> compare Node taint with Pod tolerations

node(s) were unschedulable
-> check cordon / spec.unschedulable

unbound PersistentVolumeClaims
-> inspect PVC -> StorageClass -> provisioner -> PV
```

---

## 6. Standard reset between Deployment scenarios

For scenarios that only modify the Deployment Pod template:

```powershell
kubectl rollout undo deployment/pending-demo -n sre-lab-003
kubectl rollout status deployment/pending-demo -n sre-lab-003
kubectl get pods -n sre-lab-003 -o wide
```

Target:

```text
2 Pods
1/1 Running
0 Restarts
```

Scenarios that change taints, cordon state, or PVCs have cleanup steps inside their own section.

---

# 7. Scenario 01 - Insufficient CPU

### Refer to

```text
scenarios\1-insufficient-cpu.yaml
```

This scenario changes the new Pod CPU request from the healthy baseline value to an intentionally impossible request.

### Inject

```powershell
kubectl patch deployment pending-demo `
  -n sre-lab-003 `
  --type strategic `
  --patch-file .\scenarios\1-insufficient-cpu.yaml
```

Observe and investigate:

```powershell
kubectl get pods -n sre-lab-003 -o wide
kubectl describe pod <pending-pod> -n sre-lab-003
```

### Expected evidence

```text
FailedScheduling:
0/3 nodes are available:
1 node(s) had untolerated taint(s),
2 Insufficient cpu.
```

### CPU units

```text
50m   = 0.05 CPU
100m  = 0.10 CPU
200m  = 0.20 CPU
500m  = 0.50 CPU
1000m = 1 CPU

1     = 1 CPU
16    = 16 CPUs
100   = 100 CPUs
```

The scheduler compares:

```text
New Pod CPU request
<=
Node Allocatable CPU - existing scheduled Pod CPU requests
```

Example from this lab:

```text
Worker Allocatable CPU     = 16 CPUs
Existing requests          ~= 0.37 CPU
Remaining schedulable CPU  ~= 15.63 CPUs
New Pod request            = 100 CPUs
```

Therefore the Pod cannot fit.

### Requests vs limits

```text
requests.cpu
-> scheduling reservation

limits.cpu
-> runtime CPU ceiling / throttling behavior
```

Actual CPU utilization may be low while scheduling still fails because scheduling is based on requests and allocatable capacity.

### Why preemption is not helpful

Even if every existing Pod were evicted, a 100-CPU Pod cannot fit on a 16-CPU node.

### Cleanup

Use Section 6.

---

# 8. Scenario 02 - nodeSelector mismatch

### Refer to

```text
scenarios\2-node-selector-mismatch.yaml
```

The scenario requires a node label that does not exist on the worker nodes.

### Inject

```powershell
kubectl patch deployment pending-demo `
  -n sre-lab-003 `
  --type strategic `
  --patch-file .\scenarios\2-node-selector-mismatch.yaml
```

Investigate:

```powershell
kubectl describe pod <pending-pod> -n sre-lab-003
kubectl get nodes --show-labels
kubectl get nodes -L workload-type
```

### Expected evidence

```text
node(s) didn't match Pod's node affinity/selector
```

The Pod requires:

```text
workload-type=high-memory
```

but neither worker has that label.

### Key lesson

`nodeSelector` is a hard requirement. If no feasible node has the required label/value, the scheduler cannot ignore it.

### Cleanup

Use Section 6.

---

# 9. Scenario 03 - Required node affinity mismatch

### Refer to

```text
scenarios\3-node-affinity-mismatch.yaml
```

The scenario uses:

```text
requiredDuringSchedulingIgnoredDuringExecution
```

with a node label expression that no worker satisfies.

### Inject

```powershell
kubectl patch deployment pending-demo `
  -n sre-lab-003 `
  --type strategic `
  --patch-file .\scenarios\3-node-affinity-mismatch.yaml
```

Investigate:

```powershell
kubectl describe pod <pending-pod> -n sre-lab-003
kubectl get pod <pending-pod> -n sre-lab-003 -o yaml
kubectl get nodes --show-labels
```

### Expected evidence

```text
node(s) didn't match Pod's node affinity/selector
```

The hard rule in this lab requires:

```text
node-tier=gpu
```

but no worker has that label/value.

### Important terminology

Node affinity evaluates:

```text
Pod-defined node-affinity expressions
against
Node labels
```

It does not match Pod labels against Node labels.

### nodeSelector vs nodeAffinity

```text
nodeSelector
-> simple exact key=value matching

required nodeAffinity
-> hard scheduling requirement
-> richer operators

preferred nodeAffinity
-> preference only
-> scheduler may still place elsewhere
```

Common operators:

```text
In
NotIn
Exists
DoesNotExist
Gt
Lt
```

### Cleanup

Use Section 6.

---

# 10. Scenario 04 - Untolerated taint

### Refer to

```text
scenarios\4-untolerated-taint.yaml
```

This scenario also modifies the worker nodes directly.

### Apply taints

```powershell
kubectl taint nodes sre-lab-001-worker workload=restricted:NoSchedule
kubectl taint nodes sre-lab-001-worker2 workload=restricted:NoSchedule
```

Verify:

```powershell
kubectl get nodes `
  -o custom-columns="NAME:.metadata.name,TAINTS:.spec.taints"
```

### Inject

```powershell
kubectl patch deployment pending-demo `
  -n sre-lab-003 `
  --type strategic `
  --patch-file .\scenarios\4-untolerated-taint.yaml
```

Investigate:

```powershell
kubectl describe pod <pending-pod> -n sre-lab-003
```

### Expected evidence

```text
0/3 nodes are available:
3 node(s) had untolerated taint(s)
```

The workers have:

```text
workload=restricted:NoSchedule
```

while the new Pod initially lacks a matching toleration.

### Taint vs toleration

```text
Taint
-> applied to Node
-> repels Pods

Toleration
-> applied to Pod
-> allows matching taint to be tolerated
```

A toleration does not force the Pod onto that node.

### NoSchedule vs NoExecute

```text
NoSchedule
-> blocks new incompatible Pods
-> existing Pods normally remain

NoExecute
-> can also evict existing Pods without matching tolerations
```

### Happy-path validation

Update the scenario so the Pod template contains the matching toleration under:

```text
spec.template.spec.tolerations
```

Then patch the Deployment again.

If Kubernetes reports:

```text
Warning: unknown field "tolerations"
deployment.apps/pending-demo patched (no change)
```

then `tolerations` was placed at the wrong API hierarchy.

Correct hierarchy:

```text
Deployment
`-- spec
    `-- template
        `-- spec
            `-- tolerations
```

### Cleanup

Remove the lab taints:

```powershell
kubectl taint nodes sre-lab-001-worker workload=restricted:NoSchedule-
kubectl taint nodes sre-lab-001-worker2 workload=restricted:NoSchedule-
```

Then rollback:

```powershell
kubectl rollout undo deployment/pending-demo -n sre-lab-003
kubectl rollout status deployment/pending-demo -n sre-lab-003
kubectl get pods -n sre-lab-003 -o wide
```

---

# 11. Scenario 05 - Cordoned / unschedulable nodes

### Refer to

```text
scenarios\5-cordoned-node.yaml
```

This scenario demonstrates:

```text
Ready != Schedulable
```

### Cordon both workers

```powershell
kubectl cordon sre-lab-001-worker
kubectl cordon sre-lab-001-worker2
```

Verify:

```powershell
kubectl get nodes
```

Expected:

```text
worker-1   Ready,SchedulingDisabled
worker-2   Ready,SchedulingDisabled
```

### Inject

```powershell
kubectl patch deployment pending-demo `
  -n sre-lab-003 `
  --type strategic `
  --patch-file .\scenarios\5-cordoned-node.yaml
```

Investigate:

```powershell
kubectl describe pod <pending-pod> -n sre-lab-003
kubectl describe node sre-lab-001-worker
```

### Expected evidence

```text
0/3 nodes are available:
1 node(s) had untolerated taint(s),
2 node(s) were unschedulable.
```

Node evidence:

```text
Ready:           True
Unschedulable:   true
```

### Key lesson

```text
Ready
-> node health is OK

Schedulable
-> scheduler is allowed to place new normal Pods there
```

### cordon vs drain

```text
cordon
-> prevent new normal scheduling
-> existing Pods remain

drain
-> cordon + evict eligible workload Pods
```

### Cleanup

```powershell
kubectl uncordon sre-lab-001-worker
kubectl uncordon sre-lab-001-worker2
kubectl get nodes
```

Then rollback:

```powershell
kubectl rollout undo deployment/pending-demo -n sre-lab-003
kubectl rollout status deployment/pending-demo -n sre-lab-003
kubectl get pods -n sre-lab-003 -o wide
```

---

# 12. Scenario 06 - PVC / StorageClass scheduling failure

### Refer to

```text
scenarios\6-pvc.yaml
scenarios\6-pvc-scheduling.yaml
```

Troubleshooting path:

```text
Pod
 |
PVC
 |
StorageClass
 |
Provisioner
 |
PV
```

### Create PVC

```powershell
kubectl apply -f .\scenarios\6-pvc.yaml
kubectl get pvc -n sre-lab-003
```

### Inject PVC-consuming Deployment revision

```powershell
kubectl patch deployment pending-demo `
  -n sre-lab-003 `
  --type strategic `
  --patch-file .\scenarios\6-pvc-scheduling.yaml
```

Investigate:

```powershell
kubectl get pods -n sre-lab-003 -o wide
kubectl describe pod <pending-pod> -n sre-lab-003
kubectl describe pvc pending-demo-data -n sre-lab-003
kubectl get sc
kubectl get pv
```

### Expected failure evidence

Typical scheduler event:

```text
pod has unbound immediate PersistentVolumeClaims
```

The initial failure in this lab is an invalid StorageClass reference.

```text
Pod references PVC
        |
PVC requests invalid StorageClass
        |
StorageClass not found
        |
No dynamic provisioning / binding
        |
PVC Pending
        |
Pod Pending
```

### StorageClass name vs provisioner name

In this kind cluster:

```text
StorageClass name: standard
Provisioner:        rancher.io/local-path
```

These are different values.

Using:

```text
storageClassName: local-path
```

fails if no StorageClass object literally named `local-path` exists, even though the provisioner is `rancher.io/local-path`.

PVC Events may show:

```text
storageclass.storage.k8s.io "local-path" not found
```

### Happy path

Use the actual StorageClass name:

```text
standard
```

or omit `storageClassName` if `standard` is the default StorageClass.

Verify:

```powershell
kubectl get sc
```

Expected in this lab environment:

```text
standard (default)
PROVISIONER       = rancher.io/local-path
VOLUMEBINDINGMODE = WaitForFirstConsumer
```

Recreate the PVC after correcting `6-pvc.yaml`:

```powershell
kubectl delete pvc pending-demo-data -n sre-lab-003
kubectl apply -f .\scenarios\6-pvc.yaml
```

Watch separately:

```powershell
kubectl get pvc -n sre-lab-003 -w
```

In another terminal:

```powershell
kubectl get pods -n sre-lab-003 -w
```

Expected happy path:

```text
PVC Pending
    |
Scheduler identifies consumer placement
    |
rancher.io/local-path provisions PV
    |
PVC Bound
    |
Pod scheduled
    |
ContainerCreating
    |
Running
```

Verify:

```powershell
kubectl get pvc -n sre-lab-003
kubectl get pv
kubectl get pods -n sre-lab-003 -o wide
```

Target:

```text
PVC = Bound
PV  = Bound
Pod = Running
```

### WaitForFirstConsumer nuance

A PVC may legitimately remain Pending for a short period when the StorageClass uses:

```text
volumeBindingMode: WaitForFirstConsumer
```

because provisioning/binding waits for the consuming Pod's placement context.

This is different from:

```text
StorageClass not found
```

which is a configuration failure.

### Cleanup

```powershell
kubectl rollout undo deployment/pending-demo -n sre-lab-003
kubectl rollout status deployment/pending-demo -n sre-lab-003
kubectl delete pvc pending-demo-data -n sre-lab-003
kubectl get pvc -n sre-lab-003
kubectl get pods -n sre-lab-003 -o wide
```

---

# 13. Scenario comparison matrix

| Scenario | Scheduler clue | What to inspect |
|---|---|---|
| Insufficient CPU | `Insufficient cpu` | Pod requests, Node Allocatable, allocated requests |
| nodeSelector mismatch | `didn't match Pod's node affinity/selector` | nodeSelector, node labels |
| Required node affinity | `didn't match Pod's node affinity/selector` | affinity expressions, node labels |
| Untolerated taint | `had untolerated taint(s)` | Node taints, Pod tolerations |
| Cordoned nodes | `node(s) were unschedulable` | `kubectl get nodes`, `.spec.unschedulable` |
| PVC / StorageClass | `unbound PersistentVolumeClaims` | PVC, StorageClass, provisioner, PV |

---

# 14. Production troubleshooting decision tree

```text
Pod Pending
 |
 +-- NODE assigned?
 |     |
 |     +-- YES
 |     |    -> scheduler already placed it
 |     |    -> investigate image, mount, startup, runtime
 |     |
 |     `-- NO
 |          |
 |          v
 |     kubectl describe pod
 |          |
 |          +-- Insufficient cpu/memory?
 |          |     -> Requests vs Allocatable / allocated requests
 |          |
 |          +-- affinity/selector mismatch?
 |          |     -> Pod rule vs node labels
 |          |
 |          +-- untolerated taint?
 |          |     -> Node taint vs Pod tolerations
 |          |
 |          +-- node unschedulable?
 |          |     -> cordon / spec.unschedulable
 |          |
 |          `-- unbound PVC?
 |                -> PVC -> StorageClass -> provisioner -> PV
 |
 `-- correlate with recent rollout / infrastructure change
```

---

# 15. Important Kubernetes behaviors learned

## Ready does not mean schedulable

A node may be healthy while administratively blocked from new normal scheduling.

## Scheduler uses resource requests, not live utilization

A node can be using little CPU while the scheduler reports `Insufficient cpu` because placement is based on requests and allocatable capacity.

## FailedScheduling can contain multiple reasons

Different candidate nodes can fail different scheduler filters in the same event.

## Preemption is not always useful

Preemption cannot fix:

- impossible resource requests
- missing node labels
- hard affinity mismatch
- untolerated taints
- cordoned nodes
- invalid StorageClass configuration

## Toleration does not attract a Pod

It only removes a matching taint-based rejection. Other scheduler filters still apply.

## PVC Pending can be valid or invalid

Valid example:

```text
WaitForFirstConsumer
```

Invalid example:

```text
StorageClass does not exist
```

Always inspect PVC Events.

---

