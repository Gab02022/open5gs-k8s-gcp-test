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
    2.  **Arquitectura:** Se detectaron errores de compatibilidad de CPU (`no match for platform`) al intentar usar digests SHA256 específicos de ARM64 en nodos x86.
* **Solución:** Se implementó un Wrapper Chart que sobrescribe las imágenes en el `values.yaml` forzando el uso de tags estables (`latest` o versiones específicas testeada en AMD64).

## 4. Kubernetes State & Race Conditions
* **El Problema:** Durante las actualizaciones de configuración, servicios como el PCRF entraban en `CrashLoopBackOff`.
* **Análisis:** Se identificó una condición de carrera donde los servicios intentaban conectarse a la Base de Datos mientras esta se estaba reiniciando por una actualización de imagen.
* **Solución:** Se validó la capacidad de **Self-Healing** de Kubernetes. Tras estabilizarse la base de datos, los pods dependientes se recuperaron automáticamente o fueron forzados mediante un reinicio de pods.

---
**Conclusión:**
El despliegue de VNFs (Virtual Network Functions) en Kubernetes requiere un control granular sobre el Sistema Operativo subyacente y una gestión cuidadosa de las dependencias de protocolos legacy, más allá de la orquestación estándar de contenedores web.
