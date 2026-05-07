# kubernetes-blue-green-deployment

Sample application for the CircleCI tutorial: **How to Set Up Blue-Green Deployments on Kubernetes with CircleCI**.

Read the full tutorial: https://circleci.com/blog/blue-green-deployment-kubernetes-tutorial/

## What's in this repo

```
app/          Node.js HTTP server (zero dependencies)
k8s/          Kubernetes Deployment and Service manifests (envsubst templates)
.circleci/    CircleCI pipeline config and rollback pipeline config
```

## Prerequisites

- A CircleCI account
- A Kubernetes cluster (the tutorial uses GKE, but any cluster works)
- `kubectl` configured to reach the cluster
- Docker installed locally
- A Docker Hub account

## CircleCI context variables

Create a context named `blue-green-tutorial` in your CircleCI organization and add these variables:

| Variable | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_PASSWORD` | Your Docker Hub password or access token |
| `KUBECONFIG_DATA` | Base64-encoded kubeconfig for your cluster |

To generate `KUBECONFIG_DATA` for GKE:

```bash
gcloud container clusters get-credentials <cluster-name> --zone <zone> --project <project-id>
cat ~/.kube/config | base64 | tr -d '[:space:]'
```

## Running the app locally

```bash
docker build -t blue-green-tutorial-app:local ./app
docker run -p 3000:3000 -e DEPLOY_SLOT=blue blue-green-tutorial-app:local
```

Verify it's running:

```bash
curl localhost:3000/api
# {"version":"1.0.0","slot":"blue","hostname":"...","timestamp":"..."}

curl localhost:3000/health
# {"status":"ok"}
```

## Kubernetes manifests

The project uses four manifests:

| File | Purpose |
|---|---|
| `k8s/deployment-blue.yml` | Blue environment Deployment (envsubst template) |
| `k8s/deployment-green.yml` | Green environment Deployment (envsubst template) |
| `k8s/service.yml` | Production Service (selector switches between blue and green) |
| `k8s/service-preview.yml` | Preview Service (always points to green, for pre-switch testing) |

Deployment manifests are envsubst templates. The pipeline substitutes these variables at deploy time:

- `IMAGE_TAG` — set to `$CIRCLE_SHA1` by the pipeline
- `DOCKERHUB_USERNAME` — from the CircleCI context
- `CIRCLE_PROJECT_ID` — injected automatically by CircleCI

## Initial setup

Before the first pipeline run, bootstrap both environments manually:

```bash
export IMAGE_TAG=initial
export DOCKERHUB_USERNAME=yourusername
export CIRCLE_PROJECT_ID=your-project-id
envsubst < k8s/deployment-blue.yml | kubectl apply -f -
envsubst < k8s/deployment-green.yml | kubectl apply -f -
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/service-preview.yml
```

## Blue-green deployment workflow

The CircleCI pipeline runs this workflow on each push to `main`:

1. **build-and-push** — Build and push Docker image tagged with commit SHA
2. **deploy-to-green** — Update the green Deployment with the new image
3. **validate-green** — Run smoke tests against the preview Service
4. **hold-for-approval** — Wait for manual approval in the CircleCI UI
5. **switch-traffic** — Patch the production Service selector to `version: green`

## Rollback pipeline

`.circleci/rollback.yml` defines the rollback pipeline. Configure it in the CircleCI Deploys dashboard under **Project Settings > Deploys > Rollback Pipeline**. Once set, rollbacks can be triggered from the CircleCI web UI — the pipeline patches the production Service selector back to `version: blue`, restoring the previous version instantly.
