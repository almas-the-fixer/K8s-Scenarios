# Pod — Solutions

## Scenario 1

### Verification commands

> Note: For yml file see s1_pod.yml

**Node the Pod landed on:**

```bash
kubectl get pods -o wide
```


**Filtered env var from inside the container:**

```bash
kubectl exec web-inspector -- printenv | grep "ENVIRONMENT"
```


**Last 5 log lines:**

```bash
kubectl logs web-inspector --tail=5
```