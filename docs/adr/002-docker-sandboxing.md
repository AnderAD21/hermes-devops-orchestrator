# ADR-002: Aislamiento de Subagentes de Código (OpenCode) en Contenedores Docker sin Privilegios

* **Estado:** Aceptado
* **Fecha:** 2026-11-17
* **Autor:** Anderson Brayan Aranda Delgado

---

## 1. Contexto y Problema

En nuestra arquitectura, **Hermes Agent** actúa como orquestador principal y delega tareas de desarrollo, refactorización, ejecución de pruebas y manipulación de archivos a **OpenCode CLI** y herramientas de terminal.

Permitir que un agente de IA autónomo ejecute comandos arbitrarios de shell (`bash`/`fish`), instale dependencias (`npm`, `pip`, `cargo`) o modifique archivos directamente sobre el sistema operativo anfitrión (Fedora Linux 44 en local o Ubuntu Server en Azure) introduce riesgos severos:
1. **Exposición de Secretos del Host:** Un comando mal formado o una inyección de prompt podría leer `~/.ssh/`, `~/.config/gcloud/`, `~/.hermes/.env` o variables de entorno sensibles del sistema.
2. **Contaminación del Entorno Base:** La instalación de librerías o paquetes de distintos proyectos degrada la reproducibilidad del host.
3. **Escalación de Privilegios y Daño en Sistema de Archivos:** Comandos destructivos (`rm -rf`, modificación de permisos) ejecutados con el usuario del sistema o `root` comprometen la disponibilidad del servidor.

---

## 2. Decisión

1. **Aislar la ejecución de herramientas de terminal y del subagente OpenCode CLI dentro de contenedores Docker efímeros.**
2. **Aplicar endurecimiento (*hardening*) de contenedores bajo el principio de mínimo privilegio:**
   * **Usuario No-Root (`docker_run_as_host_user: true`):** El contenedor se ejecuta con el `UID:GID` de un usuario sin privilegios (`1000:1000`), evitando que procesos dentro del contenedor tengan permisos de superusuario o dejen archivos pertenecientes a `root` en los volúmenes montados.
   * **Eliminación de Capacidades Peligrosas:** Se descartan capacidades `SETUID` y `SETGID` innecesarias.
   * **Montaje Explícito de Directorio de Trabajo (`/workspace`):** Solo se monta el directorio del repositorio objetivo en `/workspace`, manteniendo el resto del sistema de archivos del host (`$HOME`, `/etc`, credenciales de otros servicios) completamente inaccesible.
   * **Passthrough Explícito de Variables:** Solo se inyectan las variables estrictamente necesarias para la tarea, bloqueando por defecto el reenvío automático de secretos del host.

---

## 3. Consecuencias

### Positivas
* **Frontera de Seguridad Fuerte:** Si un subagente ejecuta un comando destructivo o procesa código de terceros no confiable, el impacto queda confinado al contenedor efímero y al directorio `/workspace` del proyecto bajo prueba.
* **Entorno 100% Reproducible y Portátil:** La misma imagen Docker (`hermes-opencode-sandbox`) funciona de manera idéntica en Fedora Linux 44 local y en la máquina virtual de Microsoft Azure (Hito 3).
* **Higiene del Sistema Host:** Los compiladores, runtimes y dependencias de prueba viven dentro de la imagen del contenedor, manteniendo el servidor anfitrión limpio.

### Negativas / Trade-offs
* **Sobrecarga Mínima de Almacenamiento e Inicio:** Requiere construir y mantener actualizada la imagen Docker base con OpenCode CLI y las herramientas de desarrollo necesarias.
* **Gestión de Permisos en Bind Mounts:** Requiere alinear el `UID:GID` del usuario del contenedor con el usuario del host para evitar conflictos de escritura en `/workspace`.
