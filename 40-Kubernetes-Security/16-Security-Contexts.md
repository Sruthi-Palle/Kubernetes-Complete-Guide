# Security Contexts

> 💡 This article explains how to secure containers in Kubernetes by configuring security settings at the pod and container levels.

When you run a Docker container, you have the flexibility to define various security parameters. For example, you can set a specific user ID or modify Linux capabilities. Consider the following Docker commands:

```bash theme={null}
docker run --user=1001 ubuntu sleep 3600
docker run --cap-add MAC_ADMIN ubuntu
```

These configurations help manage the security of Docker containers, and similar settings are available in Kubernetes.

In Kubernetes, containers are always encapsulated in pods. You can define security settings either at the pod level, which affects all containers in the pod, or at the container level where the settings apply specifically to one container. Note that if the same security configuration is set at both the pod and container levels, the container-specific settings take precedence over the pod-level configurations.

Below is an example of a pod definition that configures the security context at the container level, using an Ubuntu image that runs the `sleep` command:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```

This configuration instructs Kubernetes to run the container as user ID 1000 and adds the `MAC_ADMIN` capability. If your goal is to enforce these security settings across all containers within a single pod, you can define the security context at the pod level instead.

By leveraging security contexts, you enhance the security posture of your containerized applications in Kubernetes. For additional guidance, you may find the following resources useful:

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes Concepts: Security Contexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

---

# Practical Examples Security Contexts

## Checking the User of a Running Pod

To determine which user is executing the sleep process inside an Ubuntu sleeper pod, follow these steps:

1. **List the Pods**\
   Identify the Ubuntu sleeper pod by listing all pods:

   ```bash theme={null}
   kubectl get pods
   ```

2. **Access the Pod**\
   Use `kubectl exec` to access the pod's shell:

   ```bash theme={null}
   kubectl exec -it ubuntu-sleeper -- bash
   ```

3. **Verify the User Inside the Container**\
   Once inside, run the command:

   ```bash theme={null}
   whoami
   ```

   The expected output should be:

   ```bash theme={null}
   root
   ```

Below is a sample console output demonstrating these steps:

```bash theme={null}
kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
ubuntu-sleeper   1/1     Running   0          7m58s
```

And, inside the container:

```bash theme={null}
whoami
root
```

This confirms that the sleep process is running as the root user.

---

## Updating the Pod to Run as a Specific Non-Root User (UID 1010)

To update the Ubuntu sleeper pod so that the process runs with the user ID 1010, follow these steps:

1. **Export the Current Pod Configuration**\
   Retrieve the current configuration and save it to a file:

   ```bash theme={null}
   kubectl get pod ubuntu-sleeper -o yaml > ubuntu-sleeper.yaml
   ```

2. **Edit the Configuration File**\
   Open `ubuntu-sleeper.yaml` in your favorite text editor. Locate the pod's `securityContext`, which should currently look like:

   ```yaml theme={null}
   securityContext: {}
   ```

3. **Modify the Security Context**\
   Update the file by adding the `runAsUser` property with UID 1010:

   ```yaml theme={null}
   securityContext:
     runAsUser: 1010
   ```

4. **Apply the Updated Configuration**\
   Save the changes, then delete the existing pod (forcing deletion if necessary) and reapply the configuration:

   ```bash theme={null}
   kubectl delete pod ubuntu-sleeper --force
   kubectl apply -f ubuntu-sleeper.yaml
   ```

An excerpt of the modified configuration is as follows:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
  namespace: default
spec:
  containers:
    - command:
        - sleep
        - "4800"
      image: ubuntu
      imagePullPolicy: Always
      name: ubuntu
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-vwn5z
          readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: controlplane
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext:
    runAsUser: 1010
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
    - effect: NoExecute
      key: node.kubernetes.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: node.kubernetes.io/unreachable
      operator: Exists
```

After applying these changes, you can verify that the pod is running:

```bash theme={null}
kubectl get pods
```

> At the container level, the `securityContext` settings override those defined at the pod level.

---

## Determining the User for Multiple Containers in a Pod

When dealing with pods that contain multiple containers, it's important to understand how user contexts are inherited. Consider the following pod definition from the `multi-pod.yaml` file:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  securityContext:
    runAsUser: 1001
  containers:
    - image: ubuntu
      name: web
      command: ["sleep", "5000"]
      securityContext:
        runAsUser: 1002
    - image: ubuntu
      name: sidecar
      command: ["sleep", "5000"]
```

Key observations:

- The pod-level security context sets `runAsUser` to 1001.
- The web container defines its own `securityContext` with `runAsUser: 1002`.

Since the container-level security context takes precedence, the processes in the **web** container run as user **1002**. The **sidecar** container, which lacks a container-specific security context, inherits the pod-level setting, so its processes run as user **1001**.

The final configuration is provided below for reference:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  securityContext:
    runAsUser: 1001
  containers:
    - image: ubuntu
      name: web
      command: ["sleep", "5000"]
      securityContext:
        runAsUser: 1002
    - image: ubuntu
      name: sidecar
      command: ["sleep", "5000"]
```

---

## Updating the Ubuntu Sleeper Pod to Run as Root with SYS_TIME Capability

To modify the Ubuntu sleeper pod so that:

1. The process runs as the default root user.
2. The pod is granted the `SYS_TIME` capability,

follow these steps:

1. **Edit the Pod Configuration**\
   Open the current configuration file (e.g., `ubuntu-sleeper.yaml`), and remove any pod-level `securityContext` that enforces a non-root user.

2. **Add Capabilities at the Container Level**\
   Under the container specification responsible for running the sleep command, add a `securityContext` that includes the required capability:

   ```yaml theme={null}
   securityContext:
     capabilities:
       add: ["SYS_TIME"]
   ```

A sample updated configuration is shown below:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-07-26T22:12:07Z"
  name: ubuntu-sleeper
  namespace: default
  resourceVersion: "945"
  uid: 34c800d-d278-498b-b6be-f9e41086be9f
spec:
  containers:
    - command:
        - sleep
        - "4800"
      image: ubuntu
      imagePullPolicy: Always
      name: ubuntu
      securityContext:
        capabilities:
          add: ["SYS_TIME"]
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-km279
          readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: controlplane
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
    - effect: NoExecute
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
```

After saving the configuration, delete the existing pod and apply the new configuration:

```bash theme={null}
kubectl delete pod ubuntu-sleeper --force
kubectl apply -f ubuntu-sleeper.yaml
```

Verify that the pod is running as expected:

```bash theme={null}
kubectl get pods
```

---

## Adding the NET_ADMIN Capability

To enhance the security context further by adding the `NET_ADMIN` capability in addition to `SYS_TIME`, follow these steps:

1. **Modify the Container’s Security Context**\
   Open the `ubuntu-sleeper.yaml` file and update the `securityContext` under the container section to include both capabilities:

   ```yaml theme={null}
   securityContext:
     capabilities:
       add: ["SYS_TIME", "NET_ADMIN"]
   ```

2. **Apply the Changes**\
   Save the file and reapply the pod configuration by deleting the current pod and applying the updated configuration:

   ```bash theme={null}
   kubectl delete pod ubuntu-sleeper --force
   kubectl apply -f ubuntu-sleeper.yaml
   ```

3. **Verify the Update**\
   Optionally, check that the pod is running with the updated capabilities:

   ```bash theme={null}
   kubectl get pods
   ```

A sample console interaction may look like this:

```bash theme={null}
controlplane ~ ➜ vi ubuntu-sleeper.yaml
controlplane ~ ➜ kubectl delete pod ubuntu-sleeper --force
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "ubuntu-sleeper" force deleted

controlplane ~ ➜ kubectl apply -f ubuntu-sleeper.yaml
pod/ubuntu-sleeper created

controlplane ~ ➜ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
ubuntu-sleeper   1/1     Running   0          10s
```

This completes the guide on updating Kubernetes pod configurations with various security contexts and capabilities. For more detailed information on Kubernetes security practices, see the [Kubernetes Documentation](https://kubernetes.io/docs/).
