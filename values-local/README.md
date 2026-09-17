# values-local/ — local-only per-environment Helm values

**This entire directory is gitignored.** It exists on this machine only — never committed, because
`jag-servebc-forms-flow-ai-charts` is a **public** `bcgov` fork (the private licensed fork is
`bcgov-c/jag-servebc-forms-flow-ai-ee`, a different repo for the EE app source, not this charts
repo). Real secrets (DB passwords, client secrets, S3 keys, JWT secrets, the formio root
password, etc.) live in these files and must not end up in the public repo.

**This is an interim measure**, not a final answer — see the `dr-handbook-and-runbook-drift-audit`
memory for the fuller discussion (Kustomize+ArgoCD, like RSBC's `tenant-gitops-be78d6`, is the
more complete fix but a bigger lift). Revisit once a real secrets-management story exists
(a proper vault, sealed-secrets, ArgoCD+SOPS, etc.) — at that point these files should migrate
there instead of staying as loose local YAML.

## Layout

`values-local/<env>/<chart-name>.yaml` — one file per chart per environment, containing exactly
what that chart's live Helm release actually has recorded (`helm get values <release>`), captured
2026-09-16 for `dev`. Only `dev/` exists so far.

## Usage

```bash
helm upgrade --install <release> charts/<chart> -f values-local/dev/<chart>.yaml
```

This replaces long inline `--set` chains — the point is that "what's actually supposed to be
running" becomes a readable, diffable file instead of scattered `--set` flags in a runbook or
shell history, and a future `helm upgrade --reuse-values` for an unrelated reason won't silently
carry forward stale values the way it has twice already (see
`formsflow-formio-root-cred-and-formio-url-bugs` and `formsflow-ee-v826-upgrade` memories).

## Keeping these current

**These files are only as good as the last time someone updated them.** Whenever a live
`helm upgrade --set ...` or an imperative `oc set image`/`oc set resources`/`oc set env` is
applied directly to a release, update the matching file here too — otherwise this directory
itself becomes exactly the kind of stale reference it was created to replace. Sanity-check with
`helm get values <release>` before trusting a file here is still accurate.
