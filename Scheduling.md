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