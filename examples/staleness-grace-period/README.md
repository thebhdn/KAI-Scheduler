# Staleness Grace Period Example

This example demonstrates how the staleness grace period works.

## Scenario

A 2-pod gang is created. When both 2 pods are running, the group is gang-satisfied.
If one pod fails, the remaining 1 pod is still running but the gang condition is no longer met
(minMember: 2, but only 1 active). The scheduler starts a 30 second grace period before evicting
the stale pod.

## Steps

### 1. Create the namespace

```bash
kubectl apply -f 00-podgroup.yaml
```

### 2. Create 2 pods (gang-satisfied)

```bash
kubectl apply -f 01-pods.yaml
```

This creates the PodGroup (`minMember: 2`) and both pods in a single manifest file.

Verify all pods are running:

```bash
kubectl get pods -n stale-grace-demo
```

Expected: both pods in `Running` status, podgroup in `Running` status.

### 3. Simulate a pod failure (triggers staleness)

Delete one pod to simulate a failure:

```bash
kubectl delete pod stale-gang-pod-1 -n stale-grace-demo
```

Expected: the remaining pod stays running (staleness grace period is 30 seconds).
The podgroup becomes **stale** - it has a running pod but the gang condition is no longer met.
Verify:

```bash
kubectl get podgroup -n stale-grace-demo -o yaml | grep -A5 "staleness"
```

### 4. Wait for the grace period to expire

After 30 seconds, the stalegangeviction action evicts the remaining running pod:

```bash
kubectl get pods -n stale-grace-demo
```

Expected: all pods in `Terminating` or `Terminated` status.

## Key points

- **Grace period prevents immediate eviction**: Without the grace period, stale pods would be
  evicted on the next scheduler cycle. The grace period gives time for transient failures to
  resolve (e.g., node recovery, image pull retries).

- **Staleness requires active pods**: A pod in `Pending` status does not count toward staleness.
  The gang must have some Running/Allocated pods for staleness to trigger.

- **Negative values disable eviction**: Setting the grace period to a negative value (e.g., `-1s`)
  disables stale eviction entirely for the pod group.
