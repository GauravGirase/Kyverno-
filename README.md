# Kyverno
## Installation
### Using Helm (recommended)
```bash
# dev-values.yaml
replicaCount: 1
image:
  tag: "v1.10.0"
resources:
  requests:
    memory: 256Mi
    cpu: 100m
  limits:
    memory: 512Mi
    cpu: 500m
config:
  webhooks: # “Only apply admission webhooks to namespaces that have this label:”
    - namespaceSelector:
        matchExpressions:
          - key: kyverno.io/audit
            operator: In
            values:
              - "true"
```

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno \
 --namespace kyverno \
 --create-namespace \
 --values dev-values.yaml
```
### or using kubectl
```bash
kubectl apply -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml
```
## Validation Modes
- **audit:** Logs violations but allows resources (good for testing)
- **enforce:** Rejects violations (production mode)
## Use exclusions to exempt system namespaces:
```bash
validationFailureAction: enforce
   excludeResources:
     namespaces:
     - kyverno
     - kube-system
```
