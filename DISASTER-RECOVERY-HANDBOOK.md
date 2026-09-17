# formsflow.ai EE — Disaster Recovery Handbook (dev)

**Purpose:** rebuild the `a60371-dev` formsflow.ai EE deployment from nothing, step by step. No narrative, no history — just what to run. If you want the story of *why* each step is the way it is, see `formsflow-ai-silver-runbook.md` in this repo.

**Scope:** `a60371-dev` only. Repeat the same steps against `a60371-test`/`a60371-prod` with their own namespace, hostnames, and secrets once dev is validated.

**Values convention:** every Helm chart below is installed with `-f values-local/<env>/<chart>.yaml`, not long inline `--set` chains. That directory is gitignored (real secrets live in it) — see `values-local/README.md`. **`values-local/dev/` already exists and reflects live `a60371-dev` today** (last captured 2026-09-16, real values already substituted) — for a dev rebuild, just use those files as-is, skip the YAML templates shown in each step below.

**For a new environment (`test`/`prod`):** the YAML shown in each step is a *template*, not a file you can save verbatim — `${GATEWAY}`/`${NS}`/`${DOMAIN}`/etc. inside a YAML block are **not** shell-expanded just because they look like it (that only happens inside actual bash strings, never inside file content). Either hand-substitute each placeholder before saving, or generate the real file with `envsubst` once the env vars from Step 0 are exported:
```bash
envsubst < template.yaml > values-local/${ENV}/forms-flow-ai.yaml   # repeat per chart
```

**Before you start, have these in hand** — this handbook tells you where each one gets *used*, not how to obtain it:

| Secret | Source | Can it be freshly generated instead? |
|---|---|---|
| CHES email API `clientSecret` | BC Gov CHES team / your CHES integration owner | No — registered externally, must match |
| S3 `accessKeyId` / `secretAccessKey` | Object storage bucket owner (`ag-pssg-sharedservices`) | No — must match the bucket's actual credentials |
| IDIR broker `clientSecret` (client `serve-legal-documents-4299`) | BC Gov CSS/SSO Requests portal (`bcgov.github.io/sso-requests`) | No — registered externally, must match |
| Everything else below (DB passwords, JWT secrets, Keycloak admin password, BPM client secret, formio root password, Redis password) | — | **Yes** — generate fresh with `openssl rand`, nothing external depends on the old value |

---

## 0. Prerequisites

- `oc` logged in with edit access to `a60371-dev`, `a60371-tools`
- `helm` v3, `git`, `jq`, `yq`, `openssl`, `curl`, `gettext` (for `envsubst`)
- This repo cloned: `git clone https://github.com/bcgov/jag-servebc-forms-flow-ai-charts.git`
- EE app source cloned: `git clone https://github.com/bcgov-c/jag-servebc-forms-flow-ai-ee.git` (private — needs `bcgov-c` org access)
- `jag-servebc` cloned (for `servebc-api`): `git clone https://github.com/bcgov/jag-servebc.git`, branch `release/2026.9.1` (or whatever's current — check with the team)

```bash
export ENV=dev
export NS=a60371-dev
export TOOLS_NS=a60371-tools
export DOMAIN=apps.silver.devops.gov.bc.ca
export GATEWAY="forms-flow-web-${NS}.${DOMAIN}"   # the one public hostname everything routes through
export EE_VERSION=v8.2.6
cd /path/to/jag-servebc-forms-flow-ai-charts
```

`oc get namespace "$NS" || oc new-project "$NS"` if it doesn't exist yet. Check storage/compute quota isn't already exhausted (`oc get resourcequota -n "$NS"`) before starting — dev has historically run tight on the default storage class.

---

## 1. Shared databases

Formsflow uses the existing shared Patroni Postgres cluster and a dedicated Mongo instance — it does **not** get its own Postgres via Helm (`postgresql-ha.enabled=false` everywhere below).

```bash
DB_SUFFIX=_82   # bump this if reusing an environment that already has a "_82" generation

# Create one DB + user per component on the shared Patroni cluster (formsflowdb, keycloakdb,
# bpmdb, webapidb, analyticsdb, servebcdb) — see Phase 1 of the main runbook for the exact
# `oc exec patroni-0 -- psql` commands if this cluster doesn't already have them.

# One consolidated secret holds every DB connection field components read via secretKeyRef —
# create/patch it with real values for whichever DBs you just created:
oc create secret generic "formsflow-db${DB_SUFFIX}" -n "$NS" \
  --from-literal=FORMSFLOW_API_HOSTNAME=patroni-master \
  --from-literal=FORMSFLOW_API_DB_NAME=webapidb${DB_SUFFIX} \
  --from-literal=FORMSFLOW_API_DB_USER=webapiuser \
  --from-literal=FORMSFLOW_API_DB_PASSWORD="$(openssl rand -base64 24)" \
  --from-literal=BPM_DB_HOST=patroni-master \
  --from-literal=BPM_DB_NAME=bpmdb${DB_SUFFIX} \
  --from-literal=BPM_DB_USER=bpmuser \
  --from-literal=BPM_DB_PASSWORD="$(openssl rand -base64 24)" \
  --from-literal=KEYCLOAK_DB_HOST=patroni-master \
  --from-literal=KEYCLOAK_DB_PORT=5432 \
  --from-literal=KEYCLOAK_DB_NAME=keycloakdb${DB_SUFFIX} \
  --from-literal=KEYCLOAK_DB_USER=keycloakuser \
  --from-literal=KEYCLOAK_DB_PASSWORD="$(openssl rand -base64 24)" \
  --from-literal=ANALYTICS_DB_CONNECTION_STRING="postgresql://analyticsuser:$(openssl rand -base64 24)@patroni-master:5432/analyticsdb${DB_SUFFIX}"

# Mongo (formio) — separate small dedicated instance, not Patroni
oc scale dc/formio-mongodb-dev -n "$NS" --replicas=1   # if the DC already exists from a prior install
oc rollout status dc/formio-mongodb-dev -n "$NS" --timeout=120s
```

Record the Mongo user/password/db name you use — you'll pass them into Step 2 (`mongodb.auth.*`).

---

## 2. `forms-flow-ai` (shared config + Redis)

`values-local/dev/forms-flow-ai.yaml` (template for a new environment — dev's copy already has real values):
```yaml
Domain: apps.silver.devops.gov.bc.ca   # ⚠️ bare, never namespace-prefixed — chart templates prepend it themselves
forms-flow-forms:
  admin:
    email: "<real formio root email — decide this now, also used in Step 5>"
    password: "<openssl rand -base64 20 | tr -d '+/=' — strip +/=, see curl gotcha note in Step 4>"
mongodb:
  enabled: false
  auth:
    usernames: ["formiouser"]
    passwords: ["<mongo password from Step 1>"]
    databases: ["formio_82"]
  service:
    nameOverride: formio-mongodb-dev
    ports:
      mongodb: 27017
postgresql-ha:
  enabled: false
redisExporter:
  persistence:
    storageClass: netapp-block-standard
```

```bash
helm upgrade --install forms-flow-ai charts/forms-flow-ai -f values-local/${ENV:-dev}/forms-flow-ai.yaml --namespace "$NS"
oc wait --namespace "$NS" --for=condition=ready pod \
  --selector=app.kubernetes.io/name=redis-exporter --timeout=120s
```

---

## 3. `forms-flow-servebc-config`

`values-local/dev/forms-flow-servebc-config.yaml` (template — `ches.clientSecret`/`s3.accessKeyId`/`s3.secretAccessKey` are external, see the table at the top):
```yaml
bpm:
  timezone: America/Vancouver
ches:
  clientId: "<from CHES team>"
  clientSecret: "<from CHES team>"
  keycloakUrl: https://dev.loginproxy.gov.bc.ca/auth/realms/comsvcauth/protocol/openid-connect/token
  serviceUrl: https://ches-dev.api.gov.bc.ca/api/v1/email
database:
  host: patroni-master
  port: 5432
  dbName: "servebcdb_82"   # match this env's DB_SUFFIX from Step 1
  username: servebcuser
  password: "<openssl rand -base64 24>"
s3:
  host: ag-pssg-sharedservices.objectstore.gov.bc.ca
  bucketName: "<this env's bucket>"
  accessKeyId: "<from bucket owner>"
  secretAccessKey: "<from bucket owner>"
  useSsl: true
servebc:
  externalUrl: "https://servebc-api-${NS}.${DOMAIN}"   # substitute your actual NS/DOMAIN
  internalUrl: http://servebc-api:3003
```

```bash
helm upgrade --install forms-flow-servebc-config charts/forms-flow-servebc-config \
  -f values-local/${ENV:-dev}/forms-flow-servebc-config.yaml --namespace "$NS"
```

---

## 4. `forms-flow-idm` (Keycloak)

`values-local/dev/forms-flow-idm.yaml` (template):
```yaml
keycloak:
  auth:
    adminUser: admin
    adminPassword: "<openssl rand -base64 24 | tr -d '+/='>"
  ingress:
    hostname: "forms-flow-idm-${NS}.${DOMAIN}"   # substitute your actual NS/DOMAIN
    tls: true
  externalDatabase:
    existingSecret: "formsflow-db_82"   # match this env's DB_SUFFIX from Step 1
    existingSecretHostKey: KEYCLOAK_DB_HOST
    existingSecretPortKey: KEYCLOAK_DB_PORT
    existingSecretUserKey: KEYCLOAK_DB_USER
    existingSecretDatabaseKey: KEYCLOAK_DB_NAME
    existingSecretPasswordKey: KEYCLOAK_DB_PASSWORD
postgresql-ha:
  enabled: false
```

```bash
KEYCLOAK_ADMIN_PASS="<same value as keycloak.auth.adminPassword above>"   # needed below for the realm setup

helm upgrade --install forms-flow-idm charts/forms-flow-idm -f values-local/${ENV:-dev}/forms-flow-idm.yaml --namespace "$NS"
oc wait --namespace "$NS" --for=condition=ready pod \
  --selector=app.kubernetes.io/name=keycloak --timeout=300s
```

**⚠️ Before merging any future upstream chart sync:** check `charts/forms-flow-idm/values.yaml`'s `keycloak.initContainers` (`formsflow-themes`) doesn't have a hardcoded `securityContext.runAsUser` — OpenShift's restricted SCC rejects it. Already stripped in this fork; a straight upstream merge could reintroduce it.

### Realm setup

**If you have a saved realm export** (`openshift-export/old-keycloak-realm-export-*.json` or similar), import it directly via the Admin Console (Realm settings → Import) or `POST /admin/realms/forms-flow-ai/partial-import`. That's the fast path — skip to "Verify" below.

**Otherwise, build it from scratch:**

```bash
KC_HOST="https://forms-flow-idm-${NS}.${DOMAIN}/auth"
TOKEN=$(curl -sk -X POST "${KC_HOST}/realms/master/protocol/openid-connect/token" \
  --data-urlencode "client_id=admin-cli" --data-urlencode "username=admin" \
  --data-urlencode "password=${KEYCLOAK_ADMIN_PASS}" --data-urlencode "grant_type=password" \
  | jq -r .access_token)
# ⚠️ always --data-urlencode for password fields — plain -d silently mangles a "+" as a space

curl -sk -X POST "${KC_HOST}/admin/realms" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"realm":"forms-flow-ai","enabled":true,"registrationAllowed":true,"loginWithEmailAllowed":true,"bruteForceProtected":true}'
```

**Client `forms-flow-web`** (public, direct-access-grants enabled, wildcard redirect for dev flexibility — tighten before prod):
```bash
curl -sk -X POST "${KC_HOST}/admin/realms/forms-flow-ai/clients" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{
  "clientId":"forms-flow-web","publicClient":true,"standardFlowEnabled":true,"directAccessGrantsEnabled":true,
  "redirectUris":["*"],"webOrigins":["*"],
  "protocolMappers":[
    {"name":"formsflow-web-mapper","protocol":"openid-connect","protocolMapper":"oidc-audience-mapper",
     "config":{"included.client.audience":"forms-flow-web","access.token.claim":"true"}},
    {"name":"camunda-rest-api","protocol":"openid-connect","protocolMapper":"oidc-audience-mapper",
     "config":{"included.custom.audience":"camunda-rest-api","access.token.claim":"true"}},
    {"name":"groups","protocol":"openid-connect","protocolMapper":"oidc-group-membership-mapper",
     "config":{"full.path":"true","claim.name":"groups","access.token.claim":"true","id.token.claim":"true","userinfo.token.claim":"true","multivalued":"true"}},
    {"name":"dashboard-mapper","protocol":"openid-connect","protocolMapper":"oidc-usermodel-attribute-mapper",
     "config":{"user.attribute":"dashboards","claim.name":"dashboards","access.token.claim":"true","id.token.claim":"true","userinfo.token.claim":"true","multivalued":"true","aggregate.attrs":"true"}}
  ]
}'
```
Then create the 26 client roles on this client (see `formsflow-ai-silver-runbook.md`'s Keycloak Exhibit A list, or the running realm's `GET .../clients/{id}/roles`, for the full names — grouped: design, submissions, tasks, filters, dashboards/metrics, administration).

**Client `forms-flow-bpm`** (confidential, service account):
```bash
BPM_CLIENT_SECRET="$(openssl rand -hex 16)"
curl -sk -X POST "${KC_HOST}/admin/realms/forms-flow-ai/clients" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"clientId\":\"forms-flow-bpm\",\"publicClient\":false,\"secret\":\"${BPM_CLIENT_SECRET}\",\"serviceAccountsEnabled\":true}"
```

**Group tree** (create top-level via `POST .../groups`, subgroups via `POST .../groups/{parentId}/children`), then assign the client roles listed above to each group per the runbook's Exhibit A/B mapping:
```
/camunda-admin
/formsflow
  /formsflow-admin        (+ realm-management client roles: manage-users, manage-clients, create-client,
                             query-users, query-clients, query-groups, view-clients, view-users, view-authorization)
  /formsflow-client
  /formsflow-designer
  /formsflow-reviewer
    /management
    /access-allow-submissions
    /staff
/formsflow-analytics
  /staff-reports
```

**Test users** (`formsflow-admin`, `formsflow-client`, `formsflow-designer`, `formsflow-reviewer`) — `POST .../users`, `PUT .../users/{id}/reset-password`, `PUT .../users/{id}/groups/{groupId}` into the matching group (put `formsflow-reviewer` in `/formsflow/formsflow-reviewer/staff` specifically).

**IDIR identity provider** (optional — only if federated staff login is needed):
```bash
curl -sk -X POST "${KC_HOST}/admin/realms/forms-flow-ai/identity-provider/instances" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{
  "alias":"keycloak-oidc-gold","displayName":"IDIR","providerId":"keycloak-oidc","enabled":true,
  "trustEmail":true,"storeToken":true,"firstBrokerLoginFlowAlias":"first broker login",
  "config":{
    "clientId":"serve-legal-documents-4299",
    "clientSecret":"<GET FROM BC GOV CSS/SSO PORTAL — see table at top>",
    "clientAuthMethod":"client_secret_basic",
    "authorizationUrl":"https://dev.loginproxy.gov.bc.ca/auth/realms/standard/protocol/openid-connect/auth",
    "tokenUrl":"https://dev.loginproxy.gov.bc.ca/auth/realms/standard/protocol/openid-connect/token",
    "userInfoUrl":"https://dev.loginproxy.gov.bc.ca/auth/realms/standard/protocol/openid-connect/userinfo",
    "jwksUrl":"https://dev.loginproxy.gov.bc.ca/auth/realms/standard/protocol/openid-connect/certs",
    "issuer":"https://dev.loginproxy.gov.bc.ca/auth/realms/standard",
    "logoutUrl":"https://dev.loginproxy.gov.bc.ca/auth/realms/standard/protocol/openid-connect/ext/legacy/logout",
    "useJwksUrl":"true","validateSignature":"true","syncMode":"IMPORT"
  }
}'
```
**⚠️ Before this actually works:** the BC Gov CSS/SSO client registration for `serve-legal-documents-4299` must have this instance's broker callback URL registered as a valid redirect URI: `https://forms-flow-idm-${NS}.${DOMAIN}/auth/realms/forms-flow-ai/broker/keycloak-oidc-gold/endpoint`. This is external — only whoever owns that SSO integration can add it. Cannot be worked around from our side.

---

## 5. `forms-flow-forms` (Form.io)

`values-local/dev/forms-flow-forms.yaml` (template — `admin.email`/`password` must match Step 2's `forms-flow-forms.admin.*` exactly):
```yaml
admin:
  email: "<same FORMIO_ROOT_EMAIL as Step 2>"
  password: "<same FORMIO_ROOT_PASS as Step 2>"
jwt:
  secret: "<openssl rand -hex 32>"
ingress:
  hostname: "${GATEWAY}"   # substitute this env's actual gateway host
  tls: false
  selfSigned: false
  extraTls: [{}]
```
`ingress.tls=false` + `extraTls: [{}]` is required exactly as shown — `ingress.tls=true` alone, or `selfSigned=true`, produces a broken/untrusted cert.

```bash
oc scale dc/formio-mongodb-dev -n "$NS" --replicas=1   # leave running from here on, don't scale back to 0
helm upgrade --install forms-flow-forms charts/forms-flow-forms -f values-local/${ENV:-dev}/forms-flow-forms.yaml --namespace "$NS"
```
See Verify step for confirming the root credential actually works (not just that the pod is reachable).

---

## 6. `forms-flow-api` (webapi)

`values-local/dev/forms-flow-api.yaml` (template):
```yaml
image:
  registry: image-registry.openshift-image-registry.svc:5000
  repository: "${TOOLS_NS}/forms-flow-api"
  tag: "dev-${EE_VERSION}"
  pullSecrets: ["default-dockercfg-ng9fs"]   # confirm this pull-secret name exists in your namespace
ingress:
  hostname: "${GATEWAY}"
  tls: false
  selfSigned: false
  extraTls: [{}]
ExternalDatabase:
  ExistingSecretName: "formsflow-db_82"   # match this env's DB_SUFFIX
  ExistingDatabaseHostKey: FORMSFLOW_API_HOSTNAME
  ExistingDatabaseNameKey: FORMSFLOW_API_DB_NAME
  ExistingDatabaseUserNameKey: FORMSFLOW_API_DB_USER
  ExistingDatabasePasswordKey: FORMSFLOW_API_DB_PASSWORD
FormioJWTExpire: 240
# ⚠️ Chart default (1Gi/512Mi) OOM-kills gunicorn's default 5 workers × 10 threads —
# always override, don't rely on the chart default.
resources:
  limits:
    memory: 1536Mi
  requests:
    memory: 768Mi
```

```bash
helm upgrade --install forms-flow-api charts/forms-flow-api -f values-local/${ENV:-dev}/forms-flow-api.yaml --namespace "$NS"
```

The shared `forms-flow-ai` secret (created in Step 2) must carry `FORMIO_ROOT_EMAIL`/`FORMIO_ROOT_PASSWORD` matching what you set in Step 2/5 — this is how `forms-flow-api` authenticates to formio as root. If it ever drifts from what formio's database actually has seeded (check by logging in directly: `POST /formio/user/login`), role/user-resource fetching fails with `Expecting value: line 1 column 1` in the logs, not an obvious error.

---

## 7. `forms-flow-analytics` (Redash)

`values-local/dev/forms-flow-analytics.yaml` (template — note `ingress.tls: [{}]`, a different shape than the `extraTls` pattern other charts use; this chart's own ingress template doesn't have a `selfSigned`/`extraTls` knob):
```yaml
ingress:
  hosts:
    - host: "${GATEWAY}"
      paths: ["/redash", "/setup"]
  tls: [{}]
externalPostgreSQLSecret:
  name: "formsflow-db_82"   # match this env's DB_SUFFIX
  key: ANALYTICS_DB_CONNECTION_STRING
server:
  env:
    REDASH_HOST: "https://${GATEWAY}/redash"
    REDASH_PROXIES_COUNT: 1
```

```bash
helm upgrade --install forms-flow-analytics charts/forms-flow-analytics -f values-local/${ENV:-dev}/forms-flow-analytics.yaml --namespace "$NS"
```

---

## 8. `forms-flow-bpm` (Camunda)

`values-local/dev/forms-flow-bpm.yaml` (template — `forms-flow-bpm.clientsecret` must match the Keycloak client secret from Step 4's realm setup):
```yaml
image:
  registry: image-registry.openshift-image-registry.svc:5000
  repository: "${TOOLS_NS}/forms-flow-bpm-ee"
  tag: "dev-${EE_VERSION}"
  pullSecrets: ["default-dockercfg-ng9fs"]
ingress:
  hostname: "${GATEWAY}"
  tls: false
  selfSigned: false
  extraTls: [{}]
forms-flow-bpm:
  clientsecret: "<same BPM_CLIENT_SECRET as Step 4's forms-flow-bpm Keycloak client>"
camunda:
  database:
    port: 5432
  ExternalDatabase:
    ExistingSecretName: "formsflow-db_82"   # match this env's DB_SUFFIX
    ExistingDatabaseHostKey: BPM_DB_HOST
    ExistingDatabaseNameKey: BPM_DB_NAME
    ExistingDatabaseUsernameKey: BPM_DB_USER
    ExistingDatabasePasswordKey: BPM_DB_PASSWORD
  websocket:
    securityOrigin: "https://${GATEWAY}"
```

```bash
helm upgrade --install forms-flow-bpm charts/forms-flow-bpm -f values-local/${ENV:-dev}/forms-flow-bpm.yaml --namespace "$NS"
```
Vanilla EE `forms-flow-bpm` — no custom Java extensions ported (real-time task-list push, custom Keycloak service-account token class). Users see stock poll/refresh behavior for task updates.

---

## 9. `servebc-api` (this org's own `/api`)

Build the image first (from `jag-servebc`, `api/` subdirectory — Dockerfile lives there, not repo root):
```bash
cd /path/to/jag-servebc/api
SHA=$(git rev-parse --short HEAD)
oc start-build servebc-api --from-dir=. -n "$TOOLS_NS" --follow
oc tag "${TOOLS_NS}/servebc-api:latest" "${TOOLS_NS}/servebc-api:dev-${SHA}"
```
`values-local/dev/servebc-api.yaml` (template — update `image.tag` to the `$SHA` just built):
```yaml
image:
  registry: image-registry.openshift-image-registry.svc:5000
  repository: "${TOOLS_NS}/servebc-api"
  tag: "dev-<SHA from build step above>"
  pullPolicy: Always
  pullSecrets: ["default-dockercfg-ng9fs"]
ingress:
  hostname: "servebc-api-${NS}.${DOMAIN}"
  tls: false
  selfSigned: false
  extraTls: [{}]
formsflow:
  servebcConfigmap: forms-flow-servebc
  servebcSecret: forms-flow-servebc
database:
  initJob:
    enabled: false
```

```bash
helm upgrade --install servebc-api charts/servebc-api -f values-local/${ENV:-dev}/servebc-api.yaml --namespace "$NS"
```

---

## 10. Frontend — `forms-flow-web`, `forms-flow-web-root-config`, microfrontends

Build all EE frontend images from `bcgov-c/jag-servebc-forms-flow-ai-ee` (see that repo's own OpenShift-specific Dockerfile fixes — already applied there, don't need to be redone):
```bash
cd /path/to/forms-flow-ai-ee
for d in forms-flow-web forms-flow-web-root-config forms-flow-api forms-flow-bpm; do
  ( cd "$d" && oc start-build "${d}$( [ "$d" = forms-flow-web ] || [ "$d" = forms-flow-bpm ] && echo -ee )" \
      --from-dir=. -n "$TOOLS_NS" --follow )
done
```
(`forms-flow-api`/`forms-flow-bpm` were already built in Steps 6/8 above if following in order — this is just for reference if rebuilding standalone.)

**`forms-flow-web`** (Helm-managed). `values-local/dev/forms-flow-web.yaml` (template):
```yaml
image:
  registry: image-registry.openshift-image-registry.svc:5000
  repository: "${TOOLS_NS}/forms-flow-web-ee"
  tag: "dev-${EE_VERSION}"
  pullSecrets: ["default-dockercfg-ng9fs"]
ingress:
  hostname: "${GATEWAY}"
  tls: false
  selfSigned: false
  extraTls: [{}]
IsEnterPrise: "true"
web:
  base_custom_url: "https://servebc-api-${NS}.${DOMAIN}"
  custom_theme_url: ""
  enable_applications_module: "true"
  enable_forms_module: "true"
  enable_processes_module: "true"
  enable_tasks_module: "true"
  enable_dashboards_module: "false"
```

```bash
helm upgrade --install forms-flow-web charts/forms-flow-web -f values-local/${ENV:-dev}/forms-flow-web.yaml --namespace "$NS"
```

**`forms-flow-web-root-config`** — NOT Helm-managed, plain manifests (`oc apply`/`oc set env` per the runbook's Phase 3.11). Build, then:
```bash
oc set image deployment/forms-flow-web-root-config \
  forms-flow-web-root-config=image-registry.openshift-image-registry.svc:5000/${TOOLS_NS}/forms-flow-web-root-config:dev-${EE_VERSION} -n "$NS"
oc set env deployment/forms-flow-web-root-config -n "$NS" \
  REACT_APP_API_SERVER_URL="https://${GATEWAY}/formio" \
  REACT_APP_API_PROJECT_URL="https://${GATEWAY}/formio" \
  REACT_APP_WEB_BASE_URL="https://${GATEWAY}/webapi" \
  REACT_APP_BPM_URL="https://${GATEWAY}/camunda" \
  REACT_APP_KEYCLOAK_URL="https://forms-flow-idm-${NS}.${DOMAIN}" \
  REACT_APP_KEYCLOAK_URL_REALM=forms-flow-ai \
  REACT_APP_KEYCLOAK_CLIENT=forms-flow-web \
  REACT_APP_APPLICATION_NAME="ServeLegal"
```
(Route: `forms-flow-web-root-config-${NS}.${DOMAIN}` — this is the actual public entry point users log in at, distinct from `${GATEWAY}` which is the API/asset gateway.)

**Microfrontends** — 8 packages (`forms-flow-admin`, `forms-flow-components`, `forms-flow-integration`, `forms-flow-nav`, `forms-flow-review`, `forms-flow-service`, `forms-flow-submissions`, `forms-flow-theme`) built as **one combined image** from `bcgov/jag-servebc-forms-flow-ai-micro-front-ends` (`main` branch), deployed as a plain Deployment+Service+Route (no Helm). See that repo's own `.github/workflows/README.md` for the build pipeline, or build locally with the same binary-build pattern as above. Route: `forms-flow-microfrontend-${NS}.${DOMAIN}`.

---

## 11. Verify

```bash
# Keycloak
curl -sf "https://forms-flow-idm-${NS}.${DOMAIN}/auth/realms/forms-flow-ai/.well-known/openid-configuration" | jq .issuer

# Everything behind the gateway hostname
curl -sk -o /dev/null -w "root -> %{http_code}\n" "https://${GATEWAY}/"
curl -sk -o /dev/null -w "webapi -> %{http_code} (expect 401)\n" "https://${GATEWAY}/webapi/formio/roles"
curl -sk -o /dev/null -w "camunda -> %{http_code} (expect 302)\n" "https://${GATEWAY}/camunda/"
curl -sk -o /dev/null -w "formio -> %{http_code}\n" "https://${GATEWAY}/formio/"
curl -sk -o /dev/null -w "redash -> %{http_code}\n" "https://${GATEWAY}/redash"

# formio root credential actually works, not just reachable — read straight from the
# values file rather than a shell var, since nothing above exports one anymore
FORMIO_ROOT_EMAIL=$(yq '.["forms-flow-forms"].admin.email' values-local/${ENV:-dev}/forms-flow-ai.yaml)
FORMIO_ROOT_PASS=$(yq '.["forms-flow-forms"].admin.password' values-local/${ENV:-dev}/forms-flow-ai.yaml)
curl -sk -o /dev/null -w "formio login -> %{http_code} (expect 200)\n" -X POST "https://${GATEWAY}/formio/user/login" \
  -H "Content-Type: application/json" -d "{\"data\":{\"email\":\"${FORMIO_ROOT_EMAIL}\",\"password\":\"${FORMIO_ROOT_PASS}\"}}"

# servebc-api
curl -sk -o /dev/null -w "servebc-api -> %{http_code}\n" "https://servebc-api-${NS}.${DOMAIN}/api/v1/healthcheck"

# the actual entry point — do this one in a real browser, not curl
echo "https://forms-flow-web-root-config-${NS}.${DOMAIN}/"
```

**Real browser smoke test, not optional:** log in as `formsflow-client`, submit a form, confirm a Camunda process + webapi application record is created; log in as `formsflow-reviewer`, confirm the task appears (manual refresh, not real-time — known gap); log in as `formsflow-designer`, confirm the form builder loads and a new form can be created and saved.

---

## Known permanent gaps (don't try to fix these during a rebuild — they're accepted, documented limitations)

- No real-time task-list push (custom Java extension not ported)
- No custom `ServiceFlow` public NCQ document-serving flow
- IDIR login requires an external BC Gov CSS/SSO portal update (redirect URI) that only the SSO integration owner can make
- Bitnami `keycloak`/`postgresql-ha`/`mongodb` images are on the frozen `bitnamilegacy` catalog — fine for dev, revisit before prod
