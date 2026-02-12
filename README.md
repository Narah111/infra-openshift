# Infrastructure Repository - OpenShift Guestbook

> **GitOps Infrastructure-as-Code** for the Guestbook application (ocp-guestbook repository)

This repository contains all Kubernetes/OpenShift manifests for deploying the guestbook application. It follows GitOps principles where this repository serves as the **single source of truth** for the production infrastructure.

## Purpose

This is the **operations repository** in a two-repository GitOps architecture:

- **[ocp-guestbook](https://github.com/Narah111/ocp-guestbook)** (Application Repository) - Source code and CI/CD pipelines
- **[infra_openshift](https://github.com/Narah111/infra_openshift)** (This Repository) - Kubernetes manifests and infrastructure configuration

## How It Works

1. Developers push code to the [application repository](https://github.com/Narah111/ocp-guestbook)
2. GitHub Actions builds container images and pushes them to GHCR
3. **GitHub Actions automatically updates manifests in THIS repository** with new image tags
4. OpenShift/ArgoCD watches this repository and deploys changes automatically

This enables **zero-touch deployment** from code commit to production.

## Repository Structure

```
infra_openshift/
└── .gitignore/
└── Readme.md
└── k8s/
    ├── backend.yml              # Backend Deployment
    ├── backend-service.yml      # Backend Service (ClusterIP)
    ├── backend-config.yml       # Backend ConfigMap
    ├── backend-secret.yml       # Backend Secrets (DB, Redis passwords)
    ├── frontend.yml             # Frontend Deployment
    ├── frontend-service.yml     # Frontend Service (ClusterIP)
    ├── postgres.yml             # PostgreSQL Deployment
    ├── postgres-service.yml     # PostgreSQL Service
    ├── postgres-configmap.yml   # PostgreSQL configmap
    ├── postgres-secret.yml      # PostgreSQL secret
    ├── redis.yml                # Redis Deployment
    ├── redis-service.yml        # Redis Service
    ├── redis-secret.yml         # Redis secrets

```
**Note:** 
  - All secrets and configmaps is not pushed here in github due to securitybut they are available in localy in my PC.
  - I know i should even have .gitignore in out of github but I will fix it.          

## Architecture

```
┌─────────────────────────────────────────────────┐
│           OpenShift Cluster                     │
│                                                 │
│  ┌──────────┐    ┌──────────┐                 │
│  │ Frontend │◄───│  Route   │◄─── Internet    │
│  │  (Nginx) │    │ (Ingress)│                 │
│  └────┬─────┘    └──────────┘                 │
│       │                                         │
│       │ /api/*                                  │
│       ▼                                         │
│  ┌──────────┐                                  │
│  │ Backend  │                                  │
│  │  (Go API)│                                  │
│  └─┬─────┬──┘                                  │
│    │     │                                      │
│    │     └────────┐                            │
│    ▼              ▼                            │
│  ┌────────┐   ┌───────┐                       │
│  │Postgres│   │ Redis │                       │
│  │  (DB)  │   │(Cache)│                       │
│  └────────┘   └───────┘                       │
└─────────────────────────────────────────────────┘
```

## Deployment

### Prerequisites

- OpenShift cluster access
- `oc` CLI tool installed and configured

### Deploy All Resources

```bash
# Clone this repository
git clone https://github.com/Narah111/infra_openshift.git
cd infra_openshift

# Create a new project (namespace)
oc new-project guestbook

# Deploy in order
oc apply -f k8s/backend-config.yml
oc apply -f k8s/backend-secret.yml
oc apply -f k8s/postgres-pvc.yml
oc apply -f k8s/postgres.yml
oc apply -f k8s/postgres-service.yml
oc apply -f k8s/redis-pvc.yml
oc apply -f k8s/redis.yml
oc apply -f k8s/redis-service.yml
oc apply -f k8s/backend.yml
oc apply -f k8s/backend-service.yml
oc apply -f k8s/frontend.yml
oc apply -f k8s/frontend-service.yml
oc apply -f k8s/frontend-route.yml

# Or deploy everything at once
oc apply -f k8s/
```

### Verify Deployment

```bash
# Check all pods are running
oc get pods

# Check services
oc get svc

# Get the external URL
oc get route frontend
```

Expected output:
```
NAME       HOST/PORT                                    PATH   SERVICES   PORT
frontend   frontend-guestbook.apps.cluster.example.com        frontend   8080
```

Visit the URL to access the application.

## Configuration

### Secrets Management

** Important:** The `backend-secret.yml` file contains base64-encoded secrets. In production:

1. **Never commit actual secrets to Git**
2. Use sealed secrets or external secret management (e.g., Vault, AWS Secrets Manager)
3. The current secrets are for demonstration purposes only

To create new secrets:
```bash
# Create secret from literals
oc create secret generic backend-secret \
  --from-literal=DB_PASSWORD=your_password \
  --from-literal=REDIS_PASSWORD=your_redis_password \
  --dry-run=client -o yaml > k8s/backend-secret.yml
```

### ConfigMaps

Application configuration is managed via ConfigMaps:
- `backend-config.yml` - Database connection settings, Redis configuration

To update configuration:
```bash
oc edit configmap backend-config
```

## GitOps Workflow

### Automated Updates

This repository is automatically updated by GitHub Actions from the [application repository](https://github.com/Narah111/ocp-guestbook):

1. When code is pushed to `main` branch in app repo
2. GitHub Actions builds new container image
3. Image is tagged with semantic version: `1.0.BUILD-SHA`
4. **This repository is automatically updated** with the new image tag
5. OpenShift detects the change and deploys the new version

Example commit from automation:
```
Updated backend image to 1.0.41-13275af
```

### Manual Updates

To manually update image versions:

```bash
# Edit the deployment file
vim k8s/backend.yml

# Change the image tag
# image: ghcr.io/narah111/ocp-guestbook/backend:1.0.41-13275af

# Commit and push
git add k8s/backend.yml
git commit -m "Updated backend to version 1.0.42"
git push
```

## Monitoring

### Check Application Health

```bash
# Backend health endpoint
oc port-forward svc/backend 8080:8080
curl http://localhost:8080/health

# Check pod logs
oc logs -l app=backend -f
oc logs -l app=frontend -f

# Check database connection
oc exec -it deploy/postgres -- psql -U guestbook -d guestbook -c "\dt"
```

### Scaling

```bash
# Scale backend
oc scale deployment backend --replicas=3

# Scale frontend
oc scale deployment frontend --replicas=2

# Verify
oc get pods -l app=backend
```

## Security Considerations

- Secrets are base64 encoded (not encrypted) - use SealedSecrets or external secret management in production
- Network policies should be added to restrict pod-to-pod communication
- Consider implementing resource limits and quotas
- Regular security scanning of container images

## Troubleshooting

### Common Issues

**Pods not starting:**
```bash
oc describe pod <pod-name>
oc logs <pod-name>
```

**Database connection errors:**
```bash
# Check PostgreSQL is ready
oc get pods -l app=postgres
oc logs deploy/postgres

# Verify service
oc get svc postgres
```

**Frontend can't reach backend:**
```bash
# Check service discovery
oc exec -it deploy/frontend -- curl http://backend:8080/health
```

### View All Resources

```bash
# Get everything in the namespace
oc get all

# Get persistent volumes
oc get pvc

# Get configmaps and secrets
oc get configmap,secret
```

## Related Resources

- [Application Repository](https://github.com/Narah111/ocp-guestbook) - Source code and CI/CD
- [OpenShift Documentation](https://docs.openshift.com)
- [Kubernetes Documentation](https://kubernetes.io/docs)
- [GitOps Principles](https://www.gitops.tech)

## Contributing

This repository is automatically updated by CI/CD pipelines. Manual changes should be:

1. Tested in a development environment first
2. Reviewed for security implications
3. Documented in commit messages

## License

- This project is part of a DevOps course demonstration.
- You need to have an Openshift account and make sure that you can creat project
  because in some case it's not free. 

---

**Note:** This is the infrastructure repository. Application code lives in [ocp-guestbook](https://github.com/Narah111/ocp-guestbook).
