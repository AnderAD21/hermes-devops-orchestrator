# Hermes DevOps Orchestrator

[![Cloud Provider](https://img.shields.io/badge/Google_Cloud-Vertex_AI-4285F4?logo=googlecloud&logoColor=white)](https://cloud.google.com/vertex-ai)
[![Infrastructure](https://img.shields.io/badge/Microsoft_Azure-Linux_VM-0078D4?logo=microsoftazure&logoColor=white)](https://azure.microsoft.com/)
[![Container](https://img.shields.io/badge/Docker-Rootless_Sandbox-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![OS](https://img.shields.io/badge/Fedora_Linux-44-51A2DA?logo=fedora&logoColor=white)](https://fedoraproject.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Arquitectura híbrida y multi-cloud de orquestación de agentes de IA orientada a **DevOps, Automatización e Ingeniería de Software**, diseñada bajo principios de **FinOps**, **mínimo privilegio** y **aislamiento en contenedores Docker**.

Este repositorio documenta la infraestructura, configuración declarativa, decisiones de arquitectura (*ADRs*) y *skills* personalizadas para operar **Hermes Agent** como cerebro orquestador conectado a **Google Cloud Vertex AI** y delegar tareas de ingeniería de código a subagentes efímeros (**OpenCode CLI**) en contenedores aislados tanto en entorno local como en **Microsoft Azure**.

---

## Arquitectura del Sistema

### 1. Estado Actual (As-Is: Hitos 0–1 — Operativo)
Entorno local validado en **Fedora Linux 44**, con **Hermes Agent** autenticado mediante **Application Default Credentials (ADC)** contra **Google Cloud Vertex AI** y flujo GitOps hacia GitHub:

```mermaid
flowchart LR
    subgraph LocalHost["Nodo Local (Fedora Linux 44)"]
        DEV["Usuario / Dev (CLI & Hermes Desktop)"]
        HERMES_LOCAL["Hermes Agent\n(~/.hermes)"]
        ADC["GCP ADC Auth\n(OAuth2 Credentials)"]
        DEV --> HERMES_LOCAL
        HERMES_LOCAL --> ADC
    end

    subgraph GCP["AI & FinOps Layer (Google Cloud Platform)"]
        VERTEX["Google Cloud Vertex AI\n(project: hermes-devops-agent | global)"]
        FLASH["gemini-3.5-flash\n(Reasoning: LOW)"]
        LITE["gemini-3.5-flash-lite\n(Tareas Auxiliares)"]
        VERTEX --> FLASH
        VERTEX --> LITE
    end

    subgraph GitHub["Control de Versiones"]
        REPO["GitHub Repository\n(AnderAD21/hermes-devops-orchestrator)"]
    end

    ADC -->|"Inferencia Cubierta por Crédito GCP"| VERTEX
    HERMES_LOCAL -->|"Git + gh CLI (PR Workflow)"| REPO
```

### 2. Arquitectura Objetivo (To-Be: Hitos 2–3 — Despliegue 24/7 y Sandboxing)
Evolución hacia una topología híbrida con nodo siempre activo en **Microsoft Azure**, aislamiento de subagentes de código en contenedores **Docker rootless** e interoperabilidad **A2A v1.0** (*las líneas sólidas `-->` indican flujos ya validados y las punteadas `-.->` indican componentes en despliegue*):

```mermaid
flowchart TB
    subgraph ControlPlane["Control Plane (Interacción)"]
        CLI_USER["Usuario / Dev (CLI Local)"]
        TG_USER["Usuario vía Telegram Bot (24/7)"]
    end

    subgraph FedoraNode["Nodo Local (Fedora Linux 44)"]
        HERMES_FEDORA["Hermes Agent Local\n(Orquestador de Escritorio)"]
    end

    subgraph AzureRG["Azure Resource Group (rg-hermes-devops — GitHub Student Pack $100 Credit)"]
        NSG["Network Security Group (NSG)\n(SSH Hardening / Puerto 22 Restringido)"]
        subgraph AzureVM["Azure VM Linux (Ubuntu LTS B2s)"]
            HERMES_CLOUD["Hermes Agent 24/7\n(Gateway & Orquestador Cloud)"]
            SKILL["Skill: opencode-delegate"]
            subgraph DockerSandbox["Contenedor Docker Aislado (No-Root / UID 1000)"]
                OPENCODE["OpenCode CLI + Git + Runtimes\n(Volumen /workspace aislado)"]
            end
        end
    end

    subgraph GCPLayer["AI & FinOps Layer (Google Cloud Platform — Developer Program Credit)"]
        VERTEX_AI["Google Cloud Vertex AI\n(project: hermes-devops-agent)"]
        MODELS["gemini-3.5-flash (Primary)\ngemini-3.5-flash-lite (Auxiliary)"]
        VERTEX_AI --> MODELS
    end

    subgraph VCS["GitOps & CI/CD"]
        GH_REMOTE["GitHub Repositories\n(Ramas feat/*, Commits Atómicos & PRs)"]
    end

    CLI_USER --> HERMES_FEDORA
    HERMES_FEDORA -->|"ADC Auth (Validado)"| VERTEX_AI
    HERMES_FEDORA -->|"gh CLI (Validado)"| GH_REMOTE

    TG_USER -.->|"(Planned) Webhook / Long-Polling"| HERMES_CLOUD
    CLI_USER -.->|"(Planned) SSH Key ed25519"| NSG
    NSG -.->|"(Planned) Tráfico Autorizado"| HERMES_CLOUD
    HERMES_FEDORA <-.->|"(Planned) Protocolo A2A v1.0"| HERMES_CLOUD

    HERMES_CLOUD -.->|"(Planned) Delegación"| SKILL
    SKILL -.->|"(Planned) Bind Mount /workspace"| OPENCODE
    HERMES_CLOUD -.->|"(Planned) ADC Auth"| VERTEX_AI
    OPENCODE -.->|"(Planned) Inferencia Agéntica"| VERTEX_AI
    OPENCODE -.->|"(Planned) Push & Pull Requests"| GH_REMOTE
```

---

## Estrategia FinOps y Optimización de Recursos Cloud

Uno de los pilares de ingeniería de este proyecto es operar una arquitectura cloud de nivel productivo manteniendo un **costo operativo neto de $0.00 USD de bolsillo**, mediante la combinación estratégica de créditos multi-nube y control de consumo por token:

| Proveedor / Programa | Beneficio Asignado | Rol en la Arquitectura | Mecanismo de Control FinOps |
| :--- | :--- | :--- | :--- |
| **Microsoft Azure**<br>*(GitHub Student Developer Pack)* | **$100.00 USD** en créditos cloud | Cómputo 24/7 en máquina virtual Linux (`rg-hermes-devops`, instancia Ubuntu B2s) en reemplazo de una VPS comercial pagada. | Selección de SKU de bajo consumo, apagado/redimensionamiento programado si no hay carga y almacenamiento estándar. |
| **Google Cloud Platform**<br>*(Google Developer Program — AI Pro)* | **S/. 34.52 / mes** (~$10.00 USD renovables mensualmente) | Inferencia LLM centralizada en **Vertex AI** (`aiplatform.googleapis.com`) tanto para el nodo local como para el nodo en Azure. | Uso de `gemini-3.5-flash` con `reasoning.effort: "low"` por defecto y `gemini-3.5-flash-lite` para tareas auxiliares/compresión. |
| **Host Local**<br>*(Fedora Linux 44)* | Cómputo local propio | Desarrollo, pruebas de contenedores Docker y orquestación interactiva sin consumir horas de VM cuando se trabaja en escritorio. | Descarga de tareas pesadas de compilación local antes de promover cambios al entorno cloud. |

> **Resultado FinOps:** Separar el plano de **cómputo continuo (Azure)** del plano de **inferencia agéntica (GCP Vertex AI)** permite absorber el 100% de la facturación dentro de los programas de desarrollador y educación vigentes, evitando cargos recurrentes de tarjeta y manteniendo observabilidad total de cuotas e IAM.

---

## Stack Tecnológico

| Capa | Tecnología | Propósito en la Arquitectura |
| :--- | :--- | :--- |
| **Sistema Operativo Host** | Fedora Linux 44 (`fish` shell) / Ubuntu LTS (Azure) | Entorno de desarrollo local y nodo cloud 24/7 |
| **Agente Orquestador** | [Hermes Agent](https://github.com/NousResearch/hermes-agent) | Gestión de sesiones, memoria SQLite, delegación de tareas y pasarela Telegram |
| **Interoperabilidad Agéntica** | Protocolo A2A v1.0 / SSH | Comunicación entre el agente local en Fedora y el agente 24/7 en Azure |
| **Proveedor LLM & FinOps** | Google Cloud Vertex AI (`aiplatform.googleapis.com`) | Inferencia con `gemini-3.5-flash` y `gemini-3.5-flash-lite` cubierta por créditos GCP |
| **Autenticación Cloud** | GCP Application Default Credentials (ADC) | Autenticación segura sin API Keys estáticas expuestas en código |
| **Sandboxing & Subagente** | Docker Engine + OpenCode CLI | Ejecución aislada de generación/refactorización de código sin permisos `root` |
| **Infraestructura Cloud** | Microsoft Azure (`rg-hermes-devops` + VM B2s + NSG) | Despliegue continuo del orquestador y pasarela de mensajería |
| **Flujo GitOps** | Git + GitHub CLI (`gh`) | Convención de ramas, *Conventional Commits* y automatización de Pull Requests |

---

## Estructura del Repositorio

```text
hermes-devops-orchestrator/
├── config/
│   ├── .env.example          # Plantilla sanitizada de variables de entorno (Vertex AI, Docker, Telegram)
│   └── config.example.yaml   # Configuración base de Hermes Agent (FinOps + backend Docker)
├── docs/
│   └── adr/                  # Architecture Decision Records (decisiones técnicas y financieras)
│       ├── 001-vertex-ai-finops.md
│       └── 002-docker-sandboxing.md
├── scripts/                  # Scripts de automatización, aprovisionamiento y diagnóstico
├── skills/                   # Skills personalizadas de Hermes (ej. opencode-delegate)
├── .gitignore                # Reglas estrictas de bloqueo de secretos, credenciales JSON y estados
└── README.md                 # Documentación principal del proyecto
```

---

## Guía de Despliegue y Configuración

### 1. Prerrequisitos
* **Linux** (Fedora 44 / Ubuntu 22.04+) con `git`, `gh`, `docker` y `gcloud` CLI instalados.
* Proyecto activo en **Google Cloud Platform** con la API `aiplatform.googleapis.com` habilitada.
* **Hermes Agent** instalado en `~/.hermes/`.

### 2. Autenticación en Google Cloud (ADC)
En lugar de utilizar llaves estáticas de AI Studio, autenticamos el entorno mediante **Application Default Credentials**:

```bash
gcloud auth login
gcloud config set project hermes-devops-agent
gcloud auth application-default login
gcloud services enable aiplatform.googleapis.com
```

### 3. Configuración de Hermes Agent
Copia las plantillas sanitizadas de este repositorio hacia tu directorio de configuración local:

```bash
cp config/.env.example ~/.hermes/.env
cp config/config.example.yaml ~/.hermes/config.yaml
```

Edita `~/.hermes/.env` ajustando la ruta local de tu archivo `application_default_credentials.json` y, opcionalmente, el token de tu bot de Telegram.

### 4. Despliegue en Nube (Hito 3 — Microsoft Azure)
1. Aprovisionar el grupo de recursos `rg-hermes-devops` y la VM Linux en Azure con grupo de seguridad de red (**NSG**) restringido exclusivamente a SSH por llave pública (`ed25519`).
2. Instalar Docker Engine (modo sin privilegios `root`) y Hermes Agent.
3. Sincronizar este repositorio y levantar el servicio de `hermes gateway` conectado a Telegram, Vertex AI y enlace opcional **A2A v1.0**.

---

## Índice de Architecture Decision Records (ADRs)

Las decisiones arquitectónicas clave del proyecto están documentadas bajo el formato estándar de ingeniería en [`docs/adr/`](docs/adr/):

* **[ADR-001: Migración a Google Cloud Vertex AI con ADC bajo Estrategia FinOps](docs/adr/001-vertex-ai-finops.md)** — Justificación financiera y técnica del paso de Google AI Studio a Vertex AI para capitalizar créditos mensuales del *Google Developer Program* con control estricto de cuotas y costos por token.
* **[ADR-002: Aislamiento de Subagentes de Código (OpenCode) en Contenedores Docker sin Privilegios](docs/adr/002-docker-sandboxing.md)** — Diseño de seguridad para ejecutar herramientas de modificación de código y terminal dentro de contenedores efímeros `non-root` con montajes restringidos.

---

## Hoja de Ruta (Roadmap)

- [x] **Hito 0 — Entorno Local y FinOps:** Instalación de Hermes Agent en Fedora 44, configuración de `gcloud` ADC y validación de consumo en Vertex AI (`hermes-devops-agent`).
- [x] **Hito 1 — Repositorio DevOps y Documentación:** Estructura base, protección de secretos, diagramas de arquitectura e historial de ADRs.
- [ ] **Hito 2 — Sandboxing con Docker + OpenCode:** Construcción de imagen Docker rootless con OpenCode CLI y creación de la skill `opencode-delegate`.
- [ ] **Hito 3 — Despliegue en Azure (IaC):** Aprovisionamiento automatizado de VM en Azure (`rg-hermes-devops`), hardening SSH/NSG, Telegram Gateway 24/7 e interoperabilidad A2A v1.0.

---

## Autor

**Anderson Brayan Aranda Delgado** (`@AnderAD21`)  
Ingeniería Informática — Universidad Nacional de Trujillo (UNT, Perú)  
Enfoque en *DevOps, Cloud Engineering & AI Automation*.
