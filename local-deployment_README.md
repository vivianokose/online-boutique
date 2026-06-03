# Online Boutique — Local Deployment

> Deployed locally using **kind** (Kubernetes IN Docker). No cloud account needed.
> Part of my DevOps engineering training — documenting everything publicly.

![All 11 pods Running](./images/LOCAL_07_all_pods_running.png)
*All 11 services Running. Zero restarts.*

---

## Quick Start

```bash
# Prerequisites: Docker, kind, kubectl

# 1. Create cluster
kind create cluster --name online-boutique

# 2. Clone repo
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo

# 3. Deploy all 11 services
kubectl apply -f ./release/kubernetes-manifests.yaml

# 4. Wait for all pods to be Running (~5-10 min first run)
kubectl get pods --watch

# 5. Access the app
kubectl port-forward deployment/frontend 8080:8080
# Open: http://localhost:8080
```

---

## Environment Setup

Before anything runs, these are the tool versions used in this deployment.

![Tool versions](./images/LOCAL_01_tool_versions.png)
*Docker, kind, and kubectl versions confirmed before starting.*

---

## Cluster Creation

![kind cluster created](./images/online_boutique_created.png)
*kind cluster spun up successfully.*

![kind start output](./images/LOCAL_03_kind_start_output.png)
*kind bootstrapping the control plane and worker node inside Docker.*

![kubectl get nodes](./images/LOCAL_04_kubectl_get_nodes.png)
*One node, Ready status. The cluster is live.*

---

## Repo Structure

![Repo folder structure](./images/LOCAL_02_repo_folder_structure.png)
*The microservices-demo repo after cloning. The `release/` folder holds the manifests we apply.*

---

## Deploying the App

![kubectl apply output](./images/LOCAL_05_kubectl_apply_output.png)
*All 11 Kubernetes resources created in one command.*

Pods do not start instantly. They go through `Pending` and `ContainerCreating` first while Kubernetes pulls images and schedules workloads.

![Pods in Pending and ContainerCreating](./images/LOCAL_06_pods_pending_containerCreating.png)
*Normal behavior. Nothing is broken here — images are still being pulled.*

Once all images are pulled and containers start, every pod moves to `Running`.

![All 11 pods Running](./images/LOCAL_07_all_pods_running.png)
*All 11 services Running. Zero restarts.*

![kubectl get services](./images/LOCAL_08_kubectl_get_services.png)
*Services and their ClusterIP addresses. The frontend service is the entry point.*

---

## The App

![Online Boutique Homepage](./images/LOCAL_09_homepage.png)
*Online Boutique running at localhost:8080*

An e-commerce store made up of **11 microservices across 5 programming languages**, all communicating over gRPC.

![Product detail page](./images/LOCAL_10_product_detail.png)
*Navigating to a product — frontend is calling productcatalogservice and recommendationservice behind the scenes.*

![Add to cart](./images/LOCAL_11_add_to_cart.png)
*Adding an item to the cart — cartservice storing the session in Redis over gRPC.*

![Checkout complete](./images/LOCAL_12_checkout_complete.png)
*Full checkout completed. checkoutservice coordinated 6 services to make this happen.*

---

## What Gets Deployed

```
NAME                          READY   STATUS    RESTARTS
adservice                     1/1     Running   0
cartservice                   1/1     Running   0
checkoutservice               1/1     Running   0
currencyservice               1/1     Running   0
emailservice                  1/1     Running   0
frontend                      1/1     Running   0
loadgenerator                 1/1     Running   0
paymentservice                1/1     Running   0
productcatalogservice         1/1     Running   0
recommendationservice         1/1     Running   0
shippingservice               1/1     Running   0
```

> **Note:** The `loadgenerator` starts sending simulated user traffic automatically the moment pods are Running. You will see live requests in frontend logs immediately — no configuration needed.

---

## Service Map

| Service | Language | Responsibility |
|---|---|---|
| frontend | Go | HTTP entry point. Calls every other service. |
| cartservice | C# | Stores cart data in Redis over gRPC. |
| productcatalogservice | Go | Returns product listings from a static JSON file. |
| checkoutservice | Go | Orchestrates the full order flow across 6 services. |
| paymentservice | Node.js | Simulates credit card processing. |
| emailservice | Python | Sends order confirmations (simulated). |
| currencyservice | Node.js | Handles currency conversion. |
| shippingservice | Go | Calculates shipping cost. |
| recommendationservice | Python | Suggests related products. |
| adservice | Java | Returns contextual ads. |
| loadgenerator | Python/Locust | Simulates real user traffic automatically. |

---

## Inspecting a Running Pod

![kubectl describe pod frontend A](./images/LOCAL_13_describe_pod_A.png)
*Pod metadata: node assignment, IP, image pulled, resource requests and limits.*

![kubectl describe pod frontend B](./images/LOCAL_13_describe_pod_B.png)
*Events section — shows the full lifecycle: Scheduled, Pulled, Created, Started.*

![kubectl logs frontend](./images/LOCAL_14_kubectl_logs.png)
*Live request logs from the frontend. The loadgenerator is already sending traffic.*

---

## Resource Usage

![kubectl top pods](./images/LOCAL_15_kubectl_top_pods.png)
*CPU and memory consumption across all pods. adservice and frontend are the heaviest.*

---

## Fault Injection Experiment

After confirming everything was healthy, I injected a CPU stressor into `recommendationservice`:

```bash
kubectl exec -it deployment/recommendationservice -- sh -c "while true; do true; done"
```

![After fault injection](./images/fault_injection_result.png)
*The recommendation service under CPU pressure. The rest of the app kept running normally.*

**What happened:**
- The recommendation service slowed down under CPU pressure
- The rest of the app (cart, checkout, payment) kept working normally
- Kubernetes did **not** restart the pod — because it wasn't crashing, just degrading

**Recovery:**

```bash
kubectl delete pod -l app=recommendationservice
```

Kubernetes created a fresh pod in ~30 seconds. This experiment taught me that Kubernetes protects against failures, not degradation. For that you need resource limits, liveness probes, and readiness probes properly configured.

---

## Cleanup

```bash
# Remove all application resources
kubectl delete -f ./release/kubernetes-manifests.yaml

# Delete the cluster
kind delete cluster --name online-boutique
```

![kubectl delete output](./images/LOCAL_16_kubectl_delete.png)
*All 11 resources deleted cleanly.*

---

## Full Walkthrough

Step-by-step guide including architecture breakdown, troubleshooting, and the full fault injection experiment:

- 📖 [Read on Dev.to](https://dev.to/vivianchiamaka)
- 📖 [Read on Hashnode](https://vivianchiamaka.hashnode.dev)

---

## About

**Vivian Chiamaka Okose** — Cloud DevOps Engineer, documenting the journey publicly.

[LinkedIn](https://linkedin.com/in/okosechiamaka) · [GitHub](https://github.com/vivianokose) · [X](https://x.com/vivianchiamaka)
