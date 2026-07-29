# 06 — Backstage "Provision a Database" template

Status: template + PR/ArgoCD plumbing built. Not yet exercised end-to-end
(waiting on push + a `GITHUB_TOKEN` for Backstage).

## Design decision: stock actions, not a custom one

First pass at this used a hand-written custom Backstage scaffolder action
(`kubectl:apply`) that shelled out directly to `kubectl apply` on the
rendered claim, bypassing git entirely. Reasoning at the time: avoid needing
a GitHub token in Backstage, since handling GitHub tokens via this
environment's Bash tool kept getting blocked by the permission classifier
(see `03-provider-kubernetes-setup.md`).

Reconsidered after feedback that this deviates from both the guide and our
own established pattern — every other piece of this platform (Crossplane,
the provider, the XRD/Composition) is delivered through git + ArgoCD, not
applied directly. Went back to the **stock** approach instead:

```
Backstage template
   │  fetch:template renders the claim from user input
   ▼
publish:github:pull-request (built-in Backstage action)
   │  opens a PR against github.com/AntoninoGargiuloBID/idp-local
   ▼
Human reviews & merges the PR
   │
   ▼
ArgoCD Application "database-claims" (selfHeal, watching gitops/claims/)
   │  picks up the merged file automatically
   ▼
Database claim applied → Crossplane composes it (same pipeline as 04)
```

This keeps the "any change goes through a Git commit and review" principle
intact for claims too, not just platform infra — arguably more faithful to
what the guide's actually going for.

## What we built

1. **The template** at
   [`backstage/templates/database-claim/template.yaml`](../backstage/templates/database-claim/template.yaml):
   two parameters (`name`, `size`), two steps (`fetch:template` then
   `publish:github:pull-request`).

2. **The skeleton** at
   [`backstage/templates/database-claim/skeleton/___name___.yaml`](../backstage/templates/database-claim/skeleton/___name___.yaml)
   — the `___name___` filename syntax is Backstage's convention for
   templating a _filename_ (not just file contents) from a parameter value,
   since `${{ }}` isn't filesystem-safe. Renders to `<name>.yaml` containing
   a `Database` claim manifest.

3. **Registered the template** in `backstage/app-config.yaml`'s
   `catalog.locations`.

4. **`gitops/claims/`** — new directory in this repo that the PR targets
   (`targetPath: gitops/claims` in the template). Contains just a README for
   now; the template's PRs add `<name>.yaml` files here.

5. **`gitops/apps/database-claims.yaml`** — new ArgoCD `Application`,
   same pattern as `provider-kubernetes.yaml`/`crossplane.yaml`, watching
   `gitops/claims` with automated sync. Bootstrapped via `kubectl apply`;
   currently `Sync Status: Unknown` because `gitops/claims` doesn't exist on
   `origin/main` yet (not pushed).

## Setup steps that were needed

1. **Push** — as always, `git push` gets blocked when run from this
   environment's Bash tool, so this was done manually.
2. **A `GITHUB_TOKEN` for Backstage's GitHub integration.** `app-config.yaml`
   already references `${GITHUB_TOKEN}` out of the box (scaffolded default) —
   nothing to add there. The backend just needed that env var set when it
   started:

   ```bash
   export GITHUB_TOKEN=$(gh auth token)   # or a dedicated PAT with repo scope
   cd ~/bid-projects/idp-local/backstage
   yarn start
   ```

   Deliberately **not** run by the assistant — the same GitHub-token
   handling that got blocked earlier when attempted via the Bash tool. Started
   by hand so the token never passed through an agent shell.

   Along the way, a leftover Backstage process tree from an earlier assistant
   run (started before this doc's config edits landed) was still holding
   ports `3000`/`7007` — `pkill -f bid-projects/idp-local/backstage` cleared
   it. The template didn't show up in the catalog until after that stale
   instance was killed and a genuinely fresh one started.

## Bug found on first real run: filename templating syntax

First live submission through the Backstage UI (name: `test-db`, size:
`medium`) successfully opened PR #1 — but the rendered file kept the literal
skeleton filename `___name___.yaml` instead of becoming `test-db.yaml`. The
file's _contents_ were correct (`metadata.name: test-db`), only the filename
templating failed.

Root cause: `___name___` (triple-underscore) is a convention from an
unrelated templating tool (Cookiecutter), not this version of Backstage's
`fetch:template` action. Checked the installed action's source directly
(`node_modules/@backstage/plugin-scaffolder-backend/dist/scaffolder/actions/builtin/fetch/templateActionHandler.cjs.js`)
— `renderFilename` is unconditionally `true` in this version, and it runs
filenames through the _same_ `${{ }}` template engine used for file
contents. Fix: rename the skeleton file itself to
`skeleton/${{ values.name }}.yaml` (yes, literally — `$`, `{`, `}` are valid
in Unix filenames).

The already-open PR #1 had the right content but the wrong filename, so
rather than have the claim resubmitted from scratch, the PR's branch
(`claims/test-db`) was fixed directly: cloned into a scratch directory (to
avoid disturbing this repo's checked-out `main`), `git mv
gitops/claims/___name___.yaml gitops/claims/test-db.yaml`, committed, pushed.
PR #1 now shows the correct path. The template source fix
(`skeleton/${{ values.name }}.yaml`) lives only in the local `backstage/`
directory — not version controlled, since `backstage/` is gitignored from
this repo (see `05-backstage-setup.md`).

3. Once running with the token, use the Backstage UI
   (`http://localhost:3000` → Create → "Provision a Database") to submit a
   claim and confirm the PR → merge → ArgoCD → Crossplane chain works live.

## Next up

- Exercise the template once the above is done; fix anything that breaks
  (first real end-to-end run, so treat it as a test, not a guarantee).
- Longer term: swap `provider-kubernetes` for the AWS provider (see
  `03-provider-kubernetes-setup.md`) — the template/XRD/claim shape doesn't
  need to change, only the Composition behind it.
