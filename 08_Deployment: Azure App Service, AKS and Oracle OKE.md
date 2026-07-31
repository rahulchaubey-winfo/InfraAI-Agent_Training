# Deployment: Azure App Service, AKS and Oracle OKE

**Document 08 of the InfraAI Agent technical series**

---

## Purpose of this document

This document covers how the platform is deployed, what infrastructure it requires, how changes
reach production, and what the current recovery position is.

Three deployment targets are supported. The Azure App Service path is fully automated. The
Kubernetes paths, for Azure AKS and Oracle OKE, are documented and manual.

Two sections are directly relevant to customer conversations. Section 3 describes the private
networking design, which is stronger than our commercial material currently reflects. Section 5
describes the OKE path, which is the appropriate recommendation for Oracle-centric customers and is
a genuine differentiator in that segment.

Section 8 records the current recovery position honestly, including what cannot presently be undone.

<img width="2589" height="1926" alt="alt_image (4)" src="https://github.com/user-attachments/assets/cfca01a3-4191-41ff-bdd9-e52eeb30cf4f" />


---

## 1. Deployment targets

| Target | Method | Status |
|---|---|---|
| Azure App Service | GitHub Actions, fully automated | Primary. The documented recommendation |
| Azure Kubernetes Service | `kubectl apply -k k8s/overlays/aks` | Documented, manual |
| Oracle Kubernetes Engine | `kubectl apply -k k8s/overlays/oke` | Documented, manual |

A self-hosted virtual machine path also exists, corresponding to the `deploy/self-hosted-vm` branch
and the `deploy-vm.yml` workflow. It is used for internal demonstration deployment.

No AWS deployment target is supported. An AWS EKS environment exists from an earlier phase and is
not part of the current deployment model. Its status is recorded in document 10.

---

## 2. What the platform requires

Regardless of target, the platform requires four things.

| Requirement | Detail |
|---|---|
| Container runtime | Two images: backend and frontend |
| PostgreSQL 16 | With the `pgvector` extension available |
| An AI provider | OpenAI, Anthropic, Google, or Azure AI Foundry |
| Network reachability | To monitored hosts and databases |

Everything else is configuration. There is no separate vector database, message broker, cache tier
or search cluster.

The database requirement is worth stating precisely for customer conversations. The platform needs a
PostgreSQL server and an empty database named `infraai`. It does not require tables to be created,
users to be provisioned or schema to be applied. Migrations run automatically on container start.

---

## 3. The Azure App Service architecture

This is the primary deployment path and the one to describe in a technical evaluation.

```text
Azure Resource Group
  |
  +-- Virtual Network  10.0.0.0/16
        |
        +-- snet-private-endpoints   10.0.2.0/24
        |     Container Registry, private endpoint, public access disabled
        |
        +-- snet-apps                10.0.1.0/24
        |     Backend App Service    port 8000, VNet integrated
        |     Frontend App Service   port 80,   VNet integrated
        |
        +-- snet-postgres            10.0.3.0/24
              PostgreSQL Flexible Server, VNet injected
```

### The private networking design

Five properties are worth stating in a security review, and all five are verified in the deployment
workflow.

**PostgreSQL is VNet-injected** with a private DNS zone. It has no public endpoint. It is not
reachable from the internet under any configuration.

**The container registry is Premium tier with public network access disabled**, reached through a
private endpoint with its own private DNS zone.

**App Services authenticate to the registry using managed identity.** No registry credentials exist
anywhere in the deployment. There is no username, no password and nothing to rotate.

**`WEBSITE_VNET_ROUTE_ALL` is set**, forcing all outbound traffic through the virtual network rather
than the public internet.

**Container images are built server-side** using `az acr build`. No Docker daemon is required in the
pipeline, and the build succeeds against a registry with public access disabled.

This is a well-constructed private topology. It is better than most products at comparable maturity
and it is currently under-represented in our commercial material.

### One correction to the current state

The deployment workflow contains a step titled "Configure access restrictions" whose body consists
only of comments and status output. No access restriction is applied.

Both App Services are therefore publicly reachable over HTTPS. The database and registry are
genuinely private; the application tier is not.

The workflow comment records the reason: the alert webhook endpoint must remain reachable by
Alertmanager, which cannot easily present credentials. That is a legitimate constraint with a
straightforward remedy, being IP restriction to known monitoring egress addresses or a shared secret
on the webhook.

This is recorded in document 10. Until it is closed, the accurate statement is that the data tier is
private and the application tier is internet-facing.

---

## 4. Continuous integration and delivery

Five workflows are defined.

| Workflow | Trigger | Purpose |
|---|---|---|
| `deploy.yml` | Manual | Component selector: infrastructure, backend, frontend, apps, all |
| `deploy-backend.yml` | Push to `backend/**` | Backend redeployment |
| `deploy-frontend.yml` | Push to `frontend/**` | Frontend redeployment |
| `deploy-vm.yml` | Manual | Self-hosted virtual machine target |
| `setup-azure.yml` | Manual | Azure bootstrap |

The principal workflow accepts a component selector, so infrastructure provisioning and application
deployment are separable:

| Component | Effect |
|---|---|
| `infrastructure` | Provisions network, registry, database and app services. First-time setup |
| `backend` | Builds and deploys the backend only |
| `frontend` | Builds and deploys the frontend only |
| `apps` | Both applications, no infrastructure change. **The routine case** |
| `all` | Infrastructure and both applications |

Job dependencies are chained with conditional guards so partial deployments behave correctly. Each
application job concludes with a health check that polls the health endpoint and fails the workflow
if the service does not respond.

Resource names are workflow inputs rather than hard-coded values, so the same workflow deploys to
differently named environments without modification. This matters for per-customer deployment.

---

## 5. Kubernetes deployment

The `k8s` directory uses Kustomize with a base and per-platform overlays.

```text
k8s/
  base/
    namespace.yaml  secrets.yaml  postgres.yaml
    backend.yaml    frontend.yaml  ingress.yaml
    kustomization.yaml
  overlays/
    aks/    ingress-patch  postgres-patch  kustomization
    oke/    backend-patch  frontend-patch  ingress-patch
            postgres-patch  ocir-secret  kustomization
```

The base holds platform-neutral manifests. Each overlay patches the platform-specific differences.

| Concern | AKS | OKE |
|---|---|---|
| Storage class | `managed-csi` | `oci-bv` |
| Ingress | NGINX Ingress Controller | OCI Native Ingress Controller |
| Registry | Azure Container Registry | OCI Container Registry with pull secret |

Deployment is a single command per platform:

```bash
kubectl kustomize k8s/overlays/oke      # preview
kubectl apply -k k8s/overlays/oke       # apply
```

### The OKE advantage

For Oracle-centric customers, OKE is the deployment to lead with, and the reason is architectural
rather than commercial.

An OKE cluster reaches Oracle Autonomous Database and Database Systems through the same virtual
cloud network, through an OCI Service Gateway, or through cross-VCN peering. No VPN, no
ExpressRoute, no public internet transit.

Since the platform's differentiator is live database diagnosis, the quality of that database
connection is not incidental. Running on OKE alongside the customer's Oracle estate makes the
diagnostic path shorter, faster and simpler to approve.

For Black Box, ADNOC and any customer whose estate is predominantly Oracle Cloud, this is the
recommendation.

---

## 6. Database provisioning

The platform requires PostgreSQL 16 with `pgvector` available.

| Environment | Provisioning |
|---|---|
| Azure App Service | Flexible Server created by the workflow, VNet injected |
| AKS | Azure Database for PostgreSQL recommended over in-cluster |
| OKE | OCI Database with PostgreSQL, or in-cluster with `oci-bv` storage |
| Local development | Container image with pgvector included |

### A required correction

The deployment workflow enables the `uuid-ossp` extension. It does not enable `vector`.

Consequently the retrieval subsystem described in document 06 cannot function on an environment
provisioned by that workflow. Either the extension has been enabled manually and is undocumented,
in which case a rebuild would not reproduce it, or retrieval has not been exercised in that
environment.

The correction is one line in the workflow:

```bash
az postgres flexible-server parameter set \
  --resource-group $RG --server-name $PG_SERVER \
  --name azure.extensions --value uuid-ossp,vector
```

Followed by `CREATE EXTENSION IF NOT EXISTS vector;` in a migration.

The same applies to local development. A standard `postgres:16-alpine` image does not include
pgvector. The `pgvector/pgvector:pg16` image should be used instead.

---

## 7. Local development

```bash
docker run -d --name infraai-pg \
  -e POSTGRES_DB=infraai \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 pgvector/pgvector:pg16

cd backend
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

cd frontend
npm install
npm run dev
```

The frontend serves on port 5173 and the backend on 8000. A combined `docker-compose` configuration
is also available.

Migrations run on backend start, so no manual database preparation is required locally either.

---

## 8. Recovery position

This section records the current state accurately rather than aspirationally.

### What is possible

Container images are tagged with the commit SHA, so a previous application version can be restored:

```bash
az webapp config container set \
  --resource-group infraai-rg --name infraai-backend \
  --container-image-name infraaiacr.azurecr.io/infraai/backend:PREVIOUS_SHA \
  --container-registry-url https://infraaiacr.azurecr.io

az webapp restart --resource-group infraai-rg --name infraai-backend
```

### What is not possible

**The database cannot be rolled back.** Migrations execute automatically on container start. A
deployment including a schema change has already applied it before the application becomes
available. Reverting the container image does not revert the schema.

`alembic downgrade` exists in principle. It has not been exercised on this platform, and downgrade
paths are not routinely tested.

**There is no pre-migration backup.** No dump is taken before migrations run, so there is no
restore point specific to the deployment.

**There are no deployment slots.** Azure App Service supports staging slots with atomic swap, which
would provide both validated deployment and instant rollback. They are not configured.

### The operational consequence

Any deployment containing a migration should be treated as forward-only. If it fails, recovery is
forward: identify the fault, correct it, deploy again.

### The remedy

Three changes address this and are recorded in document 11.

Add a `pg_dump` step before migrations execute, retained for a defined period.

Configure deployment slots, so new versions are validated before receiving traffic and rollback is
a swap.

Exercise downgrade paths in a non-production environment so `alembic downgrade` is a tested
procedure rather than a theoretical one.

---

## 9. Secrets in deployment

Secrets are supplied to the workflow as GitHub repository secrets and written to App Service
application settings.

| Secret | Purpose |
|---|---|
| `AZURE_CREDENTIALS` | Service principal for Azure authentication |
| `DB_PASSWORD` | PostgreSQL administrator password |
| `JWT_SECRET_KEY` | Token signing key |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Seeded administrator account |

Two observations.

Application settings hold these values in plaintext, including a database connection string with an
embedded password. Azure Key Vault references are supported by App Service and are not used. The
platform's own security checklist recommends Key Vault; the deployment does not implement it.

The seeded administrator password appears as a default value in the workflow and in documentation.
Any deployment where it has not been changed is trivially accessible. It should be removed from all
documentation and defaults.

Both items are recorded in document 10.

---

## 10. Operational commands

```bash
# Health
curl https://BACKEND_HOST/api/health

# Live logs
az webapp log tail --resource-group infraai-rg --name infraai-backend

# Service state
az webapp show -g infraai-rg -n infraai-backend --query state

# Migration state
az webapp ssh --resource-group infraai-rg --name infraai-backend
cd /app && alembic current && alembic history

# Kubernetes
kubectl get pods -n infraai
kubectl logs -n infraai deployment/backend --tail=100
```

### Common conditions

| Condition | Cause | Action |
|---|---|---|
| Container does not start | Invalid `DATABASE_URL` | App Service configuration settings |
| Migration fails | Database unreachable from VNet | VNet integration and subnet delegation |
| Authentication fails universally | `JWT_SECRET_KEY` changed between deployments | Keep the secret stable |
| Registry pull fails | Managed identity not assigned | Assign identity, grant `AcrPull` |
| Retrieval returns nothing | `pgvector` not enabled | See section 6 |
| Health check fails after deploy | Migration still running | Inspect logs |

---

## 11. Summary

1. Three deployment targets. Azure App Service is automated; AKS and OKE are documented and manual.
2. Requirements are a container runtime, PostgreSQL 16 with pgvector, an AI provider and network
   reachability. Nothing else.
3. The Azure private networking design is sound: VNet-injected database, private-endpoint registry,
   managed identity for image pull, all outbound traffic through the VNet.
4. The access restriction step in the workflow is not implemented. The data tier is private; the
   application tier is internet-facing.
5. Five CI/CD workflows. Resource names are inputs, so the same workflow serves differently named
   environments.
6. Kustomize base and overlays handle platform differences cleanly. OKE is the recommendation for
   Oracle-centric customers because the database path is shorter.
7. The `pgvector` extension is not enabled by the deployment workflow. Retrieval cannot function
   until this is corrected.
8. Application rollback is possible by image tag. Database rollback is not. Treat any deployment
   containing a migration as forward-only.
9. Secrets are held as plaintext application settings. Key Vault references are supported and
   unused.

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | Conceptual basis and architectural principle |
| 02 | Operational flow | A single alert traced from ingestion to notification |
| 03 | System architecture | The eight architectural layers |
| 04 | Execution modes | Single-call and orchestrated analysis |
| 05 | Data collection | Host and database diagnostic execution |
| 06 | Knowledge and retrieval | Organisational knowledge integration and vector search |
| 07 | Data model | Schema, storage and migration strategy |
| 08 | Deployment | This document |
| 09 | Safety and control | Command classification, approval workflow, guardrails |
| 10 | Current state assessment | Verified gaps and remediation priorities |
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set.*
