## Scenario 2: Troubleshooting (hostPath)

**Diagnosis**
After applying the buggy YAML, the Pod was stuck in `ContainerCreating`. Running `kubectl describe pod vol-s2-pod` and checking the Events section at the bottom reveals the exact failure:
`MountVolume.SetUp failed for volume "host-logs": hostPath type check failed: /opt/strict-logs is not a directory`

**Root Cause**
The `hostPath` volume was configured with `type: Directory`. This strict type tells the kubelet to verify that the path exists on the node's filesystem and is actually a directory before starting the container. Because `/opt/strict-logs` did not exist on the target node, the kubelet refused to mount it, causing the Pod to fail.

**Fix**
Change the `type` field in the `hostPath` volume definition from `Directory` to `DirectoryOrCreate`. This tells the kubelet to create the directory on the node if it does not already exist.

```yaml
volumes:
name: host-logs
hostPath:
path: /opt/strict-logs
type: DirectoryOrCreate
```

**Verification**
1. Apply the fixed manifest: `kubectl apply -f s2_pod_fixed.yml`.
2. Verify the Pod is running: `kubectl get pods`.
3. **Crucial step:** Verify the directory was created on the correct node. Since the Pod was scheduled on `node01` (visible via `kubectl get pods -o wide`), you must check `node01`'s filesystem, not the `controlplane`'s filesystem. 
   - `ssh node01`
   - `ls -ld /opt/strict-logs` (This will confirm the directory exists on the actual host).

**Key Takeaways**
- `hostPath` volumes mount the filesystem of the *specific worker node* where the Pod is scheduled, not the control plane. If you check the host filesystem on the wrong node, the directory will not be there.
- Understand the `hostPath` type enforcement:
  - `Directory`: The directory must exist. If it doesn't, the Pod fails.
  - `DirectoryOrCreate`: The kubelet will create it if it's missing.
  - `File`: The file must exist.
  - `FileOrCreate`: The kubelet will create an empty file if it's missing.
- In a real-world production environment, using `hostPath` is generally discouraged due to security risks and node-specific coupling. It is mostly used for specific node-level agents (like log collectors or monitoring tools). For standard app data, PersistentVolumes (PV/PVC) are the standard approach.
***