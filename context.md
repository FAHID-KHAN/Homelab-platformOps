# Homelab GitOps / Platform Engineering — Conversation Context

**Context captured:** 2026-10-25  
**Status:** Guides for milestones 1–12 provided. **Not executed or validated on the user's infrastructure.**  
**Working agreement:** User executes commands and writes/configures files; assistant explains each step, reviews outputs and helps debug. Never assume success without evidence.

## 1. Objective

Build a portfolio-grade, production-style, self-managed Kubernetes delivery platform, progressing from Kustomize/Helm/Argo CD to Jenkins CI, GitOps promotion, monitoring, controlled releases, security and disaster recovery. Learn how each component works internally, rather than blindly copying manifests. The lab is not genuinely highly available while it relies on one K3s server.

## 2. Existing / planned environment

- Existing homelab as previously described: ThinkPad running Ubuntu and **K3s**, **Flannel**, **Argo CD**, **Jenkins**, **Prometheus**, **Grafana**, **InfluxDB**. Actual current health, namespaces, tool versions, and installed components **not validated in this chat**.
- Development machine: MacBook, using terminal, `kubectl`, `git`, `helm`, `gh`, and optionally Docker.
- Repositories proposed: `homelab-gitops` (Kubernetes/GitOps config) and `homelab-api` (Python FastAPI code, tests, Dockerfile, Jenkinsfile).
- Image registry: GHCR, owner and repository URLs are placeholders to replace with real values.
- Future resources: second Ubuntu worker or VM, dedicated Jenkins build agent, optional Argo Rollouts controller.
- All potentially sensitive credentials (K3s join token, GHCR token, SSH credentials, kubeconfig) must remain outside Git and chat transcripts.

## 3. Architecture

```text
MacBook / developer
      |
      v
GitHub homelab-api --> Jenkins CI --> test --> build Docker image --> GHCR
                           |
                           v
                     GitOps change / PR
                           |
                           v
GitHub homelab-gitops --> Argo CD --> Kustomize or Helm templates
                                         |
                                         v
                                   K3s / Flannel
                                         |
                      Dev / Staging / Prod workloads
                                         |
                                   Prometheus / Grafana
```

- **Helm:** renders reusable chart templates and values into Kubernetes manifests. Argo CD Helm integration generally runs Helm as a template renderer, not a managed `helm install` release.
- **Kustomize:** transforms shared plain-YAML base into environment-specific manifests via overlays.
- **Argo CD:** tracks desired Git state against live cluster, syncs and reconciles; `Synced` can coexist with `Degraded`.
- **Jenkins:** builds/tests/packages/publishes, proposes GitOps image change, not direct cluster deployment.
- **Kubernetes:** schedules Pods, rolls Deployments, routes Services. Argo CD does not individually create pods.

## 4. Milestone roadmap and specific material delivered

### M1 — Inspect actual homelab

Read-only checks:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A
kubectl get svc -A
kubectl get ingress -A
kubectl get applications.argoproj.io -A
kubectl get applicationsets.argoproj.io -A
kubectl get daemonsets -n kube-system
kubectl get storageclass
kubectl top nodes
```

Determine Argo CD health, ingress, Prometheus, storage, capacity, Jenkins capabilities and existing applications before changes.

### M2 — GitOps repository with Podinfo/Kustomize

Proposed:

```text
homelab-gitops/
  applications/podinfo/
    base/{deployment.yaml,service.yaml,kustomization.yaml}
    overlays/dev/kustomization.yaml
    overlays/prod/kustomization.yaml
  argocd/
  helm/values/
  README.md
```

Podinfo image example `ghcr.io/stefanprodan/podinfo:6.15.0`, HTTP 9898, metrics 9797, liveness `/healthz`, readiness `/readyz`. Dev namespace `podinfo-dev` 1 replica, prod `podinfo-prod` 2 replicas, customized by Kustomize overlays. Verify by `kubectl kustomize applications/podinfo/overlays/{dev,prod}` and save renders to `/tmp`. Commit and push `homelab-gitops` with `gh repo create` if wanted. Exact upstream image/tag availability still needs to be verified when executing.

### M3 — First Argo CD Application

`argocd/podinfo-dev.yaml`: Argo CD `Application` in `argocd`, Git `main`, `path: applications/podinfo/overlays/dev`, target `podinfo-dev`, `CreateNamespace=true`. Start with manual sync. Bootstrap once via `kubectl apply -f argocd/podinfo-dev.yaml`, then sync in UI. Check `kubectl get all -n podinfo-dev`, rollout status, port-forward `svc/podinfo 9898:9898`. Change replicas in Git 1→2 and manually sync; then enable `automated.enabled: true`, `selfHeal: true`, `prune: false` on the Argo CD Application. Because Application itself isn't yet Git-managed, apply changed Application manifest to cluster once. Demonstrate drift with `kubectl scale deployment/podinfo -n podinfo-dev --replicas=3`; Argo CD should reconcile back to Git's 2. Wait for verified outputs before claiming complete.

### M4 — Helm with Argo CD, second Podinfo instance

Podinfo upstream Helm repo example `https://stefanprodan.github.io/podinfo`, chart `podinfo`, pinned chart version example `6.15.0` (VERIFY actual chart/version and `helm show values` before use). `helm/values/podinfo-helm.yaml` with `replicaCount`, image, service, UI and resource settings. Render locally using `helm template podinfo-helm stefanprodan/podinfo --version ... -f ...`. Argo CD Application `podinfo-helm` targets namespace `podinfo-helm`, `sources` refers to upstream Helm chart and Git repo with `ref: values`, Helm `valueFiles: [$values/helm/values/podinfo-helm.yaml]`. Requires Argo CD multi-source support. Bootstrap via `kubectl apply` Application, then manual sync. Compare two instances with port forwards 9898 and 9899. Upgrade values in Git and sync. Ensure actual chart resource names and value keys match installed chart.

### M5 — End-to-end Jenkins CI / GHCR / Argo CD

`homelab-api` proposed files:

```text
app.py
requirements.txt
requirements-dev.txt
tests/test_app.py
Dockerfile
.dockerignore
.gitignore
Jenkinsfile
```

FastAPI endpoints `/`, `/healthz`, `/readyz`, tests using pytest/TestClient. Dockerfile uses Python 3.12 slim, non-root user, uvicorn port 8000. Jenkins Multibranch Pipeline on trusted `docker-builder` agent: checkout, Python venv/test, docker build tag from short Git SHA, GHCR push from `ghcr-credentials` only for trusted `main`. Images under `ghcr.io/USERNAME/homelab-api:TAG`. GHCR read permissions/package visibility checked separately. GitOps `applications/homelab-api/base/{deployment,service,kustomization}.yaml` and `overlays/dev/kustomization.yaml`, namespace `homelab-api-dev`, Argo CD Application `argocd/homelab-api-dev.yaml` auto sync (prune false, selfHeal true). Deploy by updating image in Git, not with Jenkins kubectl. Later optional Jenkins-created image-update PR to GitOps repo with narrowly scoped credentials (`gitops-gh-token`). **Security caveat:** Earlier example PR automation passed token-derived auth header through a process command and should **NOT be used verbatim**; use GitHub App/secure Git credential handling and review agent trust/isolation. Need test actual Jenkins build agent and permissions before implementing.

### M6 — Prometheus / Grafana + recovery

Extend API with `prometheus-fastapi-instrumentator` and `/metrics`. Exclude probe endpoints from request stats. Add metrics test; Jenkins builds/deploys new image via GitOps. Inspect whether existing Prometheus is Operator-managed (`kubectl get crd | grep monitoring.coreos.com`, `kubectl get prometheus -A`). If Operator: `ServiceMonitor` with correct selectors and namespace selectors; if standalone: add scrape target to existing Prometheus configuration via its actual management method, e.g. `homelab-api.homelab-api-dev.svc.cluster.local:8000/metrics`. Do not blindly install second monitoring stack. Confirm target UP, Grafana panels for `http_requests_total`, error rate and `http_request_duration_seconds_bucket`, adapt label names to actual metrics. Simulate bad dev image tag, observe `ImagePullBackOff`, degraded rollout vs Argo CD Synced; revert offending Git commit and verify healthy rollout. Auto-sync and historical Argo CD UI rollback are not a substitute for Git revert. Do not do failure exercises against important production workloads.

### M7 — App of Apps, production, least privilege, DR

- Root Argo CD Application `bootstrap/root-app.yaml` targets `argocd/apps`, initially manual sync and no prune. Child resources are `Application` YAML within `argocd/apps/`; correction was given: avoid `../` references that violate Kustomize load restrictions; move child manifest into same directory and update `argocd/apps/kustomization.yaml`.
- Root first manages existing `homelab-api-dev` Application. Avoid overlapping parents and destructive finalizers.
- Prod overlay `applications/homelab-api/overlays/prod/kustomization.yaml`, namespace `homelab-api-prod`, 2 replicas and tested image; AppProject `homelab-workloads` restricts source repo/destinations, with Namespace permission for CreateNamespace. Production Application manual sync. Git PR to promote same tested image instead of rebuilding; ultimately prefer immutable digest.
- Security: non-root app, seccomp RuntimeDefault, drop capabilities, no privilege escalation, read-only root filesystem only after tests. K3s/Flannel networking & NetworkPolicy enforcement need actual inspection; example ingress allowed monitoring namespace, but needs consideration of other traffic. No plaintext secrets in Git. Manage Jenkins/GHCR and runtime credentials separately.
- Backups: distinguish default K3s SQLite vs embedded etcd, backup datastore plus K3s server token safely to **external** encrypted storage, protect config and PVCs/application-specific databases. For SQLite, planned maintenance stop/copy/restart; etcd uses snapshots. GitOps alone cannot restore persistent data. Write `docs/runbooks/disaster-recovery.md`. Never trial destructive restore on the only working cluster.

### M8 — Multi-node K3s

Inspect current server and network config, prepare separate Ubuntu worker/VM, restrict API TCP 6443 and Flannel VXLAN UDP 8472 where relevant, securely retrieve agent token. Join with `K3S_URL`/`K3S_TOKEN` via K3s install, verify new worker Ready, label `workload-tier=applications`, selectively node-select a development app. Multi-node with one server **does not mean HA control plane**. Verify existing Flannel backend, routes/firewalls and version compatibility first.

### M9 — Add staging

Add `applications/homelab-api/overlays/staging/kustomization.yaml`, namespace `homelab-api-staging`; create `argocd/apps/homelab-api-staging.yaml`, ensure AppProject destinations include staging and root Kustomize references the Application. Distinct sync policy: dev automatic, staging manually reviewed, prod manual. Promote same tested immutable artifact dev → staging → prod by PR.

### M10 — Dedicated Jenkins agent

Separate controller from build workloads; dedicated trusted Linux machine/VM with Java, Git, Python, Docker or more isolated builder, Jenkins node label `docker-builder`. Simple Jenkins pipeline verifies actual agent hostname/tool versions. CI host need not join K3s. Avoid mounting container runtime sockets into controller or untrusted workloads. Prefer restricted credentials and eventually BuildKit isolation.

### M11 — Argo Rollouts canary

Install version-pinned Argo Rollouts controller/CRDs (review release/compatibility and manifest before cluster-wide install), optional kubectl plugin. Create separate `applications/rollout-demo/` with Rollout + Service and Argo CD Application in `rollout-lab`, initially manual sync; example `ghcr.io/stefanprodan/podinfo:6.15.0` and 5 replicas, canary steps ~20%, pause, ~50%, pause, 100%. Without ingress/service mesh traffic-routing integration, weights approximate via replica counts; no guaranteed HTTP traffic percentages. Verify capacity; watch `kubectl argo rollouts get rollout`, use promote/abort. Revert Git image on failed rollout for durable GitOps state. Future enhancement: Prometheus analysis and traffic routing.

### M12 — Reliability experiments

On noncritical test workloads: stop/restart *additional worker* k3s-agent and observe scheduling, delete dev Pod to watch ReplicaSet controller, inspect scheduling capacity/Pending Pods, record incident evidence and RTO/RPO. No destructive control-plane failure on only ThinkPad. Recovery testing should use isolated environment/backups.

## 5. Key repository layout by the end

```text
homelab-gitops/
  README.md
  bootstrap/root-app.yaml
  argocd/
    homelab-project.yaml
    apps/
      kustomization.yaml
      homelab-api-dev.yaml
      homelab-api-staging.yaml
      homelab-api-prod.yaml
      rollout-demo.yaml
    podinfo-dev.yaml
    podinfo-helm.yaml
  applications/
    podinfo/
      base/{deployment.yaml,service.yaml,kustomization.yaml}
      overlays/{dev,prod}/kustomization.yaml
    homelab-api/
      base/{deployment.yaml,service.yaml,kustomization.yaml}
      overlays/{dev,staging,prod}/kustomization.yaml
    rollout-demo/{rollout.yaml,service.yaml,kustomization.yaml}
  helm/values/podinfo-helm.yaml
  docs/runbooks/disaster-recovery.md

homelab-api/
  app.py
  requirements.txt
  requirements-dev.txt
  tests/test_app.py
  Dockerfile
  .dockerignore
  .gitignore
  Jenkinsfile
```

## 6. Status / critical honesty

- **Everything described above was guidance, not execution.** No `kubectl` output, Jenkins build result, or GitHub repository state was validated in this conversation.
- The user asked whether they can code independently; response: yes, this is a substantial roadmap, but these are educational examples requiring adjustments, security review and real integration testing.
- Some sample names, manifests, tool versions, image tags, chart values, Jenkins PR automation, NetworkPolicies, backup commands and Helm/Argo Rollouts installation sequences are **illustrative and should be revalidated against current official documentation and the actual homelab**. Never copy unreviewed credentials or give a guarantee all commands work unmodified.
- When returning to the project, request/inspect actual command outputs for the **current** milestone; prefer small executable and verifiable steps, diagnose errors rather than assume earlier milestones succeeded.
- The user explicitly wants explanation and commands while implementing themselves, not the assistant remotely editing their machines.

## 7. Quick verification commands for resuming

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A
kubectl get applications -n argocd
kubectl get appprojects -n argocd
kubectl get deployments -A
kubectl get svc -A
kubectl get pvc -A
kubectl get crd | grep -E 'monitoring.coreos.com|rollouts.argoproj.io'
```

Do not share live secrets in terminal output. Follow the actual state rather than assuming all 12 stages are finished.
