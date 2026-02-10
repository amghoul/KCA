# Logging and monitoring
- Metrics Server is designed to be deployed once per Kubernetes cluster. It collects metrics from nodes and pods, aggregates the data, and retains it in memory. 
- Keep in mind that because Metrics Server stores data only in memory, it does not support historical performance data. For long-term metrics, consider integrating more advanced monitoring solutions.

## Deploying Metrics Server
- If you are experimenting locally with Minikube:
```bash
  minikube addons enable metrics-server
```
- For other environments, deploy Metrics Server by cloning the GitHub repository and applying its deployment files:
```bash
git clone https://github.com/kubernetes-incubator/metrics-server.git
kubectl create -f deploy/1.8+/
```
### Viewing Metrics
```bash
kubectl top node
```
```bash
kubectl top pod
```
## Managing Application Logs
```
kubectl logs -f event-simulator-pod <container-name-for-multi-container>
```

