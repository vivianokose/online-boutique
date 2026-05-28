# Online Boutique — Local Deployment

> Deployed locally using **kind** (Kubernetes IN Docker). No cloud account needed.
> Part of my DevOps engineering training — documenting everything publicly.

![All 11 pods Running](./screenshots/LOCAL_07_all_pods_running.png)
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

## The App

![Online Boutique Homepage](./screenshots/LOCAL_09_homepage.png)
*Online Boutique running at localhost:8080*

An e-commerce store made up of **11 microservices across 5 programming languages**, all communicating over gRPC.

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

## Fault Injection Experiment

After confirming everything was healthy, I injected a CPU stressor into `recommendationservice`:

```bash
kubectl exec -it deployment/recommendationservice -- sh -c "while true; do true; done"
```

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

---

## Full Walkthrough

Step-by-step guide including architecture breakdown, troubleshooting, and the full fault injection experiment:

- 📖 [Read on Dev.to](https://dev.to/vivianchiamaka)
- 📖 [Read on Hashnode](https://vivianchiamaka.hashnode.dev)

---

## About

**Vivian Chiamaka Okose** — DevOps Engineer in training, documenting the journey publicly.

[LinkedIn](https://linkedin.com/in/okosechiamaka) · [GitHub](https://github.com/vivianokose) · [X](https://x.com/vivianchiamaka)
