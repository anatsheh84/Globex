# Globex — Microservices Application for OpenShift

Kustomize-based deployment manifests for the **Globex** reference retail application — a multi-service demo used in Red Hat application modernisation workshops. The app is composed of six microservices plus two PostgreSQL databases, all from the `quay.io/redhat-gpte` image registry.

---

## Architecture

```
                         ┌──────────────────┐
                         │    globex-ui     │  Node.js frontend (port 8080)
                         └────────┬─────────┘
              ┌──────────┬────────┴──────────┬──────────────┐
              ▼          ▼                   ▼              ▼
         ┌─────────┐ ┌──────────────────┐ ┌─────────────┐ ┌───────────────┐
         │ catalog │ │ recommendation-  │ │  activity-  │ │ order-        │
         │(Spring) │ │ engine (Quarkus) │ │  tracking   │ │ placement     │
         └────┬────┘ └──────────────────┘ │  (Quarkus)  │ │  (Quarkus)    │
              │                           └─────────────┘ └───────────────┘
     ┌────────┴──────┐
     ▼               ▼
┌──────────┐   ┌──────────┐
│ catalog- │   │inventory │
│ database │   │(Quarkus) │
│(PostgreS)│   └─────┬────┘
└──────────┘         ▼
               ┌──────────────┐
               │  inventory-  │
               │  database    │
               │ (PostgreSQL) │
               └──────────────┘
```

Additionally, **activity-tracking-simulator** generates synthetic user activity events for demo/testing purposes.

---

## Services Summary

| Service | Runtime | Image | Port | Replicas | Notes |
|---|---|---|---|---|---|
| `globex-ui` | Node.js | `quay.io/redhat-gpte/globex-recommendation-ui:app-mod-workshop` | 8080 | 1 | Frontend; calls all back-end services |
| `catalog` | Spring Boot | `quay.io/redhat-gpte/globex-catalog:app-mod-workshop` | 8080 | 1 | Product catalogue; backed by PostgreSQL |
| `inventory` | Quarkus | `quay.io/redhat-gpte/globex-inventory:app-mod-workshop` | 8080 | 1 | Stock levels; backed by PostgreSQL |
| `recommendation-engine` | Quarkus | `quay.io/redhat-gpte/globex-recommendation-engine:app-mod-workshop` | 8080 | 1 | Product scoring API (`/score/product`) |
| `activity-tracking` | Quarkus | `quay.io/redhat-gpte/globex-activity-tracking-service:app-mod-workshop` | 8080 | **0** | Tracks user events; **scaled to 0 by default** |
| `order-placement` | Quarkus | `quay.io/redhat-gpte/globex-order-placement-service:app-mod-workshop` | 8080 | 1 | Order submission service |
| `activity-tracking-simulator` | — | — | — | 1 | Generates synthetic activity events for demos |
| `catalog-database` | PostgreSQL | — | 5432 | 1 | Backing store for `catalog` |
| `inventory-database` | PostgreSQL | — | 5432 | 1 | Backing store for `inventory` |

> **Note:** `activity-tracking` is intentionally deployed with `replicas: 0`. Scale it up manually or as part of a workshop exercise to enable live event tracking.

---

## Manifest Naming Convention

Files follow a clear prefix convention:

| Prefix | Kind | Example |
|---|---|---|
| `dep-` | `Deployment` | `dep-catalog.yaml` |
| `svc-` | `Service` | `svc-catalog.yaml` |
| `sa-` | `ServiceAccount` | `sa-catalog-app.yaml` |
| `secret-` | `Secret` | `secret-catalog-database.yaml` |
| `rt-` | `Route` | `rt-globex-ui.yaml` |

---

## File Reference

### Deployments
| File | Service | Key Config |
|---|---|---|
| `dep-globex-ui.yaml` | `globex-ui` | Env vars: `API_GET_PAGINATED_PRODUCTS`, `API_CATALOG_RECOMMENDED_PRODUCT_IDS`, `API_TRACK_PLACEORDER`, etc. pointing to sibling services |
| `dep-catalog.yaml` | `catalog` | DB credentials from `catalog-database` Secret; `INVENTORY_URL=inventory:8080` |
| `dep-inventory.yaml` | `inventory` | DB credentials from `inventory-database` Secret |
| `dep-recommendation-engine.yaml` | `recommendation-engine` | Config from `recommendation-engine` Secret; liveness/readiness at `/q/health/live` and `/q/health/ready` |
| `dep-activity-tracking.yaml` | `activity-tracking` | Config from `activity-tracking` Secret; **replicas: 0** |
| `dep-order-placement.yaml` | `order-placement` | Config from `order-placement` Secret; liveness/readiness at `/q/health` |
| `dep-activity-tracking-simulator.yaml` | `activity-tracking-simulator` | Generates synthetic events |
| `dep-catalog-database.yaml` | `catalog-database` | PostgreSQL for catalog service |
| `dep-inventory-database.yaml` | `inventory-database` | PostgreSQL for inventory service |

### Routes (OpenShift Ingress)
| File | Exposes |
|---|---|
| `rt-globex-ui.yaml` | `globex-ui` — main entry point for the application |
| `rt-catalog.yaml` | `catalog` API |
| `rt-activity-tracking-simulator.yaml` | Simulator endpoint |

### Secrets
| File | Used By |
|---|---|
| `secret-catalog-database.yaml` | `catalog` + `catalog-database` |
| `secret-inventory-database.yaml` | `inventory` + `inventory-database` |
| `secret-activity-tracking.yaml` | `activity-tracking` |
| `secret-order-placement.yaml` | `order-placement` |
| `secret-recommendation-engine.yaml` | `recommendation-engine` |
| `secret-image-registry.yaml` | Pull secret for `quay.io/redhat-gpte` images |

---

## Deploy

All resources are bundled in `kustomization.yaml` for a single-command deployment:

```bash
# Create and switch to the target namespace
oc new-project globex

# Deploy everything via Kustomize
oc apply -k .

# Check rollout status
oc get pods -n globex
oc get routes -n globex
```

To scale up the activity-tracking service (disabled by default):
```bash
oc scale deployment/activity-tracking --replicas=1 -n globex
```

---

## Prerequisites

- OpenShift cluster with access to `quay.io/redhat-gpte` images
- If the cluster is air-gapped, update the image references in each `dep-*.yaml` to point to your internal registry
- `oc` CLI installed and logged in with project-admin or cluster-admin privileges
