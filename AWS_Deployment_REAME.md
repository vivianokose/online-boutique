# Online Boutique — Full DevOps Deployment on AWS EKS

A production-grade DevOps project built on Google's [microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) — an 11-service e-commerce application deployed to AWS EKS using a complete GitOps pipeline.

Built by [Vivian Chiamaka Okose](https://github.com/vivianokose) as part of a real-world DevOps engineering simulation at [Pawsible Cloud](https://github.com/pawsible-cloud).

![Online Boutique live on AWS](.assets/screenshots/site_deployed_online_via_alb.png)

---

## What This Project Does

Takes a multi-language microservices application (Go, Python, Java, C#, Node.js) and deploys it to a managed Kubernetes cluster on AWS — with full CI/CD automation, GitOps-based delivery, infrastructure as code, and live monitoring.

Every push to the main branch triggers a pipeline that builds new Docker images, pushes them to ECR, and deploys to the cluster without manual intervention.

---

## Architecture

```
Browser
  │
  ▼
frontend (Go, HTTP :80)
  ├── productcatalogservice  (Go, gRPC :3550)
  ├── currencyservice        (Node.js, gRPC :7000)
  ├── cartservice            (C#, gRPC :7070)  ──► Redis
  ├── recommendationservice  (Python, gRPC :8080)
  ├── adservice              (Java, gRPC :9555)
  └── checkoutservice        (Go, gRPC :5050)
         ├── cartservice
         ├── productcatalogservice
         ├── currencyservice
         ├── shippingservice   (Go, gRPC :50051)
         ├── emailservice      (Python, gRPC :8080)
         └── paymentservice    (Node.js, gRPC :50051)

loadgenerator (Python/Locust) — simulates real user traffic
```

### DevOps Flow

```
Developer pushes code
       │
       ▼
GitHub Actions (CI)
  - docker build per service (11 parallel jobs)
  - docker push to Amazon ECR
  - Single downstream job updates Helm values with new image tags
       │
       ▼
ArgoCD detects Git change (CD)
  - Syncs Helm chart to EKS cluster automatically
       │
       ▼
EKS Cluster (Kubernetes)
  - 11 services running as pods
  - Prometheus scrapes cluster metrics
  - Grafana displays dashboards
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Docker | Build and containerise each microservice |
| Kubernetes (EKS) | Container orchestration on AWS |
| Helm | Package and manage Kubernetes manifests |
| Terraform | Provision VPC, EKS cluster, node groups, IAM |
| GitHub Actions | CI pipeline — build, push, update image tags |
| ArgoCD | GitOps CD — syncs Git state to the cluster |
| Amazon ECR | Private container registry |
| Prometheus | Metrics collection |
| Grafana | Metrics visualisation and dashboards |

---

## Repository Structure

```
online-boutique-platform/
├── src/                        # Source code for all 11 microservices
├── helm-chart/                 # Helm chart for Kubernetes deployment
│   ├── Chart.yaml
│   ├── values.yaml             # Image tags updated automatically by CI
│   └── templates/              # Kubernetes resource templates
├── terraform/                  # AWS infrastructure as code
│   ├── main.tf                 # Provider config and S3 backend
│   ├── vpc.tf                  # VPC, subnets, NAT gateway
│   ├── eks.tf                  # EKS cluster and managed node groups
│   ├── variables.tf
│   └── outputs.tf
├── .github/
│   └── workflows/
│       └── ci.yaml             # GitHub Actions CI pipeline
├── argocd/
│   └── application.yaml        # ArgoCD Application definition
└── monitoring/
    ├── servicemonitor.yaml     # Prometheus ServiceMonitor
    └── alertrules.yaml         # PrometheusRule alert definitions
```

---

## Infrastructure

Provisioned with Terraform using the official AWS EKS and VPC modules.

- VPC with public and private subnets across 2 availability zones
- EKS cluster (Kubernetes 1.30) with managed node groups
- EC2 worker nodes (t3.medium) with auto-scaling configured
- NAT gateway for private subnet egress
- IAM roles and security groups for node-to-node and node-to-ECR access

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

![Terraform init](assets/screenshots/terraform_init_II.png)

![Terraform apply complete](assets/screenshots/terraform_apply.png)

After apply, connect kubectl:

```bash
aws eks update-kubeconfig \
  --name online-boutique-cluster \
  --region us-east-1
```

---

## CI Pipeline

GitHub Actions runs on every push to `main` or `vivian` branches.

Eleven build jobs run in parallel — one per service. Each job builds the Docker image and pushes it to ECR. A single downstream job runs after all builds complete and updates `helm-chart/values.yaml` with all new image tags in one commit, avoiding race conditions.

![CI pipeline — all 11 jobs green](assets/screenshots/CI_ran_successfully.png)

![Full workflow history showing debugging journey](assets/screenshots/succcessful_workflows_and_failed_workflows.png)

![ECR repositories — all 11 services](assets/screenshots/ecr_repositories.png)

Required GitHub secrets:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `AWS_REGION` | e.g. `us-east-1` |
| `AWS_ACCOUNT_ID` | 12-digit AWS account number |

---

## GitOps with ArgoCD

ArgoCD is installed on the cluster and watches the `main` branch. Any change merged to main — including the automated image tag updates from CI — triggers an automatic sync to the cluster.

![ArgoCD — all services synced and healthy](assets/screenshots/argocd_up_and_active.png)

![ArgoCD network view showing load balancer to pods](assets/screenshots/argocd_application_details.png)

Install ArgoCD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Apply the Application:

```bash
kubectl apply -f argocd/application.yaml
```

The application deploys to the `online-boutique` namespace with automated sync, self-healing, and pruning enabled.

---

## Monitoring

Prometheus and Grafana installed via the `kube-prometheus-stack` Helm chart.

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

![Grafana dashboard — node metrics](assets/screenshots/grafana_1.png)

![Prometheus targets scraping](assets/screenshots/prometheus_data_scrapping.png)

Access Grafana locally:

```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```

Open `http://localhost:3000` — username `admin`, password `admin123`.

Recommended dashboard imports: `315` (cluster), `6417` (pods), `1860` (nodes).

---

## Real Problems Solved

This project ran into real engineering problems. Here is what broke and how it was fixed.

**ImagePullBackOff on two services** — image references had no registry path or tag. Kubernetes didn't know where to pull from. Fixed by correcting the image field to the full ECR URI.

**CI push rejections from parallel jobs** — 11 jobs all writing to `values.yaml` at the same time caused merge conflicts and rejected pushes. Fixed by separating the Helm update into a single downstream job that runs after all builds complete.

**Pod scheduling failures** — `t3.medium` nodes on AWS have an 11-pod limit based on network interface capacity, not memory or CPU. With ArgoCD and Prometheus also running on the cluster, there weren't enough slots for all app pods.

Before fix — cart service unreachable:

![500 error before pod scheduling fix](assets/screenshots/alb_not_working.png)

After fix — all services healthy:

![Live site working on AWS ELB](assets/screenshots/site_deployed_online_via_alb.png)

**EKS authentication failure** — `kubectl` returned credential errors after cluster recreation. Fixed by running `aws eks update-kubeconfig` to refresh the local kubeconfig.

**Kubernetes version mismatch in Terraform** — Terraform tried to downgrade the cluster from 1.30 to 1.29 because the variable didn't match the live cluster version. Fixed by updating the variable to match reality.

**ArgoCD CLI segfault** — corrupted binary. Reinstalled using `--http1.1` flag to work around an HTTP/2 download error.

---

## Local Development

Run the full app locally with Docker Compose before touching Kubernetes:

```bash
docker compose up
```

Open `http://localhost:8080`.

Run on local Kubernetes with Minikube:

```bash
minikube start --cpus=4 --memory=8192 --driver=docker
kubectl apply -f ./release/kubernetes-manifests.yaml
minikube service frontend-external
```

---

## Cost Notes

EKS is not free tier. Running 24/7 costs roughly $140/month with t3.medium nodes. To minimise costs, destroy the cluster when not in use:

```bash
cd terraform
terraform destroy
```

The config stays in Git. Rebuild anytime with `terraform apply`.

---

## Cleanup

```bash
kubectl delete -f argocd/application.yaml
helm uninstall prometheus -n monitoring
cd terraform && terraform destroy
```

![Terraform destroy complete — 54 resources deleted](assets/screenshots/terraform_destroy_completed.png)

Delete ECR repositories manually if no longer needed.

---

## About

Built by **Vivian Chiamaka Okose** — DevOps Engineer.

Background in Biochemistry and Biotechnology. Pivoted into tech intentionally. Working across cloud-native infrastructure, container orchestration, and GitOps workflows.

- GitHub: [vivianokose](https://github.com/vivianokose)
- LinkedIn: [Vivian Chiamaka Okose](https://linkedin.com/in/vivian-chiamaka-okose)
- Blog: [vivianchiamaka.hashnode.dev](https://vivianchiamaka.hashnode.dev)
