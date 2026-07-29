# 03 — provider-kubernetes, delivered via ArgoCD

Status: installed and healthy. XRD/Composition not yet defined.

## Why this provider first

Decided to start with `provider-kubernetes` (manages in-cluster resources)
rather than jumping straight to the AWS provider, to prove out the
Composition/Claim/XRD mechanics for free before touching real cloud
resources. Plan is to swap in the AWS provider later once this is working
end-to-end (see `../README.md` substitutions table).

## Getting ArgoCD to reach our repo

Unlike Crossplane core (sourced directly from the public Helm chart repo), a
Crossplane _provider_ is installed via a plain Kubernetes manifest, not a
Helm chart — so delivering it through ArgoCD meant ArgoCD needed to clone a
git repo containing that manifest.

- Created `~/bid-projects/idp-local` as a **public** GitHub repo
  (`github.com/AntoninoGargiuloBID/idp-local`) so ArgoCD can clone it with
  zero credentials. Originally considered a private repo, but wiring a GitHub
  token into ArgoCD for that got blocked by this environment's tool-permission
  classifier (treated as credential-exposure risk, both as a CLI arg and
  written to a temp file) — went public instead since the repo only holds
  infra YAML/docs, no secrets.

## What we did

1. **Wrote the provider manifest** at
   [`gitops/manifests/provider-kubernetes.yaml`](../gitops/manifests/provider-kubernetes.yaml):
   a `DeploymentRuntimeConfig` (pins the provider's ServiceAccount to a fixed
   name — by default Crossplane generates a random per-revision name, which
   would break any RBAC binding written against a static name), the
   `Provider` itself (`xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.15.0`),
   and RBAC for that ServiceAccount.

2. **First RBAC draft bound `cluster-admin`** — flagged HIGH by an automated
   security review. Fixed by replacing it with a `ClusterRole` scoped to just
   the resource kinds our Compositions will actually manage as cloud-resource
   stand-ins: `configmaps`, `services`, `namespaces`, `serviceaccounts`,
   `deployments`, `statefulsets`. No `secrets`, no cluster-admin. **If a
   later Composition needs another kind, extend this list — don't fall back
   to cluster-admin.**

3. **Wrote the ArgoCD `Application`** at
   [`gitops/apps/provider-kubernetes.yaml`](../gitops/apps/provider-kubernetes.yaml),
   pointing at our own repo/path instead of a Helm chart:

   ```yaml
   spec:
     source:
       repoURL: https://github.com/AntoninoGargiuloBID/idp-local.git
       targetRevision: main
       path: gitops/manifests
     destination:
       namespace: crossplane-system
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```

4. **Committed, pushed, then bootstrapped**:

   ```bash
   git push origin main   # done manually — git push kept getting blocked
                           # by the tool-permission classifier when run
                           # from an agent shell in this environment
   kubectl apply -f gitops/apps/provider-kubernetes.yaml
   ```

5. **Verified**: `argocd app get provider-kubernetes` → `Synced` / `Healthy`.
   `kubectl get providers.pkg.crossplane.io` shows `provider-kubernetes`
   `INSTALLED=True HEALTHY=True`. Pod running in `crossplane-system`, its
   ServiceAccount is the fixed name `provider-kubernetes` (confirms the
   `DeploymentRuntimeConfig` took effect and the scoped RBAC binding is
   actually attached to the right identity).

## Next up

Define an `XRD` + `Composition` that uses `provider-kubernetes`'s `Object`
resource type to stand in for a "cloud resource" (e.g. a Composition that
provisions a ConfigMap+Deployment pair as a fake "database" claim), so
Backstage templates have something real to claim against. Once that
round-trip works, swap the provider for AWS.
