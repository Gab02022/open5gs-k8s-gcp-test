# 📑 Technical Post-Mortem & Lessons Learned

Durante la integración de Open5GS en un entorno de nube pública (GKE), se encontraron y resolvieron varios desafíos técnicos relacionados con la naturaleza específica de las aplicaciones de Telecomunicaciones (Telco).

## 1. Limitaciones del Kernel en Nubes Públicas (SCTP)
* **El Problema:** El componente **AMF** fallaba persistentemente (`CrashLoopBackOff`) con el error `Protocol not supported`.
* **Análisis:** Los nodos por defecto de GKE usan **COS (Container-Optimized OS)**, un sistema operativo minimalista que no incluye el módulo del kernel **SCTP**. SCTP es mandatorio para la interfaz N2 (conexión entre la antena gNodeB y el Core).
* **Solución:**
    1.  Migración de la infraestructura a nodos basados en **Ubuntu** (`--image-type=UBUNTU_CONTAINERD`).
    2.  Implementación de un `DaemonSet` privilegiado (`sctp-loader`) que ejecuta `nsenter` y `modprobe sctp` en el host para habilitar el protocolo.

## 2. Dependencias de Protocolos Legacy (Diameter/DNS)
* **El Problema:** El **SMF** fallaba con `Name or service not known` en librerías de `freeDiameter`, a pesar de configurar un entorno 5G SA.
* **Análisis:** Aunque se buscaba una arquitectura puramente 5G, el código de Open5GS mantiene una dependencia dura de **Diameter** para la gestión de políticas (interfaz Gx). Al deshabilitar el **PCRF**, el DNS de Kubernetes no podía resolver el host, causando un error fatal.
* **Solución:** Se habilitó el PCRF (`pcrf: enabled: true`) en el `values.yaml` para satisfacer la dependencia de resolución DNS y estabilizar el SMF.

## 3. Cadena de Suministro de Software (Supply Chain)
* **El Problema:** Múltiples componentes (MongoDB, WebUI InitContainer) fallaron con `ImagePullBackOff`.
* **Hallazgos:**
    1.  **Hardcoding:** El Helm Chart original tenía referencias *hardcoded* a repositorios deprecados (`bitnamilegacy`) dentro de los `initContainers` de la WebUI.
* **Solución:** Se implementó un Wrapper Chart que sobrescribe las imágenes en el `values.yaml` forzando el uso de tags estables (`latest` o versiones específicas testeada en AMD64).

## 4. Kubernetes State & Race Conditions
* **El Problema:** Durante las actualizaciones de configuración, servicios como el PCRF entraban en `CrashLoopBackOff`.
* **Análisis:** Se identificó una condición de carrera donde los servicios intentaban conectarse a la Base de Datos mientras esta se estaba reiniciando por una actualización de imagen.
* **Solución:** Se validó la capacidad de **Self-Healing** de Kubernetes. Tras estabilizarse la base de datos, los pods dependientes se recuperaron automáticamente o fueron forzados mediante un reinicio de pods.

## 5. Gestión de Software de Terceros (Vendoring & Patching)
* **El Problema:** El componente **WebUI** fallaba persistentemente en el estado `Init:ImagePullBackOff` o `Init:CrashLoopBackOff`.
* **Análisis Profundo:**
    1.  Se descubrió mediante auditoría del código fuente del Helm Chart (versión 2.2.6) que la imagen del `initContainer` estaba **hardcoded** (escrita en piedra) apuntando a una etiqueta inexistente: `bitnami/mongodb:4.4.1-debian-10-r39` o repositorios deprecados (`bitnamilegacy`).
    2.  Al intentar forzar la imagen `latest` mediante `values.yaml`, el contenedor fallaba con `Exit Code: 127` (Command not found), indicando que los scripts de inicialización del chart no eran compatibles con las versiones más nuevas de MongoDB (v7.0+) que han cambiado la estructura de binarios (mongo shell vs mongosh).
* **Solución (Técnica de Vendoring):**
    1.  Se optó por realizar **Vendoring** del Chart: descargar el código fuente al repositorio propio en lugar de referenciarlo remotamente.
    2.  Se aplicó un parche manual (`sed`) en el archivo `deployment.yaml` para reemplazar la imagen defectuosa por la imagen oficial de Docker **`mongo:5.0`**.
    3.  Esta versión específica (5.0) demostró tener compatibilidad binaria con los scripts legacy del chart y alta disponibilidad en Docker Hub, resolviendo el bloqueo.

---

---
**Conclusión:**
El despliegue de VNFs (Virtual Network Functions) en Kubernetes requiere un control granular sobre el Sistema Operativo subyacente y una gestión cuidadosa de las dependencias de protocolos legacy, más allá de la orquestación estándar de contenedores web.
