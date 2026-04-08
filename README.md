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
## Tier-1: Critical security policies
### Policy 1: Deny privileged containers
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-privileged-containers
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-privileged-containers
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Privileged containers are not allowed"
        foreach:
          - list: "request.object.spec.containers[]"
            deny:
              conditions:
                all:
                  - key: "{{ element.securityContext.privileged || `false` }}"
                    operator: Equals
                    value: true

    - name: deny-privileged-init-containers
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Privileged init containers are not allowed"
        foreach:
          - list: "request.object.spec.initContainers[]"
            deny:
              conditions:
                all:
                  - key: "{{ element.securityContext.privileged || `false` }}"
                    operator: Equals
                    value: true
```
### Policy 2: Deny host network access
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-host-network
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-host-network
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Host network access is not allowed"
        deny:
          conditions:
            all:
              - key: "{{ request.object.spec.hostNetwork || `false` }}"
                operator: Equals
                value: true
```
### Policy 3: Require Non-root containers
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-non-root-containers
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Containers must not run as root"
        foreach:
          - list: "request.object.spec.containers[]"
            deny:
              conditions:
                any:
                  # Explicit root user
                  - key: "{{ element.securityContext.runAsUser || request.object.spec.securityContext.runAsUser || `0` }}"
                    operator: Equals
                    value: 0
                  # Explicitly disabling non-root
                  - key: "{{ element.securityContext.runAsNonRoot || request.object.spec.securityContext.runAsNonRoot || `false` }}"
                    operator: Equals
                    value: false

    - name: require-non-root-init-containers
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Init containers must not run as root"
        foreach:
          - list: "request.object.spec.initContainers[]"
            deny:
              conditions:
                any:
                  - key: "{{ element.securityContext.runAsUser || request.object.spec.securityContext.runAsUser || `0` }}"
                    operator: Equals
                    value: 0
                  - key: "{{ element.securityContext.runAsNonRoot || request.object.spec.securityContext.runAsNonRoot || `false` }}"
                    operator: Equals
                    value: false
```
## Tier-2: Operational Standards
### Policy 4: Require resource limits (Mutation + Validation)
```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-resource-limits
spec:
  rules:
    - name: add-default-resources
      match:
        resources:
          kinds:
            - Pod
      mutate:
        foreach:
          - list: "request.object.spec.containers[]"
            patchStrategicMerge:
              spec:
                containers:
                  - name: "{{ element.name }}"
                    resources:
                      requests:
                        +(cpu): "100m"
                        +(memory): "128Mi"
                      limits:
                        +(cpu): "500m"
                        +(memory): "256Mi"

    - name: add-default-resources-init
      match:
        resources:
          kinds:
            - Pod
      mutate:
        foreach:
          - list: "request.object.spec.initContainers[]"
            patchStrategicMerge:
              spec:
                initContainers:
                  - name: "{{ element.name }}"
                    resources:
                      requests:
                        +(cpu): "100m"
                        +(memory): "128Mi"
                      limits:
                        +(cpu): "500m"
                        +(memory): "256Mi"
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
