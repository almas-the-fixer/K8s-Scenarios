## Service — Scenario 1 (Easy)

**Goal:** expose a Deployment with a ClusterIP Service, and see how endpoints track Pods automatically as the Deployment scales.

**Suggested time:** 10 minutes

### Task

1. Create a Deployment named `web-app` with **3 replicas**, image `nginx:1.25`, Pods labeled `app=web-app`, container listening on port `80`.
2. Create a **ClusterIP** Service named `web-svc` that selects `app=web-app`, serving on port `80` and targeting the container's port `80`.
3. Confirm the Service actually has live endpoints — show the command that lists them, and confirm the IPs match your 3 Pods' IPs.
4. Spin up a temporary debug Pod and, from inside it, reach `web-app` **two different ways**:
   - by the Service's ClusterIP directly
   - by the Service's DNS name (not the Pod names)
   
   Show a successful response both times.
5. Scale `web-app` to **5** replicas. Without touching the Service at all, confirm the endpoint list grew to 5 on its own.
6. Print only the Service's ClusterIP and port (not the full `get svc` output).

### Deliverable

Deployment as `s1_deployment.yml`, Service as `s1_service.yml`. No broken state — this is about seeing how selectors, endpoints, and DNS actually connect before we start breaking that connection in S2.

<details>
<summary>Hint (step 4)</summary>

`kubectl run <name> --image=busybox:1.36 --rm -it -- sh` gives you a throwaway shell inside the cluster. `wget` or `nslookup` both work from busybox; nginx doesn't need `curl` specifically to prove it responds.

</details>

<details>
<summary>Hint (step 6)</summary>

`-o jsonpath` lets you pull out just the fields you want from any object, not just Services.

</details>