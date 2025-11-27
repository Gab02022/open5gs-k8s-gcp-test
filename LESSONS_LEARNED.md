# 📑 Technical Post-Mortem & Lessons Learned

During the integration of Open5GS in a public cloud environment (GKE), several technical challenges arose related to the specific nature of Telecommunications Workloads (Telco).

## 1. Kernel Limitations in Public Cloud (SCTP)
* **The Issue:** The **AMF** component persistently failed (`CrashLoopBackOff`) with a `Protocol not supported` error.
* **Analysis:** GKE default nodes use **COS (Container-Optimized OS)**, a hardened, minimalist OS that excludes non-standard kernel modules like **SCTP**. SCTP is mandatory for the N2 interface (signaling between gNodeB and Core).
* **Solution:**
    1.  Migrated infrastructure to **Ubuntu**-based nodes (`--image-type=UBUNTU_CONTAINERD`).
    2.  Implemented a privileged `DaemonSet` (`sctp-loader`) that utilizes `nsenter` and `modprobe sctp` on the host to enable the protocol dynamically.

## 2. Legacy Protocol Dependencies (Diameter/DNS)
* **The Issue:** The **SMF** failed to start with a `Name or service not known` error within `freeDiameter` libraries, despite configuring a 5G Standalone environment.
* **Analysis:** Even though a pure 5G architecture was intended, the Open5GS codebase maintains a hard dependency on **Diameter** for policy management (Gx interface). By disabling the **PCRF**, internal Kubernetes DNS could not resolve the host, causing a fatal error.
* **Solution:** The PCRF was enabled (`pcrf: enabled: true`) in `values.yaml` to satisfy the DNS resolution dependency and stabilize the SMF.

## 3. Software Supply Chain & Architecture
* **The Issue:** Multiple components (MongoDB, WebUI InitContainer) failed with `ImagePullBackOff` or `ErrImagePull`.
* **Findings:**
    1.  **Deprecated Tags:** The Helm Chart referenced specific image tags that had been removed from Docker Hub registries.
* **Solution:** Implemented a Wrapper Chart strategy, overriding image references in `values.yaml` to force the use of stable tags (`latest` or specific versions tested on AMD64).

## 4. Kubernetes State & Race Conditions
* **The Issue:** During configuration updates via GitOps, services like PCRF entered a `CrashLoopBackOff` state.
* **Analysis:** A race condition was identified where dependent services attempted to connect to the Database while it was restarting due to an image update.
* **Solution:** Validated Kubernetes **Self-Healing** capabilities. Once the database stabilized, dependent pods automatically recovered or were forced via a pod delete/restart strategy.

## 5. Third-Party Software Management (Vendoring & Patching)
* **The Issue:** The **WebUI** component persistently failed in `Init:ImagePullBackOff`.
* **Deep Analysis:**
    1.  Source code auditing of the Helm Chart (v2.2.6) revealed that the `initContainer` image was **hardcoded** to a deprecated repository (`bitnamilegacy` or nonexistent tags), ignoring `values.yaml` overrides.
    2.  Attempts to force a `latest` image resulted in `Exit Code: 127` (Command not found), indicating the chart's initialization scripts were incompatible with newer MongoDB binaries (mongosh vs mongo shell).
* **Solution (Vendoring Technique):**
    1.  Adopted a **Chart Vendoring** strategy: downloading the source code into the repository instead of referencing it remotely.
    2.  Applied a manual patch (`sed`) to `deployment.yaml`, replacing the defective image with the official Docker **`mongo:5.0`** image.
    3.  This specific version (5.0) provided the necessary binary compatibility for legacy scripts while ensuring high availability on Docker Hub.

---
**Conclusion:**
Deploying VNFs (Virtual Network Functions) on Kubernetes requires granular control over the underlying Operating System and careful management of legacy protocol dependencies, surpassing standard web-container orchestration practices.
