# Security
## Authentication
### Basic Auth
<mark>
Setup basic authentication on Kubernetes (Deprecated in 1.19)
Note: This is not recommended in a production environment. This is only for learning purposes. Also note that this approach is deprecated in Kubernetes version 1.19 and is no longer available in later releases
</mark>

- Follow the below instructions to configure basic authentication in a kubeadm setup.

- Create a file with user details locally at /tmp/users/user-details.csv

- User File Contents
```bash
password123,user1,u0001
password123,user2,u0002
password123,user3,u0003
password123,user4,u0004
password123,user5,u0005
```

- Edit the kube-apiserver static pod configured by kubeadm to pass in the user details. The file is located at `/etc/kubernetes/manifests/kube-apiserver.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
      <content-hidden>
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
    volumeMounts:
    - mountPath: /tmp/users
      name: usr-details
      readOnly: true
  volumes:
  - hostPath:
      path: /tmp/users
      type: DirectoryOrCreate
    name: usr-details
```

- Modify the kube-apiserver startup options to include the `basic-auth` file
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
      <content-hidden>
    - --basic-auth-file=/tmp/users/user-details.csv
```
- Create the necessary roles and role bindings for these users:
```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
 
---
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1 # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```
- Once created, you may authenticate into the kube-api server using the users credentials
```bash
curl -v -k https://localhost:6443/api/v1/pods -u "user1:password123"
```
## TLS in Kubernetes Certificate Creation

> This guide explains how to generate TLS certificates for a Kubernetes cluster using OpenSSL, focusing on CA and client/server certificates.

### 1. Generating CA Certificates

First, create the Certificate Authority (CA) certificates. The process involves generating a private key, creating a Certificate Signing Request (CSR) that includes the CA's common name, and finally signing the CSR with the private key to produce the CA certificate. The completed process provides the CA with its private key (`ca.key`) and root certificate (`ca.crt`), which are essential for subsequently signing other certificates.

```bash  theme={null}
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

<Callout icon="lightbulb" color="#1CB2FE">
  The CA certificate is the cornerstone of your certificate infrastructure, so ensure that you safeguard the private key.
</Callout>

### 2. Creating Client Certificates

#### 2.1 Admin User Certificate

To generate a certificate for the admin user:

1. Create a private key for the admin.
2. Generate a CSR for the admin user specifying the common name (CN) and organizational unit (OU) to reflect group membership (e.g., `system:masters`). This consistency ensures that the admin identity is properly logged in audit trails and recognized in `kubectl` commands.
3. Sign the admin CSR with the CA certificate to produce the final admin certificate.

```bash  theme={null}
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

The resulting `admin.crt` file functions as a secure credential, akin to a username and password pair, for authenticating the admin user with the Kubernetes API server.

A similar process is followed to generate client certificates for other components such as the scheduler, controller manager, and kube-proxy.

### 3. Using Client Certificates in API Requests

Client certificates eliminate the requirement for using a username and password when making REST API calls to the Kubernetes API server. The admin certificate, for example, can be used to securely communicate with the server by specifying the key, certificate, and CA certificate in your request.

```bash  theme={null}
curl https://kube-apiserver:6443/api/v1/pods \
  --key admin.key --cert admin.crt --cacert ca.crt
```

The API server will respond with a JSON object listing the pods:

```json  theme={null}
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods"
  },
  "items": []
}
```

Most Kubernetes clients can load these connection parameters via a kubeconfig file that consolidates the information required to reach the API server.

### 4. Server-Side Certificates

For secure communication, both client and server certificates must trust the same CA root certificate. This certificate is used by both parties to verify the authenticity of the certificate they receive.

### 4.1 Etcd Server Certificate

The etcd server, a critical component in high availability configurations, also requires a certificate. If etcd is running as a cluster, remember to generate peer certificates to secure inter-member communications. Once created, the certificates are referenced in the etcd configuration file (commonly, `etcd.yaml`). See the example below:

```bash  theme={null}
cat etcd.yaml
- --advertise-client-urls=https://127.0.0.1:2379
- --key-file=/path-to-certs/etcdserver.key
- --cert-file=/path-to-certs/etcdserver.crt
- --client-cert-auth=true
- --data-dir=/var/lib/etcd
- --initial-advertise-peer-urls=https://127.0.0.1:2380
- --initial-cluster=master=https://127.0.0.1:2380
- --listen-client-urls=https://127.0.0.1:2379
- --listen-peer-urls=https://127.0.0.1:2380
- --name=master
- --peer-cert-file=/path-to-certs/etcdpeer1.crt
- --peer-client-cert-auth=true
- --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
- --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
- --snapshot-count=10000
- --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

The `--trusted-ca-file` option ensures that etcd client connections are authenticated using the CA certificate.

***

### 5. Kube API Server Certificates

The Kube API server is the primary access point for the cluster and is known by several aliases such as "kubernetes", "kubernetes.default", and "kubernetes.default.svc.cluster.local", as well as its IP address. This diversity requires that its certificate includes multiple Subject Alternative Names (SANs).

#### 5.1 Creating the API Server Certificate

Start by generating a CSR for the API server:

```bash  theme={null}
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr
```

Then, create an OpenSSL configuration file (e.g., `openssl.cnf`) to include all necessary SANs:

```ini  theme={null}
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```

After configuring the CSR with the SANs, sign the certificate using your CA certificate and key. Specify the final certificate parameters in your kube-apiserver configuration, as shown in the configuration snippet below:

```bash  theme={null}
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \
  --etcd-servers=https://127.0.0.1:2379 \
  --event-ttl=1h \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/apiserver-kubelet-client.crt \
  --kubelet-client-key=/var/lib/kubernetes/apiserver-kubelet-client.key \
  --kubelet-https=true \
  --runtime-config=api/all \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key \
  --v=2
```

<Callout icon="lightbulb" color="#1CB2FE">
  Ensure that all DNS names and IP addresses used by the API server are correctly listed in the SAN section of your OpenSSL configuration.
</Callout>

***

## 6. Kubelet Certificates

The kubelet is a critical component running on each node, managing node-specific operations and secure communication with the API server. For this purpose, every node needs its own certificate and key pair. When generating these certificates, name them according to the node's identity (e.g., node01, node02, node03).

It is also a best practice to generate a separate certificate for the node when it acts as a client to the API server. This certificate should include an identity format such as "system:node" to ensure the API server can assign the appropriate group membership (e.g., `system:nodes`).

## View Certificate Details

> Learn to inspect and verify certificates in a Kubernetes cluster, covering both manual setups and automated configurations like kubeadm.

### Understanding Your Cluster Setup

* If you deploy a cluster from scratch, you may generate and configure all certificates manually (as explored in a previous lesson).
* If you use an automated provisioning tool like kubeadm, certificate generation and configuration are handled for you. In this case, Kubernetes components are deployed as pods instead of OS services.

#### Native Service Deployment

When Kubernetes components are deployed as native services, you can review service files to understand the certificate configuration. For example, inspect the kube-apiserver service file:

```bash  theme={null}
cat /etc/systemd/system/kube-apiserver.service
[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=172.17.0.32 \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --event-ttl=1h \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certfile=/var/lib/kubernetes/kubelet-client.crt \\
  --kubelet-client-key=/var/lib/kubernetes/kubelet-client.key \\
  --kubelet-https=true \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/kube-apiserver-key.pem \\
  --v=2
```

#### Deployment Using kubeadm

When using kubeadm, components such as the kube-apiserver are defined as pods in manifest files. For example, view the kube-apiserver pod manifest:

```yaml  theme={null}
cat /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
      - kube-apiserver
      - --authorization-mode=Node,RBAC
      - --advertise-address=172.17.0.32
      - --allow-privileged=true
      - --client-ca-file=/etc/kubernetes/pki/ca.crt
      - --disable-admission-plugins=PersistentVolumeLabel
      - --enable-admission-plugins=NodeRestriction
      - --enable-bootstrap-token-auth=true
      - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
      - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
      - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
      - --insecure-port=0
      - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
      - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
      - --proxy-client-certfile=/etc/kubernetes/pki/apiserver-kubelet-client.crt
      - --proxy-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
      - --request-timeout=30s
```

### Creating a Certificate Inventory

When performing a certificate health check, it’s essential to create a checklist—perhaps using a spreadsheet—to record details such as:

* Certificate file paths
* Configured names and alternate names
* Associated organizations
* Certificate owners
* Certificate authorities (issuers)
* Expiration dates

Begin by examining configuration files (such as the kube-apiserver manifest located in `/etc/kubernetes/manifests`) to identify the certificate files in use.

For example, the kube-apiserver manifest might reveal the following options:

```yaml  theme={null}
spec:
  containers:
    - command:
      - kube-apiserver
      - --authorization-mode=Node,RBAC
      - --advertise-address=172.17.0.32
      - --allow-privileged=true
      - --client-ca-file=/etc/kubernetes/pki/ca.crt
      - --disable-admission-plugins=PersistentVolumeLabel
      - --enable-admission-plugins=NodeRestriction
      - --enable-bootstrap-token-auth=true
      - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
      - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
      - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
      - --etcd-servers=https://127.0.0.1:2379
      - --insecure-port=0
      - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
      - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
      - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
      - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
      - --secure-port=6443
      - --service-account-key-file=/etc/kubernetes/pki/sa.pub
      - --service-cluster-ip-range=10.96.0.0/12
      - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
      - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

### Inspecting Certificate Details

After identifying certificate files, use OpenSSL to decode them and check their details. For example, to review the API server certificate, run:

```bash  theme={null}
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

This command displays:

* The subject name and any alternate names
* The validity period (including expiry dates)
* The issuing certificate authority

Repeat this process for all certificates in your Kubernetes cluster. Ensure that:

* Certificate names and alternate names are correctly configured.
* Each certificate is associated with the appropriate organization.
* Certificates are issued by the correct certificate authority (e.g., kubeadm typically uses "Kubernetes" as the CA).
* None of the certificates have expired.

### Troubleshooting with Logs

When certificate issues are suspected, reviewing logs can provide valuable insights.

### For Clusters Using Native OS Services

Check service logs using system commands. For example, inspect etcd logs with:

```bash  theme={null}
journalctl -u etcd.service -l
```

Below is an example excerpt from etcd logs:

```plaintext  theme={null}
2019-02-13 02:53:28.144631 I | etcdmain: etcd Version: 3.2.18
2019-02-13 02:53:28.144680 I | etcdmain: Git SHA: eddf599c6
2019-02-13 02:53:28.144684 I | etcdmain: Go Version: go1.8.7
2019-02-13 02:53:28.144692 I | etcdmain: Go OS/Arch: linux/amd64
2019-02-13 02:53:28.144696 I | etcdmain: setting maximum number of CPUs to 4, total number of available CPUs is 4
2019-02-13 02:53:28.144734 N | etcdmain: the server is already initialized as member before, starting as etcd member...
2019-02-13 02:53:28.146651 I | etcdserver: name = master
...
WARNING: 2019/02/13 02:53:30 Failed to serve client requests on 127.0.0.1:2379
Failed to dial 127.0.0.1:2379: connection error: desc = "transport: authentication handshake failed: remote error: tls: bad certificate"; please retry.
```

#### For Clusters Using kubeadm

Since core components are deployed as pods, retrieve logs using:

* Running `kubectl logs <pod-name>` for pod-level logs.
* If the API server or etcd is down and `kubectl` is unresponsive, list all containers with:

  ```bash  theme={null}
  docker ps -a
  ```

  Then inspect container logs:

  ```bash  theme={null}
  docker logs <container-id>
  ```

## Certificates API

> This article explains managing certificates and the Certificate API in Kubernetes, detailing the lifecycle of certificate signing requests and automation for certificate rotation.

<Callout icon="lightbulb" color="#1CB2FE">
  As the number of users increases, manually signing certificate requests becomes impractical. Kubernetes addresses this challenge with a built-in Certificates API that automates CSR management and certificate rotation.
</Callout>

### Managing Certificate Signing Requests (CSRs)

The Kubernetes Certificates API allows users to submit their CSRs via an API call, creating a CertificateSigningRequest object. Administrators can then review and approve these requests using `kubectl` commands. Once approved, Kubernetes signs the certificate using the CA's key pair. The signed certificate is then available for extraction and distribution to the requesting user.
#### Step 1: User Generates Private Key and CSR

A user creates a private key and generates a certificate signing request using the following command:

```bash  theme={null}
openssl genrsa -out jane.key 2048
```

The user then sends the CSR to the administrator.

#### Step 2: Administrator Creates a CSR Object

The administrator creates a CertificateSigningRequest object with a manifest file. In the manifest, the `kind` is set to CertificateSigningRequest, and the `spec` section includes the encoded certificate signing request (CSR must be encoded in base64). Below is an example manifest:

```yaml  theme={null}
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  expirationSeconds: 600 # seconds
  usages:
    - digital signature
    - key encipherment
    - server auth
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNUUw0tLS0KTUl1Q1dEQ0NBVUFDQVFBd0V6RVJHQTFVdU0R6VjRkNHTQ0RzU0aU1yY3I0d11qYXl0c1RUVFRlQiVtNS0tLS0tLkRvd25nUIDhUnQ0dXJ0YW50YmlsZWdslNQZHYR0W1nNHh1RVFLdLtJPG0tLkFUTUJQS0w0UlRqS1JlTVUyZUl3bTJaSE44TG5NQ2czTWc9PQ==
```

Administrators can list pending CSRs with the following command:

```bash  theme={null}
kubectl get csr
```

The output may resemble:

```plaintext  theme={null}
NAME      AGE   SIGNERNAME                                   REQUESTOR                  REQUESTEDDURATION   CONDITION
jane      10m   kubernetes.io/kube-apiserver-client         admin@example.com          10m                 Pending
```

#### Step 3: Approving the CSR

To approve the CSR, run:

```bash  theme={null}
kubectl certificate approve jane
```

After approval, Kubernetes signs the CSR with the CA key pair, and the certificate is embedded in the CertificateSigningRequest object's YAML output as a base64 encoded string. You can decode it using base64 utilities to view the plain text certificate.

Below is an example output of a CertificateSigningRequest object:

```yaml  theme={null}
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  creationTimestamp: 2019-02-13T16:36:43Z
  name: new-user
spec:
  groups:
    - system:masters
    - system:authenticated
  expirationSeconds: 600
  usages:
    - digital signature
    - key encipherment
    - server auth
  username: kubernetes-admin
status:
  certificate: L$0tL1CRUdJTiBDRVJUSUZJQ0FURS9tL0t1SURDakNDQWZLZ0F3SUJBZ0lVRmwyQ2wXYXoxalW5M3JNVisreFRYQYouW3dnd0RWpL1pJaHZjTkFRRUwkQkFBd0ZVUnRVMVhQTFRUVF4TUhM1ZpHkdVpMjxkF1RncweE9UQ1NVE14TmpNeU1EQmFgdGl0dY0ZFBl2ajNuSXY3eFd3I1Rm5u440c0t520vXukwTFM5V29ge1hHZdWCMlEZ2FOMVVMRFBXTVhjN09FVnVjSk1k4weRUVtR5tD11zWeHVjS1h6g1dV0pMediMUGbXYFKWVKWMVmBjRVRTY3dod2xiO1ND0kLS0tL1F0kQg0V5VElSGUNBVEUt
  conditions:
    - lastUpdateTime: 2019-02-13T16:37:21Z
      message: This CSR was approved by kubectl certificate approve.
      reason: KubectlApprove
      type: Approved
```

### The Role of the Controller Manager

Within the Kubernetes control plane, components such as the API Server, Scheduler, and Controller Manager work together. However, all certificate-related operations—such as CSR approval and signing—are managed by the Controller Manager.

The Controller Manager includes dedicated controllers for CSR approval and CSR signing tasks. Since signing certificates requires access to the CA's root certificate and private key, its configuration specifies the file paths to these credentials. For example, the Controller Manager’s configuration file might include settings like the following:

```yaml  theme={null}
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
spec:
  containers:
  - command:
      - kube-controller-manager
      - --address=127.0.0.1
      - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
      - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
      - --controllers=*,bootstrapsigner,tokencleaner
      - --kubeconfig=/etc/kubernetes/controller-manager.conf
      - --leader-elect=true
      - --root-ca-file=/etc/kubernetes/pki/ca.crt
      - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
      - --use-service-account-credentials=true
```
## KubeConfig

> This article explains kubeconfig files in Kubernetes, focusing on certificate-based authentication and access management across multiple clusters.

### Certificate Authentication with curl and kubectl

Previously, we generated a certificate for a user and utilized the certificate along with a key to query the Kubernetes REST API for a list of pods. For instance, if your cluster is named "my kube playground," you can make a curl request to the API server as follows:

```bash  theme={null}
curl https://my-kube-playground:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

The API server then returns a response similar to this:

```json  theme={null}
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

```bash  theme={null}
kubectl get pods \
  --server https://my-kube-playground:6443 \
  --client-key admin.key \
  --client-certificate admin.crt \
  --certificate-authority ca.crt
```

The response in this case might be:

```bash  theme={null}
No resources found.
```

### Understanding the Kubeconfig File

By default, kubectl searches for a kubeconfig file named "config" in the \~/.kube directory. Once properly set up, you can simply execute:

```bash  theme={null}
kubectl get pods
```

and kubectl will automatically use the configurations defined within the file.

The kubeconfig file is organized into three key sections:

* **Clusters:** Define the Kubernetes clusters you need access to (e.g., development, production, or clusters hosted by different cloud providers).
* **Users:** Specify the user accounts and associated credentials (such as admin, dev, or prod users) that have permissions on the clusters.
* **Contexts:** Link a cluster with a user by specifying which user should access which cluster. A context can also define a default namespace.

Below is an example of a basic kubeconfig file in YAML format:

```yaml  theme={null}
apiVersion: v1
kind: Config
clusters:
- name: my-kube-playground  # values hidden…
- name: development
- name: production
- name: google
contexts:
- name: my-kube-admin@my-kube-playground
- name: dev-user@google
- name: prod-user@production
users:
- name: my-kube-admin
- name: admin
- name: dev-user
- name: prod-user
```

### Viewing and Customizing Your Kubeconfig

To view the current kubeconfig settings, run:

```bash  theme={null}
kubectl config view
```

This command outputs details about clusters, users, contexts, and the active context. An example output might look like:

```yaml  theme={null}
apiVersion: v1
kind: Config
current-context: kubernetes-admin@kubernetes
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://172.17.0.5:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

If you want to view a custom kubeconfig file, use the `--kubeconfig` option:

```bash  theme={null}
kubectl config view --kubeconfig=my-custom-config
```

A sample custom configuration may appear as follows:

```yaml  theme={null}
apiVersion: v1
kind: Config
current-context: my-kube-admin@my-kube-playground
clusters:
- name: my-kube-playground
- name: development
- name: production
contexts:
- name: my-kube-admin@my-kube-playground
- name: prod-user@production
users:
- name: my-kube-admin
- name: prod-user
```

To change the active context—for example, switching from the admin user to the production user—execute:

```bash  theme={null}
kubectl config use-context prod-user@production
```

After running this command, the kubeconfig is updated accordingly. The new configuration might look like this:

```yaml  theme={null}
apiVersion: v1
kind: Config
current-context: prod-user@production
clusters:
- name: my-kube-playground
- name: development
- name: production
contexts:
- name: my-kube-admin@my-kube-playground
- name: prod-user@production
users:
- name: my-kube-admin
- name: prod-user
```

Additional kubectl config commands can be used to update or delete entries as needed.

### Configuring Default Namespaces

Namespaces in Kubernetes help segment clusters into multiple virtual clusters. You can configure a context to automatically use a specific namespace. Consider the following kubeconfig snippet without a default namespace:

```yaml  theme={null}
apiVersion: v1
kind: Config
clusters:
- name: production
  cluster:
    certificate-authority: ca.crt
    server: https://172.17.0.51:6443
contexts:
- name: admin@production
  context:
    cluster: production
    user: admin
users:
- name: admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

To specify a default namespace (for example, "finance"), simply add the `namespace` field:

```yaml  theme={null}
apiVersion: v1
kind: Config
clusters:
- name: production
  cluster:
    certificate-authority: ca.crt
    server: https://172.17.0.51:6443
contexts:
- name: admin@production
  context:
    cluster: production
    user: admin
    namespace: finance
users:
- name: admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

When you switch to this context, kubectl will automatically operate within the specified namespace.

### Managing Certificates in Kubeconfig Files

For instance, specifying a full path looks like this:

```yaml  theme={null}
apiVersion: v1
kind: Config
clusters:
- name: production
  cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
```

Alternatively, you may embed the certificate data directly:

```yaml  theme={null}
apiVersion: v1
kind: Config
clusters:
- name: production
  cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    certificate-authority-data: LS0tLS1CRUdJTiBD...
```

To decode base64 encoded certificate data, use the following command:

```bash  theme={null}
echo "LS0...bnJ" | base64 --decode
```

The decoded output will resemble:

```text  theme={null}
-----BEGIN CERTIFICATE-----
MIICDCCAuCAQAwE...
-----END CERTIFICATE-----
```
