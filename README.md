# 5G Core Network on GKE with GitOps 📡☁️

Este proyecto demuestra el despliegue de una red **Core 5G Standalone (Open5GS)** totalmente funcional sobre **Google Cloud Platform (GCP)**, utilizando metodologías **GitOps** y prácticas modernas de **Cloud Native**.

El objetivo es demostrar la integración de cargas de trabajo de Telecomunicaciones (Telco Workloads) que requieren protocolos específicos (SCTP, Diameter) dentro de entornos de contenedores orquestados.

## 🏗️ Arquitectura y Tecnologías

* **Cloud Provider:** Google Kubernetes Engine (GKE) Standard.
* **OS:** Ubuntu (requerido para carga de módulos del Kernel).
* **5G Core:** Open5GS (v2.7.x) desplegado vía Helm Charts.
* **GitOps:** ArgoCD para sincronización automática y manejo de estado.
* **Networking:** Habilitación de **SCTP** (Stream Control Transmission Protocol) para la interfaz N2 (AMF-gNodeB).
* **Database:** MongoDB.

## 📂 Estructura del Repositorio

```text
.
├── charts/
│   └── open5gs-5g-sa/    # Wrapper Chart con la configuración "Golden" (5G SA puro)
│       ├── Chart.yaml    # Definición de dependencias
│       └── values.yaml   # Overrides (PCRF on, WebUI off, Mongo latest)
├── infra/
│   └── sctp-loader.yaml  # DaemonSet privilegiado para inyección de módulos SCTP
├── LESSONS_LEARNED.md    # Bitácora técnica de resolución de problemas (Troubleshooting)
└── README.md             # Documentación principal


🚀 Despliegue Rápido1. Infraestructura (GKE)Se requiere un clúster con nodos Ubuntu para soportar los drivers de telecomunicaciones.Bashgcloud container clusters create open5gs-gitops \
    --zone us-central1-a \
    --machine-type e2-standard-4 \
    --num-nodes 1 \
    --image-type=UBUNTU_CONTAINERD

2. Instalación de ArgoCDBashkubectl create namespace argocd
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
3. Sincronización (GitOps)Una vez dentro de ArgoCD, se configuran dos aplicaciones:infra-setup: Despliega el sctp-loader para preparar el kernel de los nodos.open5gs-core: Despliega la aplicación 5G completa basada en el Helm Chart personalizado.✅ Estado Final de los ServiciosMicroservicioEstadoFunción / Protocolo ClaveAMFRunningAccess Management. Escucha en SCTP (N2).SMFRunningSession Management. Conecta a PCRF vía Diameter.UPFRunningUser Plane. Maneja tráfico de datos (GTP-U).PCRFRunningPolicy Control. Habilitado para resolver dependencias del SMF.MongoDBRunningBase de datos de suscriptores (Imagen latest).
