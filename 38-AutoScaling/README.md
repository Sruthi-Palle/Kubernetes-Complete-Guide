# Introduction to Autoscaling

> 💡 Before diving into autoscaling in Kubernetes, it's beneficial to understand the basic concepts of scaling using traditional physical servers.

## Traditional Scaling Concepts

Historically, applications were deployed on physical servers with fixed CPU and memory capacities. When demand increased and resources were exhausted, the only option was to perform vertical scaling. This involved:

- Shutting down the application.
- Upgrading the CPU or memory.
- Restarting the server.

This process is referred to as vertical scaling since it focuses on enhancing the capacity of an existing server.

Conversely, if an application supported multiple instances, additional servers could be added to handle increased loads without any downtime. This method, known as horizontal scaling, distributes the workload by creating more instances of the application.

### Key Points:

- **Vertical Scaling:** Increases resources (CPU, memory) of an existing server.
- **Horizontal Scaling:** Increases server count by adding more instances.

![alt text](../images/autoscaling.png)

## Scaling in Kubernetes

Kubernetes is specifically designed for hosting containerized applications and incorporates scaling based on current demands. It supports two main scaling types:

1. **Workload Scaling:** Adjusting the number of containers or Pods running in the cluster.
2. **Cluster (Infrastructure) Scaling:** Adding or removing nodes (servers) from the cluster.

![alt text](../images/autoscaling-1.png)

When scaling in a Kubernetes cluster, consider the following:

- **Cluster Infrastructure Scaling:**
  - **Horizontal Scaling:** Add more nodes.
  - **Vertical Scaling:** Enhance the resources (CPU, memory) of existing nodes.(Less common approach due to downtime)
- **Workload Scaling:**
  - **Horizontal Scaling:** Create additional Pods.
  - **Vertical Scaling:** Modify the resource limits and requests for existing Pods.

![alt text](../images/autoscaling-2.png)

## Approaches to Scaling in Kubernetes

Kubernetes supports both manual and automated scaling methods.

### Manual Scaling

Manual scaling requires intervention from the user:

- **Infrastructure Scaling (Horizontal):** Provision new nodes and join them to the cluster using:

  ```bash theme={null}
  kubeadm join ...
  ```

- **Workload Scaling (Horizontal):** Adjust the number of Pods manually with:

  ```bash theme={null}
  kubectl scale ...
  ```

- **Pod Resource Adjustment (Vertical):** Edit the deployment, stateful set, or replica set to modify resource limits and requests:

  ```bash theme={null}
  kubectl edit ...
  ```

### Automated Scaling

Automation in Kubernetes simplifies scaling and ensures efficient resource management:

- **Kubernetes Cluster Autoscaler:** Automatically adjusts the number of nodes in the cluster by adding or removing nodes when needed.
- **Horizontal Pod Autoscaler (HPA):** Monitors metrics and adjusts the number of Pods dynamically.
- **Vertical Pod Autoscaler (VPA):** Automatically changes resource allocations for running Pods based on observed usage.

> 💡 Automated scaling mechanisms in Kubernetes allow your applications and infrastructure to adapt quickly to changing loads, reducing manual effort and ensuring optimal performance.

## Why Vertical scaling of Nodes(servers) is not recommended?

> 💡 Note: Vertical scaling of cluster nodes is less common in Kubernetes because it often requires downtime. In virtualized environments, it may be easier to provision a new VM with higher resources, add it to the cluster, and then decommission the older node.

### Infrastructure Downtime

The most significant barrier to vertical scaling is the requirement to take down both the server and the applications running on it. The process involves:

1. Shutting down active servers and applications.
2. Manually adding resources to the hardware or virtual instance.
3. Rebooting the system to bring services back online.

Because modern service-level agreements (SLAs) prioritize continuous availability, this disruptive cycle is deemed undesirable for professional environments.

### Provisioning and Cluster Integration

Rather than attempting to modify an active server, administrators utilize the flexibility of VMs to maintain cluster health. The preferred workflow involves:

- Provisioning: Creating a new server with higher resource specifications.
- Integration: Adding this high-capacity server to the existing Kubernetes cluster.
- Decommissioning: Removing the older, lower-resource server from the cluster.

This method allows for the enhancement of cluster capacity without the service interruptions associated with vertical scaling.

## Next Topics

- [Horizontal-Pod-Autoscaler](../38-AutoScaling/Horizontal-Pod-Autoscaler.md)
- [In-Place-Resize-of-Pods](../38-AutoScaling/In-Place-Resize-of-Pods.md)
- [Vertical-Pod-Autoscaler](../38-AutoScaling/Vertical-Pod-Autoscaler.md)
