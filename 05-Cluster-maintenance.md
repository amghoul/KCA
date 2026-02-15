# Cluster Maintenance
- **Draining a Node:** Draining involves gracefully terminating the pods running on that node so they are recreated on other nodes. At the same time, draining marks the node as unschedulable (cordoned), preventing new pods from being scheduled on it until explicitly allowed.
```bash
kubectl drain node-1
```
- **To reverse the drain operation:**
```bash
kubectl uncordon node-1
```
- **Cordon a node:** cordon only marks a node as unschedulable without terminating or relocating the currently running pods. This ensures that no new pods will be scheduled on the node.
```bash
kubectl cordon node-1
```
## Cluster Upgrade Introduction
- It is important to note that not all components are required to run on the same version. 
- Although different components can operate on varying release versions, the Kube API Server remains the primary control plane component that all others communicate with. 
- Consequently, no component should ever run on a version higher than the API Server. For example:
- The `controller manager` and `scheduler` may be `one version lower than` the `API Server`.
- The `Kubelet` and `Kube Proxy` components may be `two versions lower than` the `API Server`.
- For instance, if the API Server is at version 1.10, then:
The controller manager and scheduler can run on version 1.10 or 1.9.
The Kubelet and Kube Proxy can run on version 1.8. Running any component on a version higher than the API Server (e.g., 1.11 when the API Server is 1.10) is not recommended.
- **Note:** `Kube Control utility` is an exception and may run on a version that `is higher, lower, or the same as` the `API Server`. This flexibility supports live, rolling upgrades where components can be upgraded individually.

-- Upgrade Process Overview
Consider a production cluster with master and worker nodes running version 1.10. The upgrade process generally involves two major steps:
- Upgrading the master nodes.
 Upgrading the worker nodes.

## Upgrading with kubeadm
- kubeadm upgrade plan
```bash
apt-get upgrade -y kubeadm=1.12.0-00
kubeadm upgrade apply v1.12.0
# Output:
# [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.12.0". Enjoy!
# [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```
- For Worker Nodes:
```bash
kubectl drain node-1
kubectl uncordon node-1
```
## Backup and Restore Methods

### What to Back Up

For most Kubernetes deployments, consider backing up:

* **Declarative Configuration Files:** Files defining resources like Deployments, Pods, and Services.
* **Cluster State:** Information stored in the etcd cluster.
* **Imperative Objects:** Resources created on the fly (e.g., namespaces, secrets, configMaps) which might not be documented in files.

Using a declarative approach — creating definition files and applying them with kubectl — not only documents your configuration but also makes it reusable and shareable. For example, here’s a simple Pod definition:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```

Apply the Pod definition with:

```bash  theme={null}
kubectl apply -f pod-definition.yml
```

### Imperative vs. Declarative Backup Approaches

While the declarative method is preferred, sometimes resources are created using imperative commands. These changes might not be stored in your version control system, which can lead to gaps in your backups. To capture all configurations, you can query the Kubernetes API server directly.

For instance, back up all resources across every namespace by running:

```bash  theme={null}
kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml
```

This command generates a comprehensive YAML snapshot of pods, deployments, services, and other resources. To simplify and automate this process in production, consider using tools like [Velero](https://velero.io).

### Backing Up the etcd Cluster

The etcd cluster is the backbone of your Kubernetes system, storing critical state and configuration details. Typically located on the master nodes, etcd’s data resides in a dedicated directory determined during setup.

Below is an example of how etcd might be configured on a master node:

```bash  theme={null}
ExecStart=/usr/local/bin/etcd \\
   --name ${ETCD_NAME} \\
   --cert-file=/etc/etcd/kubernetes.pem \\
   --key-file=/etc/etcd/kubernetes-key.pem \\
   --peer-cert-file=/etc/etcd/kubernetes.pem \\
   --peer-key-file=/etc/etcd/kubernetes-key.pem \\
   --trusted-ca-file=/etc/etcd/ca.pem \\
   --peer-trusted-ca-file=/etc/etcd/ca.pem \\
   --client-cert-auth \\
   --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
   --listen-peer-urls https://${INTERNAL_IP}:2380 \\
   --advertise-client-urls https://${INTERNAL_IP}:2379 \\
   --initial-cluster etcd-cluster-0 \\
   --initial-cluster-token etcd-cluster-0 \\
   --initial-cluster controller-0=https://${CONTROLLER0_IP}:2379 \\
   --initial-cluster-state new \\
   --data-dir=/var/lib/etcd
```

etcd offers a built-in snapshot feature via the etcdctl command. To create a snapshot called "snapshot.db", run:

```bash  theme={null}
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

After creating the snapshot, you can verify its existence:

```bash  theme={null}
ls
```

And check the snapshot status:

```bash  theme={null}
ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

## Restoring from an etcd Backup

In the event of a failure, restoring your cluster from an etcd backup involves several steps:

1. **Stop the Kubernetes API Server:** The restore process requires stopping the API server.

2. **Restore the Snapshot:** Restore the snapshot to a new data directory (e.g., `/var/lib/etcd-from-backup`):

   ```bash  theme={null}
   ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
   --data-dir /var/lib/etcd-from-backup
   ```

   This command initializes a new etcd data directory and reinitializes cluster members.

3. **Update etcd Configuration:** Modify your etcd configuration file to point to the new data directory.

4. **Restart Services:** Reload the system daemon, restart the etcd service, and finally restart the Kubernetes API server.

<Callout icon="lightbulb" color="#1CB2FE">
  Always supply the required certificate files (CA certificate, etcd server certificate, and key) during backup and restore operations to ensure secure communications.
</Callout>

For an authenticated backup, use:

```bash  theme={null}
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/etcd/ca.crt \
--cert=/etc/etcd/etcd-server.crt \
--key=/etc/etcd/etcd-server.key
```

## Choosing the Right Backup Approach

Depending on your environment, the backup strategy might vary:

| Backup Approach                 | Use Case                                     | Command Example                                                       |
| ------------------------------- | -------------------------------------------- | --------------------------------------------------------------------- |
| Declarative File Backup         | When configurations are maintained as code   | `kubectl apply -f pod-definition.yml`                                 |
| API Server Configuration Backup | Capturing all cluster resources imperatively | `kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml` |
| etcd Snapshot                   | Backing up the critical cluster state        | `ETCDCTL_API=3 etcdctl snapshot save snapshot.db`                     |

For managed Kubernetes environments where you might not have direct etcd access, relying on API queries is often the more practical solution.

### How to restore a backup from snapshot

- Step 1: Stop the kube-apiserver
Execute the following commands:
```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 30
```
- Step 2: Restore the etcd snapshot

Run the command below to restore the snapshot:

```bash
etcdutl snapshot restore /opt/snapshot-pre-boot.db --data-dir /var/lib/etcd-from-backup
```
```bash
Expected output:

2025-04-24T09:38:07Z    info    snapshot/v3_snapshot.go:265     restoring snapshot
2025-04-24T09:38:07Z    info    membership/store.go:141 Trimming membership information from the backend...
2025-04-24T09:38:07Z    info    membership/cluster.go:421       added member
2025-04-24T09:38:07Z    info    snapshot/v3_snapshot.go:293     restored snapshot
```
- Step 3: Update the etcd configuration

Open the etcd configuration file for editing:
```bash
vi /etc/kubernetes/manifests/etcd.yaml
```
Modify the volumes section as follows:
From:
```yaml
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd                    # OLD directory
      type: DirectoryOrCreate
    name: etcd-data
```
To:
```yaml
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-from-backup        # NEW restored directory
      type: DirectoryOrCreate
    name: etcd-data
```
Upon saving this file, the etcd pod will restart automatically due to static pod behavior.

- Step 4: Restart the kube-apiserver

Move the kube-apiserver manifest back to its original location:
```bash
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```
Wait for 60 seconds to allow the kube-apiserver to start.

- Step 5: Restart other control plane components

Execute the following commands to restart additional components:

- Restart kube-controller-manager
```bash
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
sleep 20
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
```
- Restart kube-scheduler
```bash
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
sleep 20
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
```
- Restart kubelet
```bash
systemctl restart kubelet
```
- Step 6: Monitor the restart process

Use the following command to monitor the restart:
```bash
watch crictl ps
```
Key indicators to observe:

All components should show STATUS = Running
The entire process should take approximately 2-3 minutes.
Step 7: Verify the restore
