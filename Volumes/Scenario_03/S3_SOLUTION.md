## Scenario 3: Hard (Stacked Bugs)

**Diagnosis**
After applying the initial buggy YAML and the ConfigMap, the Pod failed to start.
1. `kubectl describe pod vol-s3-pod` showed the init container failing. Looking at the init container's command, it was trying to write to `/wrong-data/index.html`, but the `volumeMount` for `shared-data` was mounted at `/data`. Path mismatch.
2. After fixing the path, the Pod still failed. Events showed: `MountVolume.SetUp failed for volume "host-logs": hostPath type check failed: /var/log/nginx is not a directory, it's a File`. The `hostPath` was set to `type: File`, but the application expects to write logs to a directory. Furthermore, the node's filesystem actually had `/var/log/nginx` as a regular file, causing a clash when trying to mount it as a directory.
3. After fixing the `hostPath` type, the Pod still failed with `CreateContainerConfigError`. The volume `config-vol` referenced a ConfigMap named `my-config`, but the actual ConfigMap applied to the cluster was named `app-config`.

**Root Cause**
1. Init container command path did not match the `volumeMount` path.
2. `hostPath` volume type was incorrectly set to `File` instead of a directory type, and clashed with an existing file on the worker node.
3. ConfigMap volume referenced a non-existent ConfigMap name (`my-config` instead of `app-config`).

**Fix**
1. Changed the init container command to write to `/data/index.html` to match the `mountPath: /data`.
2. Changed the `hostPath` type from `File` to `DirectoryOrCreate`. (Note: On the Killercoda playground, `/var/log/nginx` existed as a file, requiring manual deletion via `ssh node01` and `rm /var/log/nginx` before the Pod could start.)
3. Changed the ConfigMap volume `name` reference from `my-config` to `app-config`.

    initContainers:
    - name: init-busybox
      command: ['sh', '-c', 'echo "Welcome to Scenario 3" > /data/index.html']
      volumeMounts:
      - name: shared-data
        mountPath: /data
    ...
    volumes:
    - name: host-logs
      hostPath:
        path: /var/log/nginx
        type: DirectoryOrCreate
    - name: config-vol
      configMap:
        name: app-config

**Verification**
1. `kubectl get pod vol-s3-pod -o wide` confirmed the Pod reached `1/1 Running`.
2. `curl <Pod-IP>` returned `Welcome to Scenario 3`, proving the init container wrote to the `emptyDir` and the main container served it.
3. `kubectl exec -it vol-s3-pod -c app-nginx -- cat /etc/nginx/conf.d/custom.conf` returned the nginx config, proving the ConfigMap was mounted correctly.

**Key Takeaways**
- Init Containers and Shared Volumes: Init containers run to completion before the main containers start. They are frequently used with `emptyDir` volumes to prepare data, files, or configurations that the main containers will consume. Ensure the paths in the init container's commands exactly match the `mountPath` defined in its `volumeMounts`.
- Pod Immutability: You cannot edit a running Pod's volumes or volumeMounts. If you make a mistake in a Pod's volume configuration, you must delete the Pod and reapply the corrected YAML. (Deployments handle this automatically via rolling updates, but raw Pods do not.)
- hostPath Node Coupling: `hostPath` volumes are strictly tied to the filesystem of the specific node where the Pod is scheduled. If the node's filesystem has unexpected files, permissions, or missing directories, the Pod will fail. Always check the correct worker node's filesystem (via `ssh`) when debugging `hostPath` issues. In production, avoid `hostPath` for application data; use PersistentVolumes instead.
- Scenario Design Note: The `hostPath` bug in this scenario exposed a real Killercoda quirk where `/var/log/nginx` existed as a regular file on the node. This was not an intentional part of the scenario design but turned into a valuable real-world lesson about node filesystem state affecting Pod scheduling.