# formsflow.ai v7.3.1 Upgrade — Progress Status

> **Superseded 2026-08-17:** the target was switched from open-source v7.3.1 to Enterprise Edition v8.2.5, and scope expanded to include `forms-flow-data-layer`/`forms-flow-documents-api`/`forms-flow-analytics`. The runbook has been updated to match — **phase numbers in Phase 3 below no longer match the current runbook** (three new components were inserted after 3.5, shifting everything from the old 3.6 onward). The factual record below (what was actually executed against `dev` on 2026-08-11) is still accurate as a historical log of that specific session; only the "resume checklist" and "next step" framing was stale — see the **2026-08-17 update** immediately below for the current state instead. See the runbook's revision note at the top of `formsflow-ai-silver-runbook.md` for full detail.

## 2026-08-17 update — current status

**Environment in progress:** `dev` (`a60371-dev`)
**Reference:** [`formsflow-ai-silver-runbook.md`](../formsflow-ai-silver-runbook.md) — current version, EE v8.2.5.

| Phase | Status |
|---|---|
| -1 — Repo forks | `bcgov` forks (charts, OSS app, microfrontends) done. **`bcgov-c/jag-servebc-forms-flow-ai-ee` (private, EE source) still not obtained** — authorized but not yet created. Confirmed non-blocking: Phase 3's local component builds source directly from the AOT-upstream clone at `~/Documents/ServeLegal/code/forms-flow-ai-ee` (pinned near `v8.2.5`), not from the `bcgov-c` fork path. |
| 0 — Pre-flight (`a60371-dev`) | Done — see [`2026-08-11-phase0-preflight-a60371-dev.md`](2026-08-11-phase0-preflight-a60371-dev.md) |
| 1 — New databases (Patroni + Mongo) | **Fully done (1.1–1.4)**, including the EE-scope `forms-flow-analytics` DB (1.4) and the same-day Mongo rename below |
| 2 — Chart checkout | **Done** — this working directory is already the `bcgov` fork checkout (not a separate scratch clone); `helm dependency build` succeeded for `forms-flow-ai` and `forms-flow-idm`; bitnami repo resolves. Custom charts (`servebc-api`, `forms-flow-servebc-config`) and the `secrets.yaml` Mongo wiring fix committed and pushed to `origin/feature/ee-8.2.5-migration` (`e9f732d`, `9a4edd6`). |
| 3 — Ordered `helm upgrade --install` | Not started — next step |
| 4 — Verification / go-no-go | Not started |
| 5 — Rollback | N/A (nothing deployed yet) |

**DB naming superseded again since 2026-08-11:** `DB_SUFFIX` moved from `731` to `_82`, applied to database names only — existing plain-named Postgres roles (`bpmuser`/`webapiuser`/`keycloakuser`) are reused rather than creating suffixed roles; only `servebcuser`/`servebcdb` are new (servebcdb unsuffixed, see the runbook's `DB_SUFFIX` note for why). The per-component K8s Secrets from the 2026-08-11 run below (`bpm-db-731` etc.) were **deleted** and replaced with one consolidated secret, `formsflow-db-82`.

**Mongo also renamed 2026-08-17** (it had been left on the old `731` naming when the rest of Phase 1 moved to `_82`): `formio731`/`formiouser731` dropped, replaced with database `formio_82` and a **plain, unsuffixed** user `formiouser` — matching the Postgres convention (database tagged, user not). Credentials now live in `formsflow-db-82` (`FORMIO_DB_HOST/PORT/NAME/USER/PASSWORD`) instead of only a shell variable, which is what caused the original `731` pass's password to be unrecoverable this time around.

**`forms-flow-web` architecture question (container vs. AOT's S3-static pipeline) is resolved**: staying on the self-hosted nginx container via the OSS-shaped Helm chart, not switching to S3/CDN.

**Resume checklist (current):**
1. Run Phase 3 in order starting with `forms-flow-ai` (3.1) — reads Mongo creds from `formsflow-db-82` automatically now, no manual `$FORMIO_DB_PASS` needed.
2. Obtain the `bcgov-c` EE fork when possible (Phase -1) — not urgent, doesn't block Phase 3.
3. Obtain the AOT private registry key when possible — lets later components switch from local OpenShift builds to pulling prebuilt images; also not urgent for `dev`.

---

## 2026-08-11 session (historical record — see update above for current status)

**Last updated:** 2026-08-11
**Environment in progress:** `dev` (`a60371-dev`)
**Reference:** [`formsflow-ai-silver-runbook.md`](../formsflow-ai-silver-runbook.md) — phase numbers below match that document **as of 2026-08-11, not the current version**.

Session paused here — this was the resume point at the time; superseded by the 2026-08-17 update above.

## Status at a glance (as of 2026-08-11)

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
5. **`DB_SUFFIX=731`** for this install (per the runbook's env parameterization block as of 2026-08-11 — **superseded 2026-08-17, moved to `_82`**, see the update at the top of this file).

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

## Resume checklist (as of 2026-08-11 — superseded, see the 2026-08-17 update at the top of this file for the current one)

1. Decide/verify the Mongo driver-compatibility question above.
2. Proceed to **Phase 2** — clone `bcgov/jag-servebc-forms-flow-ai-charts` (this repo) into the scratch checkout dir, copy in `servebc-api` and `forms-flow-servebc-config` from the validated local reference deployment (`/Users/jaisethomas/Documents/ServeLegal/DockerDesktopBasedSetup/forms-flow-ai-charts/charts/`), commit them into the fork, then `helm dependency build` for `forms-flow-ai` and `forms-flow-idm`.
3. Continue into Phase 3 with the `--set` adjustments noted above (omit `ingress.ingressClassName`, add `redisExporter.persistence.storageClass=netapp-block-standard`, use the `-db-731` secrets from Phase 1).

## Related documents
- [`2026-08-11-phase0-preflight-a60371-dev.md`](2026-08-11-phase0-preflight-a60371-dev.md) — full Phase 0 detail.
- [`formsflow-ai-silver-runbook.md`](../formsflow-ai-silver-runbook.md) — full plan and command reference.
- `~/Documents/ServeLegal/code/jag-servebc/openshift-export/` — old-stack export (2026-08-06) + new v7.3.1 dev credentials (2026-08-11), outside this repo, gitignored, not duplicated here.
