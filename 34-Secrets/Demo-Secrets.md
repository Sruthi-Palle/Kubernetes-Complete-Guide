# Demo Secrets Optional

> 💡 Learn to work with Kubernetes Secrets by inspecting existing ones, creating a custom secret, and configuring a web application pod to use it.

In this demo, you'll learn how to work with Kubernetes Secrets. We'll start by inspecting the default Namespace's secrets, then create a custom secret for database credentials and configure a web application pod to use it.

---

## Inspecting Existing Secrets

### Question 1: How many Secrets exist in the default Namespace?

Run the following command:

```bash theme={null}
kubectl get secrets
```

You'll see output similar to:

```text theme={null}
NAME                  TYPE                                   DATA   AGE
default-token-cr4sr   kubernetes.io/service-account-token      3      7m50s
```

This indicates that only one secret exists in the default Namespace.

---

### Question 2: How many pieces of secret data are defined in the default token secret?

Inspect the details by running:

```bash theme={null}
kubectl describe secret default-token-cr4sr
```

The output shows:

```text theme={null}
Name:         default-token-cr4sr
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 2751dc84-936d-4b41-865d-847a99a2b2b

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:      570 bytes
namespace:   7 bytes
token:       eyJhbGciOiJUUzI1NiIsImtpZCI6ImEyREFqZBTXdsZ2lL...k0pIHRhS
```

This secret contains three key data fields:

- ca.crt
- namespace
- token

Thus, there are three pieces of secret data defined.

---

### Question 3: What is the type of the default token secret?

To confirm the secret type, first list the secrets:

```bash theme={null}
kubectl get secrets
```

```text theme={null}
NAME                  TYPE                                   DATA   AGE
default-token-cr4sr   kubernetes.io/service-account-token      3      7m50s
```

Then describe the secret:

```bash theme={null}
kubectl describe secret default-token-cr4sr
```

The output reiterates:

```text theme={null}
Name:         default-token-cr4sr
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 2751dc84-936d-4b41-86d5-847a99a2b2b

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:      570 bytes
namespace:   7 bytes
token:       eyJhbGciOiJSUzI1NiIsImtpZCI6ImEyZjM0ZjE0Y2R...iYjI4Njk4hB3aB2Y2E8Z2Y4ZSRjbLW
```

The secret type is clearly "kubernetes.io/service-account-token". Remember, the "namespace" listed in the secret data is not the secret type.

---

## Deploying an Application with Secrets

The application deployment follows a specific architecture where required pods and services are already running. Verify their current state by running:

```bash theme={null}
kubectl get secrets
```

```text theme={null}
NAME                  TYPE                                   DATA   AGE
default-token-cr4sr   kubernetes.io/service-account-token      3      7m50s
```

Then, check the secret details:

```bash theme={null}
kubectl describe secret default-token-cr4sr
```

```text theme={null}
Name:         default-token-cr4sr
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 2751dc84-936d-4b41-865d-847a99a2b2b

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     570 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjI6j... (truncated for brevity)
```

Next, review the pods and services:

- **List Deployments:**

  ```bash theme={null}
  kubectl get deploy
  ```

  ```text theme={null}
  No resources found in default namespace.
  ```

- **List Pods:**

  ```bash theme={null}
  kubectl get pods
  ```

  ```text theme={null}
  NAME          READY   STATUS    RESTARTS   AGE
  webapp-pod    1/1     Running   0          26s
  mysql         1/1     Running   0          26s
  ```

- **List Services:**

  ```bash theme={null}
  kubectl get svc
  ```

  ```text theme={null}
  NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
  kubernetes        ClusterIP   10.43.0.1       <none>        443/TCP           10m
  webapp-service    NodePort    10.43.73.93     <none>        8080:30000/TCP    40s
  sql01             ClusterIP   10.43.128.20    <none>        3306/TCP          40s
  ```

Note that the default token secret is not used by the web application. It is instead intended to enable the web application to connect to the MySQL database.

---

## Application Error and the Need for a New Secret

The web application is failing to connect to the MySQL database. The error message is:

```text theme={null}
Environment Variables: DB_Host=Not Set; DB_Database=Not Set; DB_User=Not Set; DB_Password=Not Set; 2003: Can't connect to MySQL server on 'localhost:3306' (111 Connection refused)
```

> 💡 This error indicates that the secret containing the database credentials has not been created. To resolve this, we need to create a new secret named "db-secret" with the necessary data fields.

Data required for the new secret:

- DB_Host
- DB_User
- DB_Password

---

## Creating the DB Secret

Before creating the new secret, check the help documentation:

```bash theme={null}
kubectl create secret --help
```

You'll see that subcommands include:

- docker-registry
- generic
- tls

Since you're creating a generic secret, the syntax is:

```bash theme={null}
kubectl create secret generic NAME [--type=string] [--from-file=key=source] [--from-literal=key1=value1] [--dry-run=server|client|none] [options]
```

For this lab, create the secret with:

```bash theme={null}
kubectl create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123
```

Verify the creation by listing the secrets:

```bash theme={null}
kubectl get secret
```

Expected output:

```text theme={null}
NAME                  TYPE                                  DATA      AGE
default-token-cr4sr   kubernetes.io/service-account-token     3         12m
db-secret             Opaque                                3         6s
```

And review the secret details:

```bash theme={null}
kubectl describe secret db-secret
```

```text theme={null}
Name:         db-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type: Opaque

Data
====
DB_Password: 11 bytes
DB_User:     4 bytes
DB_Host:     5 bytes
```

---

## Configuring the Web Application Pod to Use the New Secret

At present, the web application pod (webapp-pod) does not load the environment variables from the new "db-secret." To pass these variables into the pod, update its configuration by referencing the secret.

### Example Pod Configuration

Below is an example configuration that demonstrates how to load environment variables from a secret:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: ["/bin/sh", "-c", "env"]
      envFrom:
        - secretRef:
            name: db-secret
  restartPolicy: Never
```

For the existing web application pod, find the container section in its pod definition and add an entry under `envFrom` to reference `db-secret`. For example:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    name: webapp-pod
spec:
  containers:
    - name: webapp
      image: kodekloud/simple-webapp-mysql
      imagePullPolicy: Always
      envFrom:
        - secretRef:
            name: db-secret
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
  volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-dxllf
      readOnly: true
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
```

After modifying the configuration, update the pod:

```bash theme={null}
kubectl replace --force -f <your-updated-pod-definition.yaml>
```

Once the pod has been recreated, verify that the environment variables are correctly loaded by checking:

```text theme={null}
Environment Variables: DB_Host=sql01; DB_Database=Not Set; DB_User=root; DB_Password=password123;
```

This confirms that the web application is now receiving the required database credentials.

---

## Reviewing Best Practices for Using Secrets

Using secrets to store sensitive data such as database credentials is a common practice. However, by default, these secrets are stored in etcd without encryption, potentially leaving them exposed to anyone with access to the Kubernetes API server or the etcd database.

> 💡 For enhanced security, consider enabling encryption at rest and proper role-based access controls (RBAC) to protect your secrets.

For more details on safely managing your secrets, refer to the [Kubernetes Documentation on Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

---

## Conclusion

In this lab, you learned how to:

1. Inspect the default service account secret in Kubernetes.
2. Create a new generic secret named "db-secret" to store database credentials.
3. Configure a web application pod to load environment variables from the newly created secret.
4. Understand key security considerations when working with Kubernetes Secrets.

This concludes the lab on managing Kubernetes Secrets. Happy deploying!
