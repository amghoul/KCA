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