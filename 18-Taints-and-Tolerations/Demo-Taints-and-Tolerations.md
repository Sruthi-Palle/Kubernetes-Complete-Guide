# Demo Taints and Toleration

> 💡 This guide explains how to manage taints and tolerations in Kubernetes for controlling pod scheduling. We start by inspecting the cluster nodes, then proceed to apply a taint to a node and create pods with and without the appropriate tolerations.

---

## Step 1: Count the Nodes

Begin by verifying the total number of nodes (including the control plane) in your cluster. Run the following command:

```bash theme={null}
root@controlplane:~# kubectl get nodes
NAME           STATUS   ROLES                    AGE   VERSION
controlplane   Ready    control_plane,master     17m   v1.20.0
node01         Ready    <none>                   16m   v1.20.0
root@controlplane:~#
```

There are two nodes in the cluster.

> 💡 This output confirms that both the control plane and node01 are active and ready.

---

## Step 2: Check Taints on node01

Next, examine node01 for any existing taints. Use the `kubectl describe` command:

```bash theme={null}
root@controlplane:~# kubectl describe node node01
Name:               node01
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=node01
                    kubernetes.io/os=linux
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"8e:62:74:26:35:47"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.58.27.11
                    kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Fri, 15 Apr 2022 22:57:19 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:   node01
  AcquireTime:      <unset>
  RenewTime:        Fri, 15 Apr 2022 23:14:20 +0000
Conditions:
  Type                    Status   LastHeartbeatTime                  LastTransitionTime                 Reason
  ----                    ------   -----------------                  ---------------------             ----------------
  NetworkUnavailable      False    Fri, 15 Apr 2022 22:57:25 +0000   Fri, 15 Apr 2022 22:57:25 +0000   Flannel is running on this node
  MemoryPressure          False    Fri, 15 Apr 2022 23:12:35 +0000   Kubelet has sufficient memory available
  DiskPressure            False    Fri, 15 Apr 2022 23:11:20 +0000   Kubelet has no disk pressure
...
root@controlplane:~#
```

Since there are no taints on node01, you can conclude that there are no scheduling constraints for pods on this node at this moment.

---

## Step 3: Apply a Taint on node01

Now, add a taint to node01 by specifying a key-value pair and an effect. The command below applies a taint with the key "app", a value of "blue", and an effect of "NoSchedule":

```bash theme={null}
kubectl taint node node01 app=blue:NoSchedule
```

This taint ensures that only pods with a matching toleration will be scheduled on node01.

---

## Step 4: Create the "nginx" Pod Without a Toleration

Create a pod named "nginx" using the nginx image without specifying any toleration:

```bash theme={null}
root@controlplane:~# kubectl run nginx --image=nginx
```

After creating the pod, check its status:

```bash theme={null}
root@controlplane:~# kubectl get pods
NAME        READY   STATUS              RESTARTS   AGE
nginx    0/1     Pending             0          3m37s
root@controlplane:~#
```

Since the pod lacks a toleration for the taint applied on node01, it remains in a pending state. To investigate further, describe the pod:

```bash theme={null}
root@controlplane:~# kubectl describe pod nginx
Name:           nginx
Namespace:      default
Priority:       0
Node:           <none>
Labels:         run=nginx
Status:         Pending
Containers:
  nginx:
    Image:      nginx
    Port:       <none>
    Host Port:  <none>
...
Tolerations:  node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
              node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Warning  FailedScheduling  45s (x2 over 45s)   default-scheduler  0/2 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 1 node(s) had taint {app: blue:NoSchedule}, that the pod didn't tolerate.
...
root@controlplane:~#
```

The event message clarifies that "nginx" cannot be scheduled due to the untolerated "app" taint on node01.

> 💡 Ensure that you add the proper tolerations when you need a pod to be scheduled on a tainted node.

---

## Step 5: Create the "blueapp" Pod With a Toleration

To schedule a pod on node01 despite the taint, create a new pod named "blueapp" with a toleration for "app". Follow these steps:

1. Generate an initial YAML manifest using dry-run:

   ```bash theme={null}
   kubectl run blueapp --image=nginx --dry-run=client -o yaml > blueapp.yaml
   ```

2. Open the generated `blueapp.yaml` file and add a `tolerations` section under the spec. The corrected YAML should resemble the following:

   ```yaml theme={null}
   apiVersion: v1
   kind: Pod
   metadata:
     name: blueapp
     labels:
       run: blueapp
   spec:
     containers:
       - name: blueapp
         image: nginx
         resources: {}
     dnsPolicy: ClusterFirst
     restartPolicy: Always
     tolerations:
       - key: "app"
         operator: "Equal"
         value: "blue"
         effect: "NoSchedule"
   ```

3. Apply the manifest to create the pod:

   ```bash theme={null}
   kubectl apply -f blueapp.yaml
   ```

4. Monitor the pod creation:

   ```bash theme={null}
   kubectl get pods --watch
   ```

After a few seconds, you should see the "blueapp" pod transition to the Running state. Verify the status by running:

```bash theme={null}
root@controlplane:~# kubectl get pods -o wide
NAME       READY   STATUS              RESTARTS   AGE     IP           NODE        NOMINATED NODE   READINESS GATES
blueapp        1/1     Running             0          36s     10.244.1.2   node01      <none>           <none>
nginx   0/1     Pending             0          4m6s    <none>       <none>           <none>
root@controlplane:~#
```

The "blueapp" pod successfully schedules on node01 because it contains the correct toleration, while "nginx" continues to remain pending.

---

## Step 6: Remove the Taint from the Control Plane

Finally, inspect the control plane node to confirm it has a taint restricting regular pod scheduling:

```bash theme={null}
root@controlplane:~# kubectl describe node controlplane
Name:               controlplane
Roles:              control-plane,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/hostname=controlplane
                    node-role.kubernetes.io/control-plane=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"f6:6e:ba:7d:23:ca"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    kubeadm.alpha.coreos.com/cri-socket: /var/run/dockershim.sock
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Fri, 15 Apr 2022 22:56:44 +0000
Taints:             node-role.kubernetes.io/master:NoSchedule
Unschedulable:      false
...
root@controlplane:~#
```

To allow pods to be scheduled on the control plane, remove its taint with the following command:

```bash theme={null}
kubectl taint node controlplane node-role.kubernetes.io/master:NoSchedule-
```

Once the taint is removed, the "nginx" pod can now find a suitable node. Confirm the new pod status:

```bash theme={null}
root@controlplane:~# kubectl get pods -o wide
NAME       READY   STATUS    RESTARTS   AGE     IP           NODE         NOMINATED NODE   READINESS GATES
blueapp        1/1     Running   0          2m52s   10.244.1.2   node01       <none>           <none>
nginx   1/1     Running   0          6m22s   10.244.0.4   controlplane <none>           <none>
root@controlplane:~#
```

Now, "nginx" is running on the control plane since the scheduling conflict has been resolved.

---

## Summary

This walkthrough demonstrates the use of taints and tolerations to control pod placement in a Kubernetes cluster:

- Initially, node01 had a taint (`app=blue:NoSchedule`) and the control plane had the default master taint, which prevented the "nginx" pod from scheduling.
- Creating the "blueapp" pod with the appropriate toleration allowed it to be scheduled on node01.
- Removing the taint from the control plane enabled the "nginx" pod to be scheduled there.

Using taints and tolerations effectively can help you control where pods are deployed and maintain a balanced and secure Kubernetes environment.
