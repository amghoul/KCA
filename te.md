## Developing network policies

> This article explores advanced network policy configurations in Kubernetes to secure database pods by controlling ingress and egress traffic.

### Blocking All Ingress Traffic to the Database Pod

To begin, we block all incoming traffic to the database pod. This is achieved by creating a network policy that targets the database pod using labels and selectors. In our example, the database pod is labeled `role: db`. The following YAML file defines a policy that denies all ingress traffic by default since no specific ingress rules are provided:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
```

<Callout icon="lightbulb" color="#1CB2FE">
  If no ingress rules are specified, Kubernetes treats the policy as a complete block for incoming traffic.
</Callout>

### Allowing Ingress Traffic from the API Pod

The next step is to allow ingress traffic from the API pod on port 3306. Because responses to permitted traffic are automatically allowed, configuring an ingress rule is sufficient. In this rule, the source is defined by pod selectors (and optionally namespace selectors) while the destination port is specified.

For instance, to allow traffic only from pods labeled `name: api-pod` within namespaces labeled `prod`—and also permit traffic from a backup server with IP `192.168.5.10`—update the network policy as follows:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
  ports:
  - protocol: TCP
    port: 3306
```

In this configuration, the first entry under the `from` section restricts traffic to API pods in the production namespace (an AND condition). The second entry allows an external backup server by specifying its IP block.

<Callout icon="lightbulb" color="#1CB2FE">
  Combining pod selectors with namespace selectors ensures that the rule applies only to the intended pods within the correct namespace.
</Callout>

### Configuring Egress Traffic

In scenarios where the database pod must initiate outbound connections (for example, sending backups to an external server), an egress rule becomes necessary. To support both ingress and egress traffic, include `Egress` in the `policyTypes` and specify an egress rule.

The revised policy below permits the database pod to send traffic to a backup server at IP `192.168.5.10` on port `80` while still restricting all other outbound connections:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
  ports:
  - protocol: TCP
    port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```

<Callout icon="triangle-alert" color="#FF6B6B">
  Ensure that your egress rules cover all required outbound connections. Missing an egress rule may inadvertently block critical communication between your services.
</Callout>
