# GitOps demo app (`nitte-demo`)

A minimal, node-agnostic nginx workload used to demonstrate the full
**push → Jenkins build → Nexus → ArgoCD sync** loop on the single-node
(`mastervm`) cluster. The real `nitte-dev` / `nitte-prod` apps are pinned to the
now-removed `dev` / `prod` worker nodes, so they sit `Progressing` (pods
`Pending`). This app has **no nodeSelector**, so it schedules on `mastervm` and
shows green **Synced + Healthy**.

## Register the ArgoCD Application (run once on mastervm)

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nitte-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/pall111/HPE-merchendise-latest.git
    targetRevision: main
    path: k8s/demo
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated: { selfHeal: true, prune: true }
    syncOptions: [ CreateNamespace=true ]
EOF
```

## Verify

```bash
kubectl get application nitte-demo -n argocd
kubectl get pods -n demo
```

Expected: `nitte-demo  Synced  Healthy` and one `gitops-demo-*` pod `Running`.

## Demo the loop

Edit `k8s/demo/demo-app.yaml` (e.g. bump `replicas` to 2), commit, push.
Within ArgoCD's poll interval the app goes `OutOfSync → Syncing → Synced` and a
second pod appears — no `kubectl apply` needed.
