# Kyverno
## Installation
### Using Helm (recommended)
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
```
### or using kubectl
```bash
kubectl apply -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml
```
## Validation Modes
- **audit:** Logs violations but allows resources (good for testing)
- **enforce:** Rejects violations (production mode)
