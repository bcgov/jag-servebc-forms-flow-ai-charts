# formsflow.ai v7.3.1 Upgrade — Progress Status

> **Superseded 2026-08-17:** the target was switched from open-source v7.3.1 to Enterprise Edition v8.2.5, and scope expanded to include `forms-flow-data-layer`/`forms-flow-documents-api`/`forms-flow-analytics`. The runbook has been updated to match — **phase numbers in Phase 3 below no longer match the current runbook** (three new components were inserted after 3.5, shifting everything from the old 3.6 onward). The factual record below (what was actually executed against `dev` on 2026-08-11) is still accurate; only the "resume checklist" and "next step" framing is stale. See the runbook's revision note at the top for the current state.

**Last updated:** 2026-08-11 (see supersession note above)
**Environment in progress:** `dev` (`a60371-dev`)
**Reference:** [`formsflow-ai-silver-runbook.md`](../formsflow-ai-silver-runbook.md) — phase numbers below match that document **as of 2026-08-11, not the current version**.

Session paused here — use this as the resume point.

## Status at a glance

| Phase | Status |
|---|---|
| -1 — Repo forks | Done (this repo, `bcgov/jag-servebc-forms-flow-ai-charts`, is itself the Phase -1 result; `origin`→fork, `upstream`→AOT, default branch `main`) |
| 0 — Pre-flight (`a60371-dev`) | Done — see [`2026-08-11-phase0-preflight-a60371-dev.md`](2026-08-11-phase0-preflight-a60371-dev.md) |
| 1 — New databases (Patroni + Mongo) | **Done** — see below |
| 2 — Chart checkout | Not started |
| 3 — Ordered `helm upgrade --install` | Not started |
| 4 — Verification / go-no-go | Not started |
| 5 — Rollback | N/A (nothing to roll back yet) |

## Key decisions made this pass (all confirmed with the user)

1. **Postgres/Mongo**: reuse existing in-namespace Patroni + `formio-mongodb-dev` instead of the chart's bundled Bitnami `postgresql-ha`/`mongodb` subcharts (those moved to the frozen/unpatched `bitnamilegacy` catalog in Aug 2025 — a real prod-readiness problem). `postgresql-ha.enabled=false` / `mongodb.enabled=false` on the `forms-flow-ai` install.
2. **Keycloak**: keep ServeLegal's own self-hosted `forms-flow-idm` Keycloak, not OCIO's shared BC Gov realm. OCIO's shared realm doesn't expose the custom role-mapping (`designer`/`reviewer`/`client`) formsflow.ai needs; a custom realm inside OCIO is possible but needs project-level approvals/audits that take time — the user may pursue that separately later. The new Keycloak is a **parallel/side-by-side install**, not an in-place upgrade of the old 18.0.2-legacy instance — new release name, new route (`forms-flow-idm-${NS}.${DOMAIN}`), new DB (`keycloakdb731`), fresh realm rebuilt via the admin API, not a DB migration. Old Keycloak keeps running untouched until cutover.
   - **Open risk, not yet addressed**: `forms-flow-idm`'s Bitnami `keycloak` subchart also pulls its image from the frozen `bitnamilegacy/keycloak:26.1.4-debian-12-r2` catalog — same class of problem as #1, but no workaround is planned yet. Fine for `dev`; needs a decision (pin a patched image, or accept the risk) before `test`/`prod`.
3. **Ingress class**: this account can't list/get `IngressClass` at cluster scope (RBAC). Resolved by reading the chart templates directly — `ingressClassName` is optional and safely omitted (OpenShift's router falls back to the cluster default). Decision: **omit `ingress.ingressClassName` from every Phase 3 `--set`**, rather than block on discovering the value.
4. **Storage**: default storage class (`netapp-file-standard`) is ~89% full (28.5Gi/32Gi). Only `forms-flow-ai`'s Redis StatefulSet creates a new PVC in this chart set (2Gi default). Decision: route it to `netapp-block-standard` (currently empty) via `--set redisExporter.persistence.storageClass=netapp-block-standard`, to avoid eating further into the tight default class.
5. **`DB_SUFFIX=731`** for this install (per the runbook's env parameterization block).

## Phase 1 — what was actually done (2026-08-11)

**Postgres, on `patroni-master`:**
- `bpmdb731` / `bpmuser731`
- `webapidb731` / `webapiuser731`
- `keycloakdb731` / `keycloakuser731`
- `servebcdb731` / `servebcuser731`
- K8s Secrets created in `a60371-dev`: `bpm-db-731`, `webapi-db-731`, `keycloak-db-731`, `servebc-db-731` (each with `host`/`port`/`database`/`username`/`password` keys, matching what Phase 3's chart installs expect).

**Mongo, on `formio-mongodb-dev`:**
- `formio731` / `formiouser731` created (readWrite role on `formio731`).
- `formio-mongodb-dev` was scaled 0→1 for the operation, then back to 0 afterward (it's normally idle along with most of the old stack).

**Plaintext passwords** for all of the above are saved to `~/Documents/ServeLegal/code/jag-servebc/openshift-export/v731-dev-new-secrets.env` (chmod 600, same gitignored directory as the rest of the old-stack export — **not** in this repo, not committed anywhere).

**Two runbook assumptions corrected during execution** (details in memory, not repeated in full here):
- `formio-mongodb-dev`'s pod selector label is `name=formio-mongodb-dev`, not `app=formio-mongodb-dev` as Phase 0's own check assumed.
- The Mongo root/admin credential is secret key `admin-password` + fixed username `admin` (classic OpenShift `mongodb-persistent` template convention), not `MONGO_INITDB_ROOT_USERNAME`/`PASSWORD`.

## ⚠️ Open risk to resolve before Phase 3.4 (forms-flow-forms)

The `formio-mongodb-dev` server is **MongoDB 3.6.3** (old — wire version 6). While creating the new Mongo user, `mongosh` (bundled in modern `mongo:5.0`+ client images) refused to connect at all: *"requires at least 8 (MongoDB 4.2)"*. Had to fall back to the legacy `mongo` shell (`mongo:4.4` image) to complete the task.

**Why this matters:** Phase 3.4 deploys the new `forms-flow-forms` chart, which bundles a modern Form.io Node.js image. If its MongoDB driver has the same ≥4.2 floor that `mongosh` just hit, **the new pod may not be able to connect to this Mongo instance at all** — a potential hard blocker for Phase 3.4, not currently scoped anywhere in the runbook (upgrading `formio-mongodb-dev` itself past 3.6 would be new, separate work).

**Next step when resuming:** check what MongoDB driver version ships in the v7.3.1 `forms-flow-forms` image before running Phase 3.4.

## Resume checklist

1. Decide/verify the Mongo driver-compatibility question above.
2. Proceed to **Phase 2** — clone `bcgov/jag-servebc-forms-flow-ai-charts` (this repo) into the scratch checkout dir, copy in `servebc-api` and `forms-flow-servebc-config` from the validated local reference deployment (`/Users/jaisethomas/Documents/ServeLegal/DockerDesktopBasedSetup/forms-flow-ai-charts/charts/`), commit them into the fork, then `helm dependency build` for `forms-flow-ai` and `forms-flow-idm`.
3. Continue into Phase 3 with the `--set` adjustments noted above (omit `ingress.ingressClassName`, add `redisExporter.persistence.storageClass=netapp-block-standard`, use the `-db-731` secrets from Phase 1).

## Related documents
- [`2026-08-11-phase0-preflight-a60371-dev.md`](2026-08-11-phase0-preflight-a60371-dev.md) — full Phase 0 detail.
- [`formsflow-ai-silver-runbook.md`](../formsflow-ai-silver-runbook.md) — full plan and command reference.
- `~/Documents/ServeLegal/code/jag-servebc/openshift-export/` — old-stack export (2026-08-06) + new v7.3.1 dev credentials (2026-08-11), outside this repo, gitignored, not duplicated here.
