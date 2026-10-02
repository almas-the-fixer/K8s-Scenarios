## Scenario 3

### Diagnosis

**Setup:** applied `s3_configmap_primary.yml`, `s3_configmap_override.yml`, then `s3_pod_bad.yml`.

**Pod failed to mount:**
```bash
kubectl get pods
```
Events showed:
```
Warning  FailedMount  MountVolume.SetUp failed for volume "nginx-conf-vol" :
configmap references non-existent config key: nginx.cfg
```
The volume's `items.key` referenced `nginx.cfg`, but the ConfigMap's real key is `nginx.conf`. Unlike an env-var ConfigMap bug (`CreateContainerConfigError`), a bad key in a **volume mount** fails as `FailedMount` — a mount-time error, not a container-config error.

### Root cause

A typo'd `items.key` (`nginx.cfg` instead of `nginx.conf`) in the volume definition.

### Fix

```yaml
volumes:
  - name: nginx-conf-vol
    configMap:
      name: db-primary
      items:
        - key: nginx.conf
          path: nginx.conf
```

### `envFrom` collision — which ConfigMap wins

```bash
kubectl logs config-demo-3
```
```
DB_HOST=primary-db.local
DB_PORT=5432
server_name example.com;
listen 80;
```
Both `db-override` and `db-primary` define `DB_HOST`. `envFrom` was listed as `[db-override, db-primary]`. **The last source in the list wins** — it overwrites any earlier value for the same key. `db-override` set `DB_HOST=override-db.local` first, then `db-primary` immediately overwrote it with `primary-db.local`, which is the value that actually survived.

### Live-update test: `subPath` vs. a plain ConfigMap volume mount

```bash
kubectl edit configmap db-primary   # changed nginx.conf's content
kubectl exec config-demo-3 -- cat /etc/nginx/nginx.conf
```
Content never changed, even after waiting. This is **not a longer delay than S1** — it's a permanent limitation. A plain ConfigMap volume mount (S1) works via a symlink: Kubernetes writes fresh data into a new hidden folder on each sync, then atomically swaps a symlink to point at it, so the file you read picks up the change automatically. `subPath`, used here to mount a single file into an existing directory without wiping out everything else in that directory, bypasses this entirely — it's a direct bind-mount of just that one file, fixed at container start, with no symlink to ever swap. **A `subPath`-mounted file will never live-update, under any circumstances, without a Pod restart.**

A separate mistake happened while testing this: editing the ConfigMap via `kubectl edit` accidentally renamed the key itself (`nginx.conf` → `nginx.config`), which triggered a new `FailedMount` warning pointing the other direction. This was unrelated noise from a typo during live-editing — it didn't change the underlying finding, since `subPath` wasn't going to pick up content changes either way.

### Verification

```bash
kubectl exec -it config-demo-3 -- sh -c "cat /etc/nginx/nginx.conf && printenv | grep DB_HOST"
```
```
server_name example.com;
listen 80;
DB_HOST=primary-db.local
```
Both the file and the env var confirmed stable/frozen as expected.

### Key takeaways

- A bad key in a ConfigMap **volume** mount fails as `FailedMount` (mount-time), distinct from `CreateContainerConfigError` for a bad key in `env`/`envFrom` (container-config-time). Same root problem (wrong key), different failure signature depending on *how* the ConfigMap is consumed.
- When `envFrom` pulls from multiple ConfigMaps that define the same key, **the last one in the list wins**, silently overwriting earlier values. Order matters, and there's no warning when this happens.
- `subPath` mounts a single file directly, bypassing the symlink-swap mechanism that makes whole-ConfigMap volume mounts live-update. **A file mounted via `subPath` never picks up ConfigMap changes, ever, without a Pod restart** — this is permanent, not a longer sync delay, and it's a well-known, deliberately confusing trap because it contradicts the behavior from a plain volume mount.
- `kubectl exec <pod> -- cmd1 && cmd2` runs `cmd1` inside the Pod but `cmd2` on your **local shell** — the `&&` is outside what gets sent to `exec`. To run multiple commands inside the container, wrap them: `kubectl exec <pod> -- sh -c "cmd1 && cmd2"`.
- `kubectl edit` opens an object live in your editor for free-form changes — same end effect as `kubectl patch`, just interactive instead of scripted, and therefore easier to accidentally change something you didn't mean to.