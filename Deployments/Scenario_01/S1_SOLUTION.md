# Deployment — Solutions

## Scenario 1

### Create and verify

```bash
kubectl apply -f s1_deployment.yml
kubectl get deployments.apps
kubectl get pods -w
```
```
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
shop-web   0/3     3            0           5s
```
```
shop-web-79b75dc769-7mr5h   0/1   ContainerCreating   0   9s
shop-web-79b75dc769-945zv   0/1   ContainerCreating   0   9s
shop-web-79b75dc769-vrqpk   0/1   ContainerCreating   0   9s
...
shop-web-79b75dc769-7mr5h   1/1   Running             0   10s
```
`shop-web-79b75dc769` is the ReplicaSet the Deployment created. The hash suffix is generated from the Pod template, so it changes whenever the template changes.

### Rolling update (imperative image change)

```bash
kubectl set image deployments/shop-web nginx-container=nginx:1.25
kubectl get pods -w
```
```
shop-web-79b75dc769-6xh7k   1/1   Running             0   4m57s
shop-web-79b75dc769-rl8mq   1/1   Running             0   4m58s
shop-web-79b75dc769-w6nf6   1/1   Running             0   4m56s
shop-web-75868cc8f6-9nzqw   0/1   Pending             0   0s
shop-web-75868cc8f6-9nzqw   1/1   Running             0   1s
shop-web-79b75dc769-rl8mq   1/1   Terminating         0   4m59s
...
```
One new Pod comes up on the new ReplicaSet before one old Pod terminates, staggered rather than all at once. That's `RollingUpdateStrategy: 25% max unavailable, 25% max surge` at work.

### Confirm rollout completed correctly

```bash
kubectl get pods -o wide
```
```
shop-web-75868cc8f6-8dr6t   1/1   Running   0   30s   controlplane
shop-web-75868cc8f6-9nzqw   1/1   Running   0   31s   node01
shop-web-75868cc8f6-f6jxz   1/1   Running   0   29s   node01
```
```bash
kubectl get rs
```
```
NAME                  DESIRED   CURRENT   READY   AGE
shop-web-75868cc8f6   3         3         3       9m8s
shop-web-79b75dc769   0         0         0       13m
```
New ReplicaSet has all 3 Pods. Old ReplicaSet still exists but has 0.

### Rollback

Check history before targeting a revision:
```bash
kubectl rollout history deployment shop-web
```
```
REVISION  CHANGE-CAUSE
5         <none>
6         <none>
```
```bash
kubectl rollout undo deployment shop-web --to-revision=5
kubectl get rs
```
```
NAME                  DESIRED   CURRENT   READY   AGE
shop-web-75868cc8f6   0         0         0       9m52s
shop-web-79b75dc769   3         3         3       14m
```

### Verify the rollback actually changed the image

```bash
kubectl describe deployments.apps shop-web
```
```
Annotations:  deployment.kubernetes.io/revision: 7
...
Containers:
 nginx-container:
  Image:  nginx:1.24
```

### Key takeaways

- `kubectl set image deployment/<name> <container-name>=<image>` targets the **container name** inside the Pod template, not the Deployment's own name.
- A rolling update creates a **new** ReplicaSet from the updated template and shifts Pods across gradually, controlled by `maxUnavailable`/`maxSurge`. The old ReplicaSet isn't deleted, just scaled to 0, which is how rollback works instantly without recreating anything.
- Always run `kubectl rollout history` before `--to-revision=N`. Guessing a number that was never created (or was pruned) fails with `unable to find specified revision`.
- `rollout undo` is **not** a true rewind. It copies the target revision's template and pushes it as a **new** revision on top of history. Revision numbers only ever go up. "Rolling back" content-wise still means "rolling forward" number-wise — rolling back to revision 5's content produced revision 7.
- Trust `kubectl describe`'s actual `Image:` field to confirm a rollback worked, not just the `rolled back` success message.