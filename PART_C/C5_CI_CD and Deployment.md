## C5: CI/CD and Deployment

***

### What This Chapter Covers

How InfraAI Agent gets built, tested, packaged, and deployed to production. This chapter covers the entire pipeline from code commit to running application — GitHub Actions, Docker, Kubernetes, multi-cloud deployment, and environment management.

***

### Source Transparency

| Source            | Content                                                                                                                                  |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Your data (v1)    | GitHub Actions, Docker, Kubernetes (AKS + OKE), Kustomize overlays, ACR, Alembic migrations, entrypoint.sh, docker-compose for local dev |
| My expansion (v2) | Pipeline stages, multi-cloud mapping, environment strategy. All standard DevOps practices, fully implementable.                          |

***

### The Big Picture — Code to Production

    Developer commits code
         |
         v
    GitHub Actions triggers pipeline
         |
         v
    Build stage: Run tests, lint code, security scan
         |
         v
    Package stage: Build Docker images (backend + frontend)
         |
         v
    Push stage: Push images to container registry (ACR / ECR)
         |
         v
    Deploy stage: Apply Kubernetes manifests via Kustomize
         |
         v
    Runtime: Containers start, Alembic runs migrations, app serves traffic

This is a standard CI/CD pipeline. Nothing unusual. The key decisions are in the details — what gets containerised, how environments are managed, and how multi-cloud deployment works.

***

### The Repository Structure

InfraAI is a monorepo with backend and frontend in the same repository.

    infraai-agent/
         |
         +--- backend/
         |      +--- app/              (FastAPI application code)
         |      +--- alembic/          (database migrations)
         |      +--- Dockerfile        (backend container definition)
         |      +--- entrypoint.sh     (startup script)
         |      +--- requirements.txt  (Python dependencies)
         |
         +--- frontend/
         |      +--- src/              (React application code)
         |      +--- Dockerfile        (frontend container definition)
         |      +--- package.json      (JavaScript dependencies)
         |
         +--- k8s/
         |      +--- base/             (base Kubernetes manifests)
         |      +--- overlays/
         |           +--- dev/         (dev environment overrides)
         |           +--- staging/     (staging environment overrides)
         |           +--- production/  (production environment overrides)
         |
         +--- .github/
         |      +--- workflows/        (GitHub Actions pipeline definitions)
         |
         +--- docker-compose.yml       (local development setup)

***

### Stage 1: Build and Test

Triggered on every pull request and merge to main branch.

    GitHub Actions workflow:

      Step 1: Checkout code

      Step 2: Backend tests
        - Set up Python 3.12
        - Install dependencies (pip install -r requirements.txt)
        - Run linting (flake8 or ruff)
        - Run unit tests (pytest)
        - Run security scan (bandit for Python vulnerabilities)

      Step 3: Frontend tests
        - Set up Node.js
        - Install dependencies (npm install)
        - Run linting (eslint)
        - Run build (vite build — catches compilation errors)

      Step 4: Pass/fail gate
        - All tests must pass before proceeding
        - Failed tests block the merge

This stage ensures no broken code reaches the container build phase.

***

### Stage 2: Docker Build

Two Docker images are built — one for backend, one for frontend.

**Backend Dockerfile (simplified):**

    Base image: Python 3.12-slim
    Install: system dependencies (libpq for PostgreSQL, SSH libraries)
    Copy: requirements.txt, install Python packages
    Copy: application code
    Copy: entrypoint.sh
    Entrypoint: entrypoint.sh

**Frontend Dockerfile (simplified):**

    Stage 1 (build):
      Base image: Node 20
      Copy: package.json, install dependencies
      Copy: source code
      Run: vite build (produces static files)

    Stage 2 (serve):
      Base image: nginx
      Copy: built static files from Stage 1
      Serve via nginx on port 80

The frontend uses a multi-stage build. The first stage compiles the React application. The second stage copies only the compiled output into a lightweight nginx container. The final image is small and contains no build tools or source code.

**Image tagging convention:**

    registry/infraai-backend:1.2.3        (semantic version)
    registry/infraai-backend:main-abc1234  (branch + commit hash)
    registry/infraai-backend:latest        (latest build)

***

### Stage 3: Push to Container Registry

Built images are pushed to a private container registry.

| Cloud | Registry Service                 | Access Control                                      |
| ----- | -------------------------------- | --------------------------------------------------- |
| Azure | Azure Container Registry (ACR)   | Private endpoint within VNet, managed identity auth |
| AWS   | Elastic Container Registry (ECR) | VPC endpoint, IAM role auth                         |
| GCP   | Artifact Registry                | VPC Service Controls, service account auth          |

The registry is private. Images are not publicly accessible. Authentication is handled through cloud-native identity (managed identity on Azure, IAM roles on AWS, service accounts on GCP).

***

### Stage 4: Deploy via Kubernetes and Kustomize

InfraAI uses Kubernetes for container orchestration and Kustomize for environment-specific configuration.

**Why Kubernetes:**

*   Handles container lifecycle (start, stop, restart on failure)
*   Scales horizontally (more replicas for more load)
*   Rolling updates (zero-downtime deployments)
*   Health checks (automatic restart of unhealthy containers)
*   Works on every cloud (AKS on Azure, EKS on AWS, GKE on GCP, OKE on OCI)

**Why Kustomize (instead of Helm):**

*   Built into kubectl (no additional tooling)
*   Simple overlay model: base manifests + per-environment patches
*   No templating language to learn
*   Easy to understand what each environment changes

**How Kustomize works:**

    k8s/base/
      Contains the default Kubernetes manifests:
      - deployment.yaml (backend container spec)
      - service.yaml (internal networking)
      - configmap.yaml (non-sensitive configuration)
      - secrets.yaml (reference to cloud secret store)

    k8s/overlays/dev/
      Patches for development environment:
      - Smaller resource limits (less CPU, less memory)
      - 1 replica
      - Dev database connection string
      - Debug logging enabled

    k8s/overlays/staging/
      Patches for staging:
      - Medium resource limits
      - 2 replicas
      - Staging database connection string
      - Standard logging

    k8s/overlays/production/
      Patches for production:
      - Full resource limits
      - 3+ replicas
      - Production database connection string
      - Structured logging
      - Tighter security policies

**Deployment command:**

    kubectl apply -k k8s/overlays/production/

This applies the base manifests with production-specific overrides. One command. Repeatable. Version-controlled.

***

### Runtime: What Happens When Containers Start

**Backend container startup sequence (entrypoint.sh):**

    Step 1: Run Alembic migrations
      alembic upgrade head
      - Creates tables if first deployment
      - Applies new migrations if existing deployment
      - Existing data preserved

    Step 2: Seed default data (if first deployment)
      - Default admin user
      - Default AI provider configurations
      - Default agent profiles (8 domains)
      - Default application settings

    Step 3: Start FastAPI application
      uvicorn app.main:app --host 0.0.0.0 --port 8000

**Key point:** The database migration runs automatically on every deployment. No manual SQL scripts. No DBA involvement. If the deployment includes a new table or column, Alembic handles it. If nothing changed, Alembic does nothing.

***

### Environment Strategy

    ENVIRONMENT    PURPOSE                    DATABASE        AI PROVIDER
    -----------    -------                    --------        -----------
    Local dev      Developer's laptop         Docker postgres  OpenAI Direct (test key)
                   docker-compose up                          or local Ollama
                   
    Dev            Shared development         Managed DB       OpenAI Direct or
                   Testing new features       (small instance) Azure OpenAI (dev)

    Staging        Pre-production testing     Managed DB       Azure OpenAI
                   Customer demos             (medium)         (same model as prod)
                   
    Production     Live customer deployment   Managed DB       Azure OpenAI
                   Real alerts, real data     (production-     (private endpoint,
                                              grade, HA)       customer's tenant)

**Local development with docker-compose:**

    docker-compose.yml defines:

      Service 1: backend
        Build from: ./backend/Dockerfile
        Ports: 8000
        Depends on: postgres
        Environment: DATABASE_URL, AI_API_KEY, etc.

      Service 2: frontend
        Build from: ./frontend/Dockerfile
        Ports: 80
        
      Service 3: postgres
        Image: pgvector/pgvector:pg16
        Ports: 5432
        Volume: persistent data

    One command to start everything:
      docker-compose up --build

    Developer has full InfraAI running locally in minutes.

***

### Multi-Cloud Deployment

InfraAI is designed to deploy on any major cloud. The application code is identical. Only the infrastructure layer changes.

**Azure deployment (primary, from your documentation):**

| Component            | Azure Service                   |
| -------------------- | ------------------------------- |
| Kubernetes           | AKS (Azure Kubernetes Service)  |
| Alternative (no K8s) | App Service (VNet-injected)     |
| Database             | PostgreSQL Flexible Server      |
| Container registry   | ACR (private endpoint)          |
| AI                   | Azure OpenAI (private endpoint) |
| Secrets              | Key Vault                       |
| DNS                  | Azure DNS                       |
| Monitoring           | Application Insights            |

**OCI deployment (from your documentation):**

| Component          | OCI Service                                     |
| ------------------ | ----------------------------------------------- |
| Kubernetes         | OKE (Oracle Kubernetes Engine)                  |
| Database           | PostgreSQL on OKE or OCI Database               |
| Container registry | OCI Container Registry                          |
| AI                 | OCI Generative AI or Azure OpenAI (cross-cloud) |
| Secrets            | OCI Vault                                       |

**AWS deployment (supported):**

| Component            | AWS Service                    |
| -------------------- | ------------------------------ |
| Kubernetes           | EKS                            |
| Alternative (no K8s) | ECS Fargate or App Runner      |
| Database             | RDS PostgreSQL (with pgvector) |
| Container registry   | ECR                            |
| AI                   | Bedrock (private)              |
| Secrets              | Secrets Manager                |
| Monitoring           | CloudWatch + X-Ray             |

**GCP deployment (supported):**

| Component            | GCP Service                    |
| -------------------- | ------------------------------ |
| Kubernetes           | GKE                            |
| Alternative (no K8s) | Cloud Run                      |
| Database             | Cloud SQL PostgreSQL           |
| Container registry   | Artifact Registry              |
| AI                   | Vertex AI                      |
| Secrets              | Secret Manager                 |
| Monitoring           | Cloud Monitoring + Cloud Trace |

**Key point for management:** InfraAI is not locked to any cloud. The customer deploys on whichever cloud they already use. The application code does not change. Only the infrastructure provisioning is different.

***

### The Complete CI/CD Pipeline — End to End

    DEVELOPER                GITHUB                 REGISTRY            KUBERNETES
    ---------                ------                 --------            ----------

    Writes code
         |
         v
    Pushes to branch
         |         -------->  PR created
         |                    Actions trigger
         |                         |
         |                    Lint + Test
         |                         |
         |                    Pass? ---------> No --> PR blocked
         |                         |
         |                        Yes
         |                         |
         |                    Docker build
         |                    (backend + frontend)
         |                         |
         |                    Push images ----------> Images stored
         |                                            (ACR / ECR)
         |                                                  |
         |                    Apply Kustomize  <------------+
         |                    (kubectl apply -k)
         |                         |
         |                         +--------------------> Rolling update
         |                                                 Pods restart
         |                                                 Alembic migrates
         |                                                 App serves traffic
         |                                                      |
         |                                                 Health check passes
         |                                                      |
         v                                                 Deployment complete
    Done

***

### Rollback Strategy

If a deployment introduces a bug:

    OPTION 1: Kubernetes rollback
      kubectl rollout undo deployment/infraai-backend
      Reverts to the previous container image instantly.
      Database migrations are forward-only (Alembic does not auto-rollback).

    OPTION 2: Redeploy previous version
      kubectl apply -k k8s/overlays/production/
      with the previous image tag in the Kustomize overlay.

    OPTION 3: Alembic downgrade (if migration caused the issue)
      alembic downgrade -1
      Reverts the last database migration.
      Use with caution — may cause data loss if the migration added data.

Standard practice: test migrations thoroughly in staging before production. Forward-only migrations are safer than frequent rollbacks.

***

### Management Q\&A

**Q: How long does a deployment take?**
From code merge to running in production: typically 10-15 minutes. Build and test: 3-5 minutes. Docker build: 2-3 minutes. Push to registry: 1-2 minutes. Kubernetes rolling update: 2-3 minutes. Zero downtime during the update.

**Q: Can we deploy without Kubernetes?**
Yes. Azure App Service and AWS ECS Fargate run containers without Kubernetes. Simpler to manage but less flexible for scaling. For smaller deployments or single-customer instances, App Service is sufficient.

**Q: How do we handle database changes?**
Alembic migrations run automatically when the container starts. New tables and columns are added without manual intervention. Existing data is preserved. No DBA needed for routine deployments.

**Q: What if we need to deploy to a customer's cloud?**
The same Docker images work on any cloud. We change only the Kustomize overlay to point to the customer's database, AI service, and registry. The application code is identical across all deployments.

**Q: How do we manage secrets across environments?**
Secrets (API keys, database passwords, encryption keys) are stored in the cloud provider's secret management service (Key Vault, Secrets Manager). They are injected into containers as environment variables at runtime. Secrets are never in the code repository, never in Docker images, never in Kubernetes manifests.

**Q: What about monitoring the deployed application?**
Cloud-native monitoring: Application Insights (Azure), CloudWatch (AWS), Cloud Monitoring (GCP). Application-level health check endpoint (/health) for Kubernetes liveness and readiness probes. Structured logging from the FastAPI backend for troubleshooting.

***

### Summary — C5 in 30 Seconds

    1. CI/CD: GitHub Actions pipeline (test, build, push, deploy)
    2. Packaging: Docker images for backend (Python) and frontend (React + nginx)
    3. Registry: Private container registry (ACR / ECR / Artifact Registry)
    4. Deployment: Kubernetes with Kustomize overlays per environment
    5. Environments: Local (docker-compose), Dev, Staging, Production
    6. Database: Alembic auto-migration on container startup, zero manual SQL
    7. Multi-cloud: Same images deploy on Azure (AKS), AWS (EKS), GCP (GKE), OCI (OKE)
    8. No cloud lock-in: Application code identical, only infrastructure layer changes
    9. Rollback: Kubernetes rollout undo or redeploy previous image tag
    10. Secrets: Cloud-native secret management, never in code or images


