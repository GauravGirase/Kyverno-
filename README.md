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
### Discovery-policy
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: audit-require-imagepullpolicy
spec:
  validationFailureAction: Audit # Audit only , other key is enforce
  rules:
    - name: check-imagepullpolicy
      match:
        resources:
          kinds:
            - Pod
          namespaceSelector:
            matchLabels:
              audit-enabled: "true"
      validate:
        message: "imagePullPolicy must be Always"
        pattern:
          spec:
            containers:
              - imagePullPolicy: Always
```
### Label the application namespaces for audit
```bash
kubectl label namespace default audit-enabled=true
```
### Deploy audit-only policy
```bash
kubectl apply -f 01-discovery-policy.yaml
```
## Test
### Bad Pod (missing imagePullPolicy)
```bash
apiVersion: v1
kind: Pod
metadata:
  name: test-bad
spec:
  containers:
    - name: nginx
      image: nginx
      imagePullPolicy: IfNotPresent
```
### Check policy
```bash
kubectl get policyreport --all-namespaces
```
### o/p:
```bash
NAMESPACE   NAME                                   KIND   NAME         PASS   FAIL   WARN   ERROR   SKIP   AGE
default     025ff02e-bf09-46e1-9d6b-806ebc7e7e72   Pod    test-bad-2   0      1      0      0       0      7s
default     9631c311-6f8f-41f4-bcba-2d47fa9ed04c   Pod    test-bad     1      0      0      0       0      4m34s
```
### Get detailed report
```bash
kubectl get policyreport -n default -o yaml
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
