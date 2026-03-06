# Demo 1 Backup and Restore

> Learn to back up and restore an etcd cluster in Kubernetes, including verification, snapshot creation, and restoration procedures.

We begin by verifying the existing deployments, inspecting the etcd container setup, and then proceed with the backup and restore procedures.

---

## 1. Checking the Current Deployments

Assuming a running Kubernetes cluster with two applications deployed ("red" and "blue"), first confirm that the deployments exist:

```bash theme={null}
root@controlplane ~ ⟶ k get deploy
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
blue   3/3     3            3           18s
red    2/2     2            2           18s
root@controlplane ~ ⟶
```

---

## 2. Verifying the ETCD Version and Pod Details

To identify the version of etcd and verify pod details, locate the etcd pod in the kube-system namespace. Typically configured as a static pod, you can review its description to inspect container details and command-line parameters.

For example, examining the etcd container shows:

```bash theme={null}
Status:             Running
IP:                 10.44.214.9
IPs:
  IP:               10.44.214.9
Controlled By:     Node/controlplane
Containers:
  etcd:
    Container ID:   docker://5930818c18aacf4b00d3eb301cec4427e88249a2ab5291743f05bfa4f5dbf4b7
    Image:          k8s.gcr.io/etcd:3.5.1-0
    Image ID:       docker-pullable://k8s.gcr.io/etcd@sha256:64b9ea357325d5db9fa723dcf503b5a449177b17ac263
    Port:           <none>
    Host Port:      <none>
    Command:
      etcd
      --advertise-client-urls=https://10.44.214.9:2379
      --cert-file=/etc/kubernetes/pki/etcd/server.crt
      --client-cert-auth=true
      --data-dir=/var/lib/etcd
      --initial-advertise-peer-urls=https://10.44.214.9:2380
      --initial-cluster=controlplane=http://10.44.214.9:2380
      --key-file=/etc/kubernetes/pki/etcd/server.key
      --listen-metrics-urls=https://127.0.0.1:2381,https://10.44.214.9:2380
      --listen-peer-urls=https://10.44.214.9:2380
      --name=controlplane
      --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
      --peer-client-auth=true
      --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
      --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
      --snapshot-count=1000
      --etcd-var=100m
State:              Running
Started:            Sun, 17 Apr 2022 20:19:42 +0000
Ready:              True
Restart Count:      0
Requests:
  cpu:            100m
  memory:         100Mi
Liveness:
  http-get:       http://127.0.0.1:2381/health delay=10s timeout=15s period=10s #success=1 #failure=3
```

From the information provided, the etcd version is 3.5.1 running on port 2379 (loopback address). The container’s command-line options also specify critical certificate file locations:

- Server certificate: `/etc/kubernetes/pki/etcd/server.crt`
- CA certificate: `/etc/kubernetes/pki/etcd/ca.crt`

A similar configuration is confirmed in the second block:

```bash theme={null}
etcd:
  Container ID: docker://5930818c18aacfa00d3eb301cec4427e88249a2ab5291743f05bfa4f5dbf4b7
  Image: k8s.gcr.io/etcd:3.5.1-0
  Image ID: docker-pullable://k8s.gcr.io/etcd@sha256:64b9ea357325d5db9f8a723dcf503b5a449177b17ac8
  Port: <none>
  Host Port: <none>
  Command:
    --advertise-client-urls=https://10.44.214.9:2379
    --cert-file=/etc/kubernetes/pki/etcd/server.crt
    --client-cert-auth=true
    --data-dir=/var/lib/etcd
    --initial-advertise-peer-urls=https://10.44.214.9:2380
    --initial-cluster=controlplane=https://10.44.214.9:2380
    --key-file=/etc/kubernetes/pki/etcd/server.key
    --listen-client-urls=http://127.0.0.1:2379,https://10.44.214.9:2379
    --listen-metrics-urls=http://127.0.0.1:2381
    --peer-urls=https://10.44.214.9:2380
    --name=controlplane
    --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    --peer-client-cert-auth=true
    --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    --snapshot-count=1000
  State: Running
  Started: Sun, 17 Apr 2022 20:19:42 +0000
  Ready: True
  Restart Count: 0
  Requests:
    cpu: 100m
    memory: 100Mi
  Liveness:
    http-get http://127.0.0.1:2381/health delay=10s timeout=15s period=10s #success=1 #failure=1
  Startup:
    http-get http://127.0.0.1:2381/health delay=10s timeout=15s period=10s #success=1 #failure=1
  Environment: <none>
  Mounts:
    /etc/kubernetes/pki/etcd from etcd-certs (rw)
```

---

## 3. Preparing for Maintenance

When maintenance, such as a master node reboot, is scheduled, it is best practice to back up the etcd data. Using the built-in snapshot functionality ensures that your Kubernetes cluster data can be restored if needed.

Review the static pod definition fragment in `/etc/kubernetes/manifests/etcd.yaml`. Notice the configuration for volume mounts and probes:

```yaml theme={null}
scheme: HTTP
initialDelaySeconds: 10
periodSeconds: 10
timeoutSeconds: 15
name: etcd
resources:
  requests:
    cpu: 100m
    memory: 100Mi
startupProbe:
  failureThreshold: 24
  httpGet:
    host: 127.0.0.1
    path: /health
    port: 2381
    scheme: HTTP
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 15
volumeMounts:
  - mountPath: /var/lib/etcd
    name: etcd-data
  - mountPath: /etc/kubernetes/pki/etcd
    name: etcd-certs
hostNetwork: true
priorityClassName: system-node-critical
securityContext:
  seccompProfile:
    type: RuntimeDefault
volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
    type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd
    type: DirectoryOrCreate
    name: etcd-data
status: {}
```

> The use of hostPath volumes maps the control plane node’s physical directories for data and certificates directly into the etcd container.

Verify these paths to confirm the etcd data and certificates exist on the host:

```bash theme={null}
root@controlplane ~ ⟶ ls /var/lib/etcd
member
root@controlplane ~ ⟶ ls /etc/kubernetes/pki/etcd
ca.crt  healthcheck-client.crt  peer.crt  server.crt
ca.key  healthcheck-client.key  peer.key  server.key
root@controlplane ~ ⟶
```

---

## 4. Taking an ETCD Snapshot

Before initiating maintenance, take an etcd snapshot for backup. Ensure that the etcdctl API version is set to 3. The snapshot command requires specifying the endpoint along with the proper certificates.

Run the following command to create a snapshot:

```bash theme={null}
root@controlplane ~ ✗ etcdctl snapshot save --endpoints=127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
/opt/snapshot-pre-boot.db
Snapshot saved at /opt/snapshot-pre-boot.db
root@controlplane ~ ✗
```

Verify the backup has been saved:

```bash theme={null}
root@controlplane ~ ➜ ls /opt/snapshot-pre-boot.db
/opt/snapshot-pre-boot.db
root@controlplane ~ ➜
```

---

## 5. Post-Maintenance Issue Identification

After the maintenance (for example, following a reboot), you may notice that the cluster applications are inaccessible. Confirm the current state by checking the deployments, pods, and services:

```bash theme={null}
root@controlplane ~ ➜ kubectl get deploy
No resources found in default namespace.

root@controlplane ~ ➜ k get pods
No resources found in default namespace.

root@controlplane ~ ➜ k get svc
NAME        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
kubernetes  ClusterIP   10.96.0.1     <none>        443/TCP   20s
root@controlplane ~ ➜
```

Since deployments and services are missing, you need to restore the etcd data from your backup.

---

## 6. Restoring ETCD from Backup

To restore etcd from the snapshot, use the etcdctl snapshot restore command. This process does not require contacting an active etcd endpoint; it restores the data from the backup file into a new data directory.

For example, run:

```bash theme={null}
root@controlplane ~ # etcdctl snapshot restore --data-dir /var/lib/etcd-from-backup /opt/snapshot-pre-boot.db
2022-04-17 20:54:34.669639 I | mvcc: restore compact to 2377
2022-04-17 20:54:34.676896 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
```

Confirm that the new directory (/var/lib/etcd-from-backup) contains the restored data.

---

## 7. Reconfiguring the ETCD Pod

After restoring the etcd data, update the etcd static pod manifest (located at `/etc/kubernetes/manifests/etcd.yaml`) to point to the new data directory. Modify the hostPath for the etcd-data volume from `/var/lib/etcd` to `/var/lib/etcd-from-backup`.

The updated volume section should look like:

```yaml theme={null}
scheme: HTTP
initialDelaySeconds: 10
periodSeconds: 10
timeoutSeconds: 15
volumeMounts:
  - mountPath: /var/lib/etcd
    name: etcd-data
  - mountPath: /etc/kubernetes/pki/etcd
    name: etcd-certs
hostNetwork: true
priorityClassName: system-node-critical
securityContext:
  seccompProfile:
    type: RuntimeDefault
volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
    type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-from-backup
    type: DirectoryOrCreate
    name: etcd-data
status: {}
```

Once you update and save the changes, the kubelet will automatically restart the etcd pod with the updated manifest configuration. Monitor the pod’s status to ensure it transitions to the Running state:

```bash theme={null}
root@controlplane ~ ➜ kubectl get pods -n kube-system --watch
NAME                                      READY   STATUS     RESTARTS   AGE
coredns-6487985d-btbqw                     1/1     Running    0          37m
coredns-6487985d-c5r9t                     1/1     Running    0          37m
etcd-controlplane                          1/1     Pending    0          29s
kube-apiserver-controlplane                1/1     Running    0          37m
kube-controller-manager-controlplane       1/1     Running    1 (66s ago) 37m
kube-flannel-ds-2t8m                        1/1     Running    0          37m
kube-proxy-6bcbb                           1/1     Running    0          37m
kube-scheduler-controlplane                1/1     Running    0          37m
```

> If the new etcd pod fails to reach the Running state, such as due to failed liveness or startup probes, consider manually deleting the pod to force a restart.

Finally, verify the restored state of the cluster by checking that deployments, pods, and services are now available:

```bash theme={null}
root@controlplane ~ ⟶ kubectl get deploy
root@controlplane ~ ⟶ k get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
etcd-controlplane                         1/1     Running   0          14s
...
```

---

With the etcd backup successfully restored and the static pod manifest updated, your Kubernetes control plane resumes normal operations and the applications become accessible again. This completes the backup and restore process for etcd in a Kubernetes cluster.
