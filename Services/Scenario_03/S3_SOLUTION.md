## Scenario 3

### Diagnosis

**Setup:** applied `s3_deployment.yml` (two named container ports: `web`→80, `metrics`→9090), then attempted `s3_service_bad.yml`.

**Bug 1 — rejected outright, before any Service exists**

```bash
kubectl apply -f s3_service_bad.yml
```
Error: `nodePort` value `8080` out of the allowed range. Kubernetes restricts NodePort to `30000-32767` because a NodePort opens a real listener on every node's actual network interface, not just inside the cluster — the range exists so it doesn't collide with other software already using ports on the node's OS.

**Bug 2 — endpoints existed but only for `metrics`, not `web`**

```bash
kubectl get endpoints multi-svc
```
`web`'s `targetPort` was `web-app`, which didn't match any container port's `name` (the container only has ports named `web` and `metrics`). A name-based `targetPort` that doesn't match anything produces no endpoints for that port specifically, even though the Service and selector are otherwise fine.

**Bug 3 — endpoints existed, but connections hung instead of failing cleanly**

```yaml
protocol: UDP   # on the web port
```
`protocol` tells `kube-proxy` which kind of forwarding rule to install — TCP and UDP rules are tracked entirely separately. With `protocol: UDP`, no TCP rule existed for that ClusterIP:port at all. A `curl` (TCP) request to a port with an explicit reject gets `Connection refused` immediately; a request to a port with **no matching rule at all** just hangs with no response until it times out — a different failure signature for a different kind of bug.

### Root cause (3 bugs)

1. `nodePort: 8080` outside the valid `30000-32767` range — rejected at apply time.
2. `targetPort: web-app` didn't match the container's actual port name (`web`).
3. `protocol: UDP` on a port meant to carry HTTP/TCP traffic — `kube-proxy` never created a TCP rule for it.

### Fix

```yaml
- name: web
  port: 80
  targetPort: web
  protocol: TCP
  nodePort: 30000
```

### Verification

**ClusterIP:**
```bash
kubectl get svc multi-svc -o wide
```
```
multi-svc   NodePort   10.108.17.198   80:30000/TCP,9090:32151/TCP
```
```bash
curl 10.108.17.198
```
nginx welcome page returned.

**NodePort (via a real node IP, not the ClusterIP):**
```bash
kubectl get nodes -o wide
curl 172.30.1.2:30000
```
nginx welcome page returned. NodePort opens on *every* node's real IP regardless of which node the Pod actually runs on — testing it against the ClusterIP instead of a node IP produces the same "no matching rule" hang as Bug 3, not a real failure.

**`metrics` (untouched port) — a fourth finding, not a fix:**
```bash
curl 10.108.87.195:9090
```

`Connection refused` in 0ms, not a hang — this means something *did* answer and immediately rejected the connection, because nothing inside the container is listening on 9090. `containerPort: 9090` in the Deployment spec is only metadata/documentation; it doesn't make anything bind to that port. Stock `nginx:1.25` only runs its web server on port 80, so `metrics` was never a functioning port from the very start of this scenario — `get svc` and `get endpoints` both reported it as fully configured anyway, because neither one checks whether a real process is actually listening inside the container.

### Key takeaways

- A field can be schema-valid YAML and still get rejected outright by the API server if it violates a hard rule (`nodePort` range here; `spec.selector` immutability in Deployment S3). Read the actual error — it names the exact constraint.
- `targetPort` can reference a container port by **name** instead of number. If the name doesn't match anything on the container, that specific Service port gets no endpoints, even though the rest of the Service is fine.
- `protocol` (TCP vs UDP) determines which kind of `kube-proxy` rule gets installed. A port with the wrong protocol isn't "refused" — there's no rule to even respond, so the connection just hangs instead of failing immediately. **Connection refused** = something answered and said no. **Hanging/timeout** = nothing answered at all, because no rule matches.
- `kubectl get endpoints` shows real Pod `IP:port` pairs — the actual backends behind a Service. Curling an endpoint IP directly is a valid debugging technique to isolate "is the Pod itself fine" from "is the Service's routing broken," but it isn't the normal traffic path — normal traffic always goes through the Service's own address.
- A **NodePort** is a separate listener opened on every node's real IP — it is not reachable via `<ClusterIP>:<nodePort>`. Test it against an actual node IP (`kubectl get nodes -o wide`), any node, regardless of where the Pod is scheduled.
- A **`containerPort` declared in a Pod spec is purely informational** — it does not make anything listen there. `get svc`/`get endpoints` reporting a port as "configured" only means the Service-side routing is set up correctly; it says nothing about whether a real process inside the container is actually bound to that port. `Connection refused` at the final hop (container itself) is the signature of this gap — something in the OS answered and said no, because truly nothing is there.