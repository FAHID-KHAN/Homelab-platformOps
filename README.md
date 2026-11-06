# Homelab GitOps Platform

GitOps learning project using:

- Kubernetes K3s
- Flannel CNI
- Argo CD
- Kustomize
- Helm
- Jenkins
- Prometheus and Grafana

## Application

Podinfo, a sample Kubernetes microservice.

## Environments

- Development: podinfo-dev
- Production: podinfo-prod

Both environments share a common Kubernetes base.

## Render

Development:

kubectl kustomize applications/podinfo/overlays/dev

Production:

kubectl kustomize applications/podinfo/overlays/prod

## Deployment

Argo CD will manage deployments in Milestone 3.
