# Application Lifecycle Management
## Rolling Updates and Rollbacks
- Check the rollout status:
```bash
kubectl rollout status deployment/myapp-deployment
```
- View the history of rollouts:
```bash
kubectl rollout history deployment/myapp-deployment
```
- Deployment Strategies: `Recreate` or `RollingUpdate` (default)
  
- you can update the `container image` directly using the following command:
```bash
kubectl set image deployment/myapp-deployment nginx-container=nginx:1.9.1
```
-  you can revert to the previous version using the rollback feature. To perform a rollback, run:
```bash
kubectl rollout undo deployment/myapp-deployment
```
## Commands and Arguments
- For Docker:
```bash
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```
- **Notes:**When you append an argument to the Docker run command, it overrides the default parameters defined by the CMD instruction in the Dockerfile.
  
- For Kubernetes:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]
```
- **Notes:**By specifying the args field in the pod definition file, the CMD instruction is overridden
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
    - name: ubuntu-sleeper
      image: ubuntu-sleeper
      command: ["sleep2.0"]
      args: ["10"]
```
- **Notes:**Remember that specifying the command in a pod definition replaces the Dockerfile’s ENTRYPOINT entirely, while the args field only overrides the default parameters defined by CMD.

## Configure Environment Variables in Applications
- For Dockers:
```bash
docker run -e APP_COLOR=pink simple-webapp-color
```
- For Kubernetes:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      env:
        - name: APP_COLOR
          value: pink
```
- To define an `environment variable` directly, use:
```yaml
env:
  - name: APP_COLOR
    value: pink
```
- To reference a `ConfigMap` for the environment variable, update the definition as follows:
```yaml
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: color
```
- Similarly, to source the environment variable from a `Secret`, configure it like this:
```yaml
env:
  - name: APP_COLOR
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: color
```
## Configure ConfigMaps in Applications
- 
Using `Environment Variables` in a Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 80
      env:
        - name: APP_COLOR
          value: blue
        - name: APP_MODE
          value: prod
```
- modify the pod definition to use the `envFrom` property:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 80
      envFrom:
        - configMapRef:
            name: app-color
```
- As a `volume`
```yaml
envFrom:
  - configMapRef:
      name: app-config

env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR

volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```

### Creating ConfigMaps
- Imperative Approach:
```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MOD=prod
```
- Declarative Approach:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```
- Create the ConfigMap in your cluster with:
```bash
kubectl create -f config-map.yaml
```
- Once a ConfigMap is created, you can list it using:
```bash
kubectl get configmaps
```
To check the stored configuration data, use the describe command:
```bash
kubectl describe configmaps
```
## Secrets
- Secrets encode data using Base64. Although it provides obfuscation, it is not a substitute for encryption.
- **Imperative commands**:
```bash
kubectl create secret generic app-secret \
  --from-literal=DB_Host=mysql \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=paswd
```
- **Declarative Creation of a Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```
Apply the definition with the following command:
```bash
kubectl create -f secret-data.yaml
```
- Converting Plaintext to `Base64`:
```bash
echo -n 'mysql' | base64
echo -n 'root' | base64
echo -n 'paswrd' | base64
# Output: cGFzd3Jk
```
- If you need to decode an encoded value, use the `base64 --decode` command:
```bash
echo -n 'bXlzcWw=' | base64 --decode
echo -n 'cm9vdA==' | base64 --decode
echo -n 'cGFzd3Jk' | base64 --decode
# Output: paswrd
```
- Injecting Secrets into a Pod:
```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    envFrom:
    - secretRef:
        name: app-secret
```
- **Mounting Secrets as Files:**
Alternatively, mount the Secret as files within a `volume`. Each key in the Secret becomes a separate file:
```yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret
```
- After mounting, listing the directory contents should display each key as a file:
```bash
ls /opt/app-secret-volumes
# Output: DB_Host  DB_Password  DB_User
```
## Multi Container Pods
### Co-located containers
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
    - name: log-agent
      image: log-agent
``` 
### Regular init containers:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
  initContainers:
    - name: check-db
      image: check-db
      command: [..]
```
### Sidecar Containers:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
  initContainers:
    - name: check-db
      image: check-db
      command: [..]
      restartPolicy: Always
```
## Manual & Autoscaling
### Manual Horizontal Scaling
- To monitor the pod’s resource consumption manually, you might run:
```bash
$ kubectl top pod my-app-pod
NAME         CPU(cores)   MEMORY(bytes)
my-app-pod   450m         350Mi
```
- To manually scale the deployment:
```bash
$ kubectl scale deployment my-app --replicas=3
```
- Below is a sample deployment configuration for this scenario:
```bash
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
            cpu: "250m"
          limits:
            cpu: "500m"
```
### Automated Scaling with Horizontal Pod Autoscaler
#### using imperative comamnd:
```bash
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10
```
- To check the status of your HPA, run:
```bash
kubectl get hpa
```
- If you need to remove the autoscaler later, simply run:
```bash
kubectl delete hpa my-app
```
#### Declarative HPA Configuration
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
### Updates In place Resize of Pods
- This in-place update feature is in alpha and requires manual activation via the feature flag. It is anticipated that the feature will eventually progress to beta and then achieve a stable release.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resize-demo
  namespace: qos-example
spec:
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.8
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired # Default, but explicit here
    - resourceName: memory
      restartPolicy: RestartContainer
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"

```
### Vertical Pod Autoscaling VPA
#### Manual Vertical Scaling
```yaml
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
            cpu: "250m"
          limits:
            cpu: "500m"
```
- **Notes:**Ensure that your cluster has the metrics server running to collect these metrics. When resource usage reaches a set threshold, you must manually update the deployment (e.g., using kubectl edit deployment) to adjust CPU requests and limits. This update replaces the running pod with a new one using the new configuration.
#### Vertical Pod Autoscaler (VPA)
- Note that the VPA is not included by default in Kubernetes clusters. You must deploy it separately from its GitHub repository
```bash
$ kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml

$ kubectl get pods -n kube-system | grep vpa
vpa-admission-controller-xxxx   Running
vpa-recommender-xxxx            Running
vpa-updater-xxxx                Running
```
- Configuring the VPA:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "my-app"
      minAllowed:
        cpu: "250m"
      maxAllowed:
        cpu: "2"
      controlledResources: ["cpu"]
```
- The VPA supports multiple update modes:
-- Off: Only provides recommendations without any modifications.
-- Initial: Applies recommendations only to newly created pods.
-- Recreate: Evicts pods running with suboptimal resource allocations, leading to pod restarts.
-- Auto: Currently behaves like “recreate” by evicting pods to apply updated values. In the future, auto mode may support in-place updates without restarting pods.

- To view the VPA recommendations, run:
```bash
$ kubectl describe vpa my-app-vpa 
```