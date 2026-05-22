# KubeConfig

> 💡 This article explains kubeconfig files in Kubernetes, focusing on certificate-based authentication sing both curl and kubectl and access management across multiple clusters.

## Certificate Authentication with curl and kubectl

Previously, we generated a certificate for a user and utilized the certificate along with a key to query the Kubernetes REST API for a list of pods. For instance, if your cluster is named "my kube playground," you can make a curl request to the API server as follows:

```bash theme={null}
curl https://my-kube-playground:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

The API server then returns a response similar to this:

```json theme={null}
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods"
  },
  "items": []
}
```

Likewise, when using the kubectl command-line tool, you can supply the same parameters:

```bash theme={null}
kubectl get pods \
  --server https://my-kube-playground:6443 \
  --client-key admin.key \
  --client-certificate admin.crt \
  --certificate-authority ca.crt
```

The response in this case might be:

```bash theme={null}
No resources found.
```

> 💡 Instead of typing these options each time, streamline your workflow by moving them into a kubeconfig file.

## Understanding the Kubeconfig File

# Kubeconfig

Think of a **kubeconfig** file as your all-access pass and passport to your Kubernetes clusters.

When you use the kubectl command-line tool, it doesn't automatically know where your cluster is or who you are. The kubeconfig file bridges that gap by organizing your cluster access, user credentials, and contexts. By default, kubectl looks for this file at ~/.kube/config.

---

## The Three Core Pillars of Kubeconfig

A kubeconfig file is written in YAML and is structured around three main concepts: **Clusters**, **Users**, and **Contexts**.

| Element      | What it is                                                                                                             | Example Data                                                                               |
| ------------ | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **clusters** | **Where** you are going. The API endpoints of your Kubernetes infrastructure.                                          | API Server URL ([https://1.2.3.4:6443](https://1.2.3.4:6443)), Certificate Authority data. |
| **users**    | **Who** you are. The authentication credentials used to verify your identity.                                          | Client certificates, bearer tokens, or username/password.                                  |
| **contexts** | **How** you connect. A context ties a specific **User** to a specific **Cluster**, often defining a default namespace. | dev-ctx = Connect to dev-cluster as dev-admin in the frontend namespace.                   |

There is also a current-context field at the end of the file. This tells kubectl which context to use by default if you don't explicitly pass a flag in your command.

---

## What a Kubeconfig Looks Like

Here is a simplified example of a kubeconfig file managing access to a development cluster and a production cluster:

```yaml
apiVersion: v1
kind: Config
preferences: {}

# 1. Define the infrastructure locations
clusters:
  - name: development-cluster
    cluster:
      server: https://dev-k8s.example.com:6443
      certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t...
  - name: production-cluster
    cluster:
      server: https://prod-k8s.example.com:6443
      certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t...

# 2. Define the identities
users:
  - name: developer-alice
    user:
      client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t...
      client-key-data: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0t...
  - name: admin-bob
    user:
      token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9...

# 3. Create the shortcuts (mapping identity to location)
contexts:
  - name: dev-environment
    context:
      cluster: development-cluster
      user: developer-alice
      namespace: feature-testing
  - name: prod-environment
    context:
      cluster: production-cluster
      user: admin-bob

# 4. Set the active target
current-context: dev-environment
```

---

## Handy Commands for Managing Kubeconfig

Instead of editing this YAML file manually—which can easily lead to formatting errors—you should use kubectl config commands to manage it.

- **View your current configuration:**

```shell
kubectl config view
```

- **To see all Contexts:**

```shell
kubectl config get-contexts
```

- **Switch to a different cluster/context:**

```bash
kubectl config use-context prod-environment
```

> ⚠️ **Security Warning:** Your kubeconfig files contain highly sensitive credentials (like private keys or tokens) that grant direct access to your infrastructure. Never commit a raw kubeconfig to public repositories, and ensure its file permissions are tightly restricted (chmod 600 ~/.kube/config).

- If your config file is not in the \~/.kube directory. then you have specify using --kubeconfig flag in the command. By adding the --kubeconfig flag, you are explicitly telling the tool to ignore the default and look at your specific file instead

```bash theme={null}
kubectl config view --kubeconfig=/path/to/config
```

```bash theme={null}
kubectl config use-context contextname --kubeconfig=/path/to/config
```

- If you want to view a custom kubeconfig file, use the `--kubeconfig` option:

```bash theme={null}
kubectl config view --kubeconfig=my-custom-config
```

- To change the active context—for example, switching from the admin user to the production user—execute:

```bash theme={null}
kubectl config use-context prod-user@production
```

## Managing Certificates in Kubeconfig Files

> 💡 For best practices, use full paths for certificate files in your kubeconfig file. Alternatively, you can embed the certificate data directly using the `certificate-authority-data` field.

For instance, specifying a full path looks like this:

```yaml theme={null}
apiVersion: v1
kind: Config
clusters:
  - name: production
    cluster:
      certificate-authority: /etc/kubernetes/pki/ca.crt
```

Alternatively, you may embed the certificate data directly:
Note: First convert the content of certificate file into base64 encoded format and then pass under certificate-authority-data

```yaml theme={null}
apiVersion: v1
kind: Config
clusters:
  - name: production
    cluster:
      certificate-authority: /etc/kubernetes/pki/ca.crt
      certificate-authority-data: LS0tLS1CRUdJTiBD...
```

To decode base64 encoded certificate data, use the following command:

```bash theme={null}
echo "LS0...bnJ" | base64 --decode
```

The decoded output will resemble:

```text theme={null}
-----BEGIN CERTIFICATE-----
MIICDCCAuCAQAwE...
-----END CERTIFICATE-----
```

## Conclusion

This concludes our detailed exploration of kubeconfig files in Kubernetes. Use these best practices and examples to manage your clusters efficiently and troubleshoot any configuration issues you may encounter.

For further learning, explore the following resources:

- [Kubernetes Basics](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Hub](https://hub.docker.com/)
- [Terraform Registry](https://registry.terraform.io/)
