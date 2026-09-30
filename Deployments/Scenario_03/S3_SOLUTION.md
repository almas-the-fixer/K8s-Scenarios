## Scenario 3

### Diagnosis

**Setup:** applied `s3_secret.yml`, then healthy `s3_deployment.yml` (3/3), then attempted `s3_deployment_bad.yml`.

**Failure 1 — rejected outright, before any Pod exists**
```bash
kubectl apply -f s3_deployment_bad.yml
```
```
The Deployment "shop-cart" is invalid: spec.selector: Invalid value:
{"matchLabels":{"app":"shop-cart","tier":"backend"}}: field is immutable
```
A Deployment's `spec.selector` is locked once the object is created. Kubernetes needs an unambiguous, unchanging answer to "which Pods belong to this Deployment" — allowing the selector to change later would let it silently gain or lose ownership of Pods, breaking the same ownership model seen with ReplicaSets.

**Failure 2 — `CreateContainerConfigError` (after reverting the selector)**
```bash
kubectl apply -f s3_deployment_fixed.yml
kubectl get pods -w
```
```
shop-cart-7c74ff7467-r88rm   0/1   CreateContainerConfigError   0   4s
```
```bash
kubectl describe pod shop-cart-7c74ff7467-r88rm
```
```
CART_MODE:  <set to the key 'cart_mode' in secret 'cart-secret'>  Optional: false
```
`describe` names the exact key it tried to use (`cart_mode`). Checking the Secret directly:
```bash
kubectl get secret cart-secret -o yaml
```
showed the real key is `CART_MODE` — a case mismatch, same pattern as the ConfigMap key bug from ReplicaSet Scenario 2.

**Failure 3 — stuck `ContainerCreating`, visible only in events (after fixing the Secret key)**
```bash
kubectl get pods -w
```
Pods sat in `ContainerCreating` with **zero restarts** — the container never reached `Running`, so no `Last State: OOMKilled` ever appeared under `describe`.
```bash
kubectl get events
```
```
Warning  FailedCreatePodSandBox  pod/shop-cart-...  Failed to create pod sandbox: ...
  OCI runtime create failed: runc create failed: unable to start container process:
  container init was OOM-killed (memory limit too low?)
```
The memory limit was too low for the container runtime to even start the process — a different failure signature from a container that runs and then gets OOM-killed later (that version shows up as `CrashLoopBackOff` with `Last State: Terminated, Reason: OOMKilled` and a climbing `RESTARTS` count).

### Root cause (3 bugs)

1. **Immutable field violation** — the "update" tried to change `spec.selector`, which can never change after creation.
2. **Secret key case mismatch** — `key: cart_mode` referenced a key that doesn't exist; the real key is `CART_MODE`.
3. **Memory limit far below what the container runtime needs to start** — set to `2Mi`, which fails before the container even reaches `Running`.

### Fix

Reverted the selector/template labels to match the original:
```yaml
selector:
  matchLabels:
    app: shop-cart
template:
  metadata:
    labels:
      app: shop-cart
```
Corrected the Secret key:
```yaml
key: CART_MODE
```
Raised the memory limit to a realistic value:
```yaml
resources:
  requests:
    memory: 32Mi
  limits:
    memory: 64Mi
```

### Verification

```bash
kubectl get pods -o wide
```
```
shop-cart-597c7d75bc-7x79p   1/1   Running   0   4m55s
shop-cart-597c7d75bc-8sb8g   1/1   Running   0   4m56s
shop-cart-597c7d75bc-fthbp   1/1   Running   0   4m54s
```
```bash
kubectl rollout status deployment shop-cart
```
```
deployment "shop-cart" successfully rolled out
```
```bash
kubectl exec -it shop-cart-597c7d75bc-7x79p -- sh
printenv | grep CART_MODE
```
```
CART_MODE=standard
```

### Key takeaways

- `spec.selector` on a Deployment is immutable by design — you cannot add or change match labels after creation. The error message names the exact field.
- `describe`'s Secret/ConfigMap env line (`<set to the key '...' in secret '...'>`) tells you exactly which key was requested — compare it against `kubectl get secret <name> -o yaml` rather than guessing.
- An extremely low memory limit can fail **before** the container ever starts (`FailedCreatePodSandBox`, visible only in `get events`, zero restarts), which looks and behaves differently from a container that runs and gets OOM-killed later (`CrashLoopBackOff`, `Last State: OOMKilled`, restarts climbing). Both are memory problems; the failure point differs.
- In production, size `requests`/`limits` from measured usage (Prometheus/Grafana, VPA recommendations), alert before the limit is hit, and use a namespace `LimitRange` to prevent unrealistic values from shipping at all.