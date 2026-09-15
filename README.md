# docmanager-k8s

Kubernetes manifests to deploy [docmanager-api](https://github.com/hunairali/docmanager-api) (and its PostgreSQL database) onto a real cluster, backing up the "Docker, Kubernetes" line on my CV with actual, validated YAML rather than a bullet point.

## Why this exists

`docmanager-api` ships with a Dockerfile and a `docker-compose.yml` for local development. This repo is the next step: the manifests you'd actually hand to a cluster - a namespace, config and secrets kept separate from each other, a Postgres `Deployment` backed by a `PersistentVolumeClaim`, a `docmanager-api` `Deployment`/`Service` with resource requests/limits and health probes, an `Ingress`, and an `HorizontalPodAutoscaler`. It's deliberately plain `kubectl`/`kustomize`-compatible YAML rather than a Helm chart, so every resource is easy to read top to bottom.

## What's here

```
manifests/
├── namespace.yaml              cts-platform namespace
├── configmap.yaml               Non-secret app config (DB host/port/name)
├── secret.example.yaml          Template for DB credentials - copy to secret.yaml, never commit the real one
├── postgres-pvc.yaml             1Gi persistent volume claim for Postgres data
├── postgres-deployment.yaml   Postgres 16, wired to the ConfigMap/Secret, with a pg_isready readiness probe
├── postgres-service.yaml        ClusterIP service for Postgres
├── docmanager-deployment.yaml  docmanager-api, 2 replicas, resource requests/limits, liveness+readiness probes on /actuator/health
├── docmanager-service.yaml      ClusterIP service for docmanager-api
├── ingress.yaml                       nginx Ingress routing docmanager.local -> docmanager-api
├── hpa.yaml                          Horizontal Pod Autoscaler, 2-6 replicas at 70% CPU
└── kustomization.yaml           Ties the above together for `kubectl apply -k`
```

## Deploying it locally (kind)

This is written against [kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker) so it can be tried on a laptop with no cloud account.

```bash
# 1. Create a local cluster
kind create cluster --name cts-platform

# 2. Create your real secret from the template (never commit the result)
cp manifests/secret.example.yaml manifests/secret.yaml
# edit manifests/secret.yaml with a real DB_PASSWORD

# 3. Apply the secret, then everything else via kustomize
kubectl apply -f manifests/secret.yaml
kubectl apply -k manifests/

# 4. Watch it come up
kubectl -n cts-platform get pods -w
```

`docmanager-deployment.yaml` references `ghcr.io/hunairali/docmanager-api:latest`. Until that image is published to a registry, point it at a locally built image instead:

```bash
docker build -t docmanager-api:local ../docmanager-api
kind load docker-image docmanager-api:local --name cts-platform
kubectl -n cts-platform set image deployment/docmanager-api docmanager-api=docmanager-api:local
```

To reach the API through the Ingress, install an ingress controller in the kind cluster (e.g. the [kind-specific nginx ingress setup](https://kind.sigs.k8s.io/docs/user/ingress/)) and add `docmanager.local` to `/etc/hosts` pointing at `127.0.0.1`. Without an ingress controller, `kubectl -n cts-platform port-forward svc/docmanager-api 8080:80` is the quicker path to the same result.

## How correctness is checked

There's no live cluster in CI, so `.github/workflows/validate.yml` validates every manifest client-side on each push: [`kubeconform`](https://github.com/yannh/kubeconform) checks each file against the real Kubernetes 1.30 OpenAPI schemas in `-strict` mode (catching typos and invalid fields), and `kubectl kustomize manifests/` confirms the whole `kustomization.yaml` actually resolves. Check the **Actions** tab for the current status.

## Author

**Umair Ali Shah** - Senior Java Backend Engineer, 17+ years in enterprise/government systems.
LinkedIn: https://www.linkedin.com/in/umairalishah · GitHub: https://github.com/hunairali
