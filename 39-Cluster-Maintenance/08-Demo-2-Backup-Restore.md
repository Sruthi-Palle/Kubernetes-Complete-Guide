# Demo 2 Backup and Restore

> Learn to back up and restore etcd databases across multiple Kubernetes clusters using the kubectl client on the student node.

In this lesson, you will learn how to back up and restore etcd databases across multiple Kubernetes clusters. We will use the student node—which already has the kubectl client installed—to access and work with both clusters in our environment.

---

## Verifying the Environment and Cluster Configuration

Before proceeding, it is critical to ensure that your kubectl configuration is correct. Begin by listing the nodes and reviewing the current configuration.

Run the following command to list the nodes:

```bash theme={null}
student-node ~ ⟶ kubectl get nodes
NAME                       STATUS   ROLES           AGE   VERSION
cluster1-controlplane      Ready    control-plane   51m   v1.24.0
cluster1-node01            Ready    <none>         50m   v1.24.0
```

Then, check the current kubectl configuration to verify all defined clusters:

```bash theme={null}
student-node ~ ⟶ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://cluster1-controlplane:6443
  name: cluster1
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.33.162.10:6443
  name: cluster2
contexts:
- context:
    cluster: cluster1
    user: cluster1
  name: cluster1
- context:
    cluster: cluster2
    user: cluster2
  name: cluster2
current-context: cluster1
kind: Config
preferences: {}
users:
- name: cluster1
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: cluster2
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

> There are two clusters defined in the kubectl config on the student node: cluster1 and cluster2.

---

## Inspecting Cluster Nodes

### For Cluster 1

To determine the number of nodes in cluster1, switch to the cluster1 context:

```bash theme={null}
kubectl config use-context cluster1
```

Then, list the nodes:

```bash theme={null}
kubectl get nodes
NAME                     STATUS   ROLES           AGE   VERSION
cluster1-controlplane    Ready    control-plane   52m   v1.24.0
cluster1-node01          Ready    <none>         51m   v1.24.0
```

> Cluster1 consists of two nodes: one control plane node and one worker node (cluster1-node01).

### For Cluster 2

To inspect the nodes in cluster2 and identify the control plane node, switch to the cluster2 context:

```bash theme={null}
kubectl config use-context cluster2
```

List the nodes for cluster2:

```bash theme={null}
kubectl get nodes
NAME                     STATUS   ROLES          AGE     VERSION
cluster2-controlplane    Ready    control-plane  53m     v1.24.0
cluster2-node01          Ready    <none>        52m     v1.24.0
```

> The control plane node for cluster2 is identified as "cluster2-controlplane."

_Reminder:_ You can SSH directly into any node by running a command like `ssh <node-name>`. To return to the student node, simply type `exit`.

---

## Determining etcd Configuration

### Cluster 1: Stacked etcd Setup

For cluster1, etcd is configured in a stacked mode, meaning it runs on the control plane node. To verify this, list the pods in the kube-system namespace:

```bash theme={null}
kubectl get pods -n kube-system
```

A sample output shows an etcd pod running:

```bash theme={null}
NAME                                        READY   STATUS    RESTARTS   AGE
etcd-cluster1-controlplane                  1/1     Running   0          55m
...
```

Another way to verify is by describing the kube-apiserver pod. The configuration will include the `--etcd-servers` flag. If the URL is set to `127.0.0.1:2379` (or points to the control plane node’s IP), it confirms that the stacked configuration is used. For example:

```bash theme={null}
--etcd-servers=https://127.0.0.1:2379
```

> This configuration confirms that cluster1 is running a stacked etcd where the etcd database is hosted on the control plane node.

### Cluster 2: External etcd Setup

For cluster2, the setup differs because etcd is hosted externally. First, switch to the cluster2 context and list the pods in kube-system:

```bash theme={null}
kubectl config use-context cluster2
kubectl get pods -n kube-system
```

You will notice that no etcd pod is listed. By describing the kube-apiserver pod using `kubectl describe pod`, observe that the `--etcd-servers` flag points to an external URL. For example:

```bash theme={null}
--etcd-servers=https://192.33.162.21:2379
```

> The external etcd server used by cluster2 is identified with the IP address 192.33.162.21.

---

## Identifying the Default Data Directory for etcd

For cluster1, the default data directory for etcd can be found by inspecting the etcd pod’s configuration. Look for the `--data-dir` flag:

```bash theme={null}
--data-dir=/var/lib/etcd
```

> The default data directory used by the etcd data store in cluster1 is /var/lib/etcd.

---

## Backing Up etcd on Cluster 1

Since cluster1 uses a stacked etcd, its etcd pod is running on the control plane node. Follow these steps to back up etcd:

1. Switch to the cluster1 context:

   ```bash theme={null}
   kubectl config use-context cluster1
   ```

2. SSH into the control plane node:

   ```bash theme={null}
   ssh cluster1-controlplane
   ```

3. Use `etcdctl` (API version 3) with the correct certificate files and connection details to create a snapshot of the etcd database:

   ```bash theme={null}
   ETCDCTL_API=3 etcdctl --endpoints=https://192.33.162.8:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     snapshot save /opt/cluster1.db
   ```

   If the snapshot is successful, the following message will be displayed:

   ```bash theme={null}
   Snapshot saved at /opt/cluster1.db
   ```

4. Verify that the snapshot exists by listing the contents of the `/opt` directory:

   ```bash theme={null}
   cd /opt/
   ls
   # Expected output should include: cluster1.db
   ```

5. Finally, copy the snapshot from the control plane node to the student node:

   ```bash theme={null}
   scp cluster1-controlplane:/opt/cluster1.db .
   ```

> Make sure you have the necessary permissions when copying files between nodes.

---

## Restoring etcd for Cluster 2

For cluster2, a backup file named `cluster2.db` exists on the student node at the path `/opt/cluster2.db`. Use this backup to restore etcd to a new data directory called `/var/lib/etcd-data-new`.

Follow these steps:

1. **Copy the Snapshot File to the External etcd Server**\
   Transfer the snapshot from the student node to the external etcd server (ensure proper access credentials):

   ```bash theme={null}
   scp /opt/cluster2.db <external-etcd-server>:/root/
   ```

2. **Restore the Snapshot**\
   SSH into the external etcd server, then run the following command to restore the snapshot to the new data directory:

   ```bash theme={null}
   ETCDCTL_API=3 etcdctl snapshot restore /root/cluster2.db --data-dir /var/lib/etcd-data-new
   ```

   This will create a restored etcd data directory at `/var/lib/etcd-data-new`.

3. **Update Directory Permissions**\
   Ensure that the restored directory is owned by the etcd user:

   ```bash theme={null}
   chown -R etcd:etcd /var/lib/etcd-data-new
   ```

4. **Update etcd Systemd Configuration**\
   Modify the etcd systemd service file (usually located at `/etc/systemd/system/etcd.service`) so that the `--data-dir` flag points to the new directory. Below is an example excerpt from the configuration:

   ```ini theme={null}
   [Service]
   ExecStart=/usr/local/bin/etcd \
     --name etcd-server \
     --data-dir=/var/lib/etcd-data-new \
     --cert-file=/etc/etcd/pki/etcd.pem \
     --key-file=/etc/etcd/pki/etcd-key.pem \
     --peer-cert-file=/etc/etcd/pki/peer.crt \
     --peer-key-file=/etc/etcd/pki/peer.key \
     --trusted-ca-file=/etc/etcd/pki/ca.pem \
     --peer-trusted-ca-file=/etc/etcd/pki/ca.pem \
     --client-cert-auth \
     --peer-client-cert-auth \
     --initial-advertise-peer-urls https://192.33.162.21:2380 \
     --listen-peer-urls https://192.33.162.21:2380 \
     --advertise-client-urls https://192.33.162.21:2379 \
     --listen-client-urls https://192.33.162.21:2379,https://127.0.0.1:2379 \
     --initial-cluster etcd-server=https://192.33.162.21:2380
   ```

5. **Restart etcd and Related Control Plane Components**\
   Reload the systemd daemon and restart the etcd service:

   ```bash theme={null}
   systemctl daemon-reload
   systemctl restart etcd
   ```

   Next, ensure that the control plane components, such as kube-apiserver, kube-controller-manager, and kube-scheduler in cluster2, recognize the new etcd data. You may need to restart these pods or the Kubelet process. Verify the status of control plane pods by switching to cluster2 and running:

   ```bash theme={null}
   kubectl config use-context cluster2
   kubectl get pods -n kube-system
   ```

   A sample healthy output might look like:

   ```bash theme={null}
   NAME                                             READY   STATUS    RESTARTS   AGE
   coredns-6d4b75c6bd-7fgr9                          1/1     Running   0          79m
   kube-apiserver-cluster2-controlplane             1/1     Running   0          79m
   kube-controller-manager-cluster2-controlplane      1/1     Running   0          79m
   kube-scheduler-cluster2-controlplane             1/1     Running   0          79m
   ...
   ```

6. **Validate the Kubelet Service**\
   Ensure that the Kubelet on the control plane node is running without critical errors:

   ```bash theme={null}
   systemctl status kubelet
   ```

Once all components are healthy, the restoration process for cluster2 is complete.

> Ensure that proper backup and restore procedures are followed when modifying etcd configurations to avoid data loss.

---

This concludes the practice lesson on backing up the etcd datastore on cluster1 and restoring it onto cluster2 using a new data directory.
