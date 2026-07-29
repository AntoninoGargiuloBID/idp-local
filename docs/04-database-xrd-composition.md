# 04 — Database XRD + Composition (via `provider-kubernetes`)

Status: working end-to-end, verified with a sample claim.

## What this proves

The full self-service loop the guide is built around, minus Backstage and
minus real cloud: a developer (or, later, a Backstage template) submits a
`Database` claim → Crossplane composes it into real objects → those objects
come alive in the cluster.

```
Database claim (idp-local.dev/v1alpha1)
        │  matched by the XRD
        ▼
XDatabase composite resource
        │  Composition "database-kubernetes" (Pipeline mode)
        ▼
function-patch-and-transform renders 2 provider-kubernetes Objects
        │
        ▼
Real ConfigMap + Deployment (a fake "database") in the default namespace
```

## What we added

1. **`function-patch-and-transform`** ([`gitops/manifests/function-patch-and-transform.yaml`](../gitops/manifests/function-patch-and-transform.yaml))
   — a Crossplane Function package. Required because Crossplane v2 (we're on
   core 2.3.4) removed the old "inline patches" Composition mode — every
   Composition now runs as a `Pipeline` of one or more Functions.

2. **The `Database` XRD** ([`gitops/manifests/database-xrd.yaml`](../gitops/manifests/database-xrd.yaml))
   — defines the self-service API: `group: idp-local.dev`, claim kind
   `Database`, one parameter (`spec.parameters.size`: small/medium/large).

3. **The `database-kubernetes` Composition** ([`gitops/manifests/database-composition.yaml`](../gitops/manifests/database-composition.yaml))
   — the recipe. Uses `function-patch-and-transform` in `Pipeline` mode to
   render two `provider-kubernetes` `Object` resources from one claim: a
   ConfigMap (fake connection string + the `size` parameter) and a
   Deployment (a `postgres:16-alpine` pod standing in for a real database).

4. **A `ProviderConfig`** for `provider-kubernetes`, added to
   `provider-kubernetes.yaml` — tells the provider to manage the same
   cluster it's running in, via its own in-cluster ServiceAccount identity
   (`credentials.source: InjectedIdentity`), rather than an external
   kubeconfig.

5. **A sample claim** ([`gitops/examples/database-claim-sample.yaml`](../gitops/examples/database-claim-sample.yaml))
   — kept out of the ArgoCD-managed `gitops/manifests` path deliberately:
   this represents what a _developer_ (or Backstage template) submits, not
   platform infrastructure, so it isn't part of the platform team's
   GitOps-managed baseline.

## Bugs hit while testing (both fixed, worth knowing about)

- **String transform syntax**: `transforms[0].string` requires an explicit
  `type: Format` field alongside `fmt` — omitting it fails with `invalid
Function input: ...string.type: Required value`. Easy to miss since older
  Crossplane docs/examples don't always show it.
- **`ProviderConfig` not applied**: added the `ProviderConfig` to the YAML
  file but forgot to re-apply it after editing — Objects sat in
  `CannotConnectToProvider` until it was actually applied.
- **ArgoCD self-heal fighting manual testing**: while iterating on the
  Composition fix locally (via direct `kubectl apply`, not through git),
  ArgoCD's `automated.selfHeal` kept reverting the live `Composition` back
  to the last-pushed (still-broken) version from GitHub, causing confusing
  flapping between fixed/broken states. Temporarily disabled the
  `provider-kubernetes` Application's sync policy (`argocd app set
provider-kubernetes --sync-policy none`) to test cleanly — **re-enable
  this after pushing** so the app goes back to GitOps-managed:

  ```bash
  argocd app set provider-kubernetes --sync-policy automated --self-heal --auto-prune
  ```

## Verified result

```
kubectl get database.idp-local.dev -n default
# NAME         SYNCED   READY   CONNECTION-SECRET   AGE
# my-test-db   True     True                        20s

kubectl get configmap,deployment,pods -n default | grep test-db
# configmap/my-test-db-8txpw-config      ...
# deployment.apps/my-test-db-8txpw-db    1/1  1  1
# pod/my-test-db-8txpw-db-...            1/1  Running
```

Note the generated name suffix (`-8txpw`) — Crossplane names the Composite
Resource from the claim with an auto-generated suffix, so downstream patched
names are `<claim-name>-<suffix>-*`, not exactly `<claim-name>-*`. Expected
behavior, not a bug.

## Next up

- Push this branch, then re-enable automated sync on `provider-kubernetes`.
- When ready to move past the free/local provider: swap `provider-kubernetes`
  for the AWS provider and rewrite this Composition against real AWS
  resource types (e.g. RDS) — the `Database` claim's API doesn't need to
  change, which is the actual point of this exercise.
- Backstage: wire up a software template that generates/submits a `Database`
  claim like `database-claim-sample.yaml`, so this becomes a real self-service
  flow instead of a hand-applied YAML file.
