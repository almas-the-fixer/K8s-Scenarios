## Scenario — StatefulSet

The DevOps team wants to deploy a simple stateful web app to test persistent identity and independent storage behavior. Below are the requirements.

### Requirements

- Create a headless Service named `web-headless` that selects Pods with label `app: web` on port `80`.
- Create a StatefulSet named `web` with:
  - `serviceName` pointing to your headless Service
  - `3` replicas
  - container name `web-container`, image `nginx:latest`, listening on `containerPort: 80`
  - a `volumeClaimTemplates` entry named `www`, mounted at `/usr/share/nginx/html`, requesting `1Gi` storage with `ReadWriteOnce` access

### Verification

- `kubectl get pods` shows Pod names following the `web-0`, `web-1`, `web-2` pattern (not random hashes)
- `kubectl get pvc` shows 3 separate PVCs, one per Pod
- `kubectl get pods` shows them coming up in order (`web-0` Ready before `web-1` starts)

### Twist — prove storage isolation

1. `kubectl exec` into `web-1` and write a file into the mounted volume:
```bash
   kubectl exec web-1 -- sh -c 'echo "hello from pod 1" > /usr/share/nginx/html/index.html'
```
2. `kubectl exec` into `web-0` and `web-2` and check whether that file exists there too. It shouldn't — explain *why* in your own words before checking my answer.
3. Now delete `web-1` entirely (`kubectl delete pod web-1`) and wait for it to come back. `exec` in again and check if `index.html` still has your text.
   - Prediction first, then verify: does the file survive the Pod's death, or is it gone?