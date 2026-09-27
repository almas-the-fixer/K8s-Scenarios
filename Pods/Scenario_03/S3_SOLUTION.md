## Pod — Scenario 3 Solution

### Diagnosis

**Failure 1 — Pending**
```bash
kubectl get pods
kubectl describe pod multi-check
```
`describe` showed a `FailedScheduling` event: no node matched `nodeSelector: disktype=ssd`. Confirmed with:
```bash
kubectl get nodes --show-labels
```
Neither node had `disktype=ssd`.

**Failure 2 — CrashLoopBackOff (after Bug 1 fixed)**
```bash
kubectl logs multi-check -c app-main --previous
```
Showed the `cat` failing — file not found. Root cause: `init-setup` wrote to volume `temp-data` while `app-main` mounted a completely different volume, `shared-data`. Two separate `emptyDir`s, so nothing was actually shared between the containers.

**Failure 3 — Running but restarting later (after Bug 2 fixed)**
Pod looked healthy at first (`cat` succeeded, container stayed up briefly), but restarted again after ~10-15s. The `livenessProbe` was checking for `/shared/health.txt`, a file that was never created — only `status.txt` exists.

### Root cause (3 bugs)

1. **Scheduling** — `nodeSelector: disktype=ssd` didn't match any node's labels.
2. **Volume mismatch** — init container and main container mounted two different `emptyDir` volumes (`temp-data` vs `shared-data`) instead of sharing one, so the file the init container wrote was invisible to the main container.
3. **Liveness probe** — checked for a file (`health.txt`) that nothing ever creates; only `status.txt` exists.

### Fix

Labeled the node so the nodeSelector could match:
```bash
kubectl label node node01 disktype=ssd
```

Pointed both containers at the same volume:
```yaml
initContainers:
  - name: init-setup
    volumeMounts:
      - name: shared-data
        mountPath: /shared
containers:
  - name: app-main
    volumeMounts:
      - name: shared-data
        mountPath: /shared
```

Corrected the liveness probe to check the file that actually exists:
```yaml
livenessProbe:
  exec:
    command: ["sh", "-c", "test -f /shared/status.txt"]
  initialDelaySeconds: 10
  periodSeconds: 5
```

### Verification

```bash
kubectl get nodes --show-labels
```
```
node01   Ready   <none>   7d13h   v1.36.1   ...,disktype=ssd,...
```

```bash
kubectl get pods
```