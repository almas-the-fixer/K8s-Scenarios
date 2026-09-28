# ReplicaSet — Solutions

## Scenario 1

### Create and verify

```bash
kubectl apply -f s1_replicaset.yml
kubectl get pods -w
```
```
web-rs-krt8r   0/1   ContainerCreating   0   6s
web-rs-n5zjd   0/1   ContainerCreating   0   6s
web-rs-nqmc5   0/1   ContainerCreating   0   6s
...
web-rs-n5zjd   1/1   Running             0   8s
```

**Nodes each Pod landed on:**
```bash
kubectl get pods -o wide
```
```
NAME           READY   STATUS    RESTARTS   AGE     IP              NODE
web-rs-7t2jb   1/1     Running   0          107s    192.168.1.229   node01
web-rs-n5zjd   1/1     Running   0          2m13s   192.168.0.237   controlplane
web-rs-nqmc5   1/1     Running   0          2m13s   192.168.1.254   node01
```

### Self-healing

```bash
kubectl delete pod web-rs-krt8r
```
The watch showed `web-rs-krt8r` go `Terminating` and a new Pod, `web-rs-7t2jb`, appear at `AGE 0s`. A new name and a fresh age show it's a replacement, not the old Pod recovering.

### Scaling

**Imperative (to 5):**
```bash
kubectl scale replicaset web-rs --replicas 5
kubectl get pods -w
```
5 Pods `Running`. This changed the live object, not the manifest file.

**Declarative (down to 2):** edited `replicas: 5` → `replicas: 2` in `s1_replicaset.yml`, then:
```bash
kubectl apply -f s1_replicaset.yml
kubectl get pods -w
```
```
replicaset.apps/web-rs configured
web-rs-7t2jb   1/1   Running   0   3m10s
web-rs-n5zjd   1/1   Running   0   3m36s
```

### Desired / current / ready counts

```bash
kubectl get rs
```
```
NAME     DESIRED   CURRENT   READY   AGE
web-rs   2         2         2       4m41s
```

### Key takeaways

- A ReplicaSet keeps the Pod count on target by creating replacements. The replacement is a new Pod with a new name.
- `kubectl scale` changes the live object only, and a later `apply` resets it to whatever the file says. Keep changes in the manifest.

---