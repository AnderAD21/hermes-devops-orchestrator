# ADR-001: Migración a Google Cloud Vertex AI con Autenticación ADC bajo Estrategia FinOps

* **Estado:** Aceptado
* **Fecha:** 2026-11-17
* **Autor:** Anderson Brayan Aranda Delgado

---

## 1. Contexto y Problema

El orquestador **Hermes Agent** requiere un proveedor de modelos de lenguaje (LLM) con alta capacidad de llamado a herramientas (*tool calling*), ventana de contexto amplia y baja latencia para coordinar flujos de trabajo DevOps y subagentes de código.

Inicialmente se evaluó el uso directo de **Google AI Studio** mediante una `GEMINI_API_KEY`. Sin embargo, identificamos tres limitaciones críticas para un entorno de ingeniería sostenible:
1. **Desaprovechamiento de Créditos Cloud:** Contamos con el beneficio del **Google Developer Program (Google AI Pro)**, que otorga **S/. 34.52 (~$10 USD) mensuales** en créditos de facturación de Google Cloud Platform (GCP). Dichos créditos aplican al consumo de **Google Cloud Vertex AI** (`aiplatform.googleapis.com`), pero no a llaves directas de AI Studio fuera de la facturación empresarial de GCP.
2. **Riesgo de Seguridad por API Keys Estáticas:** Las llaves estáticas en variables de entorno son más propensas a filtraciones accidentales que las credenciales delegadas basadas en IAM.
3. **Descontrol de Costos por Razonamiento Extenso:** Los modelos con razonamiento profundo (*thinking tokens*) pueden multiplicar el costo por solicitud si no se restringe el nivel de esfuerzo en tareas operativas rutinarias.

---

## 2. Decisión

1. **Migrar el proveedor de inferencia principal a Google Cloud Vertex AI** dentro del proyecto dedicado `hermes-devops-agent` en la ubicación `global`.
2. **Estandarizar la autenticación mediante Application Default Credentials (ADC)** (`~/.config/gcloud/application_default_credentials.json`), eliminando el uso de API Keys estáticas de AI Studio.
3. **Implementar una política FinOps de selección de modelos y esfuerzo de razonamiento:**
   * **Modelo Principal:** `google/gemini-3.5-flash` con esfuerzo de razonamiento configurado en `LOW` para mantener respuestas ágiles y predecibles en costo.
   * **Modelo Auxiliar:** `google/gemini-3.5-flash-lite` asignado a tareas secundarias (compresión de historial, generación de títulos de sesión y resúmenes).
   * **Presupuesto y Alertas:** Monitorear el consumo en GCP Billing para operar 100% dentro del crédito mensual recurrente de S/. 34.52 (~$10 USD).

---

## 3. Consecuencias

### Positivas
* **Costo Operativo Neto Cero ($0 USD de bolsillo):** Toda la inferencia del orquestador y de los subagentes queda cubierta por el crédito mensual renovable de GCP.
* **Seguridad Mejorada (IAM/ADC):** Las credenciales siguen el estándar de Google Cloud SDK, permitiendo revocación centralizada, rotación de tokens OAuth2 y control granular de permisos IAM.
* **Observabilidad Financiera y de Cuotas:** Acceso directo a métricas de consumo por API, latencia y límites de tasa (*quotas*) desde la consola de Google Cloud.

### Negativas / Trade-offs
* **Complejidad Inicial de Configuración:** Requiere instalar `gcloud` CLI, habilitar `aiplatform.googleapis.com` y ejecutar `gcloud auth application-default login` en el host (y montar las credenciales en modo solo-lectura dentro de contenedores autorizados).
* **Disciplina en el Uso de Modelos:** Tareas que requieran razonamiento matemático o arquitectónico extremo deberán habilitar el modo de razonamiento alto de forma puntual bajo demanda.
