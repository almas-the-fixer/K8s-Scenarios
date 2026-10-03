## Scenario 2

### Diagnosis

**Setup:** applied `s2_secret.yml` (`immutable: true`, `DB_PASS=initial-pass`), then `s2_pod.yml`. Confirmed healthy.

**Attempted "routine" password rotation:**
```bash
kubectl apply -f s2_secret_update.yml
```
```
The Secret "db-creds-v1" is invalid: data: Forbidden: field is immutable when `immutable` is set
```

### Answer — why `immutable: true` exists at all (step 1)

Two real reasons, not just an arbitrary restriction:

1. **Performance.** A mutable ConfigMap/Secret keeps the kubelet watching it for changes on every Pod that uses it, forever — even a Pod that only ever reads it once as an env var. At scale (many Secrets, many Pods), that's ongoing API server load for watches that, in an env-var-only case, never even get used. `immutable: true` tells Kubernetes "this will never change, stop watching," which is a genuine win in large clusters.
2. **Safety.** It prevents someone from editing a shared Secret in place and silently breaking every Pod that uses it, with no new object ever created to signal that a change happened.

### Answer — the correct way to rotate it (step 2)

There's no patch-around for an immutable field. The only correct approach: **delete the Secret object and create a new one with the same name.** This is how immutable Secret rotation actually works in real clusters, not a workaround specific to this exercise.

```bash
kubectl delete -f s2_secret.yml
kubectl apply -f s2_secret_update.yml
```

### Answer — how the Pod picks up the new value, and the mistake that delayed it (step 3)

Env vars from a Secret follow the **exact same rule proven in ConfigMap S1**: they're a one-time snapshot taken at container start, with no live connection afterward. Waiting longer changes nothing — this isn't a sync-delay situation like a ConfigMap volume mount.

The first attempt recreated the Pod **before** the rotated Secret existed yet — so the "new" Pod still baked in `initial-pass`, because that was genuinely the only value available in the cluster at that moment. `sleep 60` didn't help, because there was nothing to catch up on; the Pod had simply been created too early.

Correct order — Secret first, then Pod:
```bash
kubectl delete pod secret-demo-2
kubectl apply -f s2_pod.yml
kubectl exec secret-demo-2 -- sh -c 'printenv | grep DB_PASS'
```
```
DB_PASS=rotated-pass
```
(A `container not found` error can appear if `exec` runs in the brief moment before the new container has actually started — not a bug, just a timing race. A retry a second later succeeds.)

### Answer — proving the old Secret is genuinely gone, not superseded (step 4)

```bash
kubectl get secret db-creds-v1 -o yaml
```
The object's `uid` and `creationTimestamp` are different from the original — proof this is a brand-new object with the same name, not the old one edited in place (which `immutable: true` would never have allowed anyway).

### Key takeaways

- `immutable: true` exists for real production reasons: reduced API server watch load at scale, and protection against silent in-place edits to shared config.
- The only way to "update" an immutable Secret or ConfigMap is delete + recreate with the same name — never a patch or edit.
- Env vars sourced from a Secret or ConfigMap are frozen at **container creation time**, not at Pod YAML apply time. If the underlying object changes *after* the Pod already exists, the Pod must also be recreated to pick up the new value — order matters: update the source object first, then recreate the Pod, not the other way around.
- Waiting longer never fixes a frozen env var. That only applies to a ConfigMap-volume sync delay (a different mechanism entirely) — for env vars, there's no live connection to wait for at all.