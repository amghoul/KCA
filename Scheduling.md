# This is the Scheduling Notes
## Taints & Toleration
### On Node
- Imperative way
```bash
kubectl taint nodes node01 spray=mortein:NoSchedule
```

### On Pod
- No Imperative
- Declarative Way
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
  tolerations:
  - key: spray
    value: mortein
    effect: NoSchedule
    operator: Equal
```
### Notes:
- Taint on nodes and Tolerations on pods
- Taint effects are: NoSchedule, PreferNoSchulde, NoExecute

## NodeSelectors
### On Node
- we need to label the node
```bash
kubectl label nodes <node-name> <label-key>=<label-value>
```
```bash
kubectl label nodes node1 size=large
```
### ON Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
  nodeSelector:
    size: large
```
## Node Affinity
### ON Node
- you should but labels on node using `kubectl label` command
### on Pod:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
      affinity:            # affinity rules added from here
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: color
                operator: In
                values:
                - blue
```
### Notes:
- The operators are: In, NotIn, Exists, DoesNotExist, Gt and Lt

## Resource Limit
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: elephant
  namespace: default
spec:
  containers:
  - image: polinux/stress
    name: mem-stress
    resources:
      limits:
        memory: 20Mi
        cpu: 2
      requests:
        memory: 5Mi
        cpu: 1
```
## Limit Range
### For CPU
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default: # this section defines default limits
      cpu: 500m
    defaultRequest: # this section defines default requests
      cpu: 500m
    max: # max and min define the limit range
      cpu: "1"
    min:
      cpu: 100m
    type: Container

```
## Resource Quota
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
```
## DaemonSets
- DaemonSets ensure that exactly one copy of a pod runs on every node in your Kubernetes cluster. 
- When you add a new node, the DaemonSet automatically deploys the pod on the new node
```yaml 
# daemon-set-definition.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
        - name: monitoring-agent
          image: monitoring-agent
```
- Once your YAML file is ready, create the DaemonSet using the following command
```bash
kubectl create -f daemon-set-definition.yaml
```
- Verify the DaemonSet’s creation by running:
```bash
kubectl get daemonsets
```
- For more detailed information on your DaemonSet, use:
```bash
kubectl describe daemonset monitoring-daemon
```
## Static Pods
- Static pods are created directly by the kubelet without the intervention of the API server or other control plane components.
- You can place static pods in any directory on the host. The directory location is provided to the kubelet at startup by using the `--pod-manifest-path` option in kubelet service file:
```bash
ExecStart=/usr/local/bin/kubelet \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --network-plugin=cni \
  --register-node=true \
  --v=2
```
- Alternatively, you can specify a configuration file `--config` that includes the manifest directory path. For example:
```bash
# kubelet.service
ExecStart=/usr/local/bin/kubelet \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --config=kubeconfig.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --network-plugin=cni \
  --register-node=true \
  --v=2
```
```yaml
# kubeconfig.yaml
staticPodPath: /etc/kubernetes/manifests
```
- OR you can use `ps -aux | grep kubelet` to find the options

## Priority Classes
- Priority classes allow you to assign a numerical value to Pods, where a higher number indicates higher priority. 
- For user-deployed applications, the value can range from approximately -2 billion to +1 billion.
- Additionally, there is a reserved range for internal system-critical Pods (like the Kubernetes control plane) which can have values up to 2 billion.
- **Note** To check the current priority classes in your cluster, run the following command:
```bash
kubectl get priorityclass
```
### **Creating a New Priority Class:**
- **By imperative command:**
```bash
kubectl create priorityclass high-priority --value=100000 --preemption-pol
icy=PreemptLowerPriority
```
- **By Declarative command:**
```yaml
# priority-class.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000000
description: "Priority class for mission critical pods" # optional
globalDefault: true # optional
preemptionPolicy: PreemptLowerPriority # optional is the default. Other options: never
```
### On Pod:
```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 8080
  priorityClassName: high-priority
```
## Multiple Scheduler
### Configuring Schedulers with YAML
Below are examples of configuration files for both the default and a custom scheduler. Each YAML file uses a profiles list to define the scheduler’s name.
```yaml
# my-scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-scheduler
```
```yaml
# scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
```
### Deploying an Additional Scheduler
You can deploy an additional scheduler using the existing kube-scheduler binary, tailoring its configuration through specific service files.
- Begin by downloading the kube-scheduler binary:
```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler
```
- Create separate service files for each scheduler. For example, consider the following definitions:
```
# kube-scheduler.service
ExecStart=/usr/local/bin/kube-scheduler --config=/etc/kubernetes/config/kube-scheduler.yaml
```
- Define Scheduler Configuration Files
```yaml
# my-scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-scheduler
```
### Deploying the Custom Scheduler as a Pod
- In addition to running the scheduler as a service, you can deploy it as a pod inside the Kubernetes cluster. This method involves creating a pod definition that references the scheduler’s configuration file.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
    - name: kube-scheduler
      image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
      command:
        - kube-scheduler
        - --address=127.0.0.1
        - --kubeconfig=/etc/kubernetes/scheduler.conf
        - --config=/etc/kubernetes/my-scheduler-config.yaml
```
### Verifying Scheduler Operation
- To confirm which scheduler assigned a pod, review the events in your namespace:
```bash
kubectl get events -o wide
```
- If you encounter issues, view the scheduler logs with:
```bash
kubectl logs my-custom-scheduler --namespace=kube-system
```
## Admission Controllers
 Kubectl --> kubeapi-server --> authentication --> authorization --> admission controllers --> ceate pod
 - To view the admission controllers enabled by default, run:
```bash
kube-apiserver -h | grep enable-admission-plugins
```
or on kube-adm:
```bash
kubectl exec kube-apiserver-controlplan -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```
- To add an admission controller, update the `--enable-admission-plugins` flag on the Kube API server. In a kubeadm-based setup, this involves modifying the Kube API server manifest:
```bash
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --runtime-config=api/all \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --v=2 \\
  --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
```
- For `kubeadm-based` setups where the API server runs as a pod, the manifest might look like this:
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
    - --advertise-address=172.17.0.107
    - --allow-privileged=true
    - --enable-bootstrap-token-auth=true
    - --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
```
- To disable specific admission controller plugins, use the `--disable-admission-plugins` flag similarly.
- **Note:** Both the namespace auto-provision and namespace existence admission controllers are deprecated. They have been replaced by the namespace lifecycle admission controller, which enforces that requests to non-existent namespaces are rejected and protects default namespaces (default, kube-system, and kube-public) from deletion.

## Validating and Mutating Admission Controls
- Mutating, then Validating
- To instruct the API server to use your webhook for validations or mutations, create a `ValidatingWebhookConfiguration` or a `MutatingWebhookConfiguration` object. Below is an example configuration for a validating webhook that triggers on pod creation:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  clientConfig:
    service:
      namespace: "webhook-namespace"
      name: "webhook-service"
    caBundle: "Ci0tLS0tQk......tLS0K"
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"
```