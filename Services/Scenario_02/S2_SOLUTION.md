## Scenario 2

### Diagnosis

**Setup:** applied healthy `s2_deployment.yml`, then broken `s2_service.yml`.

**Symptom:**
```bash
kubectl run debug-pod --image=busybox:1.36 --rm -it -- sh
wget api-svc
```
```
Connecting to 10.x.x.x (10.x.x.x:80)
wget: can't connect to remote host: Connection refused
```

**Check endpoints first, before suspecting the selector:**
```bash
kubectl get endpoints api-svc
```
```
api-svc   192.168.0.83:8080,192.168.1.250:8080   2m49s
```
Endpoints exist and have real Pod IPs, which immediately rules out a selector/label mismatch — if the selector hadn't matched anything, this list would be empty. The problem is narrowed to something about the port.

**Confirm what the container actually listens on:**
```bash
kubectl describe pod <api-app-pod>
```
```
Port:  80/TCP
```
The Service's `targetPort` was `8080`, but the container is only listening on `80`.

### Root cause

`targetPort: 8080` in the Service didn't match the container's actual `containerPort: 80`. The selector was correct the whole time — Service traffic was being sent to a port nothing was listening on.

### Fix

```yaml
ports:
  - port: 80
    targetPort: 80
```

### Verification

```bash
kubectl apply -f s2_service_fixed.yml
kubectl run debug-pod --image=busybox:1.36 --rm -it -- sh
wget 10.101.198.172
cat index.html
```
```
<title>Welcome to nginx!</title>
...
```
Reached successfully by ClusterIP. Also reachable by DNS name (`wget api-svc`) — note: a second `wget` to the same name can appear to "fail" simply because `index.html` already exists locally from the first attempt; that's `wget` refusing to overwrite a file, not a real connectivity failure. Use `wget -O -` or a fresh debug Pod to avoid the false alarm.

### Key takeaway: two different failure modes, two different signatures

- **No endpoints at all** → the selector matched zero Pods. `kubectl get endpoints` catches this immediately — the list is empty, full stop. Fast and unambiguous.
- **Endpoints exist but point at the wrong port** → the selector is fine, Pods are found, IPs are populated — but `get endpoints` will show a port number that *looks* valid without ever checking whether anything is actually listening there. This is invisible from `get endpoints` alone. You have to cross-reference the Service's `targetPort` against the Pod's actual listening port via `describe pod`.

Both failure modes look identical from the outside ("the Service doesn't work"), but `get endpoints` empty vs. populated is what tells you which half of the problem to even look in.