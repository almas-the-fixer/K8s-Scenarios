## Scenario 2

### Diagnosis

**Setup (order matters):**
```bash
kubectl apply -f s2_stray_pod.yml
kubectl apply -f s2_rs_buggy_file.yml
kubectl get pods
```
```
api-rs-lvpfm   0/1   ContainerCreating   0   4s
api-rs-r2klm   0/1   ContainerCreating   0   4s
stray-api      1/1   Running             0   11s
```
Only 2 Pods were created by the ReplicaSet, even though `replicas: 3`.

**Proof of ownership:**
```bash
kubectl get pod stray-api -o yaml
```
```yaml
ownerReferences:
- apiVersion: apps/v1
  blockOwnerDeletion: true
  controller: true
  kind: ReplicaSet
  name: api-rs
```
The ReplicaSet adopted `stray-api` because its labels (`app: api`) matched the selector. It counted as replica #1, so only 2 new Pods were needed.

**Why the other Pods were unhealthy:**
```bash
kubectl get pods -o wide
kubectl get events
```
```
api-rs-lvpfm   0/1   ErrImagePull       0   19s
api-rs-r2klm   0/1   ImagePullBackOff   0   19s
```
```
Warning  Failed  pod/api-rs-lvpfm  Failed to pull image "nginx:1.255": ... not found
Warning  Failed  pod/api-rs-lvpfm  Error: ErrImagePull
Warning  Failed  pod/api-rs-lvpfm  Error: ImagePullBackOff
```

### Root cause (2 things)

1. **Bad image tag**: `nginx:1.255` doesn't exist. Typo for `nginx:1.25`.
2. **Fixing the template doesn't fix existing Pods**: a ReplicaSet only reads its template when it *creates* a Pod. The two already-broken Pods kept the old image.

### Fix

Copied the buggy file to `s2_replicaset_fixed.yml` and changed the image:
```yaml
image: nginx:1.25
```
```bash
kubectl apply -f s2_replicaset_fixed.yml
kubectl get rs api-rs -o wide
```
```
NAME     DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES
api-rs   3         3         1       4m49s   api-rs       nginx:1.25
```
The template was fixed, but `READY` was still 1, so the two old Pods were still broken. Deleted only those two, without deleting the ReplicaSet or the stray Pod:
```bash
kubectl delete pod api-rs-7crgj
kubectl delete pod api-rs-mbps7
```

### Verification

```bash
kubectl get pods -o wide -w
```
```
NAME           READY   STATUS    RESTARTS   AGE     NODE
api-rs-9qt6t   1/1     Running   0          17s     node01
api-rs-tr6ld   1/1     Running   0          7s      controlplane
stray-api      1/1     Running   0          6m30s   node01
```
3 Pods `Ready`. `stray-api` survived with its original age, so it's the same Pod and still owned by `api-rs`. The two new Pods were built from the fixed template.

### Key takeaways

- A ReplicaSet manages every Pod whose labels match its selector, including ones it didn't create.
- Changing a ReplicaSet's template does **not** update existing Pods. Delete them so the ReplicaSet recreates them from the new template. A Deployment automates exactly this with rolling updates.
- `apply` printing `configured` means the cluster accepted a change, and `unchanged` means there was nothing new to send.
- Deleting a ReplicaSet also deletes the Pods it owns, including adopted ones.