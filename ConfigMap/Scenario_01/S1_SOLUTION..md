# ConfigMap — Solutions

## Scenario 1

### Setup

```bash
kubectl apply -f s1_configmap.yml
kubectl apply -f s1_pod.yml
kubectl get pods -o wide
```

### Verifying all three injection methods

**Method 1 — single key as a named env var.** Initially written with the env var name matching the ConfigMap key (`LOG_LEVEL`), which made it impossible to tell apart from Method 2's output — `envFrom` was also producing a var called `LOG_LEVEL`, and when two sources define the same name, `kubectl` doesn't make it obvious which one you're actually seeing. Fixed by giving it a different name (`APP_LOG_LEVEL`) so it could be checked on its own:
```bash
kubectl exec config-demo -- printenv APP_LOG_LEVEL
```
```
debug
```

**Method 2 — entire ConfigMap injected as env vars (`envFrom`):**
```bash
kubectl exec config-demo -- printenv MAX_CONNECTIONS
```
```
100
```
This one is unambiguous proof on its own, since `MAX_CONNECTIONS` was never referenced individually anywhere — the only way it could appear is through the bulk import.

**Method 3 — ConfigMap mounted as a volume:**
```bash
kubectl exec config-demo -- cat /etc/config/app.properties
```
```
timeout=30
retries=3
```
Matches the ConfigMap's content exactly.

### A gotcha found along the way: `envFrom` doesn't filter by "shape"

```bash
kubectl exec config-demo -- printenv | grep app.properties
```
```
app.properties=timeout=30
```
`envFrom` dumps **every** top-level key in the ConfigMap as an env var — including `app.properties`, which was only meant to be a mounted file. The multi-line value gets squashed into a single env var with an embedded newline, which is a bad shape for most tools expecting a clean single-line value. `envFrom` has no concept of "this key is file-shaped, skip it" — it just exports everything.

### Live-update behavior: env var vs. mounted file

```bash
kubectl patch configmap app-config --patch '{"data":{"LOG_LEVEL":"production"}}'
```
```bash
kubectl exec config-demo -- printenv APP_LOG_LEVEL
```
```
debug          # unchanged
```
```bash
kubectl exec config-demo -- cat /etc/config/LOG_LEVEL
```
```
production     # updated, after a short sync delay
```

**Why they behave differently:** an env var is resolved **once**, at container start, and baked directly into the process's environment — there's no live link back to the ConfigMap afterward, and a running process's environment can't be changed from outside. The only way to pick up a change is to restart the container. A mounted ConfigMap volume works completely differently: the kubelet actively watches the ConfigMap and rewrites the mounted files on a periodic sync (roughly every 60 seconds), so any process reading that file fresh just sees whatever's currently there. Env vars = a one-time snapshot taken at birth. Mounted files = continuously kept in sync for the Pod's whole lifetime.

### Key takeaways

- Giving an env var a different name than the ConfigMap key isn't just style — it's necessary if you want to verify that specific injection method in isolation, since identical names from different sources collide silently.
- `envFrom` exports every key in a ConfigMap as an env var, with no awareness of which keys were meant for files. Keep env-oriented and file-oriented data in separate ConfigMaps to avoid this.
- **If a live config change needs to reach the app without a restart, it has to be read from a mounted file, not an env var** — env vars are permanently frozen at container start no matter what happens to the ConfigMap afterward.