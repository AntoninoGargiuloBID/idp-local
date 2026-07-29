# Database claims

Populated by the Backstage "Provision a Database" template — each merged
PR adds one `<name>.yaml` file here (a `Database` claim). The `database-claims`
ArgoCD Application watches this directory and applies whatever lands here
automatically.

Don't hand-edit files here directly; use the Backstage template so changes
go through PR review, matching the rest of this platform's GitOps model.

To remove a claim, use the "Deprovision a Database" Backstage template
instead of deleting the file by hand — it opens a PR that removes the
`<name>.yaml` file. Once merged, the `database-claims` Application (which
has `prune: true`) deletes the claim from the cluster, and Crossplane
cascades that deletion to the composite and its underlying resource.
