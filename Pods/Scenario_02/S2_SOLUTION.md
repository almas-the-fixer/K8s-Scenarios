## Pod — Scenario 2 Solution

### Diagnosis

```bash
kubectl describe pod config-reader
```
Showed `Status: Running` at the top but the container itself was `Terminated / Reason: Error / Exit Code: 1`, with `Restart Count: 5` and a `BackOff` event — classic CrashLoopBackOff. `describe` only confirms *that* it crashed, not *why*.

```bash
kubectl logs config-reader --previous
```
This is the key command — since the container had already restarted, current logs were empty/unhelpful, and `--previous` pulled the last crashed instance's stderr, which pointed to the actual `cat` failure.

### Root cause (2 bugs)

1. **Missing ConfigMap** — `s2_pod.yml` referenced a ConfigMap named `app-settings` that didn't exist.
2. **Wrong `mountPath`** — mounting at `/etc/config/app.conf` (the file path) instead of `/etc/config` (the parent directory) causes Kubernetes to create `app.conf` as a *directory*, with the actual file nested one level deeper at `/etc/config/app.conf/app.conf`. The Pod's command (`cat /etc/config/app.conf`) tried to `cat` a directory → exit code 1.

### Fix

Created the missing ConfigMap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  app.conf: "Hello"
```

Corrected the volume mount to point at the directory, not the file:
```yaml
volumeMounts:
  - name: config-vol
    mountPath: /etc/config
```

### Verification

```bash
kubectl logs config-reader
```
Output showed `Hello` (the ConfigMap's `app.conf` value), confirming the file was read successfully, and `kubectl get pods` showed `Running` with `Restart Count` no longer climbing.

### Key takeaway

A ConfigMap volume's key name becomes the filename on disk. Mounting it, `mountPath` must be a **directory**, not the file's own path — pointing `mountPath` at the file path itself turns that path into a directory containing the real file one level deeper.