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
