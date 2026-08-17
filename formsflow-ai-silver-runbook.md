# formsflow.ai EE v8.2.5 — BC Gov Silver Cluster Runbook (dev / test / prod)

> **Revision note (2026-08-17):** this runbook originally targeted the open-source v7.3.1 release. The target was switched to Enterprise Edition (EE) v8.2.5 after read-only access to `AOT-Technologies/forms-flow-ai-ee` was granted the same day — no Helm charts had been installed against `dev` yet (only Phase 0 pre-flight and Phase 1 database provisioning), so there was nothing to unwind. Scope was also expanded at the same time to include `forms-flow-data-layer`, `forms-flow-documents-api`, and `forms-flow-analytics` (see revised Non-goals below). Everywhere below reflects EE 8.2.5 unless a step is explicitly marked as inherited from the OSS-era plan.

**Scope:** Replace the current formsflow.ai v4.0.8 DeploymentConfig-based stack with a fresh Enterprise Edition v8.2.5 install, deployed via Helm charts, in place (old undeployed before/as new comes up — no concurrent old+new). Run in `dev` end-to-end first; only proceed to `test`, then `prod`, once the previous environment has been verified per Phase 4's checklist.

**Source repos:** public formsflow.ai repos forked into the `bcgov` org, licensed enterprise code forked into a private `bcgov-c` repo — see Phase -1 below for the one-time fork setup.
- Upstream **EE** app source: `AOT-Technologies/forms-flow-ai-ee`, tag `v8.2.5` (pin to the tag, not `master`/`develop` — HEAD on those tracks ongoing dev toward 8.3.0-alpha) — forks to a **private** `bcgov-c/jag-servebc-forms-flow-ai-ee`, required by BC Gov compliance policy regardless of whether images are pulled prebuilt or built locally (see Phase -1). **Actively used this pass** as the source for any locally-built component images.
- Upstream OSS app source: `AOT-Technologies/forms-flow-ai`, branch `release/7.3.1` — forked to `bcgov/jag-servebc-forms-flow-ai`. **Superseded by the EE fork above** for this pass; kept around for reference only (diffing EE customizations against upstream OSS, e.g. via the EE repo's own `scheduled-os-to-ee-sync` workflow).
- Upstream charts: `AOT-Technologies/forms-flow-ai-charts`, chart set `v8.4.0` (chart versioning is independent of app versioning) — forked to `bcgov/jag-servebc-forms-flow-ai-charts`, **actively deployed from** starting Phase 2. No separate EE chart repo exists upstream — this fork holds a downloaded-and-customized copy of the OSS chart set, not a passive mirror, so it's where EE-specific customization (image registry, additional component charts) lives directly. The charts for `forms-flow-data-layer`, `forms-flow-documents-api`, and `forms-flow-analytics` already exist in this repo's `charts/` directory from the upstream chart set — no need to author new ones.
- Upstream microfrontends: `AOT-Technologies/forms-flow-ai-micro-front-ends` — forked to `bcgov/jag-servebc-forms-flow-ai-micro-front-ends` (not used this pass, reserved for the follow-up `ServiceFlow` UI porting work). **Unconfirmed for EE:** the EE repo ships its own `forms-flow-web-root-config` (micro-frontend root config) alongside `forms-flow-web` — whether EE's web architecture still needs this separate microfrontends repo, or supersedes it, hasn't been checked yet. Revisit before Phase 3.11 (`forms-flow-web`).
- Custom charts (this org's own, not upstream): `servebc-api`, `forms-flow-servebc-config` — already built and validated against a local deployment, live at `/Users/jaisethomas/Documents/ServeLegal/DockerDesktopBasedSetup/forms-flow-ai-charts/charts/`. Committed into the `jag-servebc-forms-flow-ai-charts` fork as part of Phase -1/2 so they travel with every future clone.

**Why Helm, not raw DeploymentConfigs:** the current v4.0.8 stack is hand-maintained DCs/BuildConfigs. Modern formsflow.ai ships as an official Helm chart set, and this org already has a validated local Kubernetes deployment of that chart set (`deploy-stack.sh`), including custom charts wrapping this repo's own `/api` (`servebc-api`) and cross-cutting ServeBC config (`forms-flow-servebc-config`, carrying CHES email + S3 + BPM timezone + custom theme). This runbook adapts that proven local deployment to OpenShift Silver.

**Image sourcing (revised 2026-08-17):** AOT's private Docker registry access keys are **authorized but not yet obtained** — don't assume they're in hand. Checked what that actually blocks before treating it as a hard dependency: it turns out most components don't need it at all. `forms-flow-analytics`'s chart already defaults to a **public** Docker Hub image (`docker.io/formsflow/redash:24.04.0`) and `forms-flow-idm`'s Keycloak comes from Bitnami's public image — neither touches EE source or the private registry regardless. `forms-flow-forms`'s chart defaults to a public OSS-tagged image (`docker.io/formsflow/forms-flow-forms:v7.3.0`) as a fallback if EE-specific forms features aren't needed yet. That leaves five components needing a real EE image one way or another: `forms-flow-api`, `forms-flow-bpm`, `forms-flow-web`, `forms-flow-data-layer`, `forms-flow-documents-api` (plus possibly `forms-flow-web-root-config`, pending the still-open microfrontends question). **Default for `dev` right now: build all five locally via OpenShift BuildConfig from the `bcgov-c` EE fork**, the same pattern already proven for the custom `servebc-api` (Phase 3.10) — all five already have Dockerfiles in the EE source, no reverse-engineering needed, and build-pod resource use is well within the compute headroom Phase 0 confirmed. `forms-flow-bpm` is the only heavy one (Maven multi-module build, ~5-10+ min, no inter-build dependency caching by default); the other four are small Python/Node builds, ~1-3 min each. Switch back to pulling AOT's prebuilt images once the registry key is actually obtained — worth doing before `test`/`prod` regardless, since it moves base-image patching back onto AOT rather than this project owning it long-term.

**Non-goals (explicitly out of scope for this pass — do not attempt):**
- Porting the custom Camunda Java extensions (Keycloak service-account token class, Redis-backed real-time task-event websocket push, `ApplicationAccessHandler`) currently in the v4.0.8 `forms-flow-bpm` overlay, **unless** the "Camunda/API may need customizations" question above resolves to include this — TBD, don't assume either way yet. Absent that, this install uses vanilla EE `forms-flow-bpm` — real-time task-list updates will not work; users see stock poll/refresh behavior until a follow-up project ports that code forward.
- Porting the custom `ServiceFlow` React UI (task list/filters) built against the old monolithic `forms-flow-web` — incompatible with the microfrontend architecture. Concrete, possibly citizen-facing gap: the `REACT_APP_PUBLIC_FORM_ID`/`DOCUMENT_TYPES`-driven public NCQ document-serving flow has no equivalent in the stock web chart. **Confirm with the team before a prod cutover** whether this needs to be resolved first.
- `forms-flow-mcp` — skipped this pass; add later if needed.
- ~~`forms-flow-data-layer`, `forms-flow-documents-api`, `forms-flow-analytics`~~ — **now in scope** as of the 2026-08-17 EE pivot (see Phase 3 additions below). `forms-flow-data-analysis-api` remains out of scope (not requested).

---

## Phase -1 — One-time repo fork setup (run once, not per environment)

Fork the upstream repos this project depends on into `bcgov`, prefixed `jag-servebc-`, matching the existing `bcgov/jag-servebc` app repo naming (licensed enterprise code goes to the private `bcgov-c` org instead — see the EE fork below). This decouples builds from AOT's live branches (which can move/force-push under you) and gives customizations a real, reviewable git history instead of the current build-time clone-and-overlay pattern.

Simplest path for a one-time task: use GitHub's native Fork flow in the web UI — it also gives you the proper "forked from AOT-Technologies/..." lineage badge and upstream-compare view, which the scripted `git push --mirror` approach doesn't.

For each of the three repos below, on GitHub.com:
1. Go to the upstream repo (e.g. `https://github.com/AOT-Technologies/forms-flow-ai-charts`).
2. Click **Fork** (top right).
3. Set **Owner** to `bcgov` (requires your GitHub account to have repo-create rights in that org — if the dropdown doesn't offer `bcgov`, you don't have that permission yet and need to request it first).
4. Set **Repository name** to the `jag-servebc-` prefixed name below.
5. **Uncheck** "Copy the `master` branch only" — you want all branches/tags available, not just the default, since Phase -1's next step pins a specific release branch and you may want others for reference later.
6. Click **Create fork**.

Repeat for:
- `AOT-Technologies/forms-flow-ai-charts` → `bcgov/jag-servebc-forms-flow-ai-charts` — **actively used** starting at Phase 2 below (this is what Phase 3's `helm upgrade --install` commands actually deploy from).
- `AOT-Technologies/forms-flow-ai` → `bcgov/jag-servebc-forms-flow-ai` — the OSS app source, superseded by the EE fork below for this pass. Kept as-is for reference/diffing.
- `AOT-Technologies/forms-flow-ai-micro-front-ends` → `bcgov/jag-servebc-forms-flow-ai-micro-front-ends` — not used this pass (see the "Unconfirmed for EE" note in the header above); this is where the custom `ServiceFlow` UI porting (flagged as follow-up) will eventually need to live.

**EE fork — active as of 2026-08-17, done the same way but into the private `bcgov-c` org:**
- Go to `https://github.com/AOT-Technologies/forms-flow-ai-ee`, click **Fork**, set **Owner** to `bcgov-c` (this is a *compliance requirement* for licensed code, not optional — separate from and in addition to the private Docker registry access already granted for prebuilt images), **Repository name** `jag-servebc-forms-flow-ai-ee`, uncheck "Copy the `master` branch only", **Create fork**.
- Read-only access to the upstream `AOT-Technologies/forms-flow-ai-ee` repo is sufficient to fork it — GitHub's Fork only needs read on the source and create-rights on the target org, not write access upstream.
- **Pin to the `v8.2.5` tag, not `master`/`develop`** — at last check, those track ongoing dev toward `v8.3.0-alpha`, the same trap avoided for the OSS `forms-flow-ai` fork below.

**BC Gov repos use `main`, not `master`** — AOT's upstream repos still default to `master`, so forking copies that in as-is. Don't just add a differently-named pin branch alongside a lingering `master`; rename the default branch to `main`, and make `main` *be* the pinned stable content directly (sourced from the actual release branch/tag you want, not just a raw copy of whatever upstream's `master` HEAD happens to be):

```bash
# --- forms-flow-ai-charts: upstream's `master` already IS the v8.4.0 release merge point (confirmed
# at fork time — its tip is "Merge pull request #207 from AOT-Technologies/release/8.4.0"), so main
# is sourced from master here. ---
git clone https://github.com/bcgov/jag-servebc-forms-flow-ai-charts.git /tmp/charts-init
cd /tmp/charts-init && git checkout master && git checkout -b main && git push origin main
cd - && rm -rf /tmp/charts-init

# --- forms-flow-ai (OSS app source, reference only this pass): pin release/7.3.1, NOT
# upstream's master/develop (which track ongoing dev) — main is sourced from that release branch. ---
git clone https://github.com/bcgov/jag-servebc-forms-flow-ai.git /tmp/app-init
cd /tmp/app-init && git checkout release/7.3.1 && git checkout -b main && git push origin main
cd - && rm -rf /tmp/app-init

# --- forms-flow-ai-ee (EE app source, actively used): pin the v8.2.5 tag, NOT master/develop
# (tracking v8.3.0-alpha at last check) — main is sourced from that tag. ---
git clone https://github.com/bcgov-c/jag-servebc-forms-flow-ai-ee.git /tmp/ee-init
cd /tmp/ee-init && git checkout v8.2.5 && git checkout -b main && git push origin main
cd - && rm -rf /tmp/ee-init

# --- forms-flow-ai-micro-front-ends: no specific version pin needed yet (not used this pass) —
# rename its default branch too, for consistency, straight off whatever master currently is. ---
git clone https://github.com/bcgov/jag-servebc-forms-flow-ai-micro-front-ends.git /tmp/mfe-init
cd /tmp/mfe-init && git checkout -b main && git push origin main
cd - && rm -rf /tmp/mfe-init
```

Then, on GitHub for each of the four forks: **Settings → Branches → switch the default branch to `main`**, then delete the old `master` branch once `main` is confirmed as default — leaving `master` around risks someone accidentally building off it later, silently drifting away from the pin.

Add `upstream` as a remote locally for periodic manual syncing (security patches, bug fixes) — the GitHub UI's "Sync fork" button only fast-forwards the *default* branch against upstream's *default* branch, which won't line up once your default is `main` sourced from a release branch/tag rather than upstream's `master`/`develop`. Forking trades "get upstream fixes for free" for "you control what lands," so plan on an occasional manual `git fetch upstream && git merge upstream/release/7.3.1` (or, for EE, `git fetch upstream && git merge upstream/v8.2.5` then re-tag when moving to a newer EE release) maintenance pass instead:
```bash
git remote add upstream https://github.com/AOT-Technologies/forms-flow-ai-charts.git   # repeat per repo, matching upstream URL
```

Phase 2 below is updated to clone from `bcgov/jag-servebc-forms-flow-ai-charts` instead of the AOT repo directly, reflecting this fork.

---

## 0. Environment parameterization

Every command block below reuses these. Set once per environment run — this is the only thing that changes between dev/test/prod.

```bash
# ---- EDIT THESE FOUR LINES PER ENVIRONMENT, THEN RUN EVERYTHING BELOW AS-IS ----
export ENV=dev                                    # dev | test | prod
export NS="a60371-${ENV}"
export DOMAIN="apps.silver.devops.gov.bc.ca"
export DB_SUFFIX="731"        # opaque generation tag, NOT a product version — see naming note below
# ---------------------------------------------------------------------------------

export TOOLS_NS="a60371-tools"
export CHARTS_DIR="$HOME/formsflow-charts-deploy"   # scratch checkout, see Phase 2
echo "Deploying formsflow.ai EE v8.2.5 to namespace: $NS (domain: $DOMAIN)"
```

**On `DB_SUFFIX` naming (revised 2026-08-17):** this was originally chosen as a stripped-down "7.3.1". After the target version changed mid-project (7.3.1 OSS → 8.2.5 EE, with more changes possible on future patches), that coupling turned out to be a bad idea — a version bump shouldn't force renaming databases. Going forward, treat `DB_SUFFIX` as an **opaque generation tag**, not a version string. `731` is being kept as-is for the databases already provisioned in `dev` (see Phase 1 — they're empty, harmless, and renaming them would be pure churn for no benefit) and is also used for the new `forms-flow-analytics` database added in this revision, so this first generation of `dev` databases stays internally consistent. If `dev` is ever torn down and freshly re-provisioned, bump to a new opaque tag (e.g. `_v2`, a date stamp) rather than trying to encode whatever the app version happens to be at the time. This applies to *this migration* — old-DC-based → new-Helm-based — specifically, because the old stack already occupies the plain, unsuffixed db/user names (confirmed from the `dev` export) and Postgres roles are cluster-global, so some distinguishing tag is unavoidable during the coexistence/verification window regardless of scheme. Once fully cut over to the new architecture, future version bumps (8.2.5 → 8.2.6, etc.) should go back to a **stable** db+user pair with a pre-upgrade backup as the safety net, not a new generation tag each time.

---

## Phase 0 — Pre-flight

**Status for `dev`: done (2026-08-11).** Full results in [`reports/2026-08-11-phase0-preflight-a60371-dev.md`](reports/2026-08-11-phase0-preflight-a60371-dev.md). Key outcomes folded into the commands below: this account has no cluster-scoped RBAC for `ingressclass` (resolved by omitting `ingress.ingressClassName` in Phase 3 — the charts treat it as optional), the default storage class (`netapp-file-standard`) is ~89% full so the new Redis PVC gets routed to `netapp-block-standard` instead, and `formio-mongodb-dev`'s pod selector is actually `name=formio-mongodb-dev` (the `app=` label below returns nothing even when the pod is healthy — corrected from the original draft).

```bash
# Log in (get the token from the OpenShift web console: your username dropdown -> Copy login command)
oc login --token=<token> --server=https://api.silver.devops.gov.bc.ca:6443
oc project "$NS"

# Confirm real storage class name for THIS namespace/cluster — do not assume
oc get storageclass
# ingressclass is typically NOT listable with a namespace-scoped account (cluster-scoped RBAC) —
# don't burn time on this if it 403s. ingress.ingressClassName is optional in these charts; Phase 3
# omits it and lets the cluster's default IngressClass apply.

# Confirm quota headroom before installing — this stack is heavier than a typical single-service app.
# Cross-check any tight storage-class quota against what the charts you're actually installing will
# request (grep chart templates for "kind: PersistentVolumeClaim\|VolumeClaimTemplate") rather than
# assuming the worst — most components here are stateless or externalized to Patroni/Mongo.
oc describe quota -n "$NS"

# Confirm Patroni (existing HA Postgres) is healthy before building anything on top of it
oc get statefulset patroni -n "$NS"
oc get pods -n "$NS" -l app.kubernetes.io/name=patroni -o wide
# Expect at least one pod Ready with label role=master. A degraded replica (as seen in a60371-dev
# at plan time — patroni-0 unhealthy, patroni-1 healthy+master) is NOT a blocker; a missing master
# IS a blocker — stop and fix Patroni first if so.

# Confirm the existing Form.io MongoDB DC exists (it may be scaled to 0 — that's expected, not a
# failure; Phase 1.2 scales it up when it's actually needed)
oc get pods -n "$NS" -l name=formio-mongodb-dev

# Snapshot current live config before touching anything (same pattern used for a60371-dev earlier —
# re-run for test/prod, each namespace needs its own export). Skip the "decode secrets" step for
# prod unless you specifically need those values surfaced — keys-only listing is enough to know
# what needs replacing.
mkdir -p "$HOME/openshift-export-${ENV}"
oc get dc -n "$NS" -o yaml > "$HOME/openshift-export-${ENV}/dc-snapshot.yaml"
oc get routes,svc -n "$NS" -o yaml > "$HOME/openshift-export-${ENV}/routes-svc-snapshot.yaml"
```

**Stop-and-confirm point:** this is a real namespace other people may use. Before proceeding past this point, confirm the maintenance window with the team — from Phase 1 onward this runbook starts creating new state and, at cutover, undeploying the running v4.0.8 stack.

---

## Phase 1 — New databases on existing Patroni + Mongo (fresh-install scope — new empty DBs, no data migration)

**Status for `dev`: 1.1–1.3 done (2026-08-11)**, for the original 4 components (bpm/webapi/keycloak/servebc). Full results and the two corrections below in [`reports/2026-08-11-upgrade-progress-status.md`](reports/2026-08-11-upgrade-progress-status.md). **1.4 (`forms-flow-analytics`) is new as of the 2026-08-17 scope expansion and not yet run.** Two things the original draft got wrong, already corrected below:
- The Mongo root credential is **not** `MONGO_INITDB_ROOT_USERNAME`/`MONGO_INITDB_ROOT_PASSWORD` — `formio-mongodb-dev` uses the classic OpenShift `mongodb-persistent` template convention instead: secret key `admin-password` paired with the fixed username `admin`.
- `mongosh` (bundled in `mongo:5.0`+ client images) refuses to connect — `formio-mongodb-dev` is running **MongoDB 3.6.3**, and modern drivers require ≥4.2. Use the legacy `mongo` shell (`mongo:4.4` image) instead. **This is a real open risk for Phase 3.4** (`forms-flow-forms`) if its bundled Node.js driver has the same floor — check before running that step.

### 1.1 Postgres — Camunda, webapi, Keycloak, servebc-api databases

```bash
# Get Patroni superuser creds (already exists in-namespace)
PATRONI_SUPERUSER=$(oc get secret patroni-creds -n "$NS" -o jsonpath='{.data.superuser-username}' | base64 -d)
PATRONI_SUPERPASS=$(oc get secret patroni-creds -n "$NS" -o jsonpath='{.data.superuser-password}' | base64 -d)

# Generate fresh passwords for the new app users (don't reuse anything from the old stack)
BPM_DB_PASS=$(openssl rand -base64 24)
WEBAPI_DB_PASS=$(openssl rand -base64 24)
KEYCLOAK_DB_PASS=$(openssl rand -base64 24)
SERVEBC_DB_PASS=$(openssl rand -base64 24)

oc exec -n "$NS" patroni-1 -- env PGPASSWORD="$PATRONI_SUPERPASS" psql -U "$PATRONI_SUPERUSER" -h localhost <<SQL
CREATE DATABASE bpmdb${DB_SUFFIX};
CREATE USER bpmuser${DB_SUFFIX} WITH PASSWORD '${BPM_DB_PASS}';
GRANT ALL PRIVILEGES ON DATABASE bpmdb${DB_SUFFIX} TO bpmuser${DB_SUFFIX};

CREATE DATABASE webapidb${DB_SUFFIX};
CREATE USER webapiuser${DB_SUFFIX} WITH PASSWORD '${WEBAPI_DB_PASS}';
GRANT ALL PRIVILEGES ON DATABASE webapidb${DB_SUFFIX} TO webapiuser${DB_SUFFIX};

CREATE DATABASE keycloakdb${DB_SUFFIX};
CREATE USER keycloakuser${DB_SUFFIX} WITH PASSWORD '${KEYCLOAK_DB_PASS}';
GRANT ALL PRIVILEGES ON DATABASE keycloakdb${DB_SUFFIX} TO keycloakuser${DB_SUFFIX};

CREATE DATABASE servebcdb${DB_SUFFIX};
CREATE USER servebcuser${DB_SUFFIX} WITH PASSWORD '${SERVEBC_DB_PASS}';
GRANT ALL PRIVILEGES ON DATABASE servebcdb${DB_SUFFIX} TO servebcuser${DB_SUFFIX};
SQL
```

> `patroni-1` is the pod name confirmed as current master in `a60371-dev` at plan time — **verify the actual master pod name for this namespace/run** via `oc get pods -n "$NS" -l app.kubernetes.io/name=patroni -L role` before running the above; connect to whichever pod carries `role=master`, or simpler/more robust: connect via the `patroni-master` Service host instead of `oc exec`-ing a specific pod:
> ```bash
> oc run psql-client --rm -i --restart=Never -n "$NS" --image=postgres:16-alpine -- \
>   env PGPASSWORD="$PATRONI_SUPERPASS" psql -U "$PATRONI_SUPERUSER" -h patroni-master <<SQL
> ... (same SQL block as above) ...
> SQL
> ```

### 1.2 Mongo — Form.io database

```bash
# formio-mongodb-dev is normally scaled to 0 — scale it up first, this will fail against a Service
# with no endpoints otherwise.
oc scale dc/formio-mongodb-dev -n "$NS" --replicas=1
oc wait --for=condition=ready pod -l name=formio-mongodb-dev -n "$NS" --timeout=120s

# Root credential: classic OpenShift mongodb-persistent template convention — fixed username
# "admin", password from secret key "admin-password" (NOT MONGO_INITDB_ROOT_USERNAME/PASSWORD).
MONGO_ADMIN_PASS=$(oc get secret formio-mongodb-dev-secret -n "$NS" -o jsonpath='{.data.admin-password}' | base64 -d)
FORMIO_DB_PASS=$(openssl rand -base64 24)

# The server is MongoDB 3.6.3 — mongosh (mongo:5.0+ images) refuses to connect ("requires at least
# MongoDB 4.2"). Use the legacy `mongo` shell via an older client image instead.
oc run mongo-client --rm -i --restart=Never -n "$NS" --image=mongo:4.4 -- \
  mongo "mongodb://admin:${MONGO_ADMIN_PASS}@formio-mongodb-dev:27017/admin" --eval "
    db.getSiblingDB('formio${DB_SUFFIX}').createUser({
      user: 'formiouser${DB_SUFFIX}',
      pwd: '${FORMIO_DB_PASS}',
      roles: [{ role: 'readWrite', db: 'formio${DB_SUFFIX}' }]
    })
  "

# Scale back down — it's normally idle along with the rest of the old stack.
oc scale dc/formio-mongodb-dev -n "$NS" --replicas=0
```

### 1.3 Create the Kubernetes Secrets each chart's `ExternalDatabase`/`externalDatabase` block expects

```bash
oc create secret generic bpm-db-${DB_SUFFIX} -n "$NS" \
  --from-literal=host=patroni-master \
  --from-literal=port=5432 \
  --from-literal=database=bpmdb${DB_SUFFIX} \
  --from-literal=username=bpmuser${DB_SUFFIX} \
  --from-literal=password="$BPM_DB_PASS"

oc create secret generic webapi-db-${DB_SUFFIX} -n "$NS" \
  --from-literal=host=patroni-master \
  --from-literal=port=5432 \
  --from-literal=database=webapidb${DB_SUFFIX} \
  --from-literal=username=webapiuser${DB_SUFFIX} \
  --from-literal=password="$WEBAPI_DB_PASS"

oc create secret generic keycloak-db-${DB_SUFFIX} -n "$NS" \
  --from-literal=host=patroni-master \
  --from-literal=port=5432 \
  --from-literal=database=keycloakdb${DB_SUFFIX} \
  --from-literal=username=keycloakuser${DB_SUFFIX} \
  --from-literal=password="$KEYCLOAK_DB_PASS"

oc create secret generic servebc-db-${DB_SUFFIX} -n "$NS" \
  --from-literal=host=patroni-master \
  --from-literal=port=5432 \
  --from-literal=database=servebcdb${DB_SUFFIX} \
  --from-literal=username=servebcuser${DB_SUFFIX} \
  --from-literal=password="$SERVEBC_DB_PASS"
```

Keep the plaintext passwords generated above somewhere durable for this run (a password manager, not committed anywhere) — you'll need `SERVEBC_DB_PASS`/user/host/name again explicitly in Phase 3's `forms-flow-servebc-config` install, since that chart creates its *own* secret from `--set database.*` values rather than reading the one above.

### 1.4 Postgres — `forms-flow-analytics` database (new, EE scope addition)

`forms-flow-analytics` already defaults `postgresql.enabled: false` and `redis.enabled: false` in its chart values — it's already externalized, no `--set` override needed for those toggles (unlike `forms-flow-ai`/`forms-flow-idm`, which need it set explicitly). Its Redis need is satisfied for free too: `externalRedis` already defaults to `redis://redis-exporter:6379/0`, which is `forms-flow-ai`'s own Redis service — no separate Redis to provision. Only a new Postgres database is needed, and this chart takes the connection as a single URL via a Secret (`externalPostgreSQLSecret`), not split host/user/password `--set` values like the other components.

`forms-flow-data-layer` and `forms-flow-documents-api` — the other two EE-scope additions — need **no new database at all**: `forms-flow-data-layer` reuses `forms-flow-api`'s own secret directly (`FORMSFLOW_API_DB_*` keys sourced from whatever secret `--set formsflow.webapi.secret=...` points at in Phase 3.5.5) plus the shared `forms-flow-ai` secret/configmap for the Form.io Mongo connection, and `forms-flow-documents-api` is stateless (Redis + URLs to already-running services only). Nothing to add here for either.

```bash
ANALYTICS_DB_PASS=$(openssl rand -base64 24)

oc run psql-client --rm -i --restart=Never -n "$NS" --image=postgres:16-alpine -- \
  env PGPASSWORD="$PATRONI_SUPERPASS" psql -U "$PATRONI_SUPERUSER" -h patroni-master <<SQL
CREATE DATABASE analyticsdb${DB_SUFFIX};
CREATE USER analyticsuser${DB_SUFFIX} WITH PASSWORD '${ANALYTICS_DB_PASS}';
GRANT ALL PRIVILEGES ON DATABASE analyticsdb${DB_SUFFIX} TO analyticsuser${DB_SUFFIX};
SQL

oc create secret generic analytics-db-${DB_SUFFIX} -n "$NS" \
  --from-literal=connectionString="postgresql://analyticsuser${DB_SUFFIX}:${ANALYTICS_DB_PASS}@patroni-master:5432/analyticsdb${DB_SUFFIX}"
```

Referenced in Phase 3 as `--set externalPostgreSQLSecret.name=analytics-db-${DB_SUFFIX} --set externalPostgreSQLSecret.key=connectionString`.

---

## Phase 2 — Chart checkout

```bash
mkdir -p "$CHARTS_DIR" && cd "$CHARTS_DIR"
git clone --branch main https://github.com/bcgov/jag-servebc-forms-flow-ai-charts.git .

# Port the two custom charts in from the validated local setup (not in the upstream repo, and not
# yet committed into the fork — do that once, as part of this same one-time setup, so future runs
# just `git clone` a fork that already has them: `git add charts/servebc-api charts/forms-flow-servebc-config
# && git commit -m "add ServeBC custom charts" && git push`)
cp -R /Users/jaisethomas/Documents/ServeLegal/DockerDesktopBasedSetup/forms-flow-ai-charts/charts/servebc-api ./charts/
cp -R /Users/jaisethomas/Documents/ServeLegal/DockerDesktopBasedSetup/forms-flow-ai-charts/charts/forms-flow-servebc-config ./charts/

# Only forms-flow-ai and forms-flow-idm have sub-chart dependencies to resolve
helm dependency build ./charts/forms-flow-ai
helm dependency build ./charts/forms-flow-idm

# Confirm the Bitnami Keycloak sub-chart still resolves (Postgres/Mongo are being externalized
# per the plan decision below, but forms-flow-idm's Keycloak container itself still comes from
# this dependency) — if this 404s, the chart tarball needs vendoring into an internal registry
# before continuing.
helm repo add bitnami https://charts.bitnami.com/bitnami 2>/dev/null
helm repo update
```

---

## Phase 3 — Ordered `helm upgrade --install`

Deployment order (dependency-correct, matches the chart repo's own documented order and the validated `deploy-stack.sh`, extended with the 2026-08-17 EE scope additions): `forms-flow-ai` (infra shell) → `forms-flow-servebc-config` → `forms-flow-idm` (Keycloak) → `forms-flow-forms` (Form.io) → `forms-flow-api` (webapi) → `forms-flow-data-layer` → `forms-flow-documents-api` → `forms-flow-analytics` → `forms-flow-bpm` (Camunda) → `servebc-api` (custom API) → `forms-flow-web`. The three new components all land right after `forms-flow-api` since each depends on api/forms/idm already being up (data-layer and documents-api directly reference their secrets/config; analytics is more loosely coupled but keeping the order consistent costs nothing).

**Decision, confirmed with the user: Postgres via existing Patroni, not the chart's bundled Bitnami `postgresql-ha`.** Bitnami's `postgresql-ha`/`mongodb` images moved to the frozen/unpatched `bitnamilegacy` catalog (Aug 2025) — a real "blocks prod sign-off" problem the original draft of this runbook flagged. Patroni is already running, already BC-Gov-approved in this namespace, and architecturally simpler (DCS-based leader election, no separate pgpool routing tier) than repmgr+pgpool — reusing it sidesteps the Bitnami problem entirely rather than trading one HA technology for an equivalently-good one with a licensing/staleness catch. Mongo: no existing HA alternative, so reuse the existing single-instance `formio-mongodb-dev` rather than adding a second frozen-image dependency (the current v4.0.8 stack already runs this way with no reported issues).

### 3.1 forms-flow-ai (infra shell — Redis only; Postgres/Mongo externalized)

```bash
helm upgrade --install forms-flow-ai ./charts/forms-flow-ai \
  --namespace "$NS" \
  --set Domain="${NS}.${DOMAIN}" \
  --set postgresql-ha.enabled=false \
  --set mongodb.enabled=false \
  --set redisExporter.persistence.storageClass=netapp-block-standard

# Wait for Redis (the one piece of infra this chart still owns)
oc wait --namespace "$NS" --for=condition=ready pod \
  --selector=app.kubernetes.io/name=redis-exporter --timeout=120s
```

`redisExporter.persistence.storageClass` routes the new Redis PVC to `netapp-block-standard` instead of the default `netapp-file-standard` — per Phase 0, the default class is ~89% full in `dev`; this is the only PVC any Phase 3 chart creates (confirmed by grepping every chart's templates for `PersistentVolumeClaim`/`VolumeClaimTemplate`), so this one `--set` clears the storage-quota risk entirely. `ingress.ingressClassName` is intentionally omitted from every `--set` in this phase (not just here) — see the Phase 0 note above.

**`imageCredentials.*` omitted for now** — the AOT private registry access key is authorized but not yet obtained (2026-08-17). Per the revised "Image sourcing" note above, the components that actually need EE images this pass are being built locally via OpenShift BuildConfig instead of pulled from that registry (same pattern as `servebc-api`, Phase 3.10 — see 3.5/3.6/3.7/3.9/3.11 below), so there's nothing to pull credentials for yet. Add `imageCredentials.registry/username/password/email` back here once the key is obtained and you switch those steps to pulling prebuilt images.

**Local build pattern, reused by every component below that needs one** (`forms-flow-api`, `forms-flow-data-layer`, `forms-flow-documents-api`, `forms-flow-bpm`, `forms-flow-web`) — **all 5 successfully built in `a60371-tools` on 2026-08-17**, findings folded in below:

```bash
oc new-build --name=<component> --binary --strategy=docker -n "$TOOLS_NS" 2>/dev/null || true
oc start-build <component> --from-dir="/Users/jaisethomas/Documents/ServeLegal/code/forms-flow-ai-ee/<component>" -n "$TOOLS_NS" --follow
oc tag "${TOOLS_NS}/<component>:latest" "${TOOLS_NS}/<component>:${ENV}-v8.2.5"
# then in the matching helm install: --set image.registry=image-registry.openshift-image-registry.svc:5000
#                                     --set image.repository="${TOOLS_NS}/<component>" --set image.tag="${ENV}-v8.2.5"
```

**⚠️ Name `forms-flow-web` and `forms-flow-bpm` as `forms-flow-web-ee`/`forms-flow-bpm-ee` instead** — `a60371-tools` already has legacy v4.0.8 BuildConfigs at those exact names (Git-strategy, pointed at `bcgov/jag-servebc.git`), left over from the old stack. `oc new-build`'s `|| true` fallback silently reuses whatever BuildConfig already has that name, and `oc start-build --from-dir` (binary input) against a Git-strategy BuildConfig fails with a misleading "context directory does not exist" error rather than a clear name-collision error. Don't touch/delete the legacy ones (same "don't disturb the old stack" rule as everywhere else) — just use distinct names. `forms-flow-api`/`forms-flow-data-layer`/`forms-flow-documents` don't collide (no legacy BuildConfig with those exact names).

**`forms-flow-web`'s Dockerfile needed two real fixes**, applied locally to the cloned EE source (not pushed anywhere — worth reporting upstream to AOT once the `bcgov-c` fork exists):
1. `COPY .npmrc ...` (sets `legacy-peer-deps=true`, needed to resolve a genuine version conflict between `@aot-technologies/formio-react`/`formiojs` v2 and `formsflow-formio-custom-elements`' v1 peer requirement) happens *after* `RUN npm ci` — move it before.
2. `RUN npm ci --only=production` skips the `devDependencies` that `npm run build` needs (`cross-env`, `craco`) — pointless anyway since this is a multi-stage build and only the built static output reaches the final nginx image. Drop `--only=production`.

**`nginx.conf` doesn't ship in the EE repo at all** — checking AOT's actual `forms-flow-web-cd.yml` CI workflow shows their real pipeline doesn't build a Docker image for `forms-flow-web`; it runs `npm ci && npm run build` and pushes the static output to **S3**, suggesting EE's real `forms-flow-web` is meant to run as a static SPA behind S3/CDN, not a self-hosted nginx container — this Dockerfile looks unexercised by AOT's own pipeline (which explains the two bugs above going unnoticed). Worth revisiting whether `forms-flow-web`/`forms-flow-web-root-config` should be deployed differently for EE rather than forced through the OSS-shaped Helm chart — same open question as the microfrontends note in the header. For now, a working `nginx.conf` (standard SPA-serving config: security headers, CORS, `try_files ... /index.html` fallback, no RSBC- or ServeBC-specific hardcoding needed) is in place at `forms-flow-web/nginx.conf` and the container build works end to end.

`forms-flow-bpm` is the only heavy one of the five (Maven multi-module build, ~5-10+ min, no dependency caching between builds by default); the rest are small Python/Node builds, ~1-3 min each. Build-pod resource use was well within the compute headroom Phase 0 confirmed (`a60371-tools`: 0 used / 2 CPU / 8Gi memory going in).

> If `postgresql-ha.enabled=false`/`mongodb.enabled=false` still attempts to schedule those pods, check `helm template` output first (`helm template forms-flow-ai ./charts/forms-flow-ai --set postgresql-ha.enabled=false --set mongodb.enabled=false | grep -A2 "kind: StatefulSet"`) — some Bitnami sub-charts need `global.postgresql.enabled`-style flags at a different nesting level than the top-level toggle; confirm against the actual rendered manifest before assuming the `--set` above is sufficient.

> If `postgresql-ha.enabled=false`/`mongodb.enabled=false` still attempts to schedule those pods, check `helm template` output first (`helm template forms-flow-ai ./charts/forms-flow-ai --set postgresql-ha.enabled=false --set mongodb.enabled=false | grep -A2 "kind: StatefulSet"`) — some Bitnami sub-charts need `global.postgresql.enabled`-style flags at a different nesting level than the top-level toggle; confirm against the actual rendered manifest before assuming the `--set` above is sufficient.

### 3.2 forms-flow-servebc-config (CHES + S3 + BPM timezone + custom theme)

```bash
oc get secret ches-dev -n "$NS" -o jsonpath='{.data}' | jq -r 'to_entries[] | "\(.key)=\(.value | @base64d)"' > /tmp/ches-vals.env
oc get secret line-of-business-api-s3-secret -n "$NS" -o jsonpath='{.data}' | jq -r 'to_entries[] | "\(.key)=\(.value | @base64d)"' > /tmp/s3-vals.env
source /tmp/ches-vals.env   # EMAIL_KEYCLOAK_URL, EMAIL_KEYCLOAK_CLIENT_ID, EMAIL_KEYCLOAK_CLIENT_SECRET, EMAIL_SVC_URL
source /tmp/s3-vals.env     # S3_BUCKETNAME, S3_ACCESS_KEY_ID, S3_SECRET_ACCESS_KEY, S3_HOST

helm upgrade --install forms-flow-servebc-config ./charts/forms-flow-servebc-config \
  --namespace "$NS" \
  --set ches.keycloakUrl="$EMAIL_KEYCLOAK_URL" \
  --set ches.clientId="$EMAIL_KEYCLOAK_CLIENT_ID" \
  --set ches.clientSecret="$EMAIL_KEYCLOAK_CLIENT_SECRET" \
  --set ches.serviceUrl="$EMAIL_SVC_URL" \
  --set bpm.timezone="America/Vancouver" \
  --set servebc.internalUrl="http://servebc-api:3003" \
  --set servebc.externalUrl="https://servebc-api-${NS}.${DOMAIN}" \
  --set web.customThemeUrl="" \
  --set database.username="servebcuser${DB_SUFFIX}" \
  --set database.password="$SERVEBC_DB_PASS" \
  --set database.dbName="servebcdb${DB_SUFFIX}" \
  --set database.host="patroni-master" \
  --set database.port=5432 \
  --set s3.bucketName="$S3_BUCKETNAME" \
  --set s3.host="$S3_HOST" \
  --set s3.accessKeyId="$S3_ACCESS_KEY_ID" \
  --set s3.secretAccessKey="$S3_SECRET_ACCESS_KEY" \
  --set s3.useSsl="true"
rm -f /tmp/ches-vals.env /tmp/s3-vals.env
```

### 3.3 forms-flow-idm (Keycloak — self-hosted, parallel install, fresh realm)

Confirmed decision (2026-08-11): keep this self-hosted rather than switching to OCIO's shared BC Gov realm — OCIO's shared realm doesn't expose the custom role-mapping (`designer`/`reviewer`/`client`) formsflow.ai needs, and a custom realm inside OCIO would need project-level approvals/audits that take time (may be pursued separately later). This install is a parallel/side-by-side stand-up, not an in-place upgrade of the old 18.0.2-legacy instance — new release, new route, new DB, new realm rebuilt via API. **Known risk, not yet resolved:** the Bitnami `keycloak` subchart here pulls from the frozen/unpatched `bitnamilegacy/keycloak:26.1.4-debian-12-r2` catalog — same class of problem as the Postgres/Mongo Bitnami issue below, but with no workaround planned yet. Fine for `dev`; decide (pin a patched image, or accept the risk) before `test`/`prod`.

```bash
KEYCLOAK_ADMIN_PASS=$(openssl rand -base64 24)

helm upgrade --install forms-flow-idm ./charts/forms-flow-idm \
  --namespace "$NS" \
  --set keycloak.ingress.hostname="forms-flow-idm-${NS}.${DOMAIN}" \
  --set keycloak.ingress.tls=true \
  --set keycloak.auth.adminUser=admin \
  --set keycloak.auth.adminPassword="$KEYCLOAK_ADMIN_PASS" \
  --set keycloak.postgresql.enabled=false \
  --set keycloak.externalDatabase.host=patroni-master \
  --set keycloak.externalDatabase.port=5432 \
  --set keycloak.externalDatabase.database="keycloakdb${DB_SUFFIX}" \
  --set keycloak.externalDatabase.user="keycloakuser${DB_SUFFIX}" \
  --set keycloak.externalDatabase.password="$KEYCLOAK_DB_PASS"

oc wait --namespace "$NS" --for=condition=ready pod \
  --selector=app.kubernetes.io/name=keycloak --timeout=300s
```

**Realm rebuild** (fresh-install scope — recreate the structure from the old realm as a template, not a raw DB migration; reference: this session's `openshift-export/CURRENT-CONFIG-a60371-dev.md` for the exact structure being replicated):

```bash
KC_HOST="https://forms-flow-idm-${NS}.${DOMAIN}/auth"
TOKEN=$(curl -sk -X POST "${KC_HOST}/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" -d "username=admin" -d "password=${KEYCLOAK_ADMIN_PASS}" -d "grant_type=password" \
  | jq -r .access_token)

# 1. Create realm
curl -sk -X POST "${KC_HOST}/admin/realms" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"realm":"forms-flow-ai","enabled":true}'

# 2. Create clients (forms-flow-web public, forms-flow-bpm confidential — generate a fresh secret)
BPM_CLIENT_SECRET=$(openssl rand -hex 16)
curl -sk -X POST "${KC_HOST}/admin/realms/forms-flow-ai/clients" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"clientId":"forms-flow-web","publicClient":true,"redirectUris":["*"],"webOrigins":["*"]}'
curl -sk -X POST "${KC_HOST}/admin/realms/forms-flow-ai/clients" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"clientId\":\"forms-flow-bpm\",\"publicClient\":false,\"secret\":\"${BPM_CLIENT_SECRET}\",\"serviceAccountsEnabled\":true}"

# 3. Recreate groups matching the a60371-dev structure documented in CURRENT-CONFIG-a60371-dev.md:
#    /formsflow/formsflow-client, /formsflow/formsflow-reviewer (+staff, management,
#    access-allow-submissions subgroups), /formsflow/formsflow-designer,
#    /formsflow-analytics/staff-reports, /camunda-admin, /realm-management
#    (loop pattern — repeat per group path via POST .../groups, nesting children under the
#    returned parent group id for subgroups)

# 4. Recreate the 3 synthetic users (formsflow-client/reviewer/designer) in their matching groups —
#    same pattern already used successfully earlier this session for a60371-dev (POST .../users,
#    then PUT .../users/{id}/reset-password, then PUT .../users/{id}/groups/{groupId})

# 5. Configure IDIR as an Identity Provider (broker) for the 2 real staff accounts — reuse whatever
#    IDIR broker config pattern other BC Gov apps in this project use (check `dev.loginproxy.gov.bc.ca`
#    /`dev.oidc.gov.bc.ca` conventions with the platform team if this isn't already documented
#    elsewhere) rather than guessing the exact client config here.
```

> Save `$BPM_CLIENT_SECRET` — you'll pass it into both `forms-flow-bpm`'s and `forms-flow-api`'s installs below.

### 3.4 forms-flow-forms (Form.io)

**⚠️ Check before running this step:** `formio-mongodb-dev` is running MongoDB 3.6.3 (confirmed in Phase 1.2). Modern MongoDB Node.js drivers refuse to connect to anything below 4.2 — that's exactly what broke `mongosh` during Phase 1.2. Check what driver version ships in the EE `forms-flow-forms` image before running this install; if it has the same floor, this step is blocked until either the driver is downgraded (unlikely, upstream-controlled) or `formio-mongodb-dev` is upgraded past 3.6 first (separate, unscoped work).

**Image choice:** unlike the five components using the local-build pattern (Phase 3.1), this chart's default image is `docker.io/formsflow/forms-flow-forms:v7.3.0` — a **public**, OSS-tagged image, no registry key or local build needed. Fine to use as-is if EE-specific forms features aren't needed yet; otherwise the EE repo has its own `openshift_custom_Dockerfile` for this component if you want to build the EE version locally instead, same pattern as the others.

```bash
FORMIO_ROOT_PASS=$(openssl rand -base64 20)
FORMIO_JWT_SECRET=$(openssl rand -hex 32)

helm upgrade --install forms-flow-forms ./charts/forms-flow-forms \
  --namespace "$NS" \
  --set ingress.hostname="forms-flow-forms-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set admin.email="<team-email>" \
  --set admin.password="$FORMIO_ROOT_PASS" \
  --set mongodb.uri="mongodb://formiouser${DB_SUFFIX}:${FORMIO_DB_PASS}@formio-mongodb-dev:27017/formio${DB_SUFFIX}?authSource=formio${DB_SUFFIX}" \
  --set jwt.secret="$FORMIO_JWT_SECRET"
```

> Confirm the exact Mongo URI value-path (`mongodb.uri` vs a split `mongodb.host`/`.port`/`.database` set) against `charts/forms-flow-forms/values.yaml` at execution time — grep for `mongo` in that file before running, since this wasn't in the portion of the chart reviewed while planning.

### 3.5 forms-flow-api (webapi)

Build locally per the Phase 3.1 pattern (`<component>` = `forms-flow-api`) — no registry key yet. Also an explicit candidate for actual customization ("Camunda/API may need customizations" — TBD, not yet decided), which a local build accommodates either way.

```bash
helm upgrade --install forms-flow-api ./charts/forms-flow-api \
  --namespace "$NS" \
  --set image.registry=image-registry.openshift-image-registry.svc:5000 \
  --set image.repository="${TOOLS_NS}/forms-flow-api" \
  --set image.tag="${ENV}-v8.2.5" \
  --set ingress.hostname="forms-flow-api-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set database.host=patroni-master \
  --set database.port=5432 \
  --set database.dbName="webapidb${DB_SUFFIX}" \
  --set database.username="webapiuser${DB_SUFFIX}" \
  --set database.password="$WEBAPI_DB_PASS" \
  --set FormioJWTExpire="240"
```

> Confirm the exact `image.*` value paths against `charts/forms-flow-api/values.yaml` — not verified during planning, may differ from the `image.registry`/`.repository`/`.tag` split assumed here.

### 3.6 forms-flow-data-layer (EE scope addition — no new database, reuses forms-flow-api's secret)

Needs `forms-flow-api`, `forms-flow-forms`, and `forms-flow-idm` already up (it reads the webapi DB secret directly and cross-references Form.io/Keycloak config), hence its place in the order right after `forms-flow-api`. Build locally per the Phase 3.1 pattern (`<component>` = `forms-flow-data-layer`) — no registry key yet; this is a small, fast (~1-3 min) Python build.

```bash
helm upgrade --install forms-flow-data-layer ./charts/forms-flow-data-layer \
  --namespace "$NS" \
  --set image.registry=image-registry.openshift-image-registry.svc:5000 \
  --set image.repository="${TOOLS_NS}/forms-flow-data-layer" \
  --set image.tag="${ENV}-v8.2.5" \
  --set formsflow.webapi.secret=forms-flow-api \
  --set formsflow.webapi.configmap=forms-flow-api \
  --set formsflow.secret=forms-flow-ai \
  --set formsflow.configmap=forms-flow-ai
```

> Confirm the exact secret/configmap names against what Phase 3.1/3.5 actually created (`common.names.fullname` resolved these to `forms-flow-ai` and `forms-flow-api` respectively when checked against the chart templates — verify with `oc get secret,configmap -n "$NS"` if unsure) before running.

### 3.7 forms-flow-documents-api (EE scope addition — stateless, no new database)

Build locally per the Phase 3.1 pattern (`<component>` = `forms-flow-documents`, matching the EE repo's directory name — note this differs from the chart/release name `forms-flow-documents-api`) — no registry key yet, small/fast build.

```bash
helm upgrade --install forms-flow-documents-api ./charts/forms-flow-documents-api \
  --namespace "$NS" \
  --set image.registry=image-registry.openshift-image-registry.svc:5000 \
  --set image.repository="${TOOLS_NS}/forms-flow-documents" \
  --set image.tag="${ENV}-v8.2.5" \
  --set ingress.hostname="forms-flow-documents-${NS}.${DOMAIN}" \
  --set ingress.tls=true
```

> Check `charts/forms-flow-documents-api/values.yaml` for how `FORMIO_URL`/`FORMSFLOW_API_URL`/`REDIS_URL`/`KEYCLOAK_URL_HTTP_RELATIVE_PATH` are wired (configmap references similar to `forms-flow-data-layer` above) before running — not fully traced during planning.

### 3.8 forms-flow-analytics (EE scope addition — new `analyticsdb${DB_SUFFIX}` from Phase 1.4, shared Redis)

**No build needed** — this chart's default image is `docker.io/formsflow/redash:24.04.0`, a public image, unrelated to the EE source or private registry. `postgresql.enabled`/`redis.enabled` already default to `false` in this chart — no need to set them. `externalRedis` already defaults to `forms-flow-ai`'s own `redis-exporter` service, so nothing to set there either.

```bash
helm upgrade --install forms-flow-analytics ./charts/forms-flow-analytics \
  --namespace "$NS" \
  --set ingress.hostname="forms-flow-analytics-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set externalPostgreSQLSecret.name=analytics-db-${DB_SUFFIX} \
  --set externalPostgreSQLSecret.key=connectionString
```

### 3.9 forms-flow-bpm (vanilla — no custom Java extensions per plan decision)

Build locally per the Phase 3.1 pattern (`<component>` = `forms-flow-bpm`) — no registry key yet. Also an explicit candidate for actual customization ("Camunda/API may need customizations" — TBD, not yet decided), which a local build accommodates either way. This is the one heavy build in the set — Maven multi-module, no dependency caching between OpenShift BuildConfig runs by default, likely 5-10+ min. Budget for that rather than being surprised by it.

```bash
helm upgrade --install forms-flow-bpm ./charts/forms-flow-bpm \
  --namespace "$NS" \
  --set image.registry=image-registry.openshift-image-registry.svc:5000 \
  --set image.repository="${TOOLS_NS}/forms-flow-bpm-ee" \
  --set image.tag="${ENV}-v8.2.5" \
  --set ingress.hostname="forms-flow-bpm-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set camunda.jdbc.url="jdbc:postgresql://patroni-master:5432/bpmdb${DB_SUFFIX}" \
  --set camunda.jdbc.username="bpmuser${DB_SUFFIX}" \
  --set camunda.jdbc.password="$BPM_DB_PASS" \
  --set camunda.database.name="bpmdb${DB_SUFFIX}" \
  --set camunda.database.port=5432 \
  --set camunda.websocket.securityOrigin="https://forms-flow-web-${NS}.${DOMAIN}" \
  --set forms-flow-bpm.clientsecret="$BPM_CLIENT_SECRET"
# mail.* (SMTP block in values.yaml) is left at defaults — CHES integration for BPM notification
# emails is carried via forms-flow-servebc-config's ConfigMap/Secret instead (EMAIL_KEYCLOAK_URL
# etc.), NOT this chart's built-in mail.* block. Confirm in Phase 4 testing that BPM actually reads
# those CHES vars (via extraEnvVarsSecret/extraEnvVarsCM wiring — check whether forms-flow-bpm's
# values.yaml has that hook, same as servebc-api does) rather than assuming email works untested.

oc wait --namespace "$NS" --for=condition=available deployment/forms-flow-bpm --timeout=300s
```

> Confirm the exact `image.*` value paths against `charts/forms-flow-bpm/values.yaml` before running — not verified during planning, may nest differently than the `image.registry`/`.repository`/`.tag` split assumed here (the chart's `camunda.*` values suggest a Camunda-specific structure that might extend to image config too).

### 3.10 servebc-api (custom API — this repo's `/api`)

Build the image via a real OpenShift BuildConfig first (the local deployment used a manual `docker build`, not viable on Silver):

```bash
SERVEBC_API_SHA=$(git -C /Users/jaisethomas/Documents/ServeLegal/code/jag-servebc rev-parse --short HEAD)

oc new-build --name=servebc-api --binary --strategy=docker -n "$TOOLS_NS" 2>/dev/null || true
oc start-build servebc-api --from-dir=/Users/jaisethomas/Documents/ServeLegal/code/jag-servebc/api -n "$TOOLS_NS" --follow
oc tag "${TOOLS_NS}/servebc-api:latest" "${TOOLS_NS}/servebc-api:${ENV}-${SERVEBC_API_SHA}"
```

> Tag fixed to use `servebc-api`'s own git SHA (2026-08-17) — the original draft tagged this `v7.3.1-${ENV}`, coupling *this org's own, entirely separate* custom API's build tag to the formsflow.ai app version, which never made sense and would have silently broken the moment the target version changed (as it just did, OSS 7.3.1 → EE 8.2.5). `servebc-api` versions independently of whichever formsflow release it's deployed alongside.

```bash
helm upgrade --install servebc-api ./charts/servebc-api \
  --namespace "$NS" \
  --set image.registry="image-registry.openshift-image-registry.svc:5000" \
  --set image.repository="${TOOLS_NS}/servebc-api" \
  --set image.tag="${ENV}-${SERVEBC_API_SHA}" \
  --set image.pullPolicy=Always \
  --set ingress.hostname="servebc-api-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set database.initJob.enabled=false \
  --set formsflow.servebcConfigmap=forms-flow-servebc \
  --set formsflow.servebcSecret=forms-flow-servebc
```

> `database.initJob.enabled=false` because Phase 1 already created `servebcdb${DB_SUFFIX}` manually — leave the chart's own auto-creation Job off to avoid it trying (and likely failing on privilege grounds, since it expects the DB user itself to have `CREATEDB`) to redo that work.

### 3.11 forms-flow-web (frontend)

Build locally per the Phase 3.1 pattern (`<component>` = `forms-flow-web`) — no registry key yet; small/fast Node build. Still open whether `forms-flow-web-root-config` also needs its own build/step here (see the microfrontends question flagged in the header) — resolve that before this step, not after.

```bash
helm upgrade --install forms-flow-web ./charts/forms-flow-web \
  --namespace "$NS" \
  --set image.registry=image-registry.openshift-image-registry.svc:5000 \
  --set image.repository="${TOOLS_NS}/forms-flow-web-ee" \
  --set image.tag="${ENV}-v8.2.5" \
  --set ingress.hostname="forms-flow-web-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set web.base_custom_url="https://servebc-api-${NS}.${DOMAIN}" \
  --set web.custom_theme_url="" \
  --set web.enable_forms_module=true \
  --set web.enable_tasks_module=true \
  --set web.enable_dashboards_module=false \
  --set web.enable_processes_module=true \
  --set web.enable_applications_module=true
```

---

## Phase 4 — Verification (run after each install step above, not just at the end)

```bash
# Keycloak
oc rollout status deployment -n "$NS" -l app.kubernetes.io/name=keycloak
curl -sf "https://forms-flow-idm-${NS}.${DOMAIN}/auth/realms/forms-flow-ai/.well-known/openid-configuration" | jq .issuer

# Form.io
curl -sk -o /dev/null -w "forms-flow-forms -> HTTP %{http_code}\n" "https://forms-flow-forms-${NS}.${DOMAIN}/"

# webapi
curl -sk -o /dev/null -w "forms-flow-api -> HTTP %{http_code}\n" "https://forms-flow-api-${NS}.${DOMAIN}/"

# forms-flow-data-layer (no public route by default — check pod readiness instead)
oc get pods -n "$NS" -l app.kubernetes.io/instance=forms-flow-data-layer

# forms-flow-documents-api
curl -sk -o /dev/null -w "forms-flow-documents-api -> HTTP %{http_code}\n" "https://forms-flow-documents-${NS}.${DOMAIN}/"

# forms-flow-analytics
curl -sk -o /dev/null -w "forms-flow-analytics -> HTTP %{http_code}\n" "https://forms-flow-analytics-${NS}.${DOMAIN}/"

# BPM
curl -sk -o /dev/null -w "forms-flow-bpm -> HTTP %{http_code}\n" "https://forms-flow-bpm-${NS}.${DOMAIN}/camunda/actuator/health"

# servebc-api
curl -sk -o /dev/null -w "servebc-api -> HTTP %{http_code}\n" "https://servebc-api-${NS}.${DOMAIN}/api/v1/healthcheck"

# web
curl -sk -o /dev/null -w "forms-flow-web -> HTTP %{http_code}\n" "https://forms-flow-web-${NS}.${DOMAIN}/"
```

**Whole-flow smoke test (manual, in a browser):** log in as `formsflow-client`, submit a form, confirm a Camunda process instance + webapi application record is created; log in as `formsflow-reviewer`, confirm the task appears (expect a manual-refresh delay, not real-time — known gap, see Non-goals); confirm a real IDIR-federated staff login works via the broker. Explicitly test and record the actual behavior of the public NCQ form route — expected broken per the Non-goals section; don't assume, verify and document what actually happens.

**Go/no-go before promoting to the next environment:** all nine components healthy above, whole-flow smoke test passes, no unexplained pod restarts over a soak period the team is comfortable with (recommend at minimum a few hours for dev→test, longer — multi-day — for test→prod).

---

## Phase 5 — Rollback

Each chart is an independent Helm release:
```bash
helm history <release> -n "$NS"
helm rollback <release> <revision> -n "$NS"
```
Full teardown of a bad install (reverse deployment order):
```bash
for chart in forms-flow-web servebc-api forms-flow-bpm forms-flow-analytics forms-flow-documents-api forms-flow-data-layer forms-flow-api forms-flow-forms forms-flow-idm forms-flow-servebc-config forms-flow-ai; do
  helm uninstall "$chart" -n "$NS"
done
```
`helm uninstall` does not delete PVCs — clean up explicitly (`oc get pvc -n "$NS"`) once you're certain you don't need them.

**Do not drop the old v4.0.8 databases** (`bpmdb`, `webapidb`, `keycloakdb`, `analyticsdb`, the old Mongo formio DB — no `_${DB_SUFFIX}` suffix) **or delete the old DeploymentConfigs** until this environment's go/no-go checklist above has passed and an agreed soak period has elapsed. The exported DC/route/service YAML from Phase 0 plus the still-intact old databases are your rollback path back to v4.0.8 if EE 8.2.5 doesn't work out.

---

## Repeat for test, then prod

Change only the four lines in Phase 0's parameterization block (`ENV=test`, later `ENV=prod`) and re-run Phases 0–5 in full for each. Do not start `test` until `dev`'s go/no-go passes; do not start `prod` until `test`'s soak period passes and the team has explicitly signed off — in particular, reconfirm the CHES/S3 credentials and the IDIR broker configuration are the *correct, environment-specific* ones (dev CHES/S3 creds must not be reused in prod) before running Phase 3 against `-test`/`-prod`.
