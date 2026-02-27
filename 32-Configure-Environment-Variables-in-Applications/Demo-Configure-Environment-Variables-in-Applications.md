# Demo Env Variables

> 💡 Learn to manage environment variables in Kubernetes pods and update them using direct modifications and ConfigMaps for dynamic configuration.

Follow along to understand how to check running pods, update environment variables, and integrate ConfigMaps for dynamic configuration.

## Checking the Running Pod

First, check how many pods are currently running. In this example, we're working with a single pod named "webapp-color":

```bash theme={null}
k get pods
```

Output:

```bash theme={null}
NAME            READY   STATUS    RESTARTS   AGE
webapp-color    1/1     Running   0          16s
```

Next, inspect the pod details to identify the configured environment variables. Look for the `APP_COLOR` variable, which in this case is set to `pink`:

```bash theme={null}
Image ID: docker.io/kodekloud/webapp-color@sha256:99c3821ea49b89c7a22d3eebabab5c2e1ec651452e7675
Port: Host Port: <none>
State: Running
Started: Sat, 16 Apr 2022 22:49:49 +0000
Ready: True
Restart Count: 0
Environment:
  APP_COLOR: pink
...
```

When you access the web application via the provided link, you'll notice that the background color is pink.

## Updating the Pod to Change the Environment Variable

To change the background color of the web application, update the `APP_COLOR` environment variable. Start by editing the pod manifest using:

```bash theme={null}
kubectl edit pod webapp-color
```

> 💡 Pods are immutable, so while you can edit a pod's manifest, the changes cannot be applied directly. Instead, save the modified manifest locally and force replace the pod.

Below is an excerpt from the downloaded manifest where the `APP_COLOR` value is updated to `"green"`:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2022-04-16T22:49:43Z"
  labels:
    name: webapp-color
  name: webapp-color
  namespace: default
  resourceVersion: "874"
  uid: 4f24d49d-04cc-4617-88b0-8478ca19a203
spec:
  containers:
    - env:
        - name: APP_COLOR
          value: green
      image: kodekloud/webapp-color
      imagePullPolicy: Always
      name: webapp-color
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-2nbfl
          readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: controlplane
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
```

Save your changes. If you receive an error stating that editing pods is forbidden due to immutability, don't worry—the manifest is stored (e.g., in `/tmp/kubectl-edit-3135302771.yaml`). Force replace the existing pod with the updated manifest:

```bash theme={null}
kubectl replace --force -f /tmp/kubectl-edit-3135302771.yaml
```

You should see the following confirmation:

```bash theme={null}
pod "webapp-color" deleted
pod/webapp-color replaced
```

After the pod is recreated, verify that the change has taken effect:

```bash theme={null}
kubectl describe pod webapp-color
```

The output will now reflect that the environment variable `APP_COLOR` is set to `green`, and the web application's background should display green.

## Working with ConfigMaps

In addition to direct environment variable updates, Kubernetes supports managing configurations with ConfigMaps. This provides a dynamic way to update environment variables for your pods.

### Listing Existing ConfigMaps

To see the current ConfigMaps in the default namespace, run:

```bash theme={null}
kubectl get configmap
```

Output:

```bash theme={null}
NAME                     DATA   AGE
kube-root-ca.crt        1      16m
db-config               3      22s
```

### Inspecting a Specific ConfigMap

To check the database host specified in the `db-config` ConfigMap, describe it:

```bash theme={null}
kubectl describe cm db-config
```

The description will include:

```bash theme={null}
Name:         db-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
DB_HOST:
SQL01.example.com
DB_NAME:
SQL01
DB_PORT:
3306

BinaryData
==========
Events: <none>
```

This shows that the `DB_HOST` is set to `SQL01.example.com`.

### Creating a New ConfigMap for the Web Application

For the web application pod, we want to use a ConfigMap to set the environment variable dynamically. Create a new ConfigMap that sets `APP_COLOR` to `dark blue`:

```bash theme={null}
kubectl create configmap webapp-config-map --from-literal=APP_COLOR="dark blue"
```

This command creates a ConfigMap named `webapp-config-map` containing the desired environment variable.

### Updating the Pod to Use the New ConfigMap

```bash theme={null}
kubectl edit pod webapp-color
```

Next, update the pod manifest so that the web application retrieves its environment variables from the ConfigMap instead of a static declaration. Modify the manifest to remove the direct `env` key and add an `envFrom` reference:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2022-04-16T22:52:40Z"
  labels:
    name: webapp-color
  name: webapp-color
  namespace: default
  resourceVersion: "929"
  uid: f17244ec-f17f-4027-9f6c-58aa47f2cf03
spec:
  containers:
    - envFrom:
        - configMapRef:
            name: webapp-config-map
      image: kodekloud/webapp-color
      imagePullPolicy: Always
      name: webapp-color
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-2nbfl
          readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: controlplane
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
```

Save your updated manifest and force replace the pod by running:

```bash theme={null}
kubectl replace --force -f /tmp/kubectl-edit-192529677.yaml
```

You will see confirmation messages:

```bash theme={null}
pod "webapp-color" deleted
pod/webapp-color replaced
```

Finally, verify the update:

```bash theme={null}
kubectl describe pod webapp-color
```

The pod description now shows that it sources its `APP_COLOR` environment variable from the `webapp-config-map` ConfigMap. When you access the web application, the background color will have changed to dark blue.

Explore more about managing configurations in Kubernetes by visiting the [Kubernetes Documentation](https://kubernetes.io/docs/).
