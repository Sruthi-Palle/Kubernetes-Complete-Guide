# Manual Scheduling Demo

> 💡 Learn how to manually schedule a pod using a pod definition file (nginx.yaml). The steps include diagnosing a pending pod, manually assigning it to a specific node, and verifying its status.

Before creating the pod, you can verify your environment. For example, by running a command to check the cluster layout, you might see that you are working with a two-node cluster.

When you initially create the pod using the provided YAML file, the pod remains in a pending state. This happens because no scheduler has yet assigned the pod to any node. The following example demonstrates this behavior:

```bash theme={null}
pod/nginx created
root@controlplane:~# kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          12s
root@controlplane:~# kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          46s
root@controlplane:~# kubectl describe pod nginx
Name:           nginx
Namespace:      default
Priority:       0
Node:           <none>
Labels:         <none>
Annotations:    <none>
Status:         Pending
IP:             <none>
IPs:            <none>
Containers:
  nginx:
    Image:         nginx
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-kpdm6 (ro)
Volumes:
  default-token-kpdm6:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-kpdm6
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>
root@controlplane:~#
```

Notice that the Node field is `<none>`. This clearly indicates that the scheduler has not yet assigned the pod to any node. Additionally, checking the control plane components in the kube-system namespace might reveal that the scheduler pod is not running, which explains why the scheduling did not occur automatically.

---

## Manually Scheduling the Pod on a Specific Node

To manually assign the pod to node01, update your nginx.yaml file by adding the nodeName property as shown below:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: node01
  containers:
    - name: nginx
      image: nginx
```

Since the pod already exists in a pending state, you will need to delete it and recreate it. A convenient approach is to use the `kubectl replace --force` command, which deletes the existing pod and recreates it in one step:

```bash theme={null}
root@controlplane:~# vi nginx.yaml
root@controlplane:~# kubectl replace --force -f nginx.yaml
pod "nginx" deleted
pod/nginx replaced
```

You can then monitor the pod status with the following command:

```bash theme={null}
root@controlplane:~# kubectl get pods --watch
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          19s
```

The pod transitions to the Running state on node01.

---

## Scheduling the Pod on the Control Plane Node

For additional practice, you can schedule the pod on the control plane node. To do this, edit the nginx.yaml file and set the nodeName to the control plane’s name (e.g., controlplane):

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: controlplane
  containers:
    - name: nginx
      image: nginx
```

Remember that you cannot relocate a running pod from one node to another. Instead, delete the existing pod and recreate it on the desired node using the `kubectl replace --force` command:

```bash theme={null}
root@controlplane:~# vi nginx.yaml
root@controlplane:~# kubectl replace --force -f nginx.yaml
pod "nginx" deleted
pod/nginx replaced
```

Continue monitoring the pod status until it reaches the Running state:

```bash theme={null}
root@controlplane:~# kubectl get pods --watch
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          19s
```

Finally, use the wide view option to verify that the pod has been scheduled on the control plane node:

```bash theme={null}
root@controlplane:~# kubectl get pods -o wide
NAME   READY   STATUS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
nginx  1/1     Running  58s   10.244.0.4    controlplane <none>           <none>
```

---

> 💡 When manually scheduling pods, always ensure that the nodeName field is correctly set to the desired node. If the pod remains in a pending state, double-check that the targeted node is operational and available.

---

## Handling Pod Replacement and Termination

When you force replace a pod, Kubernetes sends a termination signal to the running process (in this case, the nginx process). Depending on how the process handles the termination, there might be a brief delay before the pod is fully terminated and recreated. This normal delay should not be a cause for concern.

---
