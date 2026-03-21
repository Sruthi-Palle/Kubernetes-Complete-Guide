# Role Based Access Controls

> 💡 This article explains how to implement Role-Based Access Controls in Kubernetes, including creating roles, role bindings, and verifying permissions.

## Creating a Role

To define a role, create a YAML file that sets the API version to `rbac.authorization.k8s.io/v1` and the kind to `Role`. In this example, we create a role named **developer** to grant developers specific permissions. The role includes a list of rules where each rule specifies the API groups, resources, and allowed verbs. For resources in the core API group, provide an empty string (`""`) for the `apiGroups` field.

For instance, the following YAML definition grants developers permissions on pods (with various actions) and allows them to create ConfigMaps:

```yaml theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["ConfigMap"]
    verbs: ["create"]
```

Create the role by running:

```bash theme={null}
kubectl create -f developer-role.yaml
```

> 💡 Both roles and role bindings are namespace-scoped. This example assumes usage within the default namespace. To manage access in a different namespace, update the YAML metadata accordingly.

## Creating a Role Binding

After defining a role, you need to bind it to a user. A role binding links a user to a role within a specific namespace. In this example, we create a role binding named **devuser-developer-binding** that grants the user `dev-user` the **developer** role.

Below is the combined YAML definition for both creating the role and its corresponding binding:

```yaml theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "update", "delete"]
  - apiGroups: [""]
    resources: ["ConfigMap"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Create the role binding using the command:

```bash theme={null}
kubectl create -f devuser-developer-binding.yaml
```

## Verifying Roles and Role Bindings

After applying your configurations, it's important to verify that the roles and role bindings have been created correctly.

To list all roles in the current namespace, execute:

```bash theme={null}
kubectl get roles
```

Example output:

```bash theme={null}
NAME        AGE
developer   4s
```

Next, list all role bindings:

```bash theme={null}
kubectl get rolebindings
```

Example output:

```bash theme={null}
NAME                      AGE
devuser-developer-binding 24s
```

For detailed information about the **developer** role, run:

```bash theme={null}
kubectl describe role developer
```

Sample output:

```bash theme={null}
Name:         developer
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources           Non-Resource URLs   Resource Names   Verbs
  -----------         ------------------   --------------   ----
  ConfigMap           []                   []               [create]
  pods                []                   []               [get watch list create delete]
```

To view the specifics of the role binding:

```bash theme={null}
kubectl describe rolebinding devuser-developer-binding
```

Example output:

```bash theme={null}
Name:         devuser-developer-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:    Role
  Name:    developer
Subjects:
  Kind     Name      Namespace
  ----     ----      ---------
  User     dev-user
```

## Testing Permissions with kubectl auth

As a user, You can test whether you have the necessary permissions to perform specific actions by using the `kubectl auth can-i` command. For example, to check if you can create deployments, run:

```bash theme={null}
kubectl auth can-i create deployments
```

This command might return:

```bash theme={null}
yes
```

Similarly, to verify if you can delete nodes:

```bash theme={null}
kubectl auth can-i delete nodes
```

Expected output:

```bash theme={null}
no
```

As a Administrator, To test permissions for a specific user without switching user contexts, use the `--as` flag. Although the **developer** role does not permit creating deployments, it does allow creating pods:

```bash theme={null}
kubectl auth can-i create deployments
# Output: yes
kubectl auth can-i delete nodes
# Output: no
kubectl auth can-i create deployments --as dev-user
# Output: no
kubectl auth can-i create pods --as dev-user
# Output: yes
```

You can also specify a namespace in your commands to verify permissions scoped to that particular namespace.

```bash theme={null}
kubectl auth can-i create pods --as dev-user --namespace test
```

## Limiting Access to Specific Resources

In some scenarios, you may want to restrict user access to a select group of resources. For example, if you have multiple pods in a namespace but only intend to provide access to pods named "blue" and "orange," you can utilize the `resourceNames` field in the role rule.

Start with a basic role definition without any resource-specific restrictions:

```yaml theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "create", "update"]
```

Then, update the rule to restrict access solely to the "blue" and "orange" pods:

```yaml theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "create", "update"]
    resourceNames: ["blue", "orange"]
```

### Example Scenarios

#### 1. Find which authorization modes enabled

Inspecting the API Server Configuration
Start by inspecting the kube-apiserver manifest to identify the configured authorization modes. In the manifest snippet below, note the use of “Node,RBAC” for the authorization mode:
command:

```bash theme={null}
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

```yaml theme={null}
creationTimestamp: null
labels:
  component: kube-apiserver
  tier: control-plane
name: kube-apiserver
namespace: kube-system
spec:
  containers:
    - command:
        - kube-apiserver
        - --advertise-address=10.48.174.6
        - --allow-privileged=true
        - --authorization-mode=Node,RBAC
        - --client-ca-file=/etc/kubernetes/pki/ca.crt
        - --enable-admission-plugins=NodeRestriction
        - --enable-bootstrap-token-auth=true
        - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        - --etcd-servers=https://127.0.0.1:2379
        - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
        - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
        - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
        - --requestheader-allowed-names=front-proxy-client
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-username-headers=X-Remote-User
        - --secure-port=6443
        - --service-account-issuer=https://kubernetes.default.svc.cluster.local
        - --service-account-key-file=/etc/kubernetes/pki/sa.pub
        - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
  image: k8s.gcr.io/kube-apiserver:v1.23.0
  imagePullPolicy: IfNotPresent
  livenessProbe:
```

An alternative verification method is to inspect the running processes on the control plane. For instance, execute the following command:

```bash theme={null}
root@controlplane ~ ps aux | grep authorization
root      3403  0.0  0.0  830588 115420 ?        Ssl  22:54   0:55 kube-controller-manager --allocate-node-authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --bind-address=127.0.0.1 --client-ca-file=/etc/kubernetes/pki/ca.crt --cluster-cidr=10.244.0.0/16 --kubelet-client-certificate=/etc/kubernetes/pki/ca.key --controllers=*,bootstrapsigner,tokens --kubelet-client-key=/etc/kubernetes/pki/front-proxy-client.key --service-account-privileged=false --service-cluster-ip-range=10.96.0.0/12 --use-service-account-credentials=true
root      3614  0.0  0.0  759136  55292 ?        Ssl  22:55   0:10 kube-scheduler --authentication-kubeconfig=/etc/kubernetes/scheduler.conf --bind-address=127.0.0.1 --kubeconfig=/etc/kubernetes/scheduler.conf --leader-elect=true
root      3630  0.0  0.1 111896  316984 ?        S    22:55   2:07 kube-apiserver --advertise-address=10.4.8.174.6 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=X-Remote-Group --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-client-ca.crt --requestheader-headers-prefix=X-Remote-Extra --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
root      25283  0.0  0.0  13444  1068 pts/0    S+   23:40   0:00 grep --color=auto authorization
```

The output confirms that the kube-apiserver is running with the “Node,RBAC” authorization mode.

#### 2. To count the roles across all namespaces, use this command:

```bash theme={null}

root@controlplane ~ k get roles -A --no-headers | wc -l

12

```

> 💡 Always verify RBAC changes by testing user permissions in the designated namespace to ensure that the intended access is provided while maintaining security.

For additional details on Kubernetes RBAC, refer to the [Kubernetes Documentation](https://kubernetes.io/docs/) and explore best practices for securing your clusters.
