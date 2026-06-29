# Linking Dev & Prod clusters into the Admin ArgoCD

The **Admin cluster (mastervm)** is the permanent control plane: Jenkins (CI), Nexus
(registry) and ArgoCD (GitOps) run here and stay up. The standalone **Dev**
(`192.168.56.11`) and **Prod** (`192.168.56.12`) single-node clusters are built
separately by the team. This doc is the short, ready-to-run procedure to **link** each
of those clusters into the Admin ArgoCD once they exist — no Admin rebuild required.

Everything below is already prepared in Git:

- `k8s/argocd/applications.yaml` — the `nitte-demo` / `nitte-dev` / `nitte-prod` apps.
- `k8s/argocd/cluster-secret.template.yaml` — external-cluster registration template.

## State before linking (today)

Admin ArgoCD is up and reachable; `nitte-demo` runs on Admin and is **Synced + Healthy**.
`nitte-dev` / `nitte-prod` point at the in-cluster API and stay `Progressing` (their pods
are pinned to the not-yet-built worker nodes). That's expected — they go green the moment
they're re-pointed at the real Dev/Prod clusters.

## Step 1 — Create a manager token on the target cluster

Run on the **Dev** (or **Prod**) cluster:

```bash
kubectl create serviceaccount argocd-manager -n kube-system
kubectl create clusterrolebinding argocd-manager --clusterrole=cluster-admin \
  --serviceaccount=kube-system:argocd-manager
kubectl create token argocd-manager -n kube-system --duration=8760h
```

Copy the printed token. Note the API URL (`https://192.168.56.11:6443` for Dev).

## Step 2 — Register the cluster in Admin ArgoCD

On the **Admin** cluster, copy the template, fill in `server` + `bearerToken`, apply.
**Do not commit the filled copy** (it holds a token — it's gitignored):

```bash
cp k8s/argocd/cluster-secret.template.yaml cluster-dev.yaml
# edit cluster-dev.yaml: set name, server, bearerToken
kubectl apply -f cluster-dev.yaml
```

Confirm it shows up:

```bash
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster
```

(Alternatively, with the argocd CLI + the target kubeconfig context:
`argocd cluster add <dev-context>`.)

## Step 3 — Re-point the Application at the cluster

Either edit `k8s/argocd/applications.yaml` (change `destination.server` for `nitte-dev`
to `https://192.168.56.11:6443`, commit, push, let ArgoCD apply) **or** patch live:

```bash
kubectl patch application nitte-dev -n argocd --type merge \
  -p '{"spec":{"destination":{"server":"https://192.168.56.11:6443"}}}'
```

Repeat Steps 1–3 for Prod with `192.168.56.12:6443` and `nitte-prod`.

## Step 4 — Verify

```bash
kubectl get applications -n argocd
```

`nitte-dev` (and later `nitte-prod`) should progress to **Synced + Healthy** as ArgoCD
deploys the manifests onto the remote cluster. Nexus image pulls work as long as each
target node has the Nexus `registries.yaml` trust entry (see runbook §3.3).

## Why nothing else has to change

The Applications already reference the same Git repo/path and the same Nexus images.
Linking only swaps **where** ArgoCD applies them (`destination.server`). Jenkins keeps
building on push and pushing to Nexus regardless of how many clusters are linked.
