# Pod — Scenarios

## Scenario 1 (Easy)

**Goal:** basic Pod creation, correct spec details, and reading state back out — no troubleshooting yet.

**Suggested time:** 5 minutes

### Task

1. Create a Pod named `web-inspector` running image `nginx:1.25`.
2. It must expose container port `8080` (just declare `containerPort: 8080` in the spec — no need to reconfigure nginx itself to actually listen there for this exercise).
3. Give it the labels `app=web-inspector` and `tier=frontend`.
4. Set an environment variable `ENVIRONMENT=staging` on the container.
5. Set a memory request of `64Mi` and a memory limit of `128Mi` (no CPU requirements needed).
6. Once running, without editing any files:
   - Confirm the Pod is `Running` and show which node it landed on.
   - Print only the value of the `ENVIRONMENT` env var from inside the running container (don't just dump the whole env — filter for it).
   - Retrieve the last 5 log lines from the container.

### Deliverable

Pod manifest as `s1_pod.yml`. No broken state in this one — it's just to nail the format cleanly and get the verification commands into muscle memory.