## Scenario 2

### Diagnosis

**Setup:** applied `s2_deployment.yml` (healthy, 4/4), then applied `s2_deployment_bad.yml`.

```bash
kubectl rollout status deployment/shop-api --timeout=15s
```
```
Waiting for deployment "shop-api" rollout to finish: 2 out of 4 new replicas have been updated...
error: timed out waiting for the condition
```
Timing out isn't a hard failure signal by itself — it means the rollout hadn't finished within the window checked.

**Why both ReplicaSets had live Pods at once:**
```bash
kubectl get rs
```
```
shop-api-594876868    2   2   0    (new)
shop-api-5f7bb97b46   3   3   3    (old)
```
With `replicas: 4` and the default `25% max unavailable, 25% max surge`, both round to **1** Pod. The rollout can have at most 1 extra Pod above desired, and at most 1 missing from desired, at any moment. It created 2 new Pods and removed 1 old Pod, hit both budget limits simultaneously, and froze there, since the new Pods never became Ready (nothing available to trade against).

**Root cause:**
```bash
kubectl describe deployments.apps shop-api
```
```
Readiness:  http-get http://:80/status delay=2s timeout=1s period=5s
```
```bash
kubectl get events
```
```
Warning  Unhealthy  pod/shop-api-594876868-mnlsq  Readiness probe failed: HTTP probe failed with statuscode: 404
Warning  Unhealthy  pod/shop-api-594876868-tnk7z  Readiness probe failed: HTTP probe failed with statuscode: 404
```
Same signature as before: a 404 means the app answered, and `/status` doesn't exist on a stock nginx image.

### Decision: mitigate first

Not yet 100% sure of the exact fix, but the safe first move is restoring service, not continuing to dig mid-incident:
```bash
kubectl rollout history deployment shop-api
```
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
```bash
kubectl rollout undo deployment shop-api --to-revision=1
kubectl get rs
```
```
shop-api-594876868    0   0   0
shop-api-5f7bb97b46   4   4   4
```
Back to a known-good state without needing to know the root cause first.

### Fix

Copied the bad file to `s2_deployment_fixed.yml`:
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```
Applying it came back `configured`, but produced no new ReplicaSet — the fixed template was now identical to the already-restored revision 1, so there was nothing new to roll out. To get a genuine test of the fix, forced Pod recreation:
```bash
kubectl delete pods -l app=shop-api
kubectl rollout status deployment shop-api
```
```
Waiting for deployment "shop-api" rollout to finish: 1 of 4 updated replicas are available...
Waiting for deployment "shop-api" rollout to finish: 2 of 4 updated replicas are available...
Waiting for deployment "shop-api" rollout to finish: 3 of 4 updated replicas are available...
deployment "shop-api" successfully rolled out
```

### Verification

```bash
kubectl get pods
```
```
shop-api-5f7bb97b46-8zvb8   1/1   Running   0   13s
shop-api-5f7bb97b46-q7kml   1/1   Running   0   13s
shop-api-5f7bb97b46-qs6df   1/1   Running   0   13s
shop-api-5f7bb97b46-tnmkn   1/1   Running   0   13s
```
All 4 Ready, and `rollout status` returned success cleanly, not just a Pod list that looks okay.

### Key takeaways

- `maxSurge`/`maxUnavailable` aren't just "go slow" settings. They're a hard budget. A rollout with Pods that never pass readiness freezes exactly at the edge of that budget, not somewhere arbitrary.
- A frozen-but-not-failed rollout still shows `Available: True` / `Progressing: True` in `describe`. Kubernetes only flags it as failed once `progressDeadlineSeconds` (default 600s) passes.
- In an incident, restore a known-good state first with `rollout undo`, then investigate. You don't need the root cause to roll back.
- If a fix happens to match a revision you've already rolled back to, `apply` reports `configured` but creates no new ReplicaSet, since nothing actually changed. `kubectl delete pods -l <label>` forces real recreation so you can verify the fix under an actual rollout, not just a resting state.