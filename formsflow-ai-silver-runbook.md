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
export DB_SUFFIX="_82"        # database-name tag only — usernames are NOT suffixed, see naming note below
# ---------------------------------------------------------------------------------

export TOOLS_NS="a60371-tools"
export CHARTS_DIR="$HOME/formsflow-charts-deploy"   # scratch checkout, see Phase 2
echo "Deploying formsflow.ai EE v8.2.5 to namespace: $NS (domain: $DOMAIN)"
```

**On `DB_SUFFIX` naming (revised again 2026-08-17):** two iterations on this. First pass used `731` (a stripped-down "7.3.1") on *both* database and user names — abandoned once the target version changed mid-project (a version-coupled suffix shouldn't force renaming databases). Second pass kept a generation tag but applied it to usernames too (`bpmuser731` etc.), which turned out to be unnecessary complexity: the *databases* need a distinguishing suffix (the old stack already occupies the plain, unsuffixed database names, and something has to distinguish this generation from the next one), but the *users* don't — Postgres roles are cluster-global, so a role can simply be granted access to multiple databases, and `bpmuser`/`webapiuser`/`keycloakuser`/`analyticsuser` already exist from the old stack with known passwords. **Current approach: reuse the existing plain-named roles and their existing passwords unchanged (no `ALTER USER`, no new role), and only suffix the *database* name** (`bpmdb_82`, `webapidb_82`, etc. — `_82` for the 8.2.5 generation). This is simpler to maintain (matches the old single-secret model's spirit) and safer than it might sound: reusing the password means nothing needs to change on the still-running old stack to make this work. The one real trade-off, worth being explicit about: with a shared role, the *database name* is now the only thing preventing a misconfigured new-stack install from accidentally reading/writing the old stack's live data (a wrong password would no longer catch that, since the same password is valid for both). Fine for `dev` with a single operator; reconsider before `prod` if that risk matters more there. `servebcuser` doesn't pre-exist (the old custom API used a different legacy DB/user entirely) — created fresh, plain, no suffix, reusing the password already generated for it earlier in this session. If `dev` is ever torn down and freshly re-provisioned, bump `DB_SUFFIX` to a new tag (e.g. `_83`, a date stamp) — same reasoning as before, just scoped to database names only now. Once fully cut over to the new architecture and the old stack is gone, this whole suffix concern goes away — future version bumps should be in-place migrations against the same stable database, with a pre-upgrade backup as the safety net.

**Note on `DB_SUFFIX` and Kubernetes naming:** `_82` is used directly in Postgres identifiers (`bpmdb${DB_SUFFIX}` → `bpmdb_82`, valid — Postgres allows underscores), but Kubernetes object names don't allow underscores. The consolidated secret in Phase 1.3 is therefore named literally `formsflow-db-82` (hyphen), not derived mechanically as `formsflow-db${DB_SUFFIX}` — don't copy-paste that pattern for the secret name specifically.

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

**Status for `dev`: Phase 1 fully done (1.1–1.4), as of 2026-08-17.** 1.1–1.3 first completed 2026-08-11 for the original 4 components (bpm/webapi/keycloak/servebc); full results and the two corrections below in [`reports/2026-08-11-upgrade-progress-status.md`](reports/2026-08-11-upgrade-progress-status.md). 1.4 (`forms-flow-analytics`) added and run 2026-08-17 as part of the EE scope expansion. Databases were subsequently renamed from the `731` to the `_82` suffix scheme the same day (see the `DB_SUFFIX` naming note above) — Postgres by reusing existing plain-named roles against new suffixed databases, Mongo (1.2) by dropping `formio731`/`formiouser731` and recreating as database `formio_82` with a plain (unsuffixed) user `formiouser`, matching the "database gets the tag, user doesn't" convention Postgres already used. Two things the original draft got wrong, already corrected below:
- The Mongo root credential is **not** `MONGO_INITDB_ROOT_USERNAME`/`MONGO_INITDB_ROOT_PASSWORD` — `formio-mongodb-dev` uses the classic OpenShift `mongodb-persistent` template convention instead: secret key `admin-password` paired with the fixed username `admin`.
- `mongosh` (bundled in `mongo:5.0`+ client images) refuses to connect — `formio-mongodb-dev` is running **MongoDB 3.6.3**, and modern drivers require ≥4.2. Use the legacy `mongo` shell (`mongo:4.4` image) instead. **This is a real open risk for Phase 3.4** (`forms-flow-forms`) if its bundled Node.js driver has the same floor — check before running that step.

### 1.1 Postgres — Camunda, webapi, Keycloak, servebc-api databases

**Revised 2026-08-17 — reuse the existing plain-named roles rather than creating suffixed ones** (see the `DB_SUFFIX` naming note above). `bpmuser`/`webapiuser`/`keycloakuser` already exist from the old stack — only `servebcuser` needs to be created new.

```bash
# Get Patroni superuser creds (already exists in-namespace)
PATRONI_SUPERUSER=$(oc get secret patroni-creds -n "$NS" -o jsonpath='{.data.superuser-username}' | base64 -d)
PATRONI_SUPERPASS=$(oc get secret patroni-creds -n "$NS" -o jsonpath='{.data.superuser-password}' | base64 -d)

# Existing passwords, reused as-is — NOT regenerated (regenerating would break the still-running old stack)
BPM_DB_PASS=$(oc get secret servelegal-patroni-ff-db-secret -n "$NS" -o jsonpath='{.data.BPM_DB_PASSWORD}' | base64 -d)
WEBAPI_DB_PASS=$(oc get secret servelegal-patroni-ff-db-secret -n "$NS" -o jsonpath='{.data.WEBAPI_DB_PASSWORD}' | base64 -d)
KEYCLOAK_DB_PASS=$(oc get secret servelegal-patroni-ff-db-secret -n "$NS" -o jsonpath='{.data.KEYCLOAK_DB_PASSWORD}' | base64 -d)
# servebcuser is genuinely new (old custom API used a different legacy DB/user) — fresh password
SERVEBC_DB_PASS=$(openssl rand -base64 24)

oc run psql-client --rm -i --restart=Never -n "$NS" --image=postgres:16-alpine -- \
  env PGPASSWORD="$PATRONI_SUPERPASS" psql -U "$PATRONI_SUPERUSER" -h patroni-master <<SQL
CREATE DATABASE bpmdb${DB_SUFFIX};
GRANT ALL PRIVILEGES ON DATABASE bpmdb${DB_SUFFIX} TO bpmuser;

CREATE DATABASE webapidb${DB_SUFFIX};
GRANT ALL PRIVILEGES ON DATABASE webapidb${DB_SUFFIX} TO webapiuser;

CREATE DATABASE keycloakdb${DB_SUFFIX};
GRANT ALL PRIVILEGES ON DATABASE keycloakdb${DB_SUFFIX} TO keycloakuser;

CREATE DATABASE servebcdb;
CREATE USER servebcuser WITH PASSWORD '${SERVEBC_DB_PASS}';
GRANT ALL PRIVILEGES ON DATABASE servebcdb TO servebcuser;
SQL
```

**`servebcdb` gets no `DB_SUFFIX` at all, unlike the other three** (revised 2026-08-17) — `bpmdb`/`webapidb`/`keycloakdb` hold state *internal to the formsflow.ai product* (Camunda, webapi, Keycloak), owned by AOT's upstream code, so they're coupled to which formsflow.ai generation is running and need to stay isolated during this migration. `servebcdb` holds ServeLegal's own business objects (document-serving domain data via `servebc-api`, this org's own code) — not coupled to the formsflow.ai product version at all. It's created once, plain, and just carries forward across every future formsflow.ai upgrade rather than getting a new generation-tagged name each time.

### 1.2 Mongo — Form.io database

**Status for `dev`: done (2026-08-17, renamed same day).** First pass (2026-08-11) created `formio731`/`formiouser731`, using the app-version-coupled suffix scheme that was later abandoned for Postgres (see the `DB_SUFFIX` naming note above). Revised 2026-08-17 to match: dropped `formiouser731` (the `formio731` database itself had never actually materialized — Mongo doesn't create a database until it holds data, so there was nothing to drop there, just the user), created database `formio_82` with a **plain, unsuffixed** user `formiouser` — matching the Postgres convention exactly (only the database name carries the generation tag; the user doesn't). Note this user is *not* a reuse of the old stack's existing Mongo user (which is named `formio`, not `formiouser`, and stays scoped to the old `formio` database) — it's a fresh user, just deliberately given the plain name rather than a suffixed one. **Credentials are now also written into the Phase 1.3 consolidated secret (`FORMIO_DB_HOST/PORT/NAME/USER/PASSWORD`)** rather than kept only in a shell variable — the original draft lost `$FORMIO_DB_PASS` the moment the session ended, since nothing persisted it; this fixes that for good.

**Run-order note for a from-scratch environment:** the `oc patch secret formsflow-db-82` line below depends on that secret already existing, which normally happens in **1.3, below this section** — despite 1.2 being numbered before 1.3. On a fresh environment, run 1.1 → 1.3 (base secret, no Mongo keys yet) → 1.2 (this section, patches Mongo keys in) → 1.4 (patches analytics key in, same pattern). This mirrors how 1.4 already patches into the 1.3 secret; 1.2 just wasn't following that pattern until this revision.

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
    db.getSiblingDB('formio_82').createUser({
      user: 'formiouser',
      pwd: '${FORMIO_DB_PASS}',
      roles: [{ role: 'readWrite', db: 'formio_82' }]
    })
  "

# Persist into the Phase 1.3 consolidated secret immediately — don't rely on the shell var surviving.
oc patch secret formsflow-db-82 -n "$NS" --type=merge -p \
  "{\"stringData\":{\"FORMIO_DB_HOST\":\"formio-mongodb-dev\",\"FORMIO_DB_PORT\":\"27017\",\"FORMIO_DB_NAME\":\"formio_82\",\"FORMIO_DB_USER\":\"formiouser\",\"FORMIO_DB_PASSWORD\":\"${FORMIO_DB_PASS}\"}}"

# Scale back down — it's normally idle along with the rest of the old stack.
oc scale dc/formio-mongodb-dev -n "$NS" --replicas=0
```

### 1.3 Create one consolidated Kubernetes Secret for all component DB credentials

**Revised 2026-08-17 (twice — see the `DB_SUFFIX` naming note in Phase 0).** First pass created one secret per component (`bpm-db-731` etc.); checked against the actual chart templates and confirmed **none of the charts require that** (all support `ExternalDatabase.ExistingSecretName`/`externalDatabase.existingSecret` pointed at any secret, with configurable per-field key names) — consolidated to one secret. Second pass dropped the per-user suffixing too: `bpmuser`/`webapiuser`/`keycloakuser` are reused as-is from Phase 1.1, only the database names carry the `_82` tag.

```bash
oc create secret generic formsflow-db-82 -n "$NS" \
  --from-literal=BPM_DB_HOST=patroni-master \
  --from-literal=BPM_DB_NAME=bpmdb${DB_SUFFIX} \
  --from-literal=BPM_DB_USER=bpmuser \
  --from-literal=BPM_DB_PASSWORD="$BPM_DB_PASS" \
  --from-literal=FORMSFLOW_API_HOSTNAME=patroni-master \
  --from-literal=FORMSFLOW_API_DB_NAME=webapidb${DB_SUFFIX} \
  --from-literal=FORMSFLOW_API_DB_USER=webapiuser \
  --from-literal=FORMSFLOW_API_DB_PASSWORD="$WEBAPI_DB_PASS" \
  --from-literal=KEYCLOAK_DB_HOST=patroni-master \
  --from-literal=KEYCLOAK_DB_PORT=5432 \
  --from-literal=KEYCLOAK_DB_NAME=keycloakdb${DB_SUFFIX} \
  --from-literal=KEYCLOAK_DB_USER=keycloakuser \
  --from-literal=KEYCLOAK_DB_PASSWORD="$KEYCLOAK_DB_PASS" \
  --from-literal=SERVEBC_DB_HOST=patroni-master \
  --from-literal=SERVEBC_DB_NAME=servebcdb \
  --from-literal=SERVEBC_DB_USER=servebcuser \
  --from-literal=SERVEBC_DB_PASSWORD="$SERVEBC_DB_PASS"
```

The `webapi` entry uses `FORMSFLOW_API_*` naming rather than `WEBAPI_DB_*` (inconsistent-looking on purpose) — `forms-flow-data-layer`'s chart **hardcodes** references to keys named exactly `FORMSFLOW_API_HOSTNAME`/`FORMSFLOW_API_DB_NAME`/`FORMSFLOW_API_DB_USER`/`FORMSFLOW_API_DB_PASSWORD` in whatever secret `formsflow.webapi.secret` points at (Phase 3.6), with no override mechanism of its own — checked, confirmed no other component has a hidden hardcoded dependency like this. Matching `forms-flow-api`'s own chart-default key names here means one set of keys satisfies both `forms-flow-api` (Phase 3.5) and `forms-flow-data-layer` (Phase 3.6) simultaneously.

Deliberately a *separate* object from the `forms-flow-ai` chart's own Helm-managed secret (also just named `forms-flow-ai`, see Phase 3.1) rather than merged into it — a Helm-templated secret gets fully re-rendered on every `helm upgrade`, so hand-added keys not in that chart's own template would silently disappear on the next upgrade. `ANALYTICS_DB_CONNECTION_STRING` gets added to this same secret once Phase 1.4 (below) actually creates that database — `forms-flow-analytics` wants a single connection-string key, not split host/user/password fields like the others, so it doesn't fit the pattern above directly. **`FORMIO_DB_HOST/PORT/NAME/USER/PASSWORD` are likewise patched in from Phase 1.2** (Mongo, not Postgres — split fields work fine there since `forms-flow-ai`'s `secrets.yaml` template builds its own `mongodb://` URI from `--set` values rather than reading a connection string, see Phase 3.1) rather than included in the initial `oc create secret` below, since Phase 1.2 runs before this step and patches them in directly at creation time.

**`forms-flow-servebc-config` is the one exception** — it's this org's own custom chart, not yet ported into `charts/` (that happens in Phase 2), and as of the version validated locally it has no `existingSecret` support at all; it builds its own secret directly from `--set database.*` values. Left as-is for now (Phase 3.2 still needs `SERVEBC_DB_PASS`/user/host/name passed explicitly) — revisit adding `existingSecret` support to that chart once it's actually in this repo, if it's worth the churn.

Keep the plaintext passwords generated above somewhere durable for this run (a password manager, not committed anywhere) — you'll need `SERVEBC_DB_PASS`/user/host/name again explicitly in Phase 3.2's `forms-flow-servebc-config` install, since that chart doesn't read from the secret above (see exception noted just above).

### 1.4 Postgres — `forms-flow-analytics` database (new, EE scope addition)

`forms-flow-analytics` already defaults `postgresql.enabled: false` and `redis.enabled: false` in its chart values — it's already externalized, no `--set` override needed for those toggles (unlike `forms-flow-ai`/`forms-flow-idm`, which need it set explicitly). Its Redis need is satisfied for free too: `externalRedis` already defaults to `redis://redis-exporter:6379/0`, which is `forms-flow-ai`'s own Redis service — no separate Redis to provision. Only a new Postgres database is needed, and this chart takes the connection as a single URL via a Secret (`externalPostgreSQLSecret`), not split host/user/password `--set` values like the other components.

`forms-flow-data-layer` and `forms-flow-documents-api` — the other two EE-scope additions — need **no new database at all**: `forms-flow-data-layer` reuses `forms-flow-api`'s own secret directly (`FORMSFLOW_API_DB_*` keys sourced from whatever secret `--set formsflow.webapi.secret=...` points at in Phase 3.5.5) plus the shared `forms-flow-ai` secret/configmap for the Form.io Mongo connection, and `forms-flow-documents-api` is stateless (Redis + URLs to already-running services only). Nothing to add here for either.

`analyticsuser` also already exists from the old stack (confirmed live) — same reuse pattern as Phase 1.1, only the database is new.

```bash
ANALYTICS_DB_PASS=$(oc get secret servelegal-patroni-ff-db-secret -n "$NS" -o jsonpath='{.data.ANALYTICS_DB_PASSWORD}' | base64 -d)

oc run psql-client --rm -i --restart=Never -n "$NS" --image=postgres:16-alpine -- \
  env PGPASSWORD="$PATRONI_SUPERPASS" psql -U "$PATRONI_SUPERUSER" -h patroni-master <<SQL
CREATE DATABASE analyticsdb${DB_SUFFIX};
GRANT ALL PRIVILEGES ON DATABASE analyticsdb${DB_SUFFIX} TO analyticsuser;
SQL

# Add to the same consolidated secret from 1.3, rather than a separate one
oc patch secret formsflow-db-82 -n "$NS" --type=merge -p \
  "{\"stringData\":{\"ANALYTICS_DB_CONNECTION_STRING\":\"postgresql://analyticsuser:${ANALYTICS_DB_PASS}@patroni-master:5432/analyticsdb${DB_SUFFIX}\"}}"
```

Referenced in Phase 3.8 as `--set externalPostgreSQLSecret.name=formsflow-db-82 --set externalPostgreSQLSecret.key=ANALYTICS_DB_CONNECTION_STRING`.

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

**⚠️ Pre-Phase-3 blocker found and resolved 2026-08-17: old-stack Secrets collide with every new chart's default Secret name.** Right before starting Phase 3, checked whether the "no ConfigMap/Secret naming collision" finding from Phase -1 planning (see [[formsflow-keycloak-decision-pending]] in memory — that check turned out to be wrong/incomplete for Secrets specifically) actually held at install time. It didn't: `a60371-dev` already has non-Helm-managed Secrets named exactly `forms-flow-ai`, `forms-flow-api`, `forms-flow-bpm`, `forms-flow-forms`, `forms-flow-web`, and `forms-flow-analytics` — 4+ years old, from the live v4.0.8 stack, actively mounted by its DeploymentConfigs (confirmed via `oc get dc -o json` and grepping for each name). Every chart below defaults its own Secret name to `common.names.fullname` (≈ the release name, no `fullnameOverride` set), so `helm install` for each of these would fail with *"exists and cannot be imported into the current release."* (ConfigMaps checked too — no collision there, only Secrets.)

**Decision (confirmed with the user): delete the old Secrets rather than work around the collision with `fullnameOverride`.** The alternative (renaming every new release so nothing collides) would touch resource names across every chart and every cross-chart reference to those default names (e.g. `forms-flow-data-layer`'s `formsflow.secret=forms-flow-ai`), diverging further from AOT's stock chart defaults for no real benefit. The user's call: this environment has no active users in `dev` (or `test`/`prod` — nothing has been cut over yet), so it's safer and simpler to back up the old Secrets and delete them outright, clearing the path for Helm to create its own. **This reasoning is specific to the current no-active-users state — re-evaluate before repeating this step in `test`/`prod`** if either environment has real traffic by the time Phase 3 runs there; the `fullnameOverride` route is still the fallback if a from-scratch collision check ever needs a non-destructive fix.

Executed for `dev` 2026-08-17: verified the existing 2026-08-06 export (`~/Documents/ServeLegal/code/jag-servebc/openshift-export/secrets/*.json`) still exactly matched live state for all 6 colliding Secrets (byte-identical data, confirmed via hash comparison — the old stack hadn't been touched since that export, consistent with this migration's "don't touch the old stack" rule elsewhere), so no fresh backup was needed before deleting:

```bash
for s in forms-flow-ai forms-flow-analytics forms-flow-api forms-flow-bpm forms-flow-forms forms-flow-web; do
  oc delete secret "$s" -n "$NS"
done
```

Deleting a Secret doesn't crash already-running old-stack pods (env vars were injected into the container at start and aren't re-read live) — it only means those DCs can't cleanly restart until cutover replaces them, which is expected and acceptable given the decision above. **For `test`/`prod`: re-run the same collision check** (`oc get secret <release-name> -n "$NS"` for each of the six names above) **and re-verify the backup is current before deleting** — don't assume the `dev` outcome (safe to delete) applies without checking.

**⚠️ The same collision also exists for Service and Route objects — missed in the check above, found while installing `forms-flow-forms` (Phase 3.4, 2026-08-18).** The original collision check only covered Secrets and ConfigMaps; charts also create a Service (and, via their Ingress, an OpenShift Route) named after the release, and the old stack has those too: `forms-flow-forms`, `forms-flow-analytics`, `forms-flow-bpm`, `forms-flow-web` all collided (`forms-flow-api`/`forms-flow-data-layer`/`forms-flow-documents-api`/`servebc-api` did not — no legacy Service/Route at those exact names). Same fix, same reasoning as the Secrets above — both already covered by the existing 2026-08-06 export (`services-all.yaml`/`routes-all.yaml`):
```bash
for r in forms-flow-forms forms-flow-analytics forms-flow-bpm forms-flow-web; do
  oc delete service "$r" -n "$NS"
  oc delete route "$r" -n "$NS"
done
```
**Re-run this same 4-kind check (Secret/ConfigMap/Service/Route) against every release name before each phase from here on** — don't assume Phase 3.4's discovery means everything's now clean; not all remaining releases have been checked yet.

**⚠️ zsh gotcha that caused this to be missed the first time, worth remembering for any future scripting in this runbook:** `for r in $SOME_VAR; do ... done` does **not** word-split an unquoted variable in zsh (unlike bash) — the whole space-separated string gets treated as one literal token. A collision-scan loop built this way silently checked for a resource literally named `"forms-flow-forms forms-flow-api forms-flow-..."` (with spaces, obviously never found) instead of iterating each name — no error, just silent false negatives. Either use a real zsh array (`RELEASES=(a b c)` / `for r in "${RELEASES[@]}"`) when building a list from a variable, or write the list directly as literal words in the `for` statement (`for r in a b c; do`, which parses as separate tokens regardless of shell) rather than through an intermediate variable.

### 3.1 forms-flow-ai (infra shell — Redis only; Postgres/Mongo externalized)

```bash
# Source Mongo creds from the Phase 1.3/1.2 consolidated secret rather than a shell var that may
# not exist in this session — this is what actually fixed the "$FORMIO_DB_PASS lost between
# sessions" problem the 731 -> _82 rename ran into.
FORMIO_DB_NAME=$(oc get secret formsflow-db-82 -n "$NS" -o jsonpath='{.data.FORMIO_DB_NAME}' | base64 -d)
FORMIO_DB_USER=$(oc get secret formsflow-db-82 -n "$NS" -o jsonpath='{.data.FORMIO_DB_USER}' | base64 -d)
FORMIO_DB_PASS=$(oc get secret formsflow-db-82 -n "$NS" -o jsonpath='{.data.FORMIO_DB_PASSWORD}' | base64 -d)

# Generate the Form.io root admin credential HERE, once, and reuse it identically in Phase 3.4's
# forms-flow-forms install below — see the "FORMIO_ROOT credential mismatch" warning further down
# for why this must be the same value in both places, not independently generated per chart.
FORMIO_ROOT_EMAIL="<team-email>"
FORMIO_ROOT_PASS=$(openssl rand -base64 20)
oc create secret generic formsflow-forms-admin-82 -n "$NS" \
  --from-literal=FORMIO_ROOT_EMAIL="$FORMIO_ROOT_EMAIL" \
  --from-literal=FORMIO_ROOT_PASSWORD="$FORMIO_ROOT_PASS"

helm upgrade --install forms-flow-ai ./charts/forms-flow-ai \
  --namespace "$NS" \
  --set Domain="${DOMAIN}" \
  --set postgresql-ha.enabled=false \
  --set mongodb.enabled=false \
  --set mongodb.auth.usernames[0]="$FORMIO_DB_USER" \
  --set mongodb.auth.passwords[0]="$FORMIO_DB_PASS" \
  --set mongodb.auth.databases[0]="$FORMIO_DB_NAME" \
  --set mongodb.service.nameOverride=formio-mongodb-dev \
  --set mongodb.service.ports.mongodb=27017 \
  --set redisExporter.persistence.storageClass=netapp-block-standard \
  --set "forms-flow-forms.admin.email=$FORMIO_ROOT_EMAIL" \
  --set "forms-flow-forms.admin.password=$FORMIO_ROOT_PASS"

# Wait for Redis (the one piece of infra this chart still owns)
oc wait --namespace "$NS" --for=condition=ready pod \
  --selector=app.kubernetes.io/name=redis-exporter --timeout=120s
```

`redisExporter.persistence.storageClass` routes the new Redis PVC to `netapp-block-standard` instead of the default `netapp-file-standard` — per Phase 0, the default class is ~89% full in `dev`; this is the only PVC any Phase 3 chart creates (confirmed by grepping every chart's templates for `PersistentVolumeClaim`/`VolumeClaimTemplate`), so this one `--set` clears the storage-quota risk entirely. `ingress.ingressClassName` is intentionally omitted from every `--set` in this phase (not just here) — see the Phase 0 note above.

**⚠️ `Domain` must be the bare domain (`apps.silver.devops.gov.bc.ca`), not namespace-prefixed — a real bug found running Phase 3.5, root-caused back here.** The original draft passed `Domain="${NS}.${DOMAIN}"` (namespace-prefixed). This chart's own `values.yaml` builds several shared config values (`KEYCLOAK_URL` and friends, `FORMIO_DOMAIN`) as `forms-flow-idm-{{.Release.Namespace}}.{{tpl (.Values.Domain) .}}` — i.e. it **already prepends the namespace itself**, so passing a namespace-prefixed `Domain` doubled it: `forms-flow-idm-a60371-dev.a60371-dev.apps.silver.devops.gov.bc.ca`. This silently broke every downstream component reading `forms-flow-ai`'s shared `KEYCLOAK_URL`/`KEYCLOAK_JWT_OIDC_*`/`FORMIO_DOMAIN` config (confirmed: caused `forms-flow-api` to crash-loop on a `CERTIFICATE_VERIFY_FAILED: Hostname mismatch` trying to reach Keycloak — the cert is for the real single-namespace host, not the doubled one). Grepped every use of `.Values.Domain` in this chart to confirm `${DOMAIN}` bare is correct everywhere, not just for this one key, before fixing. **Fixed here** (`Domain="${DOMAIN}"` above) — this corrects the shared configmap for every component that reads it, no per-downstream-chart patch needed. If re-running against an environment already installed with the old, wrong value: `helm upgrade` with the corrected value, then **restart every pod that already read the old config** (ConfigMap changes don't propagate to already-running pods' env vars) — `oc delete pod -l app.kubernetes.io/instance=<release>` per affected release.

**⚠️ `FORMIO_ROOT_EMAIL`/`PASSWORD` must be the exact same value passed to Phase 3.4's `forms-flow-forms` install — another real bug found the same way.** This chart's `secrets.yaml` auto-generates `FORMIO_ROOT_EMAIL`/`FORMIO_ROOT_PASSWORD` in its own shared secret from `.Values["forms-flow-forms"].admin.email/password` — if left unset (as in the original draft), it silently falls back to the chart's placeholder defaults (`me@defineme.com`/`admin`), which do **not** match whatever real admin account Phase 3.4 actually creates in Form.io. Downstream components (`forms-flow-api` here, `forms-flow-data-layer` too) read *this* chart's secret to authenticate against Form.io — a mismatch causes silent login failures (`Generate formio token using formio login API` followed by `Expecting value: line 1 column 1` — a JSON-parse error from getting an HTML error page back instead of a token), not an install-time error. **Fixed by generating the credential once, here, and passing the identical value to both this install and Phase 3.4's** (updated below) — don't let the two independently generate their own.

**Mongo renamed to match Postgres's naming convention (2026-08-17)** — the original `formio731`/`formiouser731` (created before `DB_SUFFIX` changed from `731` to `_82`) has been dropped and recreated as database `formio_82` with a plain, unsuffixed user `formiouser`, per Phase 1.2 above — same "database tagged, user not" pattern as `bpmdb_82`/`bpmuser` etc. Sourced from the `formsflow-db-82` secret rather than hardcoded here, since Mongo's database naming doesn't mechanically follow `${DB_SUFFIX}` the way Postgres's does (fresh database each generation, not a suffix applied cleanly by string substitution) — reading it from the secret avoids the two ever silently drifting apart again.

**⚠️ The `mongodb.auth.*`/`mongodb.service.*` lines above are not optional — this chart has a real wiring bug that needs working around, found 2026-08-17.** `charts/forms-flow-ai/templates/secrets.yaml` auto-generates the `NODE_CONFIG`/`MONGODB_URI` keys in this chart's own central secret (see Phase 3.4's note) **entirely from `.Values.mongodb.auth.*`/`.Values.mongodb.service.*`** — i.e. the config for the chart's *own bundled Bitnami mongodb subchart* — regardless of whether `mongodb.enabled` is `true` or `false`. Setting `mongodb.enabled=false` alone (the Patroni/Mongo-externalization decision from earlier) stops the bundled subchart from deploying, but does **not** stop this template from generating `NODE_CONFIG` out of those same values — which, left at their chart defaults, would silently point at infrastructure that doesn't exist. The `--set` lines above repurpose those "would-be bundled Mongo" values to describe the *real* external `formio-mongodb-dev` instance and its actual Phase 1.2 credentials instead, so the auto-generated `NODE_CONFIG`/`MONGODB_URI` come out correct. Confirm explicitly in Phase 4 (e.g. exec into a pod that reads `NODE_CONFIG` and check the value, or just watch whether `forms-flow-forms` connects successfully) rather than assuming this is fixed just because the values are now set correctly — this exact class of bug (wrong value, no error until runtime) is why it went unnoticed in the original draft.

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
  --set database.username="servebcuser" \
  --set database.password="$SERVEBC_DB_PASS" \
  --set database.dbName="servebcdb" \
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

**⚠️ Two real bugs found and fixed running this step for `dev` (2026-08-17) — both needed before the install actually works:**

1. **Wrong toggle for the bundled Postgres.** `keycloak.postgresql.enabled=false` (as originally drafted here) does nothing — this chart's bundled Bitnami Postgres is declared as a **top-level** dependency (`postgresql-ha`, gated by `condition: postgresql-ha.enabled` in `charts/forms-flow-idm/Chart.yaml`), not nested under `keycloak.*`. Confirmed live: the first install attempt with the old `--set` silently deployed a full `forms-flow-idm-postgresql-postgresql` StatefulSet + `pgpool` Deployment (`bitnamilegacy/pgpool` image) anyway, which then hit the same storage-quota wall as Phase 0 flagged (`8Gi` request against an already ~91%-full storage class — the PVC creation failed, so no actual storage was wasted, but the pgpool pod itself ran needlessly). **Fix: use the top-level `--set postgresql-ha.enabled=false`**, same pattern as Phase 3.1 — this is exactly the "check the rendered manifest, don't assume the toggle nesting" warning already in Phase 3.1's note, now confirmed to bite for real.
2. **Hardcoded `runAsUser: 1001` on a custom init container breaks OpenShift's SCC.** `charts/forms-flow-idm/values.yaml`'s `keycloak.initContainers` includes a custom `formsflow-themes` container (pulls `formsflow/keycloak-customizations:v7.3.0` to seed themes/providers/realm-import files) with `securityContext.runAsUser: 1001` hardcoded. OpenShift's `restricted`/`restricted-v2` SCC only allows this namespace's assigned UID range (confirmed error: `runAsUser: Invalid value: 1001: must be in the ranges: [1011120000, 1011129999]`), so the Keycloak StatefulSet couldn't schedule at all — `0/1` forever, no SCC matched. The chart's *main* Keycloak container already does this correctly (`containerSecurityContext.enabled: false` / `podSecurityContext.enabled: false`, letting OpenShift auto-assign); this one custom init container just didn't follow that pattern. **Fix: `deploy-overrides/forms-flow-idm-overrides.yaml`** (now in this repo) re-specifies the same `formsflow-themes` init container with `securityContext` dropped entirely, passed via `-f` below. **Don't try to patch just the `securityContext` field with `--set`/`--set-json`** — confirmed live that targeting a single field of an existing array element via `--set-json 'keycloak.initContainers[0].securityContext={}'` silently wiped every *other* field of that same array element (`name`/`image`/`command`/`args`/`volumeMounts` all disappeared from the rendered manifest, verified via `helm template`) — a known Helm gotcha with `--set` on array indices, not a chart bug. A full values-override file avoids it.

```bash
KEYCLOAK_ADMIN_PASS=$(openssl rand -base64 24)

helm upgrade --install forms-flow-idm ./charts/forms-flow-idm \
  -f deploy-overrides/forms-flow-idm-overrides.yaml \
  --namespace "$NS" \
  --set keycloak.ingress.hostname="forms-flow-idm-${NS}.${DOMAIN}" \
  --set keycloak.ingress.tls=true \
  --set keycloak.auth.adminUser=admin \
  --set keycloak.auth.adminPassword="$KEYCLOAK_ADMIN_PASS" \
  --set postgresql-ha.enabled=false \
  --set keycloak.externalDatabase.existingSecret=formsflow-db-82 \
  --set keycloak.externalDatabase.existingSecretHostKey=KEYCLOAK_DB_HOST \
  --set keycloak.externalDatabase.existingSecretPortKey=KEYCLOAK_DB_PORT \
  --set keycloak.externalDatabase.existingSecretUserKey=KEYCLOAK_DB_USER \
  --set keycloak.externalDatabase.existingSecretDatabaseKey=KEYCLOAK_DB_NAME \
  --set keycloak.externalDatabase.existingSecretPasswordKey=KEYCLOAK_DB_PASSWORD

# Persist the admin credential durably (same lesson as the Mongo $FORMIO_DB_PASS loss earlier) —
# don't rely on the shell variable surviving to the next session.
oc create secret generic formsflow-admin-secrets-82 -n "$NS" \
  --from-literal=KEYCLOAK_ADMIN_USER=admin \
  --from-literal=KEYCLOAK_ADMIN_PASSWORD="$KEYCLOAK_ADMIN_PASS"

oc wait --namespace "$NS" --for=condition=ready pod \
  --selector=app.kubernetes.io/name=keycloak --timeout=300s
```

Confirmed working end-to-end for `dev` 2026-08-17: Keycloak pod `1/1 Running`, route reachable (`curl .../auth/realms/master/.well-known/openid-configuration` → `HTTP 200`), no bundled-Postgres resources left behind (Helm pruned the StatefulSet/Deployment created by the first, broken attempt once `postgresql-ha.enabled=false` was set correctly on the follow-up `helm upgrade`).

**Realm rebuild — done for `dev` 2026-08-18, via a real export/import from the still-running old Keycloak rather than guessing the structure.** What actually happened, in order:

1. **Don't guess the group/client/IDP structure — export it from the old realm.** The original plan here was to recreate the group tree/IDIR broker config from memory/docs. `CURRENT-CONFIG-a60371-dev.md` doesn't actually contain the group tree (only DC env vars) — and a first attempt at guessing it (by an earlier, disconnected session) produced plausible-looking but **wrong** group names (`approver`/`clerk` instead of the real `management`/`access-allow-submissions`/`staff`; generic `group1`/`group2` instead of `staff-reports`). Caught this by pulling the real structure straight from the old Keycloak's admin API (`POST /admin/realms/forms-flow-ai/partial-export?exportClients=true&exportGroupsAndRoles=true`) — the actual source of truth, still running (scaled to 0 normally; scale up for this, same pattern as Mongo in Phase 1.2). Saved to `openshift-export/old-keycloak-realm-export-<date>.json` (outside this repo, gitignored, matches the existing export convention).
   - **The old `keycloak` DC's admin credentials live in the `forms-flow-ai` secret — the same secret this runbook's Phase 3 pre-step deleted.** Don't recreate that secret (would collide with the new Helm-owned one of the same name). Instead: create a differently-named temp secret with the old `KEYCLOAK_USER`/`KEYCLOAK_PASSWORD` values (already backed up in `openshift-export/secrets-decoded/forms-flow-ai.env`), `oc set env dc/keycloak --from=secret/<temp-name>` to repoint just those two env vars, scale up, do the export, then **revert**: scale back to 0, patch the DC's env back to reference `forms-flow-ai` again, delete the temp secret. Old stack ends up byte-identical to before, just temporarily readable.
   - **`curl -d` doesn't URL-encode form data — a generated password containing `+` breaks Keycloak token auth silently** (server decodes `+` as a space per `application/x-www-form-urlencoded`, "Invalid user credentials" with no hint why). Always use `--data-urlencode` for the token request's `username`/`password` fields, not `-d`, whenever a password may contain `+`/`&`/etc. (which `openssl rand -base64` output frequently does).
2. **Real group structure** (confirmed from the export, now live in the new realm):
   ```
   /camunda-admin
   /formsflow
     /formsflow-designer
     /formsflow-reviewer
       /management
       /access-allow-submissions
       /staff
     /formsflow-client
   /formsflow-analytics
     /staff-reports
   ```
   (`/realm-management` also appears in the old export but is a Keycloak built-in client-role group, not custom — not recreated separately here.)
3. **Real IDIR identity provider recovered — config known, secret still missing.** The old realm's broker is `keycloak-oidc-gold` (display name "IDIR"), a BC Gov Common Hosted SSO (CSS) integration federating to `dev.loginproxy.gov.bc.ca`'s `standard` realm via OIDC, `clientId=serve-legal-documents-4299`, `client_secret_basic`. All URLs (auth/token/userinfo/jwks/issuer/logout) came through the export intact — **Keycloak's partial-export always redacts `clientSecret`**, and it isn't stored anywhere else in this namespace either (checked every secret). Created in the new realm with the full recovered config but **`enabled: false`** and a placeholder `clientSecret` — get the real secret from whoever manages the `serve-legal-documents-4299` integration on the BC Gov CSS/SSO portal (`bcgov.github.io/sso-requests` or your team's equivalent), then `PUT` it in and flip `enabled: true`.
4. **A pre-existing `forms-flow-analytics` SAML client (from the same disconnected session) doesn't match anything in the old realm either** — it has a `localhost:7000` redirect URI, clearly generic/placeholder, whereas the old realm actually had **3 different real SAML clients** for external reporting callbacks (`analytics-a60371-dev.apps.silver...`, `servebc-reports.dev.jag.gov.bc.ca`, `my-reports-dev.ospg.psfs.gov.bc.ca` — all in the export). Left as-is for now since `forms-flow-analytics` itself isn't deployed until Phase 3.8 — **needs a decision before that step**: recreate the 3 real SAML clients (if that external reporting integration is still needed) or replace this placeholder with something intentional.

```bash
KC_HOST="https://forms-flow-idm-${NS}.${DOMAIN}/auth"
TOKEN=$(curl -sk -X POST "${KC_HOST}/realms/master/protocol/openid-connect/token" \
  --data-urlencode "client_id=admin-cli" --data-urlencode "username=admin" \
  --data-urlencode "password=${KEYCLOAK_ADMIN_PASS}" --data-urlencode "grant_type=password" \
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

# Persist the BPM client secret durably immediately (same lesson as $FORMIO_DB_PASS/$KEYCLOAK_ADMIN_PASS earlier)
oc patch secret formsflow-admin-secrets-82 -n "$NS" --type=merge -p \
  "{\"stringData\":{\"BPM_CLIENT_SECRET\":\"${BPM_CLIENT_SECRET}\"}}"

# 3. Recreate the real group tree (see the structure above) — top-level groups first, then subgroups
#    nested under the returned parent group id (POST .../groups for top-level, POST
#    .../groups/{parentId}/children for subgroups).

# 4. Recreate the 3 synthetic test users (formsflow-client/designer/reviewer) — POST .../users,
#    PUT .../users/{id}/reset-password, PUT .../users/{id}/groups/{groupId}. Placed formsflow-reviewer
#    in the /formsflow/formsflow-reviewer/staff subgroup specifically (not the parent formsflow-reviewer
#    group, and not management/access-allow-submissions) as the general-case reviewer role for testing.

# 5. IDIR identity provider — see point 3 above. Config is real and complete; enabled:false with a
#    placeholder clientSecret until the real one is retrieved from the CSS/SSO portal.
```

> `$BPM_CLIENT_SECRET`, `$KEYCLOAK_ADMIN_PASS`, and the 3 test users' passwords are all persisted in `formsflow-admin-secrets-82` / `formsflow-test-users-82` (`oc create secret` alongside the steps above) — don't rely on shell variables surviving between sessions, same lesson as `$FORMIO_DB_PASS` in Phase 1.2.

### 3.4 forms-flow-forms (Form.io)

**Status for `dev`: done, 2026-08-18.** Mongo driver-compat risk checked and cleared, real image choice made, and two more real bugs found (Service/Route collision — see the Phase 3 intro note above — and a TLS/Ingress gap affecting every component from here on). Detail below.

**Mongo driver-compat risk — checked, not a blocker.** Pulled the actual `docker.io/formsflow/forms-flow-forms:v7.3.0` image's dependency versions directly (`oc run ... --command -- cat node_modules/mongodb/package.json`): `mongodb` driver `4.17.2`, `mongoose` `6.12.3`. MongoDB's official Node driver compatibility matrix supports server versions 3.6–6.0 for driver v4.x — `formio-mongodb-dev` (3.6.3) is within that range. The earlier `mongosh` failure in Phase 1.2 was that specific interactive-shell tool's own stricter, hardcoded ≥4.2 requirement — unrelated to what the underlying driver protocol actually supports. Confirmed live: connects and bootstraps cleanly (see below).

**Image choice: public `v7.3.0`, not an EE local build.** Checked the EE repo's `forms-flow-forms/` directory before deciding — it has **no `package.json` at all**; its `openshift_custom_Dockerfile` just `git clone`s a separately-configured `${FORMIO_SOURCE_REPO_URL}`/`${FORMIO_SOURCE_REPO_BRANCH}` (build args, not specified anywhere in this checkout) onto a **Node 12** base image. Murkier and less proven than the known-good, already-inspected public image — used the chart's default (`docker.io/formsflow/forms-flow-forms:v7.3.0`) instead.

**⚠️ `formio-mongodb-dev` must be scaled up and left running from this point on — it's a real runtime dependency now, not just an admin-task target.** Earlier phases (1.2, and ad-hoc admin checks) scaled it 0→1→0 as a temporary measure since nothing depended on it continuously. Once `forms-flow-forms` (and later `forms-flow-data-layer`) are actually deployed, they hold an open connection to it — first install attempt without this crashed with `MongoServerSelectionError: connect ECONNREFUSED` (connection string was correct — this just confirmed the earlier `NODE_CONFIG` wiring fix from Phase 3.1 works — the instance simply wasn't running).
```bash
oc scale dc/formio-mongodb-dev -n "$NS" --replicas=1
oc rollout status dc/formio-mongodb-dev -n "$NS" --timeout=120s
# Leave at replicas=1 from here on — do NOT scale back to 0 after this step, unlike Phase 1.2/admin tasks.
```

**⚠️ Use the exact same `FORMIO_ROOT_EMAIL`/`PASSWORD` already generated and passed to Phase 3.1's `forms-flow-ai` install — do not generate a fresh one here.** See the matching warning in Phase 3.1: this chart's admin account and `forms-flow-ai`'s shared secret (which `forms-flow-api`/`forms-flow-data-layer` read to authenticate against Form.io) must agree, or those downstream components silently fail to log in. Source from the secret Phase 3.1 already created:

```bash
FORMIO_ROOT_EMAIL=$(oc get secret formsflow-forms-admin-82 -n "$NS" -o jsonpath='{.data.FORMIO_ROOT_EMAIL}' | base64 -d)
FORMIO_ROOT_PASS=$(oc get secret formsflow-forms-admin-82 -n "$NS" -o jsonpath='{.data.FORMIO_ROOT_PASSWORD}' | base64 -d)
FORMIO_JWT_SECRET=$(openssl rand -hex 32)

helm upgrade --install forms-flow-forms ./charts/forms-flow-forms \
  --namespace "$NS" \
  --set ingress.hostname="forms-flow-forms-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set ingress.selfSigned=true \
  --set admin.email="$FORMIO_ROOT_EMAIL" \
  --set admin.password="$FORMIO_ROOT_PASS" \
  --set jwt.secret="$FORMIO_JWT_SECRET"

# Add the JWT secret to the same durable secret Phase 3.1 already started.
oc patch secret formsflow-forms-admin-82 -n "$NS" --type=merge -p \
  "{\"stringData\":{\"FORMIO_JWT_SECRET\":\"${FORMIO_JWT_SECRET}\"}}"
```

**No Mongo `--set` needed here — fixed 2026-08-17, was a real bug in the original draft.** `charts/forms-flow-forms/values.yaml` has **no `mongodb.uri` value path at all** (grepped the whole file for "mongo", zero matches) — the line that used to be here did nothing. This chart actually reads `NODE_CONFIG` via `secretKeyRef` from whatever secret `.Values.formsflow.secret` points at, which defaults to `forms-flow-ai` — the same central secret Phase 3.1 populates. As long as Phase 3.1's `mongodb.auth.*`/`mongodb.service.*` values are set correctly (see the warning box there), `NODE_CONFIG` already has the right Mongo connection string by the time this step runs, automatically, for this chart and anything else reading `formsflow.secret` (e.g. `forms-flow-data-layer`'s `FORMIO_DB_URI`). Verify explicitly in Phase 4 rather than assuming — this class of bug (wrong value, no install-time error) only shows up as a runtime connection failure. **Confirmed working live 2026-08-18** — pod logs show `Opening new connection to mongodb://formiouser:...@formio-mongodb-dev:27017/formio_82`, followed by a full template/role/admin bootstrap and `Serving the Form.io API Platform`.

**⚠️ `ingress.tls=true` alone renders no TLS at all in this chart (and likely others sharing this template) — a real, universal gap found here, applies to every remaining Ingress-based component below.** This chart's `templates/ingress.yaml` only renders a `tls:` block when `ingress.tls=true` **AND** one of: a cert-manager annotation, `ingress.secrets`, or `ingress.selfSigned` is *also* set — `ingress.tls=true` on its own (as originally drafted, and as still used for `forms-flow-idm`'s Keycloak in Phase 3.3) silently produces an Ingress with no `tls:` section at all. OpenShift's Ingress→Route controller then creates an **HTTP-only** Route (no `spec.tls`), and HTTPS requests to that host get the router's generic 503 "Application is not available" page — looks exactly like a broken app, but the app and its HTTP endpoint are both actually fine (`curl http://...` returned 200 the whole time). Confirmed via `helm template` dry-run before/after: adding `--set ingress.selfSigned=true` (added above) makes the `tls:` block render.

**`ingress.selfSigned=true` changes the template condition, but does not itself create the referenced cert/secret** — the Ingress ends up with `tls: - hosts: [...] secretName: <hostname>-tls`, referencing a secret that doesn't exist yet. Unlike Keycloak's case (whose ingress template renders a *bare empty* `tls: [{}]` with no `secretName`, which OpenShift's route-generator treats as "edge-terminate with the router's own default cert, no secret needed"), a `tls:` entry that names a **specific, missing** secret makes the route-generator skip creating a Route at all rather than falling back to a default — confirmed live: after adding `selfSigned=true` alone, `oc get route` for this release returned nothing, and the Ingress's `status.loadBalancer` went empty. **Fix: manually create that exact self-signed TLS secret** — cheap, one-time per hostname, works because OpenShift's edge termination is happy with any valid cert/key pair, not specifically a CA-signed one:
```bash
HOST="forms-flow-forms-${NS}.${DOMAIN}"
openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /tmp/selfsigned.key -out /tmp/selfsigned.crt -subj "/CN=${HOST}"
oc create secret tls "${HOST}-tls" -n "$NS" --cert=/tmp/selfsigned.crt --key=/tmp/selfsigned.key
rm -f /tmp/selfsigned.key /tmp/selfsigned.crt
```
Run this (with the right `$HOST` for that component) for **every remaining Ingress-based chart below** (`forms-flow-api`, `forms-flow-documents-api`, `forms-flow-analytics`, `forms-flow-bpm`, `servebc-api`, `forms-flow-web` — check `forms-flow-data-layer` too, though earlier notes say it has no public route by default) — add `--set ingress.selfSigned=true` alongside `--set ingress.tls=true` in each chart's own install command, and create the matching `<hostname>-tls` secret either just before or just after that `helm upgrade --install`. Confirmed working end-to-end for `forms-flow-forms`: route shows `TERMINATION: edge/Redirect`, `curl -sk https://.../formio/` → `HTTP 200`.

### 3.5 forms-flow-api (webapi)

**Status for `dev`: done, 2026-08-18.** Four real bugs found and fixed running this step — two turned out to be foundational bugs in `forms-flow-ai`'s shared config (documented back in Phase 3.1, since fixing them there benefits every downstream component reading that shared secret/configmap, not just this one), plus a chart-dependency gap and a cross-namespace image-pull gap specific to this step. Detail below; **the two Phase-3.1 fixes (`Domain` bare, `FORMIO_ROOT_EMAIL`/`PASSWORD` matching) must be in place before this step will work** — if running fresh, that's already handled by the corrected Phase 3.1 above; if resuming a partially-broken install, re-run Phase 3.1's `helm upgrade` with the fix first.

Build locally per the Phase 3.1 pattern (`<component>` = `forms-flow-api`) — no registry key yet. Also an explicit candidate for actual customization ("Camunda/API may need customizations" — TBD, not yet decided), which a local build accommodates either way.

**⚠️ This chart (like `forms-flow-forms`) needs its own `helm dependency build` — not just `forms-flow-ai`/`forms-flow-idm` as Phase 2 originally said.** Every remaining chart in this phase (`forms-flow-api`, `forms-flow-data-layer`, `forms-flow-documents-api`, `forms-flow-analytics`, `forms-flow-bpm`, `forms-flow-web`, `servebc-api`) declares its own `common`-chart (or, for `forms-flow-analytics`, `redis`/`postgresql`) Bitnami dependency and will fail with *"found in Chart.yaml, but missing in charts/ directory"* otherwise. Run once, for all of them, before starting Phase 3 rather than one at a time as each fails:
```bash
for chart in forms-flow-forms forms-flow-api forms-flow-data-layer forms-flow-documents-api forms-flow-analytics forms-flow-bpm forms-flow-web servebc-api; do
  helm dependency build "./charts/$chart"
done
```

**⚠️ Cross-namespace image pull from `a60371-tools` fails by default — a real, two-part blocker, not covered by Phase 0's checks.** First symptom: `ImagePullBackOff` with `authentication required` pulling `image-registry.openshift-image-registry.svc:5000/a60371-tools/forms-flow-api:...` into `a60371-dev`. The obvious fix (`oc policy add-role-to-group system:image-puller system:serviceaccounts:a60371-dev -n a60371-tools`) **failed with a `Forbidden` error — this account doesn't have RBAC-admin rights on `a60371-tools`** to create that binding (a real, unresolved gap — a cluster/project admin would need to grant this properly for a clean, permanent fix). Confirmed the *old* v4.0.8 stack's pods (still running, e.g. `forms-flow-webapi-14-kk96l`) already pull cross-namespace successfully using the `default` service account's auto-generated `default-dockercfg-ng9fs` secret — so a working cross-namespace grant already exists, just not one this account can extend to *new* service accounts. **Workaround, needs no extra permissions (namespace-local only):**
```bash
# Link the already-working pull secret onto the new chart-created SA (name matches the release)
oc secrets link forms-flow-api default-dockercfg-ng9fs --for=pull -n "$NS"
```
This alone wasn't enough, though — **this chart's `values.yaml` hardcodes `image.pullSecrets: [forms-flow-ai-auth]`** (intended for AOT's private registry via `forms-flow-ai`'s `imageCredentials.*`, which we've deliberately left unset — see Phase 3.1's note), and the pod spec's own `imagePullSecrets` list takes priority over whatever's linked to the SA, so the pull kept failing even after the link above. **Fix: override it explicitly** (added to the install command below) to point at the working secret instead:
```bash
--set "image.pullSecrets[0]=default-dockercfg-ng9fs"
```
**Both steps are needed for every remaining locally-built component below** (`forms-flow-data-layer`, `forms-flow-documents-api`, `forms-flow-bpm`, `forms-flow-web`, `servebc-api`) — the SA-link (substituting that component's own release name) and the `image.pullSecrets[0]` override, every time, until someone with the right access grants `system:image-puller` properly on `a60371-tools`.

```bash
oc secrets link forms-flow-api default-dockercfg-ng9fs --for=pull -n "$NS"

# TLS — same pattern established in Phase 3.4, needed for every Ingress-based component from here on.
HOST="forms-flow-api-${NS}.${DOMAIN}"
openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /tmp/selfsigned.key -out /tmp/selfsigned.crt -subj "/CN=${HOST}"
oc create secret tls "${HOST}-tls" -n "$NS" --cert=/tmp/selfsigned.crt --key=/tmp/selfsigned.key
rm -f /tmp/selfsigned.key /tmp/selfsigned.crt

helm upgrade --install forms-flow-api ./charts/forms-flow-api \
  --namespace "$NS" \
  --set image.registry=image-registry.openshift-image-registry.svc:5000 \
  --set image.repository="${TOOLS_NS}/forms-flow-api" \
  --set image.tag="${ENV}-v8.2.5" \
  --set "image.pullSecrets[0]=default-dockercfg-ng9fs" \
  --set ingress.hostname="forms-flow-api-${NS}.${DOMAIN}" \
  --set ingress.tls=true \
  --set ingress.selfSigned=true \
  --set ExternalDatabase.ExistingSecretName=formsflow-db-82 \
  --set ExternalDatabase.ExistingDatabaseHostKey=FORMSFLOW_API_HOSTNAME \
  --set ExternalDatabase.ExistingDatabaseNameKey=FORMSFLOW_API_DB_NAME \
  --set ExternalDatabase.ExistingDatabaseUserNameKey=FORMSFLOW_API_DB_USER \
  --set ExternalDatabase.ExistingDatabasePasswordKey=FORMSFLOW_API_DB_PASSWORD \
  --set FormioJWTExpire="240"
```

**Revised 2026-08-17 — the original `database.host/port/dbName/username/password` values didn't do what the earlier draft assumed.** This chart has no `database.*` value path wired into its actual DB connection at all (there's a top-level `database:` key, but it feeds something else, not the `DATABASE_HOST`/`_PASSWORD`/etc. env vars — those come exclusively from the `ExternalDatabase.*` block shown above, either pointed at an external secret, as here, or left unset so the chart mints its own auto-managed secret). Using `ExternalDatabase.ExistingSecretName` explicitly, as above, is what actually wires this chart to the consolidated secret from Phase 1.3 — confirmed against `charts/forms-flow-api/templates/deployment.yaml`, not assumed. Port isn't set here — it's not in the consolidated secret (not sensitive), and the chart's own default (5432, via its own auto-created ConfigMap) applies unless overridden.

**Confirmed working live 2026-08-18** — pod `2/2 Running`, no restarts, `curl -sk https://forms-flow-api-${NS}.${DOMAIN}/webapi/` → `HTTP 200`, clean gunicorn boot with no Keycloak/Form.io connection errors (both were broken by the Phase 3.1 bugs above until those were fixed and this pod was restarted to pick up the corrected shared config).

### 3.6 forms-flow-data-layer (EE scope addition — no new database, reuses forms-flow-api's secret)

Needs `forms-flow-api`, `forms-flow-forms`, and `forms-flow-idm` already up (it reads the webapi DB secret directly and cross-references Form.io/Keycloak config), hence its place in the order right after `forms-flow-api`. Build locally per the Phase 3.1 pattern (`<component>` = `forms-flow-data-layer`) — no registry key yet; this is a small, fast (~1-3 min) Python build.

```bash
helm upgrade --install forms-flow-data-layer ./charts/forms-flow-data-layer \
  --namespace "$NS" \
  --set image.registry=image-registry.openshift-image-registry.svc:5000 \
  --set image.repository="${TOOLS_NS}/forms-flow-data-layer" \
  --set image.tag="${ENV}-v8.2.5" \
  --set formsflow.webapi.secret=formsflow-db-82 \
  --set formsflow.webapi.configmap=forms-flow-api \
  --set formsflow.secret=forms-flow-ai \
  --set formsflow.configmap=forms-flow-ai
```

**`formsflow.webapi.secret` points at the Phase 1.3 consolidated secret, not a `forms-flow-api`-named one** — revised 2026-08-17 alongside the Phase 1.3 secrets consolidation. `forms-flow-data-layer` hardcodes the exact key names it looks for (`FORMSFLOW_API_HOSTNAME`/`FORMSFLOW_API_DB_NAME`/`FORMSFLOW_API_DB_USER`/`FORMSFLOW_API_DB_PASSWORD`, no override mechanism), which is why Phase 1.3's consolidated secret uses those specific names for the webapi entry rather than the `WEBAPI_DB_*` pattern used for the other components. `formsflow.webapi.configmap` still points at `forms-flow-api`'s own auto-created ConfigMap (unaffected by the secret change, just holds the non-sensitive port) — confirm that name against what Phase 3.5 actually creates (`common.names.fullname` resolved it to `forms-flow-api` when checked against the chart templates; verify with `oc get configmap -n "$NS"` if unsure).

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
  --set externalPostgreSQLSecret.name=formsflow-db-82 \
  --set externalPostgreSQLSecret.key=ANALYTICS_DB_CONNECTION_STRING
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
  --set camunda.ExternalDatabase.ExistingSecretName=formsflow-db-82 \
  --set camunda.ExternalDatabase.ExistingDatabaseHostKey=BPM_DB_HOST \
  --set camunda.ExternalDatabase.ExistingDatabaseNameKey=BPM_DB_NAME \
  --set camunda.ExternalDatabase.ExistingDatabaseUsernameKey=BPM_DB_USER \
  --set camunda.ExternalDatabase.ExistingDatabasePasswordKey=BPM_DB_PASSWORD \
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

**DB wiring revised 2026-08-17** — dropped the old `camunda.jdbc.url`/`.username`/`.password`/`camunda.database.name` values (confirmed against `charts/forms-flow-bpm/templates/deployment.yaml`: `camunda.jdbc.url`'s default already builds itself from the same `CAMUNDA_DATABASE_SERVICE_NAME`/`_PORT`/`_NAME` env vars that `camunda.ExternalDatabase.*` populates, so hardcoding a literal URL alongside those was redundant and risked drift if one changed without the other) in favor of pointing `camunda.ExternalDatabase.ExistingSecretName` at the Phase 1.3 consolidated secret, same pattern as `forms-flow-api`. `camunda.database.port` stays as a plain value — not sensitive, not part of the consolidated secret.

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

> `database.initJob.enabled=false` because Phase 1 already created `servebcdb` manually — leave the chart's own auto-creation Job off to avoid it trying (and likely failing on privilege grounds, since it expects the DB user itself to have `CREATEDB`) to redo that work.

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
