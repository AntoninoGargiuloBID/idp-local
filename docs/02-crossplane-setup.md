# 02 — Crossplane, delivered via ArgoCD

Status: core installed. Providers not yet configured.

## Correction from the first attempt

Initially installed Crossplane with a direct `helm install` — that works, but
misses the guide's actual point: **Crossplane should be delivered by ArgoCD**,
not installed by hand, so its lifecycle (upgrades, config changes) is
GitOps-managed like everything else on the platform. Undid the direct install
and redid it as an ArgoCD `Application`.

```bash
helm uninstall crossplane -n crossplane-system
kubectl delete namespace crossplane-system
```

## What we did

1. **Turned `~/bid-projects/idp-local` into a git repo** so the platform's
   declarative config has somewhere to be committed and reviewed, per the
   guide's "any change goes through a Git commit and review" model:

   ```bash
   git init
   ```

2. **Wrote an ArgoCD `Application` manifest** at
   [`gitops/apps/crossplane.yaml`](../gitops/apps/crossplane.yaml) that sources
   the official Crossplane Helm chart directly (no need to vendor it into our
   own repo — the Application's `source` just points at
   `https://charts.crossplane.io/stable`), targets the `crossplane-system`
   namespace, and turns on `automated` sync (prune + self-heal) so future
   drift or manual `kubectl` edits get reconciled back to what's declared:

   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: crossplane
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://charts.crossplane.io/stable
       chart: crossplane
       targetRevision: 2.3.4
     destination:
       server: https://kubernetes.default.svc
       namespace: crossplane-system
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

3. **Committed the manifest**, then applied it once to bootstrap ArgoCD's
   awareness of it (this one-time `kubectl apply` of the root Application is
   standard GitOps practice — everything the Application _points to_ is then
   managed by ArgoCD, but the Application resource itself has to be
   registered somewhere first):

   ```bash
   git add gitops
   git commit -m "Add ArgoCD Application manifest for Crossplane"
   kubectl apply -f gitops/apps/crossplane.yaml
   ```

4. **Verified ArgoCD picked it up and synced it**:

   ```bash
   argocd app get crossplane
   ```

   Result: `Sync Status: Synced`, `Health Status: Healthy`. `crossplane` and
   `crossplane-rbac-manager` pods `1/1 Running` in `crossplane-system` — same
   end state as the direct Helm install, but now owned by ArgoCD.

## Open decision: which Crossplane provider?

Core Crossplane doesn't provision anything by itself — it needs a _provider_
installed on top (also delivered via an ArgoCD Application, same pattern as
above). The guide uses AWS providers (S3, RDS, IAM) against a real AWS
account. Options for this local sandbox:

- **Real AWS, minimal footprint** — install `provider-aws-s3` only, provision
  a single free-tier-eligible bucket. Real cloud calls, tiny/no cost, but
  needs real AWS credentials in the cluster.
- **`provider-kubernetes` / `provider-helm`** — Crossplane providers that
  manage in-cluster resources instead of cloud resources. Fully free/offline,
  good for proving out the Composition/Claim/XRD mechanics, but doesn't
  exercise real cloud provisioning.

Not yet decided — see `../README.md` substitutions table.

## Next up

Pick a provider option above, install it via another ArgoCD `Application`
manifest under `gitops/apps/`, then define a `CompositeResourceDefinition`
(XRD) + `Composition` so Backstage templates have something to claim against.
