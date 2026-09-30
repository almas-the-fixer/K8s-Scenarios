# Service — Solutions

## Scenario 1

### Create and verify

```bash
kubectl apply -f s1_deployment.yml
kubectl apply -f s1_service.yml
kubectl get pods -o wide -w
```
```
web-app-5f8df68655-9kgbf   1/1   Running   0   10s   192.168.0.188   controlplane
web-app-5f8df68655-cpqgp   1/1   Running   0   10s   192.168.1.191   node01
web-app-5f8df68655-cxr72   1/1   Running   0   10s   192.168.1.223   node01
```
```bash
kubectl get svc
```
```
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   10.109.61.97   <none>        80/TCP    10s
```

### Endpoints

```bash
kubectl get endpoints web-svc
```
```
web-svc   192.168.0.118:80,192.168.0.188:80,192.168.1.191:80   19m
```
The Service's endpoint list is populated automatically from any Pod whose labels match the Service's selector — same adoption mechanism seen with ReplicaSets, just applied to networking instead of Pod ownership.

### Reaching the Service two ways (from a throwaway debug Pod)

```bash
kubectl run temp-pod --image=busybox:1.36 --rm -it -- sh
```
Installed `curl` inside the debug Pod, then:

**By ClusterIP:**
```bash
curl 10.109.61.97
```
```
<title>Welcome to nginx!</title>
...
```

**By DNS name:**
```bash
curl http://web-svc
```
```
<title>Welcome to nginx!</title>
...
```
Both returned the same nginx page — the Service's DNS name resolves to the same ClusterIP under the hood.

### Scaling and endpoint tracking

```bash
kubectl scale deployment web-app --replicas=3
kubectl get endpoints web-svc
```
```
web-svc   192.168.0.118:80,192.168.0.188:80,192.168.1.191:80   19m
```
Scaled the Deployment without touching the Service at all, and the endpoint list shrank to exactly the 3 remaining Pod IPs on its own.

### ClusterIP and port only

```bash
kubectl get svc web-svc -o jsonpath='{.spec.clusterIP}{":"}{.spec.ports[*].port}{"\n"}'
```
```
10.109.61.97:80
```

### Key takeaways

- A Service never "knows" about specific Pods by name — it matches by **label selector**, same as a ReplicaSet, and the endpoint list updates automatically as Pods come and go.
- The Service's ClusterIP and its DNS name (same name as the Service object) both resolve to the same thing — DNS is just a friendlier way to reach the same ClusterIP.
- `kubectl get endpoints <svc>` is the command to prove a Service actually has something behind it — a Service can exist and look fine in `get svc` while having **zero** endpoints if the selector doesn't match anything, which is exactly the kind of bug S2 will dig into.
- `kubectl <resource> --help` / `kubectl explain <resource>` are both offline — worth the reflex over a web search, since the real exam has no internet access.