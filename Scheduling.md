# This is the Scheduling Notes
## Taints & Toleration
### On Node
- Imperative way
```bash
kubectl taint nodes node01 spray=mortein:NoSchedule
```

### On Pod
- No Imperative
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


