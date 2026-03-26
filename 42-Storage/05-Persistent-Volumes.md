# Persistent Volumes

> 💡 This article explores centralizing storage management in Kubernetes using persistent volumes and how to create and manage them effectively.

Previously, we discussed how volumes are defined within pod manifest files, where the storage settings are directly included in each pod definition. For example, a typical pod configuration with a volume might look like:

```yaml theme={null}
volumes:
  - name: data-volume
    awsElasticBlockStore:
      volumeID: <volume-id>
      fsType: ext4
```

> 💡 In environments where many users deploy multiple pods, duplicating storage configuration in every pod file can lead to redundancy and increased maintenance efforts. Any required change would need to be propagated across all pod definitions.

To solve this issue, administrators can create a centralized pool of storage. Users then request portions of this storage as needed by creating persistent volume claims (PVCs). This concept is enabled by persistent volumes (PVs).

> 💡 A persistent volume is a cluster-wide storage resource defined and managed by an administrator. Applications running on the cluster utilize these PVs by binding to them via persistent volume claims.

![alt text](../Images/Persistent-Volumes.png)

## Creating a Persistent Volume

In this section, we will create a persistent volume using a base template. First, update the API version, set the kind to PersistentVolume, and give it a name (for example, "pv-vol1"). Under the spec section, it's necessary to define the access modes. The access modes determine how a volume can be mounted on nodes, such as:

- ReadWriteOnce: The volume can be mounted as read-write by a single node.
- ReadOnlyMany: The volume can be mounted as read-only by multiple nodes.
- ReadWriteMany: The volume can be mounted as read-write by multiple nodes.

Below is an initial portion of the configuration that defines the persistent volume along with its access mode:

```yaml theme={null}
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
```

Next, specify the storage capacity for this persistent volume. In our example, we set the capacity to 1Gi. After defining the capacity, choose the volume type. Here, we use the hostPath option, which leverages storage from the node's local directory.

> 💡 The hostPath option is primarily for testing or single-node setups and is not recommended for production environments.

The complete persistent volume manifest appears as follows:

```yaml theme={null}
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

To create the persistent volume, execute the following command:

```bash theme={null}
kubectl create -f pv-d
```

After creating the volume, you can verify its status by running:

```bash theme={null}
kubectl get persistentvolume
```

In a production setup, replace the hostPath option with a supported storage solution, such as AWS Elastic Block Store, to ensure data durability and scalability.

This concludes our discussion on persistent volumes. Up next, we will explore persistent volume claims (PVCs) to see how you can request storage from the centralized persistent volume pool.

For further reading, check out these helpful resources:

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes Basics](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
- [AWS Elastic Block Store](https://aws.amazon.com/ebs/)
- [Docker Hub](https://hub.docker.com/)
