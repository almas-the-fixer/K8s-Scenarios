# Scenario 2: Volumes (Troubleshooting)

## Objective
Diagnose and fix a single targeted bug preventing a Pod from starting. The issue is related to volume configuration.

## Tasks
1. Apply the provided buggy YAML file (`s2_volumes.yml`).
2. Observe the Pod's state. It will not reach `Running`.
3. Use `kubectl describe pod` and/or `kubectl get events` to diagnose why the Pod is failing to start.
4. Identify the root cause related to the volume configuration.
5. Create a fixed version of the file named `s2_volumes_fixed.yml`. Do not edit the original file.
6. Apply the fixed file.
7. Verify the Pod reaches the `Running` state and the volume is correctly mounted.

<details>
<summary>Hint</summary>

Look closely at the `hostPath` volume definition and the `type` field. How does Kubernetes enforce the `Directory` type compared to `DirectoryOrCreate`? Check the node's filesystem if you are unsure whether the path exists.
</details>