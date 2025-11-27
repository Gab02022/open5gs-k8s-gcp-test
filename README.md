# 5G Core Network on GKE with GitOps 📡☁️

> 🇪🇸 [Leer en Español](README.es.md)

This project demonstrates the automated deployment of a **5G Standalone Core (Open5GS)** on **Google Kubernetes Engine (GKE)**, utilizing **GitOps** methodologies and modern **Cloud Native** practices.

The goal is to showcase the integration of Telecommunications Workloads requiring specific protocols (SCTP, Diameter) within orchestrated container environments.

## 🏗️ Architecture & Tech Stack

* **Cloud Provider:** Google Kubernetes Engine (GKE) Standard.
* **OS:** Ubuntu (required for kernel module loading).
* **5G Core:** Open5GS (v2.7.x) deployed via Helm Charts.
* **GitOps:** ArgoCD for automatic synchronization and state management.
* **Networking:** Enabled **SCTP** (Stream Control Transmission Protocol) for N2 interface (AMF-gNodeB).
* **Database:** MongoDB.

## 📂 Repository Structure

```text
.
├── charts/
│   └── open5gs/    # Vendored Chart with "Golden" configuration (5G SA pure)
│       ├── templates/    # Patched source code (WebUI fix)
│       └── values.yaml   # Overrides (PCRF on, WebUI fix, Mongo latest)
├── infra/
│   └── sctp-loader.yaml  # Privileged DaemonSet for SCTP kernel injection
├── LESSONS_LEARNED.md    # Technical troubleshooting log & Post-Mortem
└── README.md             # Main documentation
```

## 🚀 Quick Start
**1. Infrastructure (GKE)**
Ubuntu nodes are required to support Telco drivers.
```text
gcloud container clusters create open5gs-gitops \
    --zone us-central1-a \
    --machine-type e2-standard-4 \
    --num-nodes 1 \
    --image-type=UBUNTU_CONTAINERD
```
**2. ArgoCD Installation**
```text
kubectl create namespace argocd
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
```
**3. GitOps Sync**
Once inside ArgoCD, configure two applications:
infra-setup: Deploys sctp-loader to prepare node kernels.
open5gs-core: Deploys the full 5G application based on the custom Helm Chart.

<img width="625" height="243" alt="image" src="https://github.com/user-attachments/assets/3a8af73c-4fd0-45b7-bd95-1ccc07a06326" />
<img width="356" height="341" alt="image" src="https://github.com/user-attachments/assets/0e7c0327-5266-4420-b1e5-22618d657f63" />

