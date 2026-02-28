# 2025 Updates In place Resize of Pods

> In this article, we examine the in-place resizing of Pods in Kubernetes and explain how it can reduce disruptions during resource updates.

## Understanding Resource Updates in Kubernetes

Before discussing in-place resizing, it’s important to understand the traditional update process. In Kubernetes version 1.32, modifying a pod’s resource requirements in a Deployment causes the existing pod to be deleted and replaced with a new one that has the updated resource definitions. This approach does not perform changes in place, and the termination and replacement process can be disruptive—especially for stateful workloads.

## In-Place Pod Resource Updates

To mitigate disruption, Kubernetes is developing an in-place update feature for pod resources. This feature is currently in alpha (as of Kubernetes 1.27) and is not enabled by default. To use it, you must explicitly enable the feature flag "in-place Pod vertical scaling." When enabled, additional resize policy parameters become available. For example, you can specify that changing the CPU resource does not require a pod restart, while adjusting the memory allocation does.

> This in-place update feature is in alpha and requires manual activation via the feature flag. It is anticipated that the feature will eventually progress to beta and then achieve a stable release.

## Deployment Example with Resource Definitions

Below is an example of a Deployment configuration that sets resource requests and limits for a container:

```yaml theme={null}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx
          resources:
            requests:
              cpu: "1"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

When you update the CPU resource (for example, increasing the request), the pod can adjust its allocation without a restart. Conversely, changing the memory allocation will trigger a pod restart if the restart policy requires it.

## Limitations of In-Place Resizing

While in-place resizing offers benefits, it also comes with certain limitations:

- **Supported Resources:** Only CPU and memory can be updated in place.
- **QoS Class:** The Pod Quality of Service (QoS) class cannot be modified through in-place resizing.
- **Container Eligibility:** Init containers and ephemeral containers are not eligible for resizing.
- **Immutable Resource Positions:** Once set, a container's resource requests and limits cannot be repositioned.
- **Memory Limit Constraints:** A container’s memory limit cannot be reduced below its current usage. If an update attempts to lower the memory limit too far, the resize operation will remain in progress until a permissible limit is reached.
- **Platform Support:** Windows Pods are not supported by the in-place resizing feature at this time.

That concludes our discussion on in-place resizing of Pods in Kubernetes. For more detailed Kubernetes concepts, consider exploring the [Kubernetes Documentation](https://kubernetes.io/docs/).
