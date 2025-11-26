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
