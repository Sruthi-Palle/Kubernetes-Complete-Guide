# Demo Static Pods

> 💡 In this article, we will work through a practical examples on static pods. You will learn how to identify static pods, examine their configuration, create a new static pod, modify its image, and finally remove a static pod from a node.

---

## Identifying Static Pods in the Cluster

Start by determining the number of static pods across all namespaces. List all pods with this command:

```bash theme={null}
kubectl get pods -A
```

Static pods can be recognized by their naming convention; they often include the node name appended to the pod name. For example, a pod named `kube-apiserver-controlplane` (with "controlplane" appended) indicates that it is a static pod. In our cluster output, there are four static pods on the control plane, while other pods do not have a node name at the end.

Another way to verify if a pod is static is to inspect its YAML configuration. For example, check the Kube API Server pod with:

```bash theme={null}
kubectl get pod kube-apiserver-controlplane -n kube-system -o yaml
```

Scroll down to the `ownerReferences` section. A static pod will have an owner entry like this:

- `kind: Node`
- `name: controlplane`

This confirms that the pod is managed as a static pod. Below is a representative snippet from the YAML output:

```yaml theme={null}

...
ownerReferences:
  - apiVersion: v1
    controller: true
    kind: Node
    name: controlplane
    uid: fbab9414-1ae8-4f64-bc94-456dd8569e6c
...
```

In contrast, inspecting a pod that is not a static pod—such as one of the CoreDNS pods—will show an owner reference listing a ReplicaSet. For example:

```bash theme={null}
kubectl get pod coreDNS-74ff55c5b-brvnd -n kube-system -o yaml
```

Excerpt from its YAML output:

```yaml theme={null}

...
ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: coredns-74ff55c5b
    uid: c2935e74-20ff-44b4-b023-57e228a19b6d
...
```

This confirms that CoreDNS is not a static pod.

> 💡 You can use various filters and selectors with `kubectl` for a more in-depth analysis. However, these two methods will suffice to identify static pods.

---

## Listing Static Pod Components

To review the static pods currently running, execute:

```bash theme={null}
kubectl get pods -A
```

On the control plane, you will typically see static pods such as:

- etcd-controlplane
- kube-apiserver-controlplane
- kube-controller-manager-controlplane
- kube-scheduler-controlplane

Comparing this list with components like CoreDNS (managed by a ReplicaSet) or Kube Proxy (which is not a static pod) helps you distinguish static pods. For example, even though CoreDNS pods appear in the output, they are not static.

> 💡 When questioned on which nodes the static pods are running, remember that they typically reside on the control plane.

---

## Determining the Static Pod Manifest Directory

Static pod definitions are read by the kubelet from a specific manifest directory. To locate this directory, inspect the kubelet configuration file, typically found at `/var/lib/kubelet/config.yaml`. Look for the `staticPodPath` parameter:

```yaml theme={null}

...
staticPodPath: /etc/kubernetes/manifests
...
```

This directory contains the manifest files used by the kubelet to create static pods. For instance, listing the contents of the directory may reveal:

```bash theme={null}
ls /etc/kubernetes/manifests
```

Files you might see include:

- etcd.yaml
- kube-apiserver.yaml
- kube-controller-manager.yaml
- kube-scheduler.yaml

The number of manifest files should match the number of static pods running on the control plane.

---

## Examining the Kube API Server Static Pod

To inspect the Docker image used for the Kube API Server, open the manifest file:

```bash theme={null}
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Within the file, search for the `image:` line. You should encounter a line similar to:

```yaml theme={null}
image: k8s.gcr.io/kube-apiserver:v1.20.0
```

Ensure that you are referencing the correct version for your environment.

---

## Creating a New Static Pod

Next, create a new static pod named `static-busybox` using the BusyBox image, running the command `sleep 1000`. Generate a pod manifest with a dry-run to avoid immediate deployment in the cluster:

```bash theme={null}
kubectl run static-busybox --image=busybox --restart=Never --dry-run=client -o yaml > static-busybox.yaml
```

The generated manifest will resemble the following:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: static-busybox
  name: static-busybox
spec:
  containers:
    - command:
        - sleep
        - "1000"
      image: busybox
      name: static-busybox
      resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Next, move this file to the static pod manifest directory (on the control plane node, for example, `/etc/kubernetes/manifests`). The kubelet will then detect the new manifest and create the pod. Verify its creation by listing pods:

```bash theme={null}
kubectl get pods
```

You should observe a pod named something like `static-busybox-controlplane` in the Running state.

---

## Editing the Static Pod Manifest

To update the static pod with a different image version, edit the file located at `/etc/kubernetes/manifests/static-busybox.yaml` and change the image line to use `busybox:1.28.4`:

```yaml theme={null}

...
containers:
  - command:
      - sleep
      - "1000"
    image: busybox:1.28.4
    name: static-busybox
    resources: {}
...
```

After saving the changes, check the pod's status. Initially, the pod might be in a Pending state due to an image pull error. To monitor the status, use:

```bash theme={null}
kubectl get pods --watch
```

Once the changes are successful, the pod will transition to Running. This step underlines the importance of immediately validating any edits made to static pod manifests.

> 💡 Always monitor your pod status after making changes to ensure that the updates have been applied correctly.

---

## Deleting a Static Pod

Deleting a static pod managed by the kubelet cannot be achieved permanently by merely running a delete command. For example, executing:

```bash theme={null}
kubectl delete pod static-greenbox-node01
```

will only remove the pod temporarily. The kubelet will recreate it since its manifest file remains present.

To permanently remove the static pod named `static-greenbox-node01`, follow these steps:

1. SSH into node01 (use the internal IP if necessary).

2. Check the manifest directory on node01. Note that the static pod manifest directory might be custom configured. For example, the kubelet configuration on node01 (located at `/var/lib/kubelet/config.yaml`) could specify:

   ```yaml theme={null}
   staticPodPath: /etc/just-to-mess-with-you
   ```

3. List the manifest files on node01:

   ```bash theme={null}
   ls /etc/just-to-mess-with-you
   ```

   You should see a file named `greenbox.yaml`.

4. Remove the manifest file with:

   ```bash theme={null}
   rm /etc/just-to-mess-with-you/greenbox.yaml
   ```

After deleting the file, the kubelet will detect the change and terminate the `static-greenbox-node01` pod. Monitor the change by watching the pods:

```bash theme={null}
kubectl get pods --watch
```

Eventually, the pod will be terminated and will no longer appear in the list.

> 💡 Deleting the manifest file is the only way to permanently remove a static pod. Simply deleting the pod using `kubectl delete pod` will result in its immediate recreation by the kubelet.

---

## Summary

In this article, you have accomplished the following:

- Identified static pods by their naming conventions and owner references.
- Determined the static pod manifest directory from the kubelet configuration.
- Examined the manifest details, including the Docker image tag, for the Kube API Server.
- Created a new static pod using a dry-run to generate the YAML manifest and placed it in the manifests directory.
- Edited the static pod manifest to update the image version and verified the changes.
- Learned how to permanently delete a static pod by removing its manifest file from the node.

## K8s Reference docs

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes Basics](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
