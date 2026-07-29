# idp-local

Local, no-cloud-cost sandbox for working through freeCodeCamp's guide
["How to Build an Internal Developer Platform: A Complete Guide to Backstage, ArgoCD, and Crossplane"](https://www.freecodecamp.org/news/how-to-build-an-internal-developer-platform-a-complete-guide-to-backstage-argocd-and-crossplane/).

The original guide targets a real EKS cluster with AWS-backed Crossplane
providers. Here we're substituting local/free equivalents so the concepts
(GitOps delivery, self-service infra via Crossplane, developer portal via
Backstage) can be tested without provisioning real AWS resources.

## Substitutions vs. the guide

| Guide uses                             | We use locally                                                                 | Why                                                                                                    |
| -------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| EKS cluster (1.28+, 3x m5.xlarge)      | `kind` cluster                                                                 | Any conformant Kubernetes cluster works; EKS isn't required for ArgoCD/Crossplane/Backstage themselves |
| Crossplane AWS provider (RDS, S3, IAM) | TBD: `provider-kubernetes`/`provider-helm`, or real AWS with minimal resources | Avoid AWS spend while still testing the GitOps/composition wiring                                      |
| Backstage on EKS                       | Backstage run locally via Yarn                                                 | Backstage is just a Node app regardless of what cluster it targets                                     |

## Current status

See [`docs/`](./docs) for the running log of what's been set up and why.

- [x] Local Kubernetes cluster (`kind`)
- [x] ArgoCD installed
- [x] Crossplane installed (via ArgoCD `Application`, not direct Helm)
- [x] `provider-kubernetes` installed (via ArgoCD `Application`) — will swap/add AWS provider once satisfied with the mechanics
- [x] `Database` XRD + Composition working end-to-end (claim → ConfigMap + Deployment), verified with a sample claim
- [x] Backstage scaffolded and running locally (`backstage/`, `yarn start` — frontend `:3000`, backend `:7007`)
- [x] "Provision a Database" Backstage template built (fetch:template → publish:github:pull-request → ArgoCD `database-claims` app) — not yet exercised end-to-end (needs push + `GITHUB_TOKEN`)

## Quick reference

- Cluster: `kind` cluster named `idp-local`, kubectl context `kind-idp-local`
- ArgoCD namespace: `argocd`
- ArgoCD UI: `https://localhost:8080` (via `kubectl port-forward service/argocd-server -n argocd 8080:443`)
- ArgoCD login: `admin` / see `docs/01-argocd-setup.md` (or re-fetch, see doc for command)
- ArgoCD CLI: `argocd` (installed to `~/.local/bin`), logged in against `localhost:8080`
