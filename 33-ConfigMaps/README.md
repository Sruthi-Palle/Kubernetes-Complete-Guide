# Configure ConfigMaps in Applications

> In this article, we'll explore how to externalize and manage configuration data in Kubernetes using ConfigMaps. Instead of hard-coding environment variables in each pod definition, ConfigMaps allow you to centrally manage key–value pairs, making your Kubernetes configurations easier to maintain and scale.

> 💡 By leveraging ConfigMaps, you can separate configuration details from container images, reducing duplication and simplifying updates across multiple pods.

## Using Environment Variables in a Pod

Traditionally, environment variables are set directly within the pod specification. For example:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 80
  env:
    - name: APP_COLOR
      value: blue
    - name: APP_MODE
      value: prod
```

In this configuration, the environment variables are hard-coded into the pod definition, which can become cumbersome when managing multiple pods.

## Centralizing Configuration with ConfigMaps

To simplify the management of environment configurations, you can externalize the data using a ConfigMap. With this approach, Kubernetes inserted centrally stored key–value pairs into your pods during creation.

For instance, modify the pod definition to use the `envFrom` property:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 80
  envFrom:
    - configMapRef:
        name: app-color
```

Assume that the ConfigMap (named `app-color`) holds the following key–value pairs:

```yaml theme={null}
APP_COLOR: blue
APP_MODE: prod
```

## Creating ConfigMaps

There are two main approaches to create ConfigMaps in Kubernetes: the imperative method and the declarative method.

### Imperative Approach

If you prefer using the command line without a definition file, you can create a ConfigMap directly by specifying key–value pairs. For example, create a ConfigMap named `app-config` with specific environment variables:

```bash theme={null}
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MOD=prod
```

You can also generate a ConfigMap from a file using the `--from-file` option. For instance, if you have a file named `app_config.properties` containing configuration data:

```bash theme={null}
kubectl create configmap app-config --from-file=app_config.properties
```

### Declarative Approach

With a declarative approach, you define your ConfigMap in a YAML file and apply it with kubectl. Here is an example ConfigMap definition:

```yaml theme={null}
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

Create the ConfigMap in your cluster with:

```bash theme={null}
kubectl create -f config-map.yaml
```

For larger deployments, consider organizing multiple ConfigMaps by logical grouping, such as for your application, MySQL, and Redis:

| ConfigMap Name | Description                  | Sample Data                          |
| -------------- | ---------------------------- | ------------------------------------ |
| app-config     | Application configuration    | APP_COLOR: blue, APP_MODE: prod      |
| mysql-config   | MySQL database configuration | port: 3306, max_allowed_packet: 128M |
| redis-config   | Redis server configuration   | port: 6379, rdb-compression: yes     |

Naming ConfigMaps appropriately is essential because you will reference these names when associating them with pods.

## Viewing ConfigMaps

Once a ConfigMap is created, you can list it using:

```bash theme={null}
kubectl get configmaps
```

To check the stored configuration data, use the describe command:

```bash theme={null}
kubectl describe configmaps
```

The output will detail the key–value pairs stored in the ConfigMap, for example:

```plaintext theme={null}
Name:           app-config
Namespace:      default
Labels:         <none>
Annotations:    <none>
Data
====
APP_COLOR:
----
blue
APP_MODE:
----
prod
Events:        <none>
```

> 💡 Remember to frequently review and update your ConfigMaps to ensure your applications always run with the most current configurations.

## inserting a ConfigMap into a Pod

After creating a ConfigMap, configure your pod to use the configuration data. Below is an example pod definition that inserts the `app-config` ConfigMap into the container as environment variables:

```yaml theme={null}
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
  envFrom:
    - configMapRef:
        name: app-config
```

The corresponding ConfigMap definition might look like this:

```yaml theme={null}
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: production
```

When you create the pod with this configuration, your web application receives the configured environment variables automatically.

## Alternative insertion Methods

In addition to using `envFrom`, there are other methods to insert configuration data from ConfigMaps into your pods. You can insert a single environment variable using the `valueFrom` property or mount the entire ConfigMap as a volume. For example:

```yaml theme={null}
envFrom:
  - configMapRef:
      name: app-config

env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR

volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```

Each method provides flexibility to fit the design of your application, whether you need specific environment variables or a complete set of configuration files mounted as volumes.

> 💡 Always validate the names and key references in your ConfigMaps to ensure that your pods load the correct configuration data.

---

Practice managing and troubleshooting ConfigMaps on your live Kubernetes environment to build a robust and scalable configuration management workflow. For more detailed information, visit [Kubernetes Configuration Best Practices](https://kubernetes.io/docs/tasks/configmap/) and [Kubernetes Documentation](https://kubernetes.io/docs/).
