## Scenario 2

### Diagnosis

**Setup:** applied `s2_configmap.yml` (key `APP_MODE`), then `s2_pod.yml`.

**The trap: nothing looks wrong at a glance**
```bash
kubectl get pods -o wide
```
```
config-demo-2   1/1   Running   0   17s
```
Healthy by every normal signal — `1/1`, `Running`, zero restarts, no error events.

**Logs reveal the real problem:**
```bash
kubectl logs config-demo-2
```
```
Starting in mode:
```
`$APP_MODE` is empty. No crash, no event fired — the value just silently never loaded.

**Comparing the live ConfigMap against what the Pod referenced:**
```bash
kubectl describe configmap app-settings
```
```
Data
====
APP_MODE:
----
production
```
The ConfigMap's real key is `APP_MODE`. The Pod's manifest referenced `key: APP_MDOE` — a typo, two letters swapped.

### Root cause

A typo'd `configMapKeyRef.key`, combined with `optional: true` on the same field. Without `optional: true`, a nonexistent key would have produced `CreateContainerConfigError` and blocked the Pod from starting — the loud, familiar failure from earlier ConfigMap bugs. With it, the kubelet treats a missing key as "fine, just skip it," so the container starts normally with `$APP_MODE` resolving to an empty string.

### Fix

```yaml
env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-settings
        key: APP_MODE
        optional: true
```

Re-applying hit a Pod-immutability wall:
```bash
kubectl apply -f s2_pod_fixed.yml
```
```
The Pod "config-demo-2" is invalid: spec: Forbidden: pod updates may not change
fields other than `spec.containers[*].image`, ...
```
Most fields on a running Pod are immutable (`env` isn't on the small list of fields that can be changed in place) — the same category of restriction as the Deployment selector lock, just stricter since a bare Pod has far fewer mutable fields than a Deployment. Worked around it with delete + recreate:
```bash
kubectl delete -f s2_pod_fixed.yml
kubectl apply -f s2_pod_fixed.yml
```

### Verification

```bash
kubectl logs config-demo-2
```
```
Starting in mode: production
```

### Why this one failed silently (step 5)

`optional: true` on a `configMapKeyRef` tells the kubelet "it's fine if this key doesn't exist — just skip it, don't error." Normally a missing key blocks the Pod with `CreateContainerConfigError`, a fast and visible failure. With `optional: true`, that safety net is gone: the typo resolves to nothing, the container starts anyway, and the app just receives an empty string with zero indication anything went wrong.

### When `optional: true` is the right call vs. dangerous (step 6)

- **Right call:** config that's genuinely allowed to be absent, and the app has a real fallback for it (e.g., code that defaults `APP_MODE` to `"production"` if the env var is unset).
- **Dangerous:** config the app assumes exists and never validates — exactly this scenario, where the empty string gets interpolated straight into a log line with no check at all. `optional: true` doesn't make the config optional for the *app's logic*, only for the *kubelet's startup check*. If the app still needs the value to function correctly, `optional: true` just converts a fast, loud failure into a slow, silent one.

### A normal (not a bug) terminal-state note

Deleting and quickly re-applying a Pod can briefly show `STATUS: Error` with a `Killing: Stopping container` event while the old Pod finishes terminating. That's expected shutdown behavior, not a crash — `kubectl delete -f` blocks until the Pod is actually gone, and interrupting that wait (Ctrl+C) only stops the client from waiting, not the deletion itself.

### Key takeaways

- `1/1 Running`, zero restarts, and no error events together still don't prove an app is actually working correctly — **always check logs for the actual output**, not just the Pod's status.
- `optional: true` on a ConfigMap/Secret reference trades a loud, immediate failure (`CreateContainerConfigError`) for a silent one (empty value, no error). Use it only when the app genuinely handles the missing value.
- Don't eyeball a typo — pull the live object's real keys (`describe configmap`/`get configmap -o yaml`) and compare directly against the manifest.
- Most fields on an already-running Pod are immutable. If `apply` is rejected for a field not in the small allowed list (container image, a few others), delete and recreate rather than trying to force the update through.