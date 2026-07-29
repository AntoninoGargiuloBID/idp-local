# 01 — Local cluster + ArgoCD

Status: done.

## What we did

1. **Installed tooling** (user-writable, no sudo needed since `~/.local/bin` is
   already on `PATH`):
   - `kind` v0.27.0 → `~/.local/bin/kind`
   - `helm` v3.21.3 → `~/.local/bin/helm`
   - (`kubectl` v1.36.1 and Docker were already present)

2. **Created a local Kubernetes cluster with `kind`**, standing in for the
   guide's EKS cluster:

   ```bash
   kind create cluster --name idp-local
   ```

   This produced kubectl context `kind-idp-local`. Confirmed the node reached
   `Ready`:

   ```bash
   kubectl wait --for=condition=Ready node --all --timeout=90s
   ```

3. **Installed ArgoCD via its official Helm chart**, same install method the
   guide uses, just against `kind` instead of EKS:

   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   kubectl create namespace argocd
   helm install argocd argo/argo-cd --namespace argocd
   ```

4. **Verified all ArgoCD pods came up healthy** in the `argocd` namespace
   (application-controller, applicationset-controller, dex-server,
   notifications-controller, redis, repo-server, server — all `1/1 Running`).

   Note: `argocd-redis-secret-init` is a one-shot Job pod — it's expected to
   show `Completed`, not `Running`.

5. **Exposed the UI locally**:

   ```bash
   kubectl port-forward service/argocd-server -n argocd 8080:443
   ```

   Running in the background; UI reachable at `https://localhost:8080` (self-signed
   cert — accept the browser warning). WSL2 forwards `localhost` to Windows
   automatically, so this also opens directly from a Windows browser.

6. **Retrieved the initial admin password**:

   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```

   Login: `admin` / (output of the command above). ArgoCD's own docs recommend
   deleting this secret after first login and rotating the password.

7. **Installed the `argocd` CLI** (was missing — port-forward/UI worked, but
   the CLI binary itself hadn't been installed):

   ```bash
   curl -sSL -o ~/.local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
   chmod +x ~/.local/bin/argocd
   ```

   Logged in against the local port-forward:

   ```bash
   argocd login localhost:8080 --username admin --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)" --insecure
   ```

   `argocd app list` runs clean (empty — no Applications defined yet).

## Deviations from the guide

- No EKS, no `m5.xlarge` nodes, no AWS Cost Explorer setup — none of that is
  needed to test ArgoCD/Backstage/Crossplane mechanics locally.
- Otherwise the ArgoCD install itself matches the guide (same Helm chart, same
  namespace convention).

## Next up

Install Crossplane (see `02-crossplane-setup.md`, once written) — open
decision: point it at real AWS with minimal resources, or swap in
`provider-kubernetes`/`provider-helm` to stay fully local and free. See
`../README.md` substitutions table.
