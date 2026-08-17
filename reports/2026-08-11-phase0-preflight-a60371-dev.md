# Phase 0 Pre-flight — `a60371-dev`

**Date:** 2026-08-11
**Cluster:** `api.silver.devops.gov.bc.ca` (OpenShift Silver)
**Namespace:** `a60371-dev`
**Run by:** `jaise-aot@github`
**Reference:** [`formsflow-ai-silver-runbook.md`](../formsflow-ai-silver-runbook.md), Phase 0

Pre-flight checks ahead of the v4.0.8 → v7.3.1 Helm-based formsflow.ai upgrade, per the runbook's Phase 0. No cluster state was changed during this run — read-only checks only.

## Summary

| Check | Result | Action needed |
|---|---|---|
| Login / project access | OK — `oc login` + `oc project a60371-dev` succeed | none |
| Default storage class | `netapp-file-standard` | none |
| Ingress class | Cannot enumerate (see below) | omit `ingress.ingressClassName` in Phase 3 `--set` flags |
| Compute quota | Ample headroom (4/100 pods, 600m/4 CPU, 1.5Gi/16Gi memory) | none |
| Storage quota | Tight on default class, but new PVC need is small (see below) | pin new Redis PVC to `netapp-block-standard` |
| Patroni (HA Postgres) | `patroni-0` not ready (0/1), `patroni-1` ready + master | non-blocking, matches prior known state |
| Form.io Mongo (`formio-mongodb-dev`) | Scaled to 0/0 replicas | **must scale to 1 before Phase 1.2** |

## Details

### Ingress class — RBAC blocked, resolved without it

`oc get ingressclass` and `oc get ingress.config.openshift.io cluster` both return `Forbidden` at cluster scope — this account only has namespace-scoped access to `a60371-dev`, not cluster-viewer rights. No existing `Ingress` objects in the namespace to infer a class from either.

Resolved by reading the chart source directly rather than guessing a name: every chart's `ingress.yaml` template (e.g. `charts/forms-flow-web/templates/ingress.yaml:15`) only sets `ingressClassName` when `.Values.ingress.ingressClassName` is non-empty. Leaving it unset is the chart's own supported default — OpenShift's router picks up the cluster's default IngressClass automatically.

**Decision:** omit `ingress.ingressClassName` from all Phase 3 `--set` flags, replacing the runbook's `<confirmed-from-Phase-0>` placeholders. Revisit only if a route doesn't come up correctly.

### Storage quota — tight on the default class, but new footprint is small

`oc describe quota` shows the default storage class (`netapp-file-standard`) at 28.5Gi / 32Gi used across 23 existing PVCs (~3.5Gi headroom). This looked like a potential blocker at first glance, so it was checked against the actual chart templates rather than assumed:

```
grep -rl "kind: PersistentVolumeClaim\|VolumeClaimTemplate" charts/{forms-flow-ai,forms-flow-idm,forms-flow-forms,forms-flow-api,forms-flow-bpm,forms-flow-web}/templates
```

Only `forms-flow-ai`'s Redis (`redis-exporter`) StatefulSet creates a PVC in this chart set — `redisExporter.persistence.size: 2Gi` default, single replica. Postgres, Mongo, and Keycloak are all externalized to existing Patroni/Mongo or have their bundled subcharts disabled per the plan, so they add no new PVCs.

2Gi fits inside the 3.5Gi headroom, but leaves thin margin.

**Decision:** add `--set redisExporter.persistence.storageClass=netapp-block-standard` to the Phase 3.1 `forms-flow-ai` install — that class currently has 0/32Gi used, removing the margin risk entirely instead of eating into the already-tight `netapp-file-standard` headroom.

**Open item:** the Bitnami `keycloak` subchart under `forms-flow-idm` isn't vendored locally yet (`helm dependency build` hasn't been run against it), so its own PVC behavior is unconfirmed (likely none, since its own Postgres dependency is disabled). Check via `helm template` before Phase 3.3, per the runbook's existing caution on that chart.

### Form.io Mongo — needs a scale-up before Phase 1.2

`formio-mongodb-dev` (DeploymentConfig) is currently scaled to 0/0 replicas, along with most of the old v4.0.8 stack — only `forms-flow-web` and `forms-flow-webapi` pods are actually running right now (`forms-flow-bpm`'s last deploy completed but has no running app pod).

Phase 1.2 needs a live Mongo pod behind the `formio-mongodb-dev` Service to create the new `formio${DB_SUFFIX}` user/database — it will fail as written against a 0-replica DC (no Service endpoints).

**Action for Phase 1:** `oc scale dc/formio-mongodb-dev -n a60371-dev --replicas=1` immediately before running Phase 1.2; safe to scale back to 0 afterward.

### Patroni

`patroni-0` is still not ready (0/1), `patroni-1` is ready and holds `role=master` — the same degraded-but-non-blocking state observed in the prior session's export (2026-08-06). Not worsened; not a blocker per the runbook's own guidance (a missing *master* would be a blocker, a degraded replica is not).

## Related documents

- Prior-session export of the live v4.0.8 stack (DeploymentConfigs, ConfigMaps, Secrets, routes/services) — `~/Documents/ServeLegal/code/jag-servebc/openshift-export/` (outside this repo, gitignored there; contains plaintext secrets, not duplicated here).
- Full upgrade plan and phase-by-phase commands — [`formsflow-ai-silver-runbook.md`](../formsflow-ai-silver-runbook.md).
