## Scenario 3

### Diagnosis

**Setup:** applied `s3_secret_primary.yml` (immutable, `API_KEY=primary-key-123`), `s3_secret_override.yml` (`API_KEY=override-key-456`), then `s3_pod_bad.yml`. No bug was actually planted in this file — selector and template labels matched, so the Deployment came up cleanly on first apply.

### `envFrom` collision — same rule as ConfigMap S3

```bash
kubectl logs <pod>
```
```
API_KEY=primary-key-123
```
`envFrom` was `[db-override, db-primary]` — last source wins, so `db-primary` (listed second) overwrote `db-override`'s value. Same rule as ConfigMap S3, confirmed holding for Secrets too.

### File permissions on a mounted Secret

```bash
kubectl exec <pod> -- ls -l /etc/secret/
```
```
lrwxrwxrwx    1 root     root    API_KEY -> ..data/API_KEY
```
This is misleading at first glance — `lrwxrwxrwx` is the permission listing for the **symlink itself**, not the real file. Unix always shows symlinks as fully open regardless of the target's actual permissions. Following it to the real file:
```bash
kubectl exec <pod> -- ls -la /etc/secret/..data/
```
```
-rw-r--r--    1 root     root    15 ... API_KEY
```
`0644` — the **same default mode as a ConfigMap-mounted file.** Kubernetes does not automatically lock Secret files down tighter than ConfigMap files; `0644` is the shared default for both. Restricting it further (e.g. `defaultMode: 0400`) is a deliberate opt-in on the volume spec, not a built-in behavior. The real security boundary for Secrets isn't filesystem permissions inside a Pod that already has the data mounted — by the time a container can read the file at all, it already has full access to the value. The actual control is **RBAC**: who is allowed to `get`/`describe` the Secret object through the API in the first place, before it's ever mounted anywhere.

### Rotating an immutable Secret, with a real-world gotcha along the way

`db-primary` is `immutable: true`, so rotation requires the same delete + recreate procedure from S2:
```yaml
stringData:
  API_KEY: ridiculous-key-crash-out
```
```bash
kubectl delete secret db-primary
kubectl apply -f s3_secret_primary.yml
```
Confirmed directly against the live object, not inferred from a Pod:
```bash
kubectl get secret db-primary -o yaml
echo '<base64>' | base64 -d
```
```
ridiculous-key-crash-out
```

**A real gotcha hit along the way:** `kubectl rollout restart deployment` did not reliably produce a Pod reflecting the newly-rotated value in this session — multiple rotation attempts, each individually correct (new Secret value confirmed live via direct decode), still resulted in a new Pod reporting a **stale** `API_KEY` value from an earlier rotation. This persisted even after confirming via `describe pod` that the container's actual start time was genuinely after the new Secret's `creationTimestamp` — ruling out "it's just an old leftover Pod." The issue only resolved after a **full `kubectl delete deployment`** (removing every ReplicaSet and Pod entirely) followed by a fresh `kubectl apply` — at which point the new Pod correctly showed `ridiculous-key-crash-out` immediately.

This points to some kind of caching or sync lag between the kubelet and the API server on this specific cluster/session, rather than anything wrong with the rotation procedure itself — the procedure (edit file → delete Secret → apply → recreate the Pod/Deployment) is correct and is exactly what real clusters require. `rollout restart` is the normal, correct tool for forcing new Pods in production; it simply didn't reliably pick up the change here, and a full Deployment recreate was what actually resolved it in this environment.

### Verification

```bash
kubectl exec <new-pod> -- printenv
```
```
API_KEY=ridiculous-key-crash-out
```
Confirmed correct after the full Deployment recreate.

### Key takeaways

- `envFrom`'s "last source wins" rule holds for Secrets exactly as it does for ConfigMaps.
- Always check `ls -la` on the symlink's **target** (`..data/`), never the symlink entry itself — the symlink's own listed permissions are meaningless.
- Secret and ConfigMap volume mounts share the same default file mode (`0644`). Kubernetes does not harden Secret files automatically; the real protection is RBAC on the Secret object itself, not filesystem permissions on an already-mounted file.
- Rotating an immutable Secret always means: edit → delete the object → recreate it with the same name → recreate whatever Pod/Deployment consumes it. There is no in-place path, by design.
- **If a `rollout restart` doesn't appear to pick up a genuinely-confirmed-live config change, don't assume your procedure is wrong before checking the live object directly** (`kubectl get secret <name> -o yaml` + manual decode) — trust that over what a Pod's env var reports. If the object is confirmed correct but a restarted Pod still shows stale data, a full `delete`+`apply` of the owning Deployment is a reliable way to force a truly clean state when `rollout restart` alone doesn't resolve it.