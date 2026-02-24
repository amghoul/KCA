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
## Authorization

> This article explores authentication and authorization in Kubernetes, detailing user access, permissions, and various authorization mechanisms like RBAC and external tools.

When sharing a cluster across different organizations or teams using namespaces, authorization restricts users to their designated namespaces. Kubernetes supports multiple authorization mechanisms, including:

* Node Authorization
* Attribute-Based Authorization
* Role-Based Access Control (RBAC)
* Webhook Authorization
  
### Node Authorizor
The Kubernetes API Server is the central component accessed by both management users and internal components, such as kubelets, which retrieve and report metadata about services, endpoints, nodes, and pods.

Requests from kubelets—typically using certificates with names prefixed by `"system:node"` as part of the `system:nodes` group—are authorized by a special component known as the `node authorizer`. 

<Callout icon="lightbulb" color="#1CB2FE">
  Kubernetes supports several authorization strategies to meet diverse security requirements. Always select the most appropriate mechanism for your cluster’s needs.
</Callout>

### Attribute-Based Authorization

Attribute-based authorization associates specific users or groups with a defined set of permissions. For example, you can grant a user called "dev-user" permissions to view, create, and delete pods. This is achieved by creating a policy file in JSON format and passing it to the API server. Consider the following example policy file:

```json  theme={null}
{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"user": "dev-user-2", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"group": "dev-users", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"user": "security-1", "namespace": "*", "resource": "csr", "apiGroup": "*"}}
```

Each time security requirements change, you must manually update this policy file and restart the Kube API Server. This manual process can be tedious and set the stage for more streamlined methods such as Role-Based Access Control (RBAC).

### Role-Based Access Control (RBAC)

RBAC simplifies user permission management by defining roles instead of directly associating permissions with individual users. For example, you can create a "developer" role that encompasses only the necessary permissions for application deployment. Developers are then associated with this role, and modifications in user access can be handled by updating the role, affecting all associated users immediately.

RBAC is considered the standard method for managing access within a Kubernetes cluster.

### External Authorization Mechanisms

If you prefer managing authorization externally rather than with built-in Kubernetes mechanisms, third-party tools like [Open Policy Agent (OPA)](https://www.openpolicyagent.org/) are an excellent choice. OPA can handle both admission control and authorization by processing user details and access requirements sent via API calls from Kubernetes. Based on OPA’s response, access is either granted or denied.

### AlwaysAllow and AlwaysDeny Modes

Kubernetes also supports two basic authorization modes:

* **AlwaysAllow:** Permits all requests without performing any authorization checks.
* **AlwaysDeny:** Denies all requests.

These modes are configured using the `authorization-mode` option on the Kube API Server and are crucial when determining which authorization mechanism is active. In cases where no mode is specified, AlwaysAllow is used by default.

Below is an example configuration using AlwaysAllow:

```bash  theme={null}
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=AlwaysAllow \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --kubelet-client-key=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --service-node-port-range=30000-32767 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key \\
  -v=2
```

You can also specify a comma-separated list of multiple authorization modes. For example, to configure node authorization, RBAC, and webhook authorization, set the parameter as follows:

```bash  theme={null}
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC,Webhook \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key \\
  --v=2
```

When multiple modes are configured, each request is processed sequentially in the order specified. For example, a user’s request is first evaluated by the node authorizer. If the request does not pertain to node-specific actions and is consequently denied, it is then passed to the next module, such as RBAC. Once a module approves the request, further checks are bypassed and the user is granted access.
## Role Based Access Controls

> This article explains how to implement Role-Based Access Controls in Kubernetes, including creating roles, role bindings, and verifying permissions.

### Creating a Role

To define a role, create a YAML file that sets the API version to `rbac.authorization.k8s.io/v1` and the kind to `Role`. In this example, we create a role named **developer** to grant developers specific permissions. The role includes a list of rules where each rule specifies the API groups, resources, and allowed verbs. For resources in the core API group, provide an empty string (`""`) for the `apiGroups` field.

For instance, the following YAML definition grants developers permissions on pods (with various actions) and allows them to create ConfigMaps:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

Create the role by running:

```bash  theme={null}
kubectl create -f developer-role.yaml
```

<Callout icon="lightbulb" color="#1CB2FE">
  Both roles and role bindings are namespace-scoped. This example assumes usage within the default namespace. To manage access in a different namespace, update the YAML metadata accordingly.
</Callout>

### Creating a Role Binding

After defining a role, you need to bind it to a user. A role binding links a user to a role within a specific namespace. In this example, we create a role binding named **devuser-developer-binding** that grants the user `dev-user` the **developer** role.

Below is the combined YAML definition for both creating the role and its corresponding binding:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Create the role binding using the command:

```bash  theme={null}
kubectl create -f devuser-developer-binding.yaml
```

### Verifying Roles and Role Bindings

After applying your configurations, it's important to verify that the roles and role bindings have been created correctly.

To list all roles in the current namespace, execute:

```bash  theme={null}
kubectl get roles
```

Example output:

```bash  theme={null}
NAME        AGE
developer   4s
```

Next, list all role bindings:

```bash  theme={null}
kubectl get rolebindings
```

Example output:

```bash  theme={null}
NAME                      AGE
devuser-developer-binding 24s
```

For detailed information about the **developer** role, run:

```bash  theme={null}
kubectl describe role developer
```

Sample output:

```bash  theme={null}
Name:         developer
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources           Non-Resource URLs   Resource Names   Verbs
  -----------         ------------------   --------------   ----
  ConfigMap           []                   []               [create]
  pods                []                   []               [get watch list create delete]
```

To view the specifics of the role binding:

```bash  theme={null}
kubectl describe rolebinding devuser-developer-binding
```

Example output:

```bash  theme={null}
Name:         devuser-developer-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:    Role
  Name:    developer
Subjects:
  Kind     Name      Namespace
  ----     ----      ---------
  User     dev-user
```

### Testing Permissions with kubectl auth

You can test whether you have the necessary permissions to perform specific actions by using the `kubectl auth can-i` command. For example, to check if you can create deployments, run:

```bash  theme={null}
kubectl auth can-i create deployments
```

This command might return:

```bash  theme={null}
yes
```

Similarly, to verify if you can delete nodes:

```bash  theme={null}
kubectl auth can-i delete nodes
```

Expected output:

```bash  theme={null}
no
```

To test permissions for a specific user without switching user contexts, use the `--as` flag. Although the **developer** role does not permit creating deployments, it does allow creating pods:

```bash  theme={null}
kubectl auth can-i create deployments
# Output: yes
kubectl auth can-i delete nodes
# Output: no
kubectl auth can-i create deployments --as dev-user
# Output: no
kubectl auth can-i create pods --as dev-user
# Output: yes
```

You can also specify a namespace in your commands to verify permissions scoped to that particular namespace.

### Limiting Access to Specific Resources

In some scenarios, you may want to restrict user access to a select group of resources. For example, if you have multiple pods in a namespace but only intend to provide access to pods named "blue" and "orange," you can utilize the `resourceNames` field in the role rule.

Start with a basic role definition without any resource-specific restrictions:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "update"]
```

Then, update the rule to restrict access solely to the "blue" and "orange" pods:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "update"]
  resourceNames: ["blue", "orange"]
```

## Cluster Roles

> This article explains Kubernetes cluster roles, their management, and the distinction between namespaced and cluster-scoped resources.

### Namespaced vs Cluster-Scoped Resources

For cluster-scoped resources, you do not specify a namespace when creating them. Common examples include:

* Nodes
* Persistent Volumes
* Certificate Signing Requests
* Namespace objects

To determine which resources are namespaced and which are cluster-scoped, execute the following commands:

```bash  theme={null}
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

### Managing Cluster Roles

Cluster roles operate similarly to regular roles but are intended for cluster-scoped resources. For example, you might define a cluster administrator role that grants permissions to view, create, or delete nodes, or a storage administrator role that oversees persistent volumes and claims.

#### Example: Cluster Administrator Role

Below is an example YAML definition for a cluster administrator role:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

After creating the cluster role, you can link a user or group to it using a ClusterRoleBinding. This binding associates the defined permissions with the specified user or group.

#### Example: Cluster Role Binding

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

In this binding example, the `subjects` section specifies the user (in this case, `cluster-admin`), while the `roleRef` section links to the previously defined `cluster-administrator` role. To create the role binding, you would typically use a `kubectl create` command.

<Callout icon="lightbulb" color="#1CB2FE">
  While cluster roles and bindings are mainly used to manage cluster-scoped resources, they can also be applied to namespace-scoped resources. When used in this way, the assigned user gains access to those resources across all namespaces, unlike namespaced roles, which limit access to a specific namespace.
</Callout>

### Default Cluster Roles

By default, Kubernetes creates several cluster roles when the cluster is first set up. These default roles are ready to use and cover various administrative tasks.

## Service Accounts

> This guide explains Kubernetes service accounts, their creation, usage, and security enhancements in recent versions.

There are two main types of accounts in Kubernetes:

* **User Accounts:** Designed for human users like administrators or developers.
* **Service Accounts:** Intended for machine-to-machine interactions or application-specific tasks. For instance, monitoring tools like Prometheus use a service account to query the Kubernetes API for performance metrics, while Jenkins uses one for deploying applications.

### Example: A Kubernetes Dashboard Application

Consider an example: "my Kubernetes dashboard," a basic dashboard application built with Python. This application retrieves a list of Pods from a Kubernetes cluster by sending API requests and subsequently displays the results on a web page. To authenticate its API requests, the application uses a dedicated service account.

#### Creating a Service Account

To create a service account named `dashboard-sa`, run:

```bash  theme={null}
kubectl create serviceaccount dashboard-sa
```

To view all service accounts, use:

```bash  theme={null}
kubectl get serviceaccount
```

The output will appear similar to:

```bash  theme={null}
NAME           SECRETS   AGE
default        1         218d
dashboard-sa   1         4d
```

Upon creation, Kubernetes automatically generates a service account token stored as a Secret and links it to the account. To inspect the details of your service account and its token, execute:

```bash  theme={null}
kubectl describe serviceaccount dashboard-sa
```

Expected output:

```bash  theme={null}
Name:                dashboard-sa
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dashboard-sa-token-kbbdm
Tokens:              dashboard-sa-token-kbbdm
Events:              <none>
```

To examine the token itself, view the corresponding Secret:

```bash  theme={null}
kubectl describe secret dashboard-sa-token-kbbdm
```

Sample output:

```bash  theme={null}
Name:                dashboard-sa-token-kbbdm
Namespace:           default
Labels:              <none>
Type:                kubernetes.io/service-account-token
Data
  token: eyJhbGciOiJSUzI1NiIsImtpZCI6Ij...  (truncated for privacy)
```

This token serves as the authentication bearer token for accessing the Kubernetes API. For example, using curl:

```bash  theme={null}
curl https://192.168.56.70:6443/api -k \
--header "Authorization: Bearer eyJhbgG…"
```

<Callout icon="lightbulb" color="#1CB2FE">
  In your custom dashboard application, you would typically place the token into the appropriate configuration field to enable API authentication.
</Callout>

#### Automatic Mounting of Service Account Tokens

When deploying third-party applications (such as a custom dashboard or Prometheus) on a Kubernetes cluster, you can have Kubernetes automatically mount the service account token as a volume into the Pod. This token is typically available at the path: `/var/run/secrets/kubernetes.io/serviceaccount`.
Every namespace includes a default service account that is automatically injected into Pods. For example, consider the following simple Pod manifest using a custom dashboard image:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```

After creating this Pod, running:

```bash  theme={null}
kubectl describe pod my-kubernetes-dashboard
```

will reveal a volume mounted from a Secret (usually named something like `default-token-xxxx`). You might see an excerpt similar to:

```bash  theme={null}
Name:           my-kubernetes-dashboard
Namespace:      default
Status:         Running
IP:             10.244.0.15
Containers:
  nginx:
    Image:        my-kubernetes-dashboard
    Mounts:       /var/run/secrets/kubernetes.io/serviceaccount from default-token-j4hkv (ro)
Volumes:
  default-token-j4hkv:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-j4hkv
    Optional:    false
```

Inside the Pod, listing the contents of the service account directory shows files such as the `token` file containing the bearer token:

```bash  theme={null}
kubectl exec -it my-kubernetes-dashboard -- ls /var/run/secrets/kubernetes.io/serviceaccount
kubectl exec -it my-kubernetes-dashboard -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

#### Using a Different Service Account

By default, Pods use the `default` service account. To assign a different service account—like the previously created `dashboard-sa`—update your Pod definition to include the `serviceAccountName` field:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  serviceAccountName: dashboard-sa
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
```

<Callout icon="lightbulb" color="#1CB2FE">
  Remember that you cannot modify the service account of an existing Pod. To use a new service account, delete and recreate the Pod. Deployments will automatically roll out new Pods when changes are made to the Pod template.
</Callout>

After deploying the updated manifest, running:

```bash  theme={null}
kubectl describe pod my-kubernetes-dashboard
```

will show that the new service account is now in effect, with volume mounting information reflecting the token for `dashboard-sa` (e.g., `dashboard-sa-token-kbbdm`).

If you wish to disable the automatic mounting of the service account token, set `automountServiceAccountToken` to `false` in the Pod specification:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  automountServiceAccountToken: false
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```

### Changes in Kubernetes Versions 1.22 and 1.24

Prior to Kubernetes v1.22, service account tokens were automatically mounted from Secrets without an expiration date. Starting with v1.22, the TokenRequest API (KEP-1205) was introduced to generate tokens that are audience-bound, time-bound, and object-bound—enhancing security significantly.

Below is an example Pod definition using a projected volume sourced from the TokenRequest API:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
    - image: nginx
      name: nginx
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-6mtg8
          readOnly: true
  volumes:
    - name: kube-api-access-6mtg8
      projected:
        defaultMode: 420
        sources:
          - serviceAccountToken:
              expirationSeconds: 3607
            path: token
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
          - downwardAPI:
              items:
                - fieldRef:
                    apiVersion: v1
                    fieldPath: metadata.namespace
```

Starting with Kubernetes v1.24, Kubernetes no longer automatically creates non-expiring service account tokens stored as Secrets. Instead, after creating a new service account, you must generate a token explicitly with:

```bash  theme={null}
kubectl create token dashboard-sa
```

This command produces a token with an expiry (by default, one hour from creation). You can verify and decode this token using tools like jq or [jwt.io](https://jwt.io):

```bash  theme={null}
jq -R 'split(".") | select(length > 0) | .[0] | @base64 | fromjson' <<< <TOKEN>
```

If necessary (though not recommended), you can still create a non-expiring token by manually creating a Secret. Ensure the service account exists first:

```yaml  theme={null}
apiVersion: v1
kind: Secret
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
type: kubernetes.io/service-account-token
```

<Callout icon="triangle-alert" color="#FF6B6B">
  It is highly recommended to use the TokenRequest API to generate tokens, as API-generated tokens provide additional security features such as expiry, audience restrictions, and improved manageability.
</Callout>

## Image Security

> This article covers best practices for securing container images during deployment, including naming conventions, private registries, and Kubernetes configuration.

### Understanding Container Image Naming

Let’s start by examining a simple pod definition file that deploys an Nginx container:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

Notice the image name "nginx". This follows Docker’s image naming convention. When a repository name is provided without a user or account, Docker defaults to the "library" account. In this example, "nginx" is interpreted as "library/nginx", which represents Docker’s official image maintained by a dedicated team that follows industry best practices.

If you create your own account and build custom images, you should update the image name accordingly. For instance:

```yaml  theme={null}
image: your-account/nginx
```

By default, Docker pulls images from Docker Hub (with the DNS name docker.io) if no other registry is specified. The registry is a centralized storage where images are pushed during creation or updates, and subsequently pulled during deployment.

### Private Registry Usage

For projects that require enhanced security and privacy, you might opt for private registries. Many popular cloud service providers—such as AWS, Azure, and GCP—offer private registries built into their platforms. Alternatively, tools like [Google Container Registry](https://cloud.google.com/container-registry) (gcr.io) are frequently used for Kubernetes-related images and testing purposes.

When referencing an image from a private registry, the full image path should be specified. For example:

```yaml  theme={null}
image: docker.io/library/nginx
```

#### Authentication for Private Registries

Accessing private repositories requires prior authentication. Start by logging into your private registry using the Docker CLI:

```bash  theme={null}
docker login private-registry.io
```

After you provide your credentials, you should see a confirmation similar to this:

```plaintext  theme={null}
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: registry-user
Password:
WARNING! Your password will be stored unencrypted in /home/vagrant/.docker/config.json.
Login Succeeded
```

### Configuring Kubernetes Pods for Private Registries

To pull an image from a private registry within a pod, specify the full image path in your pod definition. For example:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: private-registry.io/apps/internal-app
```

Since Kubernetes worker nodes rely on the Docker runtime for image retrieval, they must be provided with the appropriate credentials. This is achieved by creating a Kubernetes secret of type Docker registry. Execute the following command to create the secret:

```bash  theme={null}
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=registry-user@org.com
```

Once the secret is created, reference it in your pod specification using the `imagePullSecrets` section:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: private-registry.io/apps/internal-app
  imagePullSecrets:
    - name: regcred
```

<Callout icon="lightbulb" color="#1CB2FE">
  When the pod is created, the Kubelet on the worker node will use the credentials stored in the secret to authenticate and pull the image from your private registry.
</Callout>

## Security Contexts

> This lesson explains how to secure containers in Kubernetes by configuring security settings at the pod and container levels.

```bash  theme={null}
docker run --user=1001 ubuntu sleep 3600
docker run --cap-add MAC_ADMIN ubuntu
```

These configurations help manage the security of Docker containers, and similar settings are available in Kubernetes.

In Kubernetes, containers are always encapsulated in pods. You can define security settings either at the pod level, which affects all containers in the pod, or at the container level where the settings apply specifically to one container. Note that if the same security configuration is set at both the pod and container levels, the container-specific settings take precedence over the pod-level configurations.

Below is an example of a pod definition that configures the security context at the container level, using an Ubuntu image that runs the `sleep` command:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```

This configuration instructs Kubernetes to run the container as user ID 1000 and adds the `MAC_ADMIN` capability. If your goal is to enforce these security settings across all containers within a single pod, you can define the security context at the pod level instead.

<Callout icon="lightbulb" color="#1CB2FE">
  For more hands-on practice with viewing, configuring, and troubleshooting security contexts in Kubernetes, check out the available lab resources.
</Callout>

By leveraging security contexts, you enhance the security posture of your containerized applications in Kubernetes. For additional guidance, you may find the following resources useful: