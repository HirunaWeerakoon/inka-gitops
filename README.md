# 🔄 INKA GitOps — Continuous Delivery with Argo CD & Helm

[![Argo CD](https://img.shields.io/badge/Argo%20CD-v2.13-orange.svg)](https://argoproj.github.io/cd/)
[![Helm 3](https://img.shields.io/badge/Helm-3.x-0F1689.svg)](https://helm.sh/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-k3d%20%7C%20k3s-326CE5.svg)](https://k3d.io/)
[![Sealed Secrets](https://img.shields.io/badge/Bitnami-SealedSecrets-green.svg)](https://github.com/bitnami-labs/sealed-secrets)

This repository serves as the **Single Source of Truth** for the entire **INKA Clothing Store** Kubernetes deployment. It implements declarative GitOps using **Argo CD**, **Helm**, and **Bitnami SealedSecrets**.

Whenever code is pushed to the [Application Repository](https://github.com/HirunaWeerakoon/inka_clothing_ecommerce_store), automated CI pipelines test, build, and update the image tags in this repository—triggering an instant, automated rollout in the Kubernetes cluster.

---

## 🏛️ GitOps Architecture & Directory Layout

This repository mirrors production enterprise GitOps structures using the **App-of-Apps** pattern:

```
inka-gitops/
├── argocd-apps/
│   └── microservices/
│       └── dev/
│           └── inka/
│               └── application.yaml       # Multi-source Argo CD Application manifest
└── charts/
    └── microservices/
        ├── inka/                          # Reusable Helm Chart templates
        │   ├── Chart.yaml                 # Chart metadata
        │   └── templates/
        │       ├── backend.yaml           # Spring Boot Service & Deployment
        │       ├── frontend.yaml          # React/Nginx Service & Deployment
        │       ├── mysql.yaml             # Fallback Local MySQL Deployment & PVC
        │       └── ingress.yaml           # Traefik Ingress routing rules
        └── dev/
            └── inka/                      # Environment-specific configuration
                ├── values.yaml            # Environment overrides (auto-patched by CI)
                └── resources/
                    └── sealed-secrets.yaml # Encrypted production credentials (kubeseal)
```

---

## 🔄 How the GitOps Pipeline Works

```
[ Developer ]
      │
      ▼ (git push)
[ App Repo: inka_clothing_ecommerce_store ]
      │
      ▼ (GitHub Actions CI: Tests & Multi-Stage Build)
[ Container Registries: Docker Hub & AWS ECR ]
      │
      ▼ (CI patches image tag via yq & commits to main)
[ GitOps Repo: inka-gitops / values.yaml ]
      │
      ▼ (Continuous 3-minute poll or webhook)
[ Argo CD Controller (k3d cluster) ]
      │
      ▼ (Detects drift & reconciles desired state)
[ Kubernetes Cluster: Ingress -> Frontend Pod / Backend Pod -> AWS RDS MySQL ]
```

---

## 🔐 Zero-Trust Secret Management (Bitnami SealedSecrets)

Never commit plain text passwords or API keys to Git! We use **Bitnami SealedSecrets** for asymmetric public-key encryption:

1. **The Public Key** is stored in this repository to encrypt secrets on developer laptops.
2. **The Private Key** is strictly held by the SealedSecrets controller inside the Kubernetes cluster (`kube-system`).

### How to Seal a New Secret:
```powershell
$secretYaml = kubectl create secret generic inka-secrets -n inka `
  --from-literal=MYSQL_URL="jdbc:mysql://<RDS_HOST>:3306/inka_clothing_store?..." `
  --from-literal=MYSQL_USERNAME="admin" `
  --from-literal=MYSQL_PASSWORD="<DB_PASSWORD>" `
  --from-literal=STRIPE_SECRET_KEY="<STRIPE_KEY>" `
  --dry-run=client -o yaml | kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system --format yaml

[System.IO.File]::WriteAllLines("charts/microservices/dev/inka/resources/sealed-secrets.yaml", $secretYaml, (New-Object System.Text.UTF8Encoding($false)))
```

Commit `sealed-secrets.yaml` to Git with 100% confidence. Only your cluster can decrypt it into a live Kubernetes Secret!

---

## 🚀 Bootstrapping the Cluster from Scratch

To spin up the entire cluster and deploy the application from this GitOps repo:

### 1. Create the Local Multi-Node k3d Cluster
```powershell
k3d cluster create inka-cluster --servers 1 --agents 2 --port "8080:80@loadbalancer" --port "8443:443@loadbalancer"
```

### 2. Install Sealed Secrets Controller
```powershell
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.5/controller.yaml
```

### 3. Deploy Argo CD via Terraform
```powershell
cd ../inka_clothing_ecommerce_store/terraform
terraform apply -target "helm_release.argocd" -auto-approve
terraform apply -auto-approve
```

### 4. Access Argo CD Dashboard
```powershell
# Port forward Argo CD server
kubectl port-forward svc/argo-cd-argocd-server -n argocd 8081:80

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

Open **`http://localhost:8081`** in your browser, log in as `admin`, and watch your entire infrastructure synchronize automatically!

---

## 🔗 Related Repositories
- **Application Source Code**: [inka_clothing_ecommerce_store](https://github.com/HirunaWeerakoon/inka_clothing_ecommerce_store)
- **Original Homelab Reference**: Inspired by the GitOps workshop blueprint presented by **Sysco LABS**.
