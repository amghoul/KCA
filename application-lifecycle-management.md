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