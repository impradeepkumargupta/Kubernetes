# Kubernetes Pod Troubleshooting Guide

## Comprehensive Troubleshooting Runbook

**Scope:** Kubernetes Pods, including Deployments, StatefulSets, DaemonSets, Jobs, CronJobs, and standalone Pods.

**Audience:** Cloud Engineers, DevOps Engineers, SREs, Platform Engineers, and Kubernetes administrators.

---

# 1. Kubernetes Pod Troubleshooting Philosophy

A Pod problem is usually easier to solve if you follow this order:

1. **Identify the Pod and namespace**
2. **Check Pod status and phase**
3. **Inspect Events**
4. **Inspect container state and exit codes**
5. **Check logs**
6. **Check scheduling and node health**
7. **Check resources**
8. **Check networking**
9. **Check storage**
10. **Check configuration and secrets**
11. **Check security policies and permissions**
12. **Check the application itself**
13. **Check the controller managing the Pod**

A useful high-level model is:

```text
Pod
 |
 +-- Scheduling
 |    +-- Namespace
 |    +-- Node selection
 |    +-- Taints / tolerations
 |    +-- Affinity / anti-affinity
 |    +-- Resource requests
 |
 +-- Admission / Security
 |    +-- RBAC
 |    +-- Pod Security
 |    +-- Admission Webhooks
 |    +-- SecurityContext
 |
 +-- Container Creation
 |    +-- Image
 |    +-- Registry
 |    +-- Runtime
 |
 +-- Application Startup
 |    +-- Command / args
 |    +-- Environment
 |    +-- ConfigMap / Secret
 |    +-- Volume mounts
 |
 +-- Runtime Health
 |    +-- Readiness
 |    +-- Liveness
 |    +-- Startup probes
 |    +-- Memory / CPU
 |
 +-- Networking
 |    +-- CNI
 |    +-- DNS
 |    +-- Service
 |    +-- NetworkPolicy
 |
 +-- Storage
      +-- PVC
      +-- PV
      +-- CSI
      +-- Mount permissions
```

---

# 2. First Response: Basic Pod Investigation

Start with:

```bash
kubectl get pods -A
```

Then:

```bash
kubectl get pod <pod-name> -n <namespace> -o wide
```

Example:

```bash
kubectl get pod nginx-7d8b49557c-x2abc -n production -o wide
```

Check detailed information:

```bash
kubectl describe pod nginx-7d8b49557c-x2abc -n production
```

Check events:

```bash
kubectl get events -n production --sort-by='.lastTimestamp'
```

Or:

```bash
kubectl get events -n production --sort-by='.metadata.creationTimestamp'
```

Check logs:

```bash
kubectl logs <pod-name> -n <namespace>
```

For a specific container:

```bash
kubectl logs <pod-name> -n <namespace> -c <container-name>
```

Previous crashed container:

```bash
kubectl logs <pod-name> -n <namespace> -c <container-name> --previous
```

Follow logs:

```bash
kubectl logs -f <pod-name> -n <namespace>
```

For all containers:

```bash
kubectl logs <pod-name> -n <namespace> --all-containers=true
```

---

# 3. Understand Pod STATUS Before Troubleshooting

Common Pod statuses include:

| Status | Typical Meaning |
|---|---|
| Pending | Pod has not successfully started |
| Running | At least one container is running |
| Succeeded | Containers completed successfully |
| Failed | Containers terminated unsuccessfully |
| Unknown | Kubernetes cannot reliably determine state |
| CrashLoopBackOff | Container repeatedly crashes |
| ImagePullBackOff | Kubernetes cannot pull the image |
| ErrImagePull | Initial image pull failed |
| CreateContainerConfigError | Container configuration cannot be constructed |
| CreateContainerError | Runtime failed to create container |
| ContainerCreating | Container creation is still in progress |
| Terminating | Pod deletion is in progress |
| Evicted | Kubelet evicted the Pod |
| Completed | Usually a Job/CronJob completed |

Important:

> `kubectl get pods` displays a human-friendly STATUS. The actual reason is often found in `kubectl describe pod` and the Pod's container state.

---

# 4. Group A — Pod Stuck in Pending

## Symptoms

```text
NAME                     READY   STATUS    RESTARTS   AGE
myapp-7d9f8c8f9f-abcde   0/1     Pending   0          10m
```

## Main Causes

1. Insufficient CPU
2. Insufficient memory
3. No suitable node
4. Node selector mismatch
5. Node affinity mismatch
6. Taints without matching tolerations
7. Pod anti-affinity constraints
8. Topology constraints
9. PVC not bound
10. Scheduling policy restrictions
11. Namespace resource quota
12. LimitRange constraints
13. Cluster has no schedulable nodes
14. Pod has an invalid scheduling configuration

## Diagnosis

```bash
kubectl describe pod <pod> -n <namespace>
```

Look at:

```text
Events:
  Warning  FailedScheduling
```

Example:

```text
0/3 nodes are available:
2 Insufficient memory,
1 node(s) didn't match Pod's node affinity/selector.
```

### Check Nodes

```bash
kubectl get nodes
```

```bash
kubectl describe nodes
```

### Check Allocatable Resources

```bash
kubectl describe node <node-name>
```

Look for:

```text
Allocatable:
  cpu:
  memory:
```

### Check Resource Usage

If Metrics Server is installed:

```bash
kubectl top nodes
kubectl top pods -A
```

---

## Cause A1 — Insufficient CPU

Example:

```text
0/3 nodes are available: 3 Insufficient cpu.
```

### Why it happens

The Pod requests more CPU than any available node can provide.

Example:

```yaml
resources:
  requests:
    cpu: "8"
```

If every node has less than 8 CPU allocatable, scheduling fails.

### Solutions

Reduce requests if the application does not need that much:

```yaml
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"
```

Or add/resize nodes.

Check:

```bash
kubectl describe node <node-name>
```

---

## Cause A2 — Insufficient Memory

Example:

```text
0/4 nodes are available: 4 Insufficient memory.
```

### Solutions

Reduce the memory request:

```yaml
resources:
  requests:
    memory: "512Mi"
```

Or add nodes with more memory.

Do not simply remove resource requests in production without understanding the workload.

---

## Cause A3 — Node Selector Mismatch

Pod:

```yaml
spec:
  nodeSelector:
    workload: gpu
```

But nodes have:

```text
workload=general
```

### Diagnosis

```bash
kubectl get nodes --show-labels
```

Or:

```bash
kubectl get nodes -l workload=gpu
```

### Solution

Either label an appropriate node:

```bash
kubectl label node <node-name> workload=gpu
```

Or correct the Pod specification.

---

## Cause A4 — Taint Without Toleration

Node:

```text
dedicated=gpu:NoSchedule
```

Pod has no matching toleration.

### Diagnosis

```bash
kubectl describe node <node-name>
```

Look for:

```text
Taints:
  dedicated=gpu:NoSchedule
```

### Solution

Add a toleration:

```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

Only add tolerations when the workload is actually intended to run there.

---

## Cause A5 — PVC Preventing Scheduling

Check:

```bash
kubectl get pvc -n <namespace>
```

If:

```text
STATUS   Pending
```

inspect it:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Possible causes:

- No matching PV
- StorageClass missing
- CSI driver problem
- Wrong access mode
- Insufficient storage
- Zone/topology conflict

---

# 5. Group B — ImagePullBackOff / ErrImagePull

## Symptoms

```text
STATUS
ImagePullBackOff
```

or:

```text
ErrImagePull
```

## Common Causes

1. Image does not exist
2. Wrong image tag
3. Private registry authentication failure
4. Registry unavailable
5. Network/DNS issue
6. Image architecture mismatch
7. Registry rate limiting
8. Invalid image reference
9. Image pull secret missing
10. Container runtime problem

## Diagnosis

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
Failed to pull image
```

Example:

```text
pull access denied
```

or:

```text
manifest unknown
```

---

## Cause B1 — Wrong Image Tag

Example:

```yaml
image: nginx:does-not-exist
```

Error:

```text
manifest unknown
```

### Solution

Check available tags in the registry and update:

```yaml
image: nginx:1.29
```

Avoid relying on `latest` for production deployments.

---

## Cause B2 — Private Registry Authentication

Example:

```text
pull access denied
```

Check:

```bash
kubectl get secrets -n <namespace>
```

Check Pod:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Look for:

```yaml
imagePullSecrets:
- name: regcred
```

Create a registry secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>
```

Then:

```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

---

## Cause B3 — Registry Network/DNS Problem

Check the node and cluster networking.

Inspect events:

```bash
kubectl describe pod <pod> -n <namespace>
```

On the node, inspect container runtime logs if you have node access.

For containerd:

```bash
journalctl -u containerd
```

For kubelet:

```bash
journalctl -u kubelet
```

---

# 6. Group C — CrashLoopBackOff

## Symptoms

```text
NAME       READY   STATUS             RESTARTS
myapp      0/1     CrashLoopBackOff   10
```

## Meaning

The container starts, exits/crashes, Kubernetes restarts it, and this repeats.

## Common Causes

1. Application crash
2. Incorrect command
3. Incorrect arguments
4. Missing environment variable
5. Missing ConfigMap
6. Missing Secret
7. Wrong configuration
8. Dependency unavailable
9. Permission problem
10. OOMKilled
11. Liveness probe failure
12. Port/configuration mismatch
13. Application exits immediately by design
14. Incorrect working directory
15. Missing mounted file

## First Commands

```bash
kubectl logs <pod> -n <namespace>
```

Then:

```bash
kubectl logs <pod> -n <namespace> --previous
```

The `--previous` option is particularly important when the current container has already restarted.

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

---

## Cause C1 — Application Crash

Example log:

```text
Error: DATABASE_URL is not configured
```

Solution:

Check:

```bash
kubectl get deployment <deployment> -n <namespace> -o yaml
```

Check ConfigMaps:

```bash
kubectl get configmap -n <namespace>
```

Check Secrets:

```bash
kubectl get secret -n <namespace>
```

Correct the configuration and redeploy.

---

## Cause C2 — Incorrect Command

Example:

```yaml
command:
- /app/start.sh
```

But `/app/start.sh` does not exist.

Log:

```text
exec /app/start.sh: no such file or directory
```

Solutions:

- Verify image contents
- Verify executable path
- Verify file permissions
- Correct `command`
- Correct `args`

---

## Cause C3 — Application Exits Successfully

Some applications are designed to run and exit.

Example:

```bash
echo "hello"
```

If deployed as a long-running Deployment, the container exits immediately.

For batch work, use a Job:

```yaml
apiVersion: batch/v1
kind: Job
```

For a server, ensure the application has a long-running foreground process.

---

# 7. Group D — OOMKilled

## Symptoms

```text
Reason: OOMKilled
Exit Code: 137
```

## Meaning

The container was terminated because it exceeded its available memory or the node experienced memory pressure.

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Check:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

## Common Causes

1. Memory limit too low
2. Application memory leak
3. Large workload
4. JVM heap too large
5. Python process consuming excessive memory
6. Cache growth
7. Node memory pressure

## Solutions

### Increase memory limit

```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

### Investigate the application

For JVM:

```text
-Xmx
```

must be consistent with the container memory limit.

For Python:

Investigate object growth, worker count, and caching.

Do not blindly increase memory indefinitely. Determine whether usage is expected or represents a leak.

---

# 8. Group E — CreateContainerConfigError

## Symptoms

```text
CreateContainerConfigError
```

## Common Causes

1. Missing Secret
2. Missing ConfigMap
3. Invalid environment reference
4. Invalid volume reference
5. Invalid configuration
6. Incorrect Secret key
7. Incorrect ConfigMap key

## Diagnosis

```bash
kubectl describe pod <pod> -n <namespace>
```

Example:

```text
Error: secret "database-secret" not found
```

Check:

```bash
kubectl get secret database-secret -n <namespace>
```

### Important

Secrets and ConfigMaps are namespace-scoped.

A Secret in namespace `dev` cannot automatically be referenced by a Pod in namespace `prod`.

---

# 9. Group F — CreateContainerError

## Symptoms

```text
CreateContainerError
```

The image may have been pulled, but the container could not be created.

## Common Causes

1. Invalid volume mount
2. Runtime error
3. Permission issue
4. Invalid mount configuration
5. Security restrictions
6. Container runtime problem

Start with:

```bash
kubectl describe pod <pod> -n <namespace>
```

Then inspect:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

---

# 10. Group G — ContainerCreating for Too Long

## Symptoms

```text
ContainerCreating
```

for a long time.

## Common Causes

1. Image pulling slowly
2. PVC mount failure
3. CNI networking problem
4. Volume attach problem
5. Secret/ConfigMap mount problem
6. Node problem
7. CSI driver problem

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Events usually reveal the problem.

Check PVC:

```bash
kubectl get pvc -n <namespace>
```

Check node:

```bash
kubectl get pod <pod> -n <namespace> -o wide
kubectl describe node <node-name>
```

---

# 11. Group H — Pod Running but Not Ready

## Symptoms

```text
READY
0/1
```

while:

```text
STATUS
Running
```

This usually means the container is running but failing its readiness condition.

## Common Causes

1. Readiness probe failing
2. Application not ready
3. Wrong probe path
4. Wrong probe port
5. Wrong HTTP scheme
6. Dependency unavailable
7. Startup takes longer than expected
8. Service binding only to localhost
9. DNS failure
10. Authentication/configuration issue

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
Readiness probe failed
```

---

## Example

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

If the application exposes:

```text
/healthz
```

instead of:

```text
/health
```

the probe will fail.

### Solution

Correct the probe:

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
```

---

# 12. Group I — Liveness Probe Failures

## Symptoms

Pod repeatedly restarts.

Events:

```text
Liveness probe failed
```

## Common Causes

1. Probe path incorrect
2. Probe port incorrect
3. Application startup takes too long
4. Timeout too short
5. Application is overloaded
6. Probe endpoint is expensive
7. Network issue inside Pod
8. Incorrect HTTP scheme

## Solution

Use a `startupProbe` for slow-starting applications.

Example:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
```

A startup probe prevents liveness checks from killing an application while it is still starting.

---

# 13. Group J — Startup Probe Failures

## Symptoms

```text
Startup probe failed
```

## Causes

- Application startup takes longer than configured
- Wrong startup endpoint
- Wrong port
- Wrong protocol
- Application crashes during startup

## Diagnosis

```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
```

Adjust:

```yaml
failureThreshold
periodSeconds
timeoutSeconds
```

based on measured startup time rather than arbitrary large values.

---

# 14. Group K — Pod Evicted

## Symptoms

```text
STATUS
Evicted
```

## Common Causes

1. Node memory pressure
2. Node disk pressure
3. Ephemeral storage exhaustion
4. PID pressure
5. Kubernetes eviction thresholds

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Check node:

```bash
kubectl describe node <node-name>
```

Look for:

```text
MemoryPressure
DiskPressure
PIDPressure
```

Check:

```bash
kubectl get nodes
```

---

# 15. Group L — CPU Throttling

A Pod can be `Running` and `Ready` while still performing badly.

## Symptoms

- High latency
- Slow requests
- CPU appears capped
- Application is responsive but slow

A CPU limit can cause throttling.

Example:

```yaml
resources:
  limits:
    cpu: "500m"
```

If the application needs sustained CPU above 500m, it may be throttled.

Check:

```bash
kubectl top pod <pod> -n <namespace>
```

Inspect metrics from your monitoring system.

Possible solutions:

- Increase CPU limit
- Increase CPU request
- Optimize application
- Scale horizontally
- Review whether CPU limits are appropriate for the workload

---

# 16. Group M — Pod Networking Problems

## Symptoms

- Pod cannot reach another Pod
- Pod cannot reach Service
- Pod cannot reach Internet
- Service cannot reach Pod
- DNS resolution fails
- Connection timeout
- Connection refused

## First Checks

Get Pod IP:

```bash
kubectl get pod <pod> -n <namespace> -o wide
```

Check Services:

```bash
kubectl get svc -n <namespace>
```

Check Endpoints:

```bash
kubectl get endpoints -n <namespace>
```

Or:

```bash
kubectl get endpointslices -n <namespace>
```

---

# 17. Group N — DNS Problems

## Symptoms

```text
Temporary failure in name resolution
```

or:

```text
Could not resolve host
```

## Test from Pod

```bash
kubectl exec -it <pod> -n <namespace> -- nslookup kubernetes.default
```

If `nslookup` does not exist, use a debugging image.

Check CoreDNS:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Check:

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

Check DNS Service:

```bash
kubectl get svc -n kube-system kube-dns
```

Common causes:

- CoreDNS unavailable
- NetworkPolicy blocking DNS
- Incorrect Pod DNS configuration
- Node networking problem
- CNI problem

---

# 18. Group O — Service Does Not Reach Pod

## Symptoms

```text
Service exists but application is unreachable.
```

Check:

```bash
kubectl get svc <service> -n <namespace> -o yaml
```

Check:

```bash
kubectl get endpoints <service> -n <namespace>
```

If there are no endpoints, investigate selectors.

Example Service:

```yaml
selector:
  app: frontend
```

Pod:

```yaml
labels:
  app: front-end
```

These do not match.

### Solution

Make selectors and labels consistent.

---

# 19. Group P — NetworkPolicy Blocking Traffic

## Symptoms

- Pod-to-Pod communication fails
- DNS may fail
- Service is reachable from some workloads but not others

Check:

```bash
kubectl get networkpolicy -A
```

Inspect:

```bash
kubectl describe networkpolicy <policy> -n <namespace>
```

Common mistakes:

- Default deny policy
- Missing ingress rule
- Missing egress rule
- DNS traffic not allowed
- Wrong namespace selector
- Wrong pod selector

Example DNS egress often needs UDP/TCP 53 access to the cluster DNS service.

---

# 20. Group Q — Volume / PVC Problems

## Symptoms

```text
MountVolume.SetUp failed
```

or:

```text
FailedAttachVolume
```

or:

```text
FailedMount
```

Check:

```bash
kubectl get pvc -n <namespace>
```

```bash
kubectl describe pvc <pvc> -n <namespace>
```

Check PV:

```bash
kubectl get pv
```

Check StorageClass:

```bash
kubectl get storageclass
```

---

# 21. Common Storage Causes

## Cause Q1 — PVC Pending

Possible causes:

- No StorageClass
- No provisioner
- CSI driver problem
- No suitable PV
- Requested size unavailable
- Access mode mismatch

## Cause Q2 — Wrong Access Mode

Example:

```yaml
accessModes:
- ReadWriteOnce
```

Multiple Pods across nodes may require a storage backend that supports the intended access pattern.

Do not assume `ReadWriteOnce` means only one Pod can ever mount the volume; semantics depend on the storage implementation and attachment topology.

---

# 22. Group R — Permission Denied on Mounted Volume

## Symptoms

```text
Permission denied
```

Example:

```text
/app/data/file.txt: Permission denied
```

Check:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
```

The mounted volume may be owned by another UID/GID.

Possible solution:

```yaml
securityContext:
  fsGroup: 1000
```

However, `fsGroup` behavior depends on the volume type and CSI driver.

Do not change permissions broadly with:

```bash
chmod -R 777
```

without understanding the security impact.

---

# 23. Group S — ConfigMap Problems

## Symptoms

- Missing configuration
- Application startup failure
- Missing mounted file
- Environment variable absent

Check:

```bash
kubectl get configmap -n <namespace>
```

```bash
kubectl describe configmap <name> -n <namespace>
```

Check Pod:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Common problems:

- Wrong ConfigMap name
- Wrong key
- Wrong namespace
- ConfigMap not mounted where application expects
- Application does not reload configuration automatically

---

# 24. Group T — Secret Problems

Common causes:

1. Secret does not exist
2. Wrong namespace
3. Wrong key
4. Secret mounted at wrong path
5. Application expects different variable name
6. Secret permissions/security issue

Check:

```bash
kubectl get secret -n <namespace>
```

Do not casually print Secret contents into terminals, CI logs, tickets, or chat.

To inspect keys without exposing values:

```bash
kubectl get secret <secret-name> -n <namespace> -o json
```

---

# 25. Group U — RBAC / ServiceAccount Problems

## Symptoms

Application or Kubernetes client receives:

```text
Forbidden
```

Example:

```text
User "system:serviceaccount:prod:myapp" cannot list resource "pods"
```

Check:

```bash
kubectl get serviceaccount -n <namespace>
```

Check Role:

```bash
kubectl get role -n <namespace>
```

Check RoleBinding:

```bash
kubectl get rolebinding -n <namespace>
```

Check permissions:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:<namespace>:<serviceaccount> \
  -n <namespace>
```

For cluster-wide permission:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:<namespace>:<serviceaccount> \
  --all-namespaces
```

---

# 26. Group V — SecurityContext Problems

Common causes:

- Container runs as non-root
- File permissions incompatible with UID
- Read-only filesystem
- Privileged operation denied
- Linux capability missing
- Seccomp restrictions
- AppArmor/SELinux restrictions
- Pod Security admission restrictions

Example:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
```

An application that writes to `/tmp` may fail if the filesystem is read-only unless an appropriate writable volume is mounted.

---

# 27. Group W — Pod Security Admission / Admission Webhook Failures

Symptoms may appear during Pod creation rather than after the Pod starts.

Examples:

```text
admission webhook denied the request
```

or:

```text
violates PodSecurity
```

Check:

```bash
kubectl get namespace <namespace> --show-labels
```

Inspect admission-related errors returned by:

```bash
kubectl apply -f deployment.yaml
```

Common causes:

- Privileged container
- Host network
- Host PID/IPC
- HostPath
- Running as root
- Added Linux capabilities
- Disallowed security context

Fix the workload to meet the cluster's security policy rather than disabling security controls without justification.

---

# 28. Group X — Node Problems Affecting Pods

A Pod can fail because the node is unhealthy.

Check:

```bash
kubectl get nodes
```

Look for:

```text
NotReady
```

Inspect:

```bash
kubectl describe node <node-name>
```

Important conditions:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

Check kubelet:

```bash
journalctl -u kubelet
```

Check container runtime:

```bash
journalctl -u containerd
```

Common node problems:

- Disk full
- Memory exhaustion
- Container runtime failure
- Kubelet failure
- CNI failure
- Kernel issues
- Network failure
- File descriptor exhaustion
- PID exhaustion

---

# 29. Group Y — Pod Terminating Too Long

## Symptoms

```text
Terminating
```

for a long period.

## Common Causes

1. Finalizer
2. PreStop hook
3. Application does not terminate
4. Volume detach issue
5. Node unreachable
6. API object/controller issue

Check:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Look for:

```yaml
metadata:
  finalizers:
```

Check:

```yaml
lifecycle:
  preStop:
```

Avoid immediately using:

```bash
kubectl delete pod <pod> --force --grace-period=0
```

Force deletion can leave application processes or external resources in an unexpected state. Use it only when the consequences are understood.

---

# 30. Group Z — Application Cannot Connect to Database

## Symptoms

```text
connection refused
```

or:

```text
timeout connecting to database
```

Troubleshoot in layers.

### Layer 1 — DNS

```bash
nslookup database-service
```

### Layer 2 — Network

```bash
nc -vz database-service 5432
```

### Layer 3 — Service

```bash
kubectl get svc database-service -n <namespace>
```

### Layer 4 — Endpoints

```bash
kubectl get endpoints database-service -n <namespace>
```

### Layer 5 — Database Pod

```bash
kubectl get pods -l app=database -n <namespace>
```

### Layer 6 — Credentials

Check Secret reference.

### Layer 7 — Database Configuration

Verify:

- host
- port
- database name
- username
- password
- TLS mode

---

# 31. Group AA — Wrong Environment Variables

Inspect the Pod specification:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Look for:

```yaml
env:
envFrom:
```

Typical problems:

```text
DATABASE_HOST
DATABASE_URL
DB_HOST
```

may be inconsistently named.

For safe debugging, inspect variable names rather than printing sensitive values.

---

# 32. Group AB — Wrong Port

There are several different ports to distinguish:

```text
containerPort
Service port
Service targetPort
NodePort
Ingress/backend port
```

Example:

```yaml
containers:
- ports:
  - containerPort: 8080
```

Service:

```yaml
ports:
- port: 80
  targetPort: 8080
```

This means:

```text
Client -> Service:80 -> Pod:8080
```

A mismatch can result in:

```text
connection refused
```

or traffic reaching the wrong process.

---

# 33. Group AC — Application Listening Only on localhost

A common container issue:

Application listens on:

```text
127.0.0.1:8080
```

instead of:

```text
0.0.0.0:8080
```

The application works from inside the container but cannot be reached through the Pod network.

Check from the container:

```bash
kubectl exec -it <pod> -n <namespace> -- ss -lntp
```

If `ss` is unavailable, use an appropriate debugging image or inspect application configuration.

Solution:

Configure the application to listen on:

```text
0.0.0.0
```

---

# 34. Group AD — Multi-Container Pod Problems

A Pod can contain multiple containers:

```text
Pod
 |
 +-- application
 +-- sidecar
 +-- logging agent
 +-- proxy
```

Check all containers:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].name}'
```

Logs:

```bash
kubectl logs <pod> -n <namespace> -c <container>
```

Check restart counts:

```bash
kubectl get pod <pod> -n <namespace>
```

A sidecar can cause:

- readiness failures
- startup delays
- network interception issues
- resource exhaustion
- unexpected restarts

---

# 35. Group AE — Init Container Problems

## Symptoms

Pod remains:

```text
Init:0/1
```

or:

```text
Init:CrashLoopBackOff
```

List init containers:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.initContainers[*].name}'
```

Logs:

```bash
kubectl logs <pod> -n <namespace> -c <init-container>
```

Describe:

```bash
kubectl describe pod <pod> -n <namespace>
```

Common causes:

- Database migration failure
- Configuration generation failure
- Dependency unavailable
- Permission problem
- Incorrect command
- Missing Secret/ConfigMap

---

# 36. Group AF — Deployment Problems Masquerading as Pod Problems

Sometimes the Pod is not the root cause.

Check:

```bash
kubectl get deployment -n <namespace>
```

```bash
kubectl describe deployment <deployment> -n <namespace>
```

Check ReplicaSets:

```bash
kubectl get rs -n <namespace>
```

Check rollout:

```bash
kubectl rollout status deployment/<deployment> -n <namespace>
```

Check history:

```bash
kubectl rollout history deployment/<deployment> -n <namespace>
```

Rollback if an incorrect release caused the incident:

```bash
kubectl rollout undo deployment/<deployment> -n <namespace>
```

Always verify the deployment history and change that caused the failure before rolling back production.

---

# 37. Group AG — Job / CronJob Pod Failures

For Jobs:

```bash
kubectl get jobs -n <namespace>
```

```bash
kubectl describe job <job> -n <namespace>
```

For CronJobs:

```bash
kubectl get cronjobs -n <namespace>
```

Common causes:

- Job command failure
- Incorrect schedule
- Secret/configuration missing
- Resource limits
- Backoff limit exceeded
- Application exits with non-zero code

Check:

```bash
kubectl get pods -n <namespace> --selector=job-name=<job-name>
```

---

# 38. Group AH — StatefulSet Pod Problems

StatefulSets introduce additional failure modes:

- Persistent storage
- Stable network identity
- Ordered startup
- Pod identity
- PVC retention
- Node/storage topology

Check:

```bash
kubectl get statefulset -n <namespace>
```

```bash
kubectl describe statefulset <name> -n <namespace>
```

Check PVCs:

```bash
kubectl get pvc -n <namespace>
```

A StatefulSet Pod may fail because its specific persistent volume cannot attach to the selected node.

---

# 39. Group AI — DaemonSet Pod Problems

DaemonSets normally create one Pod per eligible node.

Check:

```bash
kubectl get daemonset -n <namespace>
```

```bash
kubectl describe daemonset <name> -n <namespace>
```

If a node has no DaemonSet Pod, investigate:

- node selectors
- affinity
- taints
- tolerations
- resource availability
- DaemonSet update strategy

---

# 40. Group AJ — Pod Affinity / Anti-Affinity

Example:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: frontend
      topologyKey: kubernetes.io/hostname
```

This can prevent scheduling if there is no eligible node.

Diagnosis:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for:

```text
FailedScheduling
```

Review:

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`
- topology keys
- node labels

---

# 41. Group AK — Topology Spread Constraints

Example:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
```

If the cluster cannot satisfy the topology requirement, scheduling can fail.

Check Pod YAML:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Check node labels:

```bash
kubectl get nodes --show-labels
```

---

# 42. Group AL — Ephemeral Storage Problems

Containers also consume node disk through:

- writable container layers
- logs
- `/tmp`
- `emptyDir`
- temporary application files

Check:

```bash
kubectl describe node <node-name>
```

Look for:

```text
DiskPressure
```

You can specify:

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

Also inspect application logs and temporary-file behavior.

---

# 43. Group AM — Image Architecture Mismatch

A container image may be built for:

```text
linux/amd64
```

while the node uses:

```text
linux/arm64
```

Potential error:

```text
exec format error
```

Check node architecture:

```bash
kubectl get nodes -o wide
```

Check image manifest using your registry tooling.

Solution:

Build and publish a multi-architecture image when required:

```text
linux/amd64
linux/arm64
```

---

# 44. Group AN — Container Runtime Problems

Containerd or another runtime can fail independently of the Pod specification.

Check:

```bash
systemctl status containerd
```

```bash
journalctl -u containerd
```

Check kubelet:

```bash
systemctl status kubelet
```

```bash
journalctl -u kubelet
```

Symptoms include:

- container creation failures
- image pull failures
- sandbox creation failures
- runtime timeouts
- container startup failures

---

# 45. Group AO — CNI / Pod Sandbox Problems

Errors such as:

```text
FailedCreatePodSandBox
```

often indicate networking/CNI problems.

Check:

```bash
kubectl describe pod <pod> -n <namespace>
```

Check CNI components:

```bash
kubectl get pods -n kube-system
```

Depending on the CNI, inspect its DaemonSet and logs.

Common causes:

- CNI agent down
- IP address exhaustion
- broken node networking
- CNI configuration error
- stale network state
- cloud network integration failure

---

# 46. Group AP — Too Many Pods / IP Exhaustion

A node or cluster can run out of available Pod IP addresses.

Symptoms may include:

```text
FailedCreatePodSandBox
```

or networking allocation errors.

Check:

```bash
kubectl get nodes
```

Inspect CNI/IP allocation metrics and cloud networking configuration.

For cloud-managed Kubernetes, also check the cloud provider's subnet and Pod IP limits.

---

# 47. Group AQ — Pod Cannot Access External Internet

Test from a debugging container:

```bash
curl -I https://example.com
```

Test DNS:

```bash
nslookup example.com
```

Potential causes:

1. DNS failure
2. Egress NetworkPolicy
3. NAT gateway problem
4. Firewall
5. Proxy configuration
6. Cloud security rules
7. CNI routing problem

Separate DNS failure from TCP connectivity failure before changing firewall rules.

---

# 48. Group AR — Ingress Problems

If the application works through the Service but not through Ingress:

Test directly:

```bash
kubectl port-forward svc/<service> 8080:80 -n <namespace>
```

Then:

```bash
curl http://localhost:8080
```

If this works, investigate:

- Ingress configuration
- Ingress Controller
- DNS
- TLS
- backend Service
- backend endpoints
- path rules
- host rules

Check:

```bash
kubectl get ingress -n <namespace>
```

```bash
kubectl describe ingress <name> -n <namespace>
```

---

# 49. Group AS — TLS / Certificate Problems

Common symptoms:

```text
certificate verify failed
```

or:

```text
x509: certificate signed by unknown authority
```

Check:

- Secret exists
- TLS Secret has correct keys
- certificate matches hostname
- certificate is not expired
- CA trust is correct
- Ingress references the correct Secret

Example:

```bash
kubectl get secret <tls-secret> -n <namespace>
```

Do not expose private keys while debugging.

---

# 50. Group AT — Time / Clock Problems

Some applications fail because system time differs significantly.

Symptoms:

```text
token expired
certificate not yet valid
JWT validation failed
```

Check node time:

```bash
date
```

Check NTP/time synchronization on the node.

This is especially relevant for:

- TLS
- JWT
- cloud credentials
- signed requests

---

# 51. Group AU — File Descriptor / Process Limits

Symptoms:

```text
too many open files
```

or:

```text
cannot create new process
```

Potential causes:

- application leaks file descriptors
- excessive connections
- PID exhaustion
- node-level limits

Check node:

```bash
kubectl describe node <node-name>
```

Check application metrics and process behavior.

---

# 52. Group AV — Kubernetes API Access from Pod

If an application talks to Kubernetes API:

Check ServiceAccount:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.serviceAccountName}'
```

Check permission:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:<namespace>:<serviceaccount> \
  -n <namespace>
```

Common causes:

- wrong ServiceAccount
- missing RoleBinding
- wrong namespace
- insufficient permissions
- application uses wrong API endpoint

---

# 53. Group AW — Graceful Shutdown Problems

Applications may receive SIGTERM during:

- deployment rollout
- scaling
- node drain
- Pod deletion

If shutdown is too slow:

- requests may be dropped
- connections may terminate
- rollout may stall

Configure:

```yaml
terminationGracePeriodSeconds: 60
```

and implement proper SIGTERM handling in the application.

Use:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

only when there is a concrete reason. Prefer application-level graceful shutdown.

---

# 54. Group AX — Readiness vs Liveness Misconfiguration

A common design mistake is using liveness to test dependencies.

Bad pattern:

```text
Liveness = application + database + Redis + external API
```

If the database goes down, Kubernetes may restart a perfectly healthy application repeatedly.

Better:

```text
Liveness:
    Is the process/application alive?

Readiness:
    Can this Pod safely receive traffic?

Startup:
    Has startup completed?
```

This separation prevents cascading restarts.

---

# 55. Group AY — Resource Requests and Limits Misconfiguration

Example:

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

Requests influence scheduling.

Limits constrain resource consumption.

Typical problems:

```text
Very high requests -> Pod stays Pending
Very low memory limit -> OOMKilled
Very low CPU limit -> throttling
No requests -> poor scheduling/resource guarantees
```

Use historical monitoring data to size resources.

---

# 56. Group AZ — Namespace Quota Problems

A Pod may fail scheduling or creation because the namespace has a ResourceQuota.

Check:

```bash
kubectl get resourcequota -n <namespace>
```

```bash
kubectl describe resourcequota -n <namespace>
```

Example:

```text
pods: 10/10
requests.cpu: 8/8
requests.memory: 16Gi/16Gi
```

Solution:

- remove unused workloads
- reduce requested resources
- adjust quota
- move workload if appropriate

Quota changes should follow your cluster governance process.

---

# 57. Group BA — LimitRange Problems

Check:

```bash
kubectl get limitrange -n <namespace>
```

```bash
kubectl describe limitrange -n <namespace>
```

A LimitRange may automatically assign or enforce CPU/memory requests and limits.

This can explain unexpected resource settings.

---

# 58. Group BB — Namespace or Label Mistakes

Always verify namespace:

```bash
kubectl config view --minify --output 'jsonpath={..namespace}'
```

Then explicitly specify:

```bash
-n <namespace>
```

Check labels:

```bash
kubectl get pod <pod> -n <namespace> --show-labels
```

A surprising number of Service, NetworkPolicy, and controller problems are caused by label mismatches.

---

# 59. Group BC — Controller Selector Problems

Deployment:

```yaml
selector:
  matchLabels:
    app: frontend
```

Template:

```yaml
template:
  metadata:
    labels:
      app: frontend
```

These must align.

If selectors are wrong, the controller may not manage the intended Pods.

---

# 60. Group BD — Rollout Creates Bad Pods

Check:

```bash
kubectl rollout status deployment/<deployment> -n <namespace>
```

Then:

```bash
kubectl rollout history deployment/<deployment> -n <namespace>
```

Compare old and new ReplicaSets:

```bash
kubectl get rs -n <namespace>
```

Look for:

- changed image
- changed environment variables
- changed probes
- changed resources
- changed ConfigMaps
- changed Secrets
- changed ServiceAccount
- changed security context

---

# 61. Group BE — Debugging a Minimal Container

Many production images do not contain:

```text
curl
wget
ping
nslookup
bash
sh
```

Do not rebuild production images solely to troubleshoot unless necessary.

Use an ephemeral debug container where supported:

```bash
kubectl debug -it <pod> -n <namespace> \
  --image=nicolaka/netshoot \
  --target=<container>
```

Or run a temporary debugging Pod:

```bash
kubectl run network-debug \
  --rm -it \
  --image=nicolaka/netshoot \
  -- /bin/bash
```

Use approved debugging images in production environments.

---

# 62. Group BF — `kubectl exec` Fails

Command:

```bash
kubectl exec -it <pod> -n <namespace> -- sh
```

Possible errors:

```text
container not found
```

or:

```text
executable file not found
```

Reasons:

- wrong container
- image has no shell
- Pod is not running
- container restarted
- shell path differs

Specify container:

```bash
kubectl exec -it <pod> -n <namespace> -c <container> -- /bin/sh
```

For distroless images, use logs and ephemeral debugging rather than expecting a shell.

---

# 63. Group BG — Previous Container State

For crash diagnosis:

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState}'
```

Get exit code:

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'
```

Get reason:

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

Useful exit codes:

| Exit Code | Common Interpretation |
|---|---|
| 0 | Successful completion |
| 1 | Generic application error |
| 2 | Misuse/syntax or application-specific |
| 126 | Command found but cannot execute |
| 127 | Command not found |
| 128+N | Process terminated by signal N |
| 137 | SIGKILL; commonly OOMKilled |
| 143 | SIGTERM; often graceful termination |

Exit codes are clues, not proof of the root cause.

---

# 64. Group BH — Events Are Often the Fastest Clue

Use:

```bash
kubectl describe pod <pod> -n <namespace>
```

Pay special attention to:

```text
Events:
```

Common event reasons:

```text
FailedScheduling
Failed
FailedMount
FailedAttachVolume
FailedCreatePodSandBox
BackOff
Unhealthy
Killing
Evicted
Pulled
Pulling
Created
Started
```

Events usually tell you what Kubernetes observed, not necessarily why the application itself failed.

---

# 65. Group BI — Full Pod YAML Investigation

Use:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Inspect:

```text
metadata.labels
metadata.annotations
spec.containers
spec.initContainers
spec.volumes
spec.serviceAccountName
spec.nodeSelector
spec.affinity
spec.tolerations
spec.securityContext
spec.resources
spec.readinessProbe
spec.livenessProbe
spec.startupProbe
spec.imagePullSecrets
status.containerStatuses
status.conditions
```

---

# 66. Group BJ — Pod Conditions

Check:

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason}{"\n"}{end}'
```

Typical conditions:

```text
PodScheduled
Initialized
ContainersReady
Ready
```

Example:

```text
PodScheduled=True
Initialized=True
ContainersReady=False
Ready=False
```

This immediately narrows the problem to container readiness/runtime rather than scheduling.

---

# 67. Group BK — Service Account Token / Projected Volume Problems

Modern Kubernetes commonly uses projected ServiceAccount tokens.

Possible problems:

- ServiceAccount deleted
- projected volume failure
- API access denied
- token audience mismatch
- application expects legacy token behavior

Check:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Inspect ServiceAccount configuration and projected volumes.

---

# 68. Group BL — Admission / Mutation Changes

Admission controllers or webhooks can modify Pods.

A webhook may inject:

- sidecars
- environment variables
- volumes
- security settings
- labels
- annotations

If a Pod behaves differently from its submitted manifest, compare:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

with the original deployment manifest.

This is particularly important with service meshes and policy engines.

---

# 69. Group BM — Service Mesh Problems

Examples include:

- Istio
- Linkerd
- other sidecar-based service meshes

Symptoms:

- application works without sidecar
- traffic intercepted unexpectedly
- readiness fails
- TLS errors
- sidecar not ready
- outbound traffic blocked

Check all containers:

```bash
kubectl get pod <pod> -n <namespace>
```

Logs:

```bash
kubectl logs <pod> -n <namespace> -c <sidecar>
```

Separate application-container failures from proxy-container failures.

---

# 70. Group BN — Inconsistent Configuration Across Replicas

One Pod works while another fails.

Check:

```bash
kubectl get pods -n <namespace> -o wide
```

Compare:

```bash
kubectl get pod <pod1> -n <namespace> -o yaml
kubectl get pod <pod2> -n <namespace> -o yaml
```

Potential causes:

- different nodes
- different image versions
- stale configuration
- node-specific problems
- different mounted storage
- inconsistent admission mutations

---

# 71. Group BO — Pod Works on One Node but Not Another

This strongly suggests node-specific differences.

Compare:

```bash
kubectl get pod <pod> -o wide
```

Check node:

```bash
kubectl describe node <node>
```

Investigate:

- CNI
- kubelet
- container runtime
- disk
- memory
- kernel
- network
- cloud instance health
- node labels/taints

---

# 72. Group BP — DNS Works but Connection Fails

Example:

```text
nslookup database
```

works, but:

```bash
nc -vz database 5432
```

fails.

This indicates DNS is probably not the primary problem.

Investigate:

- Service targetPort
- endpoints
- NetworkPolicy
- application listener
- firewall
- database process

This layered approach prevents wasting time changing DNS configuration when DNS is already working.

---

# 73. Group BQ — Connection Refused vs Timeout

## Connection Refused

Usually means the destination is reachable but no process is accepting the connection on that port, or a firewall actively rejected it.

Investigate:

- wrong port
- application not listening
- Service targetPort
- application crashed

## Connection Timeout

Usually indicates packets are not reaching the destination or responses are blocked.

Investigate:

- NetworkPolicy
- firewall
- routing
- security groups
- CNI
- network path

These are not absolute rules, but they are useful first hypotheses.

---

# 74. Group BR — Pod Cannot Reach Kubernetes Service by Name

Test:

```bash
nslookup <service>
```

Use the fully qualified name:

```text
<service>.<namespace>.svc.cluster.local
```

Example:

```bash
nslookup backend.production.svc.cluster.local
```

If this works but short-name lookup does not, inspect the Pod's DNS configuration:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

---

# 75. Group BS — Incorrect Namespace in Service URL

Example:

Application in:

```text
namespace=frontend
```

tries:

```text
backend
```

Kubernetes resolves this relative to its namespace.

If backend is in `production`, use:

```text
backend.production.svc.cluster.local
```

or an appropriate namespace-qualified name.

---

# 76. Group BT — Secret/ConfigMap Updated but Application Still Uses Old Values

Mounted ConfigMaps/Secrets can be updated by Kubernetes, but applications may cache values or only read them during startup.

If configuration is injected as environment variables:

```text
env:
```

the running process does not automatically receive new environment variables.

Usually restart the workload after changing environment-based configuration:

```bash
kubectl rollout restart deployment/<deployment> -n <namespace>
```

Verify that restart behavior is compatible with your availability requirements.

---

# 77. Group BU — Image Updated but Pods Still Use Old Version

If the image tag is reused, nodes may have cached images.

Best practice:

Use immutable tags:

```text
myapp:1.4.7
```

or image digests:

```text
myapp@sha256:<digest>
```

Check:

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].imageID}'
```

This identifies the exact image used by the container runtime.

---

# 78. Group BV — `imagePullPolicy`

Typical behavior:

```yaml
imagePullPolicy: IfNotPresent
```

means an existing local image may be reused.

For `latest`, Kubernetes commonly defaults to:

```text
Always
```

when the field is omitted.

For production, immutable versioned tags or digests are generally easier to reason about.

---

# 79. Group BW — Pod Security and HostPath

HostPath example:

```yaml
volumes:
- name: host-data
  hostPath:
    path: /data
```

Potential problems:

- directory does not exist
- permissions incorrect
- security policy blocks hostPath
- Pod scheduled to a node where data does not exist

HostPath creates node-coupling and should be used deliberately.

---

# 80. Group BX — GPU Pod Problems

For GPU workloads:

Check node labels:

```bash
kubectl get nodes --show-labels
```

Check resources:

```bash
kubectl describe node <node>
```

Typical request:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

Common causes:

- GPU node unavailable
- NVIDIA device plugin unavailable
- incorrect node selector
- missing toleration
- incompatible driver
- CUDA/runtime mismatch

Check:

```bash
kubectl get pods -A | grep -i nvidia
```

Use the GPU vendor's supported diagnostics for the specific cluster.

---

# 81. Group BY — Windows vs Linux Node Problems

A workload built for Linux cannot normally run directly on Windows nodes, and vice versa.

Check:

```bash
kubectl get nodes -o wide
```

Use node selectors where required:

```yaml
nodeSelector:
  kubernetes.io/os: linux
```

Also verify:

```text
kubernetes.io/arch
kubernetes.io/os
```

---

# 82. Group BZ — Application Startup Dependency Problems

An application may start before:

- database
- cache
- message broker
- external service

is ready.

Avoid assuming Pod startup ordering provides application dependency readiness.

Use:

- retries
- exponential backoff
- readiness probes
- startup probes
- application-level dependency handling

A Pod being scheduled does not mean its dependencies are available.

---

# 83. Group CA — Probe Endpoint Authentication Problems

A health endpoint should normally be cheap and predictable.

If:

```text
/health
```

requires authentication but the kubelet probe does not supply it, the probe can fail.

Possible approaches:

- dedicated unauthenticated health endpoint
- appropriate probe headers
- TCP probe
- exec probe where justified

Avoid exposing sensitive application information through health endpoints.

---

# 84. Group CB — Probe Timeouts

Example:

```yaml
timeoutSeconds: 1
```

A slow application may fail even though it is healthy.

Measure actual response latency.

Possible adjustment:

```yaml
timeoutSeconds: 5
failureThreshold: 3
```

Do not compensate for an unhealthy application by making probes excessively permissive.

---

# 85. Group CC — HTTP 500 vs Probe Failure

A probe returning:

```text
HTTP 500
```

is a failed readiness/liveness check.

Check application logs at the exact time of the probe failure.

Useful correlation:

```text
Probe failure timestamp
        |
        v
Application log timestamp
        |
        v
Dependency / resource / application error
```

---

# 86. Group CD — Node Drain / Maintenance Issues

During:

```bash
kubectl drain <node>
```

Pods may be evicted and rescheduled.

Potential issues:

- PodDisruptionBudget
- local storage
- DaemonSets
- stateful storage
- insufficient capacity elsewhere

Check:

```bash
kubectl get pdb -A
```

A restrictive PDB can prevent voluntary disruptions from proceeding.

---

# 87. Group CE — PodDisruptionBudget

A PDB protects availability during voluntary disruptions.

Example:

```yaml
minAvailable: 2
```

If only two replicas exist, eviction of either may be blocked.

This is not normally the reason for a Pod being `Pending`, but it can explain why maintenance or draining cannot proceed.

---

# 88. Group CF — Pod Affinity Creates Hidden Capacity Constraints

A workload may require:

```text
same node
same zone
different node
same topology domain
```

Combined with:

- resource requests
- taints
- node selectors
- PDBs

these constraints can dramatically reduce available scheduling options.

When debugging scheduling, evaluate all constraints together rather than one field at a time.

---

# 89. Group CG — Admission Request Rejected Before Pod Exists

If:

```bash
kubectl apply -f deployment.yaml
```

returns an error, the Pod may never have been created.

Examples:

```text
Forbidden
Invalid
admission webhook denied
violates PodSecurity
```

Start with:

```bash
kubectl apply --dry-run=server -f deployment.yaml
```

This can reveal server-side validation/admission problems without creating the resource.

---

# 90. Group CH — YAML / Manifest Errors

Validate:

```bash
kubectl apply --dry-run=client -f deployment.yaml
```

Then:

```bash
kubectl apply --dry-run=server -f deployment.yaml
```

Common issues:

- wrong indentation
- incorrect field
- wrong API version
- wrong resource kind
- selector mismatch
- invalid probe
- invalid resource value

---

# 91. Group CI — Debugging with Temporary Pod

A generic debugging Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug
spec:
  containers:
  - name: debug
    image: busybox:1.36
    command:
    - sleep
    - "3600"
```

Apply:

```bash
kubectl apply -f debug.yaml
```

Enter:

```bash
kubectl exec -it debug -- sh
```

Test:

```bash
nslookup kubernetes.default
```

```bash
wget -O- http://my-service
```

Delete:

```bash
kubectl delete pod debug
```

Use an approved image appropriate for your environment.

---

# 92. Group CJ — A Systematic Network Debugging Sequence

When application A cannot reach application B:

```text
1. Is A Running?
2. Is A Ready?
3. Can A resolve B's DNS name?
4. Does B's Service exist?
5. Does B have Endpoints?
6. Is B listening on targetPort?
7. Is NetworkPolicy blocking traffic?
8. Is CNI healthy?
9. Is node/cloud networking healthy?
10. Is the application protocol/configuration correct?
```

Commands:

```bash
kubectl get pod -n <namespace>
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl get networkpolicy -n <namespace>
```

---

# 93. Group CK — A Systematic Storage Debugging Sequence

```text
Pod
 |
 +-- PVC exists?
 |
 +-- PVC Bound?
 |
 +-- PV available?
 |
 +-- StorageClass correct?
 |
 +-- CSI driver healthy?
 |
 +-- Volume attach successful?
 |
 +-- Volume mount successful?
 |
 +-- File permissions correct?
 |
 +-- Application path correct?
```

Commands:

```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc> -n <namespace>
kubectl get pv
kubectl get storageclass
kubectl get pods -n kube-system
```

---

# 94. Group CL — A Systematic Scheduling Debugging Sequence

```text
Pending
  |
  +-- FailedScheduling event?
       |
       +-- CPU?
       +-- Memory?
       +-- Node selector?
       +-- Affinity?
       +-- Taints?
       +-- Topology?
       +-- PVC?
       +-- Quota?
       +-- Node availability?
```

Commands:

```bash
kubectl describe pod <pod> -n <namespace>
kubectl get nodes
kubectl describe nodes
kubectl get pvc -n <namespace>
kubectl get resourcequota -n <namespace>
```

---

# 95. Group CM — A Systematic CrashLoopBackOff Sequence

```text
CrashLoopBackOff
      |
      +-- kubectl logs
      |
      +-- kubectl logs --previous
      |
      +-- describe pod
      |
      +-- exit code
      |
      +-- termination reason
      |
      +-- OOMKilled?
      |
      +-- Probe failure?
      |
      +-- Command/args?
      |
      +-- Config/Secret?
      |
      +-- Dependency?
      |
      +-- Application bug?
```

Commands:

```bash
kubectl logs <pod> -n <namespace> --previous
kubectl describe pod <pod> -n <namespace>
kubectl get pod <pod> -n <namespace> -o yaml
```

---

# 96. Group CN — A Systematic Readiness Failure Sequence

```text
Running but 0/1 Ready
        |
        +-- Readiness event?
        |
        +-- Correct port?
        |
        +-- Correct path?
        |
        +-- Application listening on 0.0.0.0?
        |
        +-- Dependency healthy?
        |
        +-- NetworkPolicy?
        |
        +-- Startup duration?
        |
        +-- Sidecar ready?
```

---

# 97. Group CO — A Systematic Image Pull Sequence

```text
ImagePullBackOff
       |
       +-- Image name correct?
       |
       +-- Tag exists?
       |
       +-- Registry reachable?
       |
       +-- Authentication correct?
       |
       +-- imagePullSecret exists?
       |
       +-- Node architecture compatible?
       |
       +-- Registry rate limit?
```

---

# 98. Group CP — Production Incident Workflow

When a production Pod fails:

## Step 1 — Establish Impact

Determine:

- How many Pods are affected?
- Is the Service still available?
- Is traffic failing?
- Is the problem isolated to one node?
- Did a deployment happen recently?

## Step 2 — Preserve Evidence

Collect:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

Avoid deleting the only failing Pod before collecting evidence unless immediate mitigation requires it.

## Step 3 — Identify the Layer

```text
Scheduling
   ↓
Image
   ↓
Container
   ↓
Application
   ↓
Readiness
   ↓
Service
   ↓
Network
   ↓
Storage
   ↓
Node
```

## Step 4 — Mitigate

Examples:

- rollback deployment
- scale replicas
- move workload
- correct configuration
- restore dependency
- add capacity

## Step 5 — Root Cause

Document:

```text
Trigger
→ Failure mechanism
→ User impact
→ Detection
→ Mitigation
→ Permanent fix
→ Prevention
```

---

# 99. Useful Command Cheat Sheet

## Pods

```bash
kubectl get pods -A
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl get pod <pod> -n <namespace> -o yaml
```

## Logs

```bash
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl logs <pod> -n <namespace> -c <container>
kubectl logs -f <pod> -n <namespace>
```

## Events

```bash
kubectl get events -A --sort-by='.lastTimestamp'
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

## Resources

```bash
kubectl top pods -A
kubectl top nodes
```

## Nodes

```bash
kubectl get nodes
kubectl describe node <node>
```

## Services

```bash
kubectl get svc -A
kubectl get endpoints -A
kubectl get endpointslices -A
```

## Storage

```bash
kubectl get pvc -A
kubectl get pv
kubectl get storageclass
```

## Configuration

```bash
kubectl get configmap -A
kubectl get secret -A
```

## Controllers

```bash
kubectl get deployment -A
kubectl get rs -A
kubectl get statefulset -A
kubectl get daemonset -A
kubectl get jobs -A
kubectl get cronjobs -A
```

## Security

```bash
kubectl get serviceaccount -A
kubectl get role -A
kubectl get rolebinding -A
kubectl get clusterrole
kubectl get clusterrolebinding
```

---

# 100. Useful JSONPath Commands

## Pod IP

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.podIP}'
```

## Node

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.spec.nodeName}'
```

## Container image

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.spec.containers[*].image}'
```

## Container state

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].state}'
```

## Restart count

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

---

# 101. Common Error → First Place to Look

| Error / Symptom | First Investigation |
|---|---|
| Pending | `describe pod` → FailedScheduling |
| ImagePullBackOff | image + registry + imagePullSecret |
| ErrImagePull | `describe pod` events |
| CrashLoopBackOff | `logs --previous` |
| OOMKilled | memory usage + limits |
| CreateContainerConfigError | Secret/ConfigMap references |
| CreateContainerError | container creation events |
| ContainerCreating | volumes/CNI/image |
| Init:CrashLoopBackOff | init container logs |
| 0/1 Ready | readiness probe |
| Liveness probe failed | probe configuration + app logs |
| FailedMount | PVC/CSI/permissions |
| FailedAttachVolume | storage attachment/CSI |
| FailedCreatePodSandBox | CNI/network |
| DNS failure | CoreDNS/CNI/NetworkPolicy |
| Service unreachable | Service selector/endpoints |
| Connection refused | target port/listener |
| Connection timeout | NetworkPolicy/firewall/routing |
| Forbidden | RBAC |
| Permission denied | UID/GID/SecurityContext |
| Evicted | node pressure |
| Pod stuck Terminating | finalizer/preStop/node |
| Exec failed | container/shell/image |
| Deployment unavailable | ReplicaSet/rollout |
| Job failed | Job Pod logs + exit code |

---

# 102. Troubleshooting Decision Tree

```text
                     Pod Problem
                          |
                          v
                  kubectl get pod
                          |
             +------------+-------------+
             |                          |
          Pending                    Not Pending
             |                          |
             v                          v
       describe pod               Is it Running?
             |                    /           \
             v                  No             Yes
       FailedScheduling            |             |
             |                     v             v
       +-----+------+         Image/Runtime    Ready?
       |     |      |              |           /   \
      CPU  Memory  Rules           |         No     Yes
       |     |      |              |         |       |
       +-----+------+              |         v       v
             |                     |      Probe     App/Network/
          Fix scheduling           |      issue     Storage issue
                                   |
                              ImagePull /
                              Create error
                                   |
                                   v
                              Check events
```

---

# 103. Golden Rules

## Rule 1

**Always check Events.**

```bash
kubectl describe pod <pod> -n <namespace>
```

## Rule 2

For crashes, always check:

```bash
kubectl logs <pod> -n <namespace> --previous
```

## Rule 3

Separate:

```text
Pod scheduling
container startup
application health
network connectivity
storage
node health
```

Do not treat all failures as application bugs.

## Rule 4

Do not immediately delete a failing Pod before collecting evidence.

## Rule 5

Do not blindly increase CPU/memory.

Find out whether the problem is:

```text
capacity
leak
traffic
configuration
application behavior
```

## Rule 6

Do not disable:

- NetworkPolicy
- Pod Security
- RBAC
- TLS
- admission controls

just to make the error disappear.

## Rule 7

Use immutable container image versions for production.

## Rule 8

Make health probes represent the correct health concept:

```text
Startup  = has initialization completed?
Readiness = can receive traffic?
Liveness = is the process still alive?
```

## Rule 9

When networking fails, debug in layers:

```text
DNS
→ Service
→ Endpoint
→ Port
→ NetworkPolicy
→ CNI
→ Node/cloud network
→ Application
```

## Rule 10

When storage fails, debug:

```text
PVC
→ PV
→ StorageClass
→ CSI
→ Attach
→ Mount
→ Permissions
→ Application
```

---

# 104. Recommended Troubleshooting Template

Use this template during incidents.

```text
Incident:
<short description>

Namespace:
<namespace>

Pod:
<pod>

Deployment/Controller:
<controller>

Node:
<node>

Pod Status:
<status>

Ready:
<yes/no>

Restart Count:
<number>

Container Exit Code:
<code>

Termination Reason:
<reason>

Recent Events:
<events>

Application Logs:
<important error>

Previous Logs:
<important error>

Image:
<image>

Image ID:
<image ID>

CPU Request/Limit:
<values>

Memory Request/Limit:
<values>

Readiness Probe:
<configuration>

Liveness Probe:
<configuration>

Startup Probe:
<configuration>

Service:
<service>

Endpoints:
<endpoints>

PVC:
<pvc>

NetworkPolicy:
<relevant policies>

ServiceAccount:
<service account>

Recent Deployment Change:
<yes/no>

Root Cause:
<root cause>

Immediate Mitigation:
<mitigation>

Permanent Fix:
<fix>

Prevention:
<monitoring/process/code/configuration improvement>
```

---

# 105. Example End-to-End Incident

## Problem

```text
frontend Pod is Running but users receive 503.
```

### Step 1

```bash
kubectl get pods -n production
```

Result:

```text
frontend-abc123   0/1   Running
```

### Step 2

```bash
kubectl describe pod frontend-abc123 -n production
```

Result:

```text
Readiness probe failed: HTTP probe failed with statuscode: 500
```

### Step 3

Check logs:

```bash
kubectl logs frontend-abc123 -n production
```

Result:

```text
Unable to connect to backend.production.svc.cluster.local:8080
```

### Step 4

Check DNS:

```bash
kubectl exec frontend-abc123 -n production -- \
  nslookup backend.production.svc.cluster.local
```

DNS works.

### Step 5

Check Service:

```bash
kubectl get svc backend -n production
```

### Step 6

Check endpoints:

```bash
kubectl get endpoints backend -n production
```

Result:

```text
<none>
```

### Step 7

Check backend Pods:

```bash
kubectl get pods -n production -l app=backend
```

Result:

```text
backend-xyz   0/1   Running
```

### Step 8

Inspect backend:

```bash
kubectl describe pod backend-xyz -n production
```

Result:

```text
Readiness probe failed
```

### Root Cause

The backend Pod was running but not Ready, so the Service had no usable endpoints. The frontend therefore could not successfully reach the backend.

### Lesson

A `Running` Pod does not necessarily mean it is serving traffic.

---

# 106. Final Troubleshooting Checklist

Before declaring a Pod incident resolved:

- [ ] Pod is scheduled
- [ ] Pod is Running
- [ ] All required containers are Running
- [ ] Pod is Ready
- [ ] Restart count is stable
- [ ] No recent warning events
- [ ] Application logs are clean
- [ ] Previous crash logs reviewed if applicable
- [ ] CPU usage is reasonable
- [ ] Memory usage is reasonable
- [ ] No OOMKilled events
- [ ] No CPU throttling concern
- [ ] Service exists
- [ ] Service selectors match Pod labels
- [ ] Endpoints exist
- [ ] DNS works
- [ ] Required NetworkPolicies allow traffic
- [ ] Required PVCs are Bound
- [ ] Volumes are mounted
- [ ] Permissions are correct
- [ ] ConfigMaps are correct
- [ ] Secrets are present
- [ ] ServiceAccount/RBAC is correct
- [ ] Node is healthy
- [ ] CNI is healthy
- [ ] Container runtime is healthy
- [ ] Recent deployment/configuration changes reviewed
- [ ] Monitoring confirms recovery
- [ ] Root cause documented
- [ ] Preventive action identified

---

# 107. One-Page Mental Model

When a Kubernetes Pod fails, think:

```text
                 POD FAILURE
                     |
     +---------------+---------------+
     |               |               |
 Scheduling       Startup          Runtime
     |               |               |
 Pending         Image pull       CrashLoop
 Resources       Config           OOM
 Affinity        Secret           Probe
 Taints          Volume           CPU
 Node            Command          App
     |               |               |
     +---------------+---------------+
                     |
                  NETWORK
                     |
          DNS → Service → Endpoint
                     |
              NetworkPolicy
                     |
                   CNI
                     |
               Node/Cloud
                     |
                  STORAGE
                     |
          PVC → PV → CSI → Mount
                     |
                 SECURITY
                     |
          RBAC → SecurityContext
                     |
             Pod Security/Policy
```

The most effective troubleshooting approach is to identify the **first failing layer** and avoid changing unrelated layers.

---

# 108. Quick Command Sequence for Any Pod

When you receive an alert for a broken Pod, start here:

```bash
# 1. Status
kubectl get pod <pod> -n <namespace> -o wide

# 2. Events and configuration
kubectl describe pod <pod> -n <namespace>

# 3. Current logs
kubectl logs <pod> -n <namespace>

# 4. Previous crash logs
kubectl logs <pod> -n <namespace> --previous

# 5. Full state
kubectl get pod <pod> -n <namespace> -o yaml

# 6. Node
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.spec.nodeName}{"\n"}'

# 7. Resources
kubectl top pod <pod> -n <namespace>

# 8. Recent events
kubectl get events -n <namespace> \
  --sort-by='.lastTimestamp'

# 9. Service/endpoints if networking is involved
kubectl get svc,endpoints,endpointslices -n <namespace>

# 10. Storage if volumes are involved
kubectl get pvc,pv -n <namespace>
```

This sequence resolves or narrows a large percentage of Pod incidents without making speculative changes.
