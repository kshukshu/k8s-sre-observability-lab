# Setup Guide

> **Goal:** spin up the full demo stack on a laptop in ≈ 15 minutes using VS Code’s integrated terminal.
>
> Tested on **macOS 14 / Windows 11 / Ubuntu 22.04**, Kind v0.23, Helm v3.15, kubectl v1.33.

---

## 0 · Clone & open

```bash
git clone https://github.com/kshukshu/Mini-Cloud-Native-Platform.git
cd Mini-Cloud-Native-Platform
code .   # optional – launches VS Code
```

---

## 1 · CLI prerequisites

| Tool        | macOS / Linux          | Windows (PowerShell)                        |
| ----------- | ---------------------- | ------------------------------------------- |
| **Kind**    | `brew install kind`    | `winget install -e --id Kubernetes.kind`    |
| **kubectl** | `brew install kubectl` | `winget install -e --id Kubernetes.kubectl` |
| **Helm**    | `brew install helm`    | `winget install -e --id Helm.Helm`          |

> **Check versions**
>
> ```bash
> kind version && kubectl version --client && helm version
> ```

---

## 2 · Create local cluster

```bash
kind create cluster --name demo  # ≈ 60 s
kubectl cluster-info --context kind-demo   # verify API server URL
```

---

## 3 · Add Helm repositories (idempotent)

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add datadog https://helm.datadoghq.com
helm repo add newrelic https://helm-charts.newrelic.com
helm repo update
```

---

## 4 · Provide SaaS credentials

```bash
# Bash / zsh
export DATADOG_API_KEY="<paste>"
export NEW_RELIC_LICENSE_KEY="<paste>"

# PowerShell
$Env:DATADOG_API_KEY="<paste>"
$Env:NEW_RELIC_LICENSE_KEY="<paste>"
```

* **Datadog** – 14‑day trial, no credit card. UI → *Integrations › APIs* → **+ New Key**.
* **New Relic** – perpetually free tier (100 GB/mo). UI → profile avatar → **API Keys** → *License* tab.

---

## 5 · Install core charts

### Bash / zsh

```bash
# Argo CD (GitOps controller)
helm install argocd argo/argo-cd -n argocd --create-namespace

# Prometheus + Grafana stack
helm install mon prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Datadog agent – Kind‑friendly (no system‑probe or trace‑agent)
helm install dd datadog/datadog \
  --set-string datadog.apiKey=$DATADOG_API_KEY \
  --set datadog.site="datadoghq.eu" \
  --set datadog.processAgent.enabled=false \
  --set datadog.apm.enabled=false \
  --set systemProbe.enabled=false \
  -n datadog --create-namespace

# New Relic agent
helm install nr newrelic/nri-bundle \
  --set global.licenseKey=$NEW_RELIC_LICENSE_KEY \
  --set global.cluster="kind-demo" \
  --set global.region="eu" \
  -n newrelic --create-namespace
```

### PowerShell continuation character is back‑tick (\`): see README Quick‑start for full example

---

## 6 · Register the GitOps Application

```bash
kubectl apply -f k8s/argocd/app.yaml    # one‑liner – Argo CD syncs the rest
```

> **Private repo?** create PAT secret first:
>
> ```bash
> kubectl -n argocd create secret generic github-creds \
>   --type=Opaque \
>   --from-literal=username=git \
>   --from-literal=password=<GITHUB_PAT> \
>   --from-literal=url=https://github.com/kshukshu/Mini-Cloud-Native-Platform.git
> ```

---

## 7 · Access dashboards

| Tool        | Command                                                      | URL                                                                                                                            |              |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ | ------------ |
| **Argo CD** | `kubectl port-forward svc/argocd-server -n argocd 8080:443`  | `https://localhost:8080` – \*admin / \$(kubectl -n argocd get secret argocd-initial-admin-secret -ojsonpath='{.data.password}' | base64 -d)\* |
| **Grafana** | `kubectl port-forward svc/mon-grafana -n monitoring 3000:80` | `http://localhost:3000` – \*admin / \$(kubectl -n monitoring get secret mon-grafana -ojsonpath='{.data.admin-password}'        | base64 -d)\* |

Datadog & New Relic links are public in the README.

---

## 8 · Inject / recover failure

```bash
kubectl scale deploy/nginx --replicas=0   # break it
sleep 120
kubectl scale deploy/nginx --replicas=1   # heal it
```

Observe SLO breach in Datadog public dashboard, Argo CD auto‑sync rollback, and metrics recovery.

---

## 9 · Teardown

```bash
kind delete cluster --name demo   # frees all local resources
```

---

### Troubleshooting

| Issue                                | Fix                                                                        |
| ------------------------------------ | -------------------------------------------------------------------------- |
| `no matches for kind "Application"`  | Wait for Argo CD Helm install to finish; CRDs aren’t ready yet.            |
| Datadog agent `1/2 CrashLoopBackOff` | Use the Kind‑friendly values provided (trace‑agent/system‑probe disabled). |
| Prometheus stack pods CrashLoop      | Allocate ≥ 4 GB RAM to Docker Desktop.                                     |
