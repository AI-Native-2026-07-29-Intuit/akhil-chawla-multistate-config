# Rogue Application rejection test (Task 2)

Prove AppProject `multistate` rejects Applications outside its allow-lists.

## Steps

```bash
export KUBECONFIG=/tmp/k3d-multistate-dev.yaml

kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rogue-bad-repo
  namespace: argocd
spec:
  project: multistate
  source:
    repoURL: https://github.com/evil/evil-config.git
    path: overlays/dev
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: multistate-dev
EOF

kubectl -n argocd get application rogue-bad-repo -o jsonpath='{range .status.conditions[*]}{.type}: {.message}{"\n"}{end}'
kubectl -n argocd delete application rogue-bad-repo
```

## Expected rejection (paste in PR #6)

`InvalidSpecError` similar to:

```
repository 'https://github.com/evil/evil-config.git' not permitted in project 'multistate'
```

Also try disallowed destination namespace (`rogue-ns`) with a permitted repo to see destination rejection.
