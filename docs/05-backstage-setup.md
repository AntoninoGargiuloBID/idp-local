# 05 — Backstage

Status: scaffolded and running locally. Not yet wired to Crossplane/ArgoCD.

## What we did

1. **Checked prerequisites** against Backstage's documented requirements:
   Node 22/24 (Active LTS — had v24.11.1), Yarn via Corepack, `make`/`gcc`/`g++`
   for native deps like `isolated-vm`, plenty of disk/memory. All satisfied
   already.

2. **Enabled Corepack** (manages the pinned Yarn version Backstage requires):

   ```bash
   corepack enable
   ```

3. **Scaffolded the app** into `backstage/` (kept out of this repo's git
   history — see `.gitignore` — since it's a large, independently-versioned
   Node project, not platform infra):

   ```bash
   cd ~/bid-projects/idp-local
   npx @backstage/create-app@latest --path backstage
   # app name: idp-local-backstage
   ```

   This installed dependencies and ran an initial TypeScript build as part
   of scaffolding.

4. **Started it**:

   ```bash
   cd backstage
   yarn start
   ```

   Runs two processes: a Rspack dev server for the frontend (`:3000`) and
   the Backstage backend (`:7007`). Verified both:

   ```bash
   curl http://localhost:3000            # 200
   curl http://localhost:7007/api/catalog/health   # 401 AuthenticationError
   ```

   The backend's 401 is _expected_ — it correctly requires credentials on
   direct API calls; the frontend authenticates automatically via guest auth
   when you use the UI. It's a sign the backend is up and enforcing auth
   correctly, not a failure.

## Deviations from the guide

- Guide assumes Backstage is deployed to/accessed via the EKS cluster
  (behind ArgoCD, likely with ingress). Here it's just running as a plain
  local Node process via `yarn start` — Backstage itself has no dependency
  on where it runs; it just needs network access to the Kubernetes API,
  ArgoCD, etc., all of which are reachable from the host already.

## Next up

Wire Backstage to the platform we've already built:

- **Kubernetes plugin**: point Backstage at the `kind-idp-local` context so
  it can show cluster resources in the catalog.
- **Software Template**: create a Backstage template that generates and
  submits a `Database` claim (like `gitops/examples/database-claim-sample.yaml`)
  — this turns the manual `kubectl apply` we did in step 04 into a real
  self-service form a developer fills out in the Backstage UI.
- Optionally, an ArgoCD plugin/integration so Application sync status shows
  up in the Backstage catalog too.
