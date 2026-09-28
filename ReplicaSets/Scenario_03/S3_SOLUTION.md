## Scenario 3

### Diagnosis

**Setup (order matters):**
```bash
kubectl apply -f s3_configmap.yml
kubectl apply -f s3_replicaset.yml
kubectl get pods
```

**Failure 1: `CreateContainerConfigError`**
```bash
kubectl describe pod <shop-pod>
```
The container couldn't start because the env var referenced key `app_mode` in ConfigMap `shop-config`, and the ConfigMap's key is `APP_MODE`. ConfigMap keys are case-sensitive.

**Failure 2: `Running` but `READY 0/1` (after Failure 1 was fixed)**
```bash
kubectl describe pod <shop-pod>
```
Events showed:
```
Warning  Unhealthy  pod/shop-rs-bwn2p  Readiness probe failed: HTTP probe failed with statuscode: 404
```
A 404 means nginx answered, so the app was up and the path didn't exist. A stock nginx image only serves `/`.

### Root cause (2 bugs)

1. **Wrong ConfigMap key**: `key: app_mode` should have been `key: APP_MODE`.
2. **Readiness probe on a nonexistent path**: `/healthz` returns 404 on stock nginx. `/health` would have failed the same way. Only `/` exists.

### Fix

Copied the buggy file to `s3_replicaset_fixed.yml` and changed:
```yaml
key: APP_MODE
```
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```
```bash
kubectl apply -f s3_replicaset_fixed.yml
```
A ReplicaSet only reads its template when it creates a Pod, so after each apply I replaced every Pod with one command:
```bash
kubectl delete pods -l app=shop
```

### Verification

```bash
kubectl get pods -o wide -w
```
```
shop-rs-789vw   1/1   Running   0   4m17s   node01
shop-rs-7vclh   1/1   Running   0   4m17s   node01
shop-rs-c8qx8   1/1   Running   0   4m17s   controlplane
shop-rs-lvnql   1/1   Running   0   4m17s   controlplane
```
4/4 `Ready`.

### Quarantine (removing a Pod from the ReplicaSet's control)

Changed one healthy Pod's label from `app: shop` to `app: shoppy`:
```bash
kubectl edit pod shop-rs-c8qx8
kubectl get pods -o wide -w
```
```
shop-rs-5j7bs   0/1   Running   0   1s     controlplane
shop-rs-789vw   1/1   Running   0   63s    node01
shop-rs-7vclh   1/1   Running   0   63s    node01
shop-rs-c8qx8   1/1   Running   0   63s    controlplane
shop-rs-lvnql   1/1   Running   0   63s    controlplane
```
The ReplicaSet saw only 3 Pods matching `app=shop` and created `shop-rs-5j7bs`, giving 5 Pods in total.

```bash
kubectl get pod shop-rs-c8qx8 -o yaml
```
```yaml
labels:
  app: shoppy
name: shop-rs-c8qx8
```
No `ownerReferences` block anymore, so the Pod is released. It's still `Running`.

```bash
kubectl get rs shop-rs
```
```
NAME      DESIRED   CURRENT   READY   AGE
shop-rs   4         4         4       24m
```

### Prediction and test

**Prediction:** putting the label back means 5 Pods match against 4 desired, so the ReplicaSet deletes one. I wasn't sure which.
```bash
kubectl edit pod shop-rs-c8qx8    # label back to app: shop
kubectl get pods -o wide -w
```
Result: 4 Pods left, and the newest one (`shop-rs-5j7bs`) was the one deleted.

### Key takeaways

- ConfigMap keys are case-sensitive, and a bad `configMapKeyRef` shows up as `CreateContainerConfigError`.
- A readiness probe returning a **404** means the app is up and the path is wrong, so don't guess new paths. Read the probe failure line.
- A ReplicaSet doesn't update existing Pods when its template changes. Replace them with `kubectl delete pods -l <label>`.
- A ReplicaSet owns Pods by **label match**. Changing a Pod's label releases it, and the ReplicaSet immediately creates a replacement. This is a real technique for keeping a misbehaving Pod alive for inspection.
- Restoring the label leaves one Pod too many, and the ReplicaSet tends to remove the newest. `controller.kubernetes.io/pod-deletion-cost` is the lever if you need to control which one goes.
- `kubectl label pod <name> app=<value> --overwrite` does the same as `kubectl edit pod` in one line.