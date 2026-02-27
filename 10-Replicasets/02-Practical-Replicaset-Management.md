> This article explores ReplicaSets in Kubernetes, covering verification, troubleshooting, scaling, and modifying ReplicaSets through practical commands and configurations.

---

## 1. Verify Existing Pods

Begin by checking the current pods in your cluster. Run the following command:

```bash theme={null}
kubectl get pods
```

The result is expected to be:

```plaintext theme={null}
No resources found in default namespace.
```

This indicates that there are no pods currently running.

---

## 2. Review Existing ReplicaSets

Next, inspect if any ReplicaSets are present:

```bash theme={null}
kubectl get replicaset
```

Initially, no ReplicaSets are available. After making configuration changes, a new ReplicaSet may be created. Check its details using the same command:

```bash theme={null}
kubectl get replicaset
```

You might see an output similar to:

```plaintext theme={null}
NAME                DESIRED   CURRENT   READY   AGE
new-replica-set     4         4         0       9s
```

---

## 3. Analyze ReplicaSet Details

To obtain a detailed description of the ReplicaSet, execute:

```bash theme={null}
kubectl describe replicaset new-replica-set
```

Focus on the pod template section; observe the container configuration:

- **Image:** `busybox777`
- **Command:**
  ```bash theme={null}
  sh -c "echo Hello Kubernetes! && sleep 3600"
  ```

Below is an excerpt from the output:

```bash theme={null}
Name:           new-replica-set
Namespace:      default
Selector:       name=busybox-pod
Labels:         <none>
Annotations:    <none>
Replicas:       4 current / 4 desired
Pods Status:    0 Running / 4 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:    name=busybox-pod
  Containers:
   busybox-container:
      Image:        busybox777
      Port:         <none>
      Host Port:    <none>
      Command:
        sh
        -c
        echo Hello Kubernetes! && sleep 3600
      Mounts:      <none>
  Volumes:     <none>
Events:
  Type     Reason             Age   From                     Message
  Normal   SuccessfulCreate   60s   replicaset-controller    Created pod: new-replica-set-7r2qw
  Normal   SuccessfulCreate   60s   replicaset-controller    Created pod: new-replica-set-wkzjh
  Normal   SuccessfulCreate   60s   replicaset-controller    Created pod: new-replica-set-tn2mp
  Normal   SuccessfulCreate   60s   replicaset-controller    Created pod: new-replica-set-vpkh8
```

The configuration reveals that the pods are configured to run the image `busybox777`.

---

## 4. Check Pod Readiness

Examine the number of ready pods in the ReplicaSet. As seen from the detailed description:

```bash theme={null}
Pods Status:    0 Running / 4 Waiting / 0 Succeeded / 0 Failed
```

This indicates that none of the pods are ready at this point.

---

## 5. Troubleshoot Pod Readiness Issues

Investigate why the pods are not transitioning to a ready state by describing one of them:

```bash theme={null}
kubectl describe pod new-replica-set-7r2qw
```

The output shows that the pod is in a waiting state with the reason `ImagePullBackOff`. This error occurs because the image `busybox777` cannot be pulled, likely due to a non-existent repository or missing authorization.

An excerpt from the pod events:

```plaintext theme={null}
State:           Waiting
Reason:          ImagePullBackOff
...
Events:
  Normal   Pulling   ...   kubelet     Pulling image "busybox777"
  Warning  Failed    ...   kubelet     Failed to pull image "busybox777": pull access denied, repository does not exist or may require authorization...
```

> 💡 Kubernetes is unable to pull the image "busybox777" because it either does not exist or you might need to provide proper credentials.

---

## 6. Delete a Pod and Observe ReplicaSet Self-Healing

To demonstrate the self-healing nature of ReplicaSets, delete one of the pods. First, list all pods:

```bash theme={null}
kubectl get pods
```

You may see output such as:

```plaintext theme={null}
NAME                           READY   STATUS             RESTARTS   AGE
new-replica-set-wkzjh         0/1     ImagePullBackOff   0          2m59s
new-replica-set-vpkh8         0/1     ImagePullBackOff   0          2m59s
new-replica-set-7r2qw         0/1     ImagePullBackOff   0          2m59s
new-replica-set-tn2mp         0/1     ImagePullBackOff   0          2m59s
```

Delete one pod:

```bash theme={null}
kubectl delete pod new-replica-set-wkzjh
```

The system confirms deletion:

```plaintext theme={null}
pod "new-replica-set-wkzjh" deleted
```

After deletion, list the pods again:

```bash theme={null}
kubectl get pods
```

A new pod will be created automatically by the ReplicaSet to maintain the desired count. Notice that the new pod has a different name and age compared to the existing ones.

---

## 7. Create a ReplicaSet Using a Definition File

Now, learn how to create a ReplicaSet from a YAML definition file. First, confirm the existence of the file:

```bash theme={null}
ls /root
```

Assuming you see a file named `ReplicaSet definition.yaml`, attempt to create the ReplicaSet:

```bash theme={null}
kubectl create -f /root/ReplicaSet\ definition.yaml
```

If an error similar to the following is returned:

```plaintext theme={null}
no matches for kind "ReplicaSet" in version "v1"
```

This indicates that the API version in the file is incorrect. The proper API version for ReplicaSets is `apps/v1`.

> 💡 Use the command `kubectl explain replicaset` to verify the correct API version.

Update the file with the correct API version and create it again.

---

## 8. Correct a Second ReplicaSet Definition File

Attempt to create a second ReplicaSet using another definition file:

```bash theme={null}
kubectl create -f replicaset-definition-2.yaml
```

If you encounter this error:

```plaintext theme={null}
The ReplicaSet "replicaset-2" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"tier":"nginx"}: selector does not match template labels
```

Open the file for editing:

```bash theme={null}
vi replicaset-definition-2.yaml
```

Examine the `spec.selector.matchLabels` and `template.metadata.labels` sections. They must match exactly. For example:

```yaml theme={null}
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-2
spec:
  replicas: 2
  selector:
    matchLabels:
      tier: nginx
  template:
    metadata:
      labels:
        tier: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

After ensuring both fields match, save the file and run:

```bash theme={null}
kubectl create -f replicaset-definition-2.yaml
```

Verify the creation of all ReplicaSets:

```bash theme={null}
kubectl get rs
```

Expected output:

```plaintext theme={null}
NAME              DESIRED   CURRENT   READY   AGE
new-replica-set   4         4         0       10m
replicaset-1      2         2         2       3m4s
replicaset-2      2         2         2       22s
```

If necessary, delete the extra ReplicaSets:

```bash theme={null}
kubectl delete rs replicaset-1
kubectl delete rs replicaset-2
```

---

## 9. Update the Original ReplicaSet Image

The original ReplicaSet is still configured to use `busybox777`. To update the image to `busybox`, edit the ReplicaSet:

```bash theme={null}
kubectl edit rs new-replica-set
```

Locate the container section and change:

```yaml theme={null}
image: busybox777
```

to

```yaml theme={null}
image: busybox
```

Save the changes and exit the editor. Bear in mind that updating the ReplicaSet does not automatically update the running pods; you must delete the existing pods so the ReplicaSet can recreate them with the correct image.

List the pods:

```bash theme={null}
kubectl get pods
```

Then, delete the current pods:

```bash theme={null}
kubectl delete pod new-replica-set-vpkh8 new-replica-set-tn2mp new-replica-set-7r2qw new-replica-set-hcmbw
```

After deletion, new pods will be created. Verify that they are transitioning from "ContainerCreating" to "Running":

```bash theme={null}
kubectl get pods
```

---

## 10. Scaling the ReplicaSet

### Scale Up to 5 Pods

Increase the ReplicaSet to five pods with the scale command:

```bash theme={null}
kubectl scale rs new-replica-set --replicas=5
```

Verify the scale-up by listing the pods again:

```bash theme={null}
kubectl get pods
```

### Scale Down to 2 Pods

To reduce the number of pods to two, edit the ReplicaSet:

```bash theme={null}
kubectl edit rs new-replica-set
```

Update the `spec.replicas` field to 2 and save the file. The ReplicaSet will automatically adjust the pod count accordingly.

---

For more details on Kubernetes concepts, visit the [Kubernetes Documentation](https://kubernetes.io/docs/).
