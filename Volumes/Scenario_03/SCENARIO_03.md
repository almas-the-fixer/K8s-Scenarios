# Scenario 3: Volumes (Hard)

## Objective
Diagnose and fix multiple stacked bugs in a Pod manifest utilizing an init container, emptyDir, hostPath, and configMap volumes. 

## Tasks
1. Apply the supporting ConfigMap file (`s3_configmap.yml`).
2. Apply the buggy Pod manifest (`s3_volumes.yml`).
3. The Pod will fail to start. Diagnose the first blocking issue using `kubectl describe pod` or events.
4. Create a fixed version of the Pod manifest named `s3_volumes_fixed.yml`. Do not edit the original file.
5. Apply the fixed file. If the Pod still fails, repeat the diagnose-and-fix loop until the Pod reaches the `Running` state.
6. Verify the final running Pod is serving the correct custom `index.html` content on port 80, and that the ConfigMap is correctly mounted.

<details>
<summary>Hint</summary>

Pay close attention to the mount paths in the initContainer versus the command it is executing. Also, verify the exact names of the resources being referenced in the volume definitions against what actually exists in the cluster.
</details>