---
name: Contexto del Proyecto Claude_SFC
description: Proyecto de migración de datos Fondos Colectivos Santander Consumer Perú — arquitectura Medallion en Databricks
type: project
---

Proyecto de migración, orquestación y modelado de datos para el área de **Fondos Colectivos** de **Banco Santander Consumer Perú (SCP)**. Fase actual: Fase 1 — Levantamiento y análisis as-is (Abril 2026).

**Producto de negocio:** Fondo Tradicional (programa SAN001) — ahorro colectivo vehicular. Grupos de asociados realizan aportes mensuales; mediante sorteo/remate uno recibe el vehículo. Canales: Ejecutivos SCP, Clusters de agentes externos, Canal Digital, concesionarios (Autoland, Astara, Derco, Interamericana, Revo Motors).

**Problema actual:** Fuentes administradas por Softek (tercero on-premise), usuarios descargan CSV manualmente desde SANTO y procesan en Excel. Lógica de negocio sin versionar ni documentar.

**Why:** Eliminar dependencia de procesos manuales en Excel y establecer plataforma de datos moderna con gobernanza Unity Catalog.

**How to apply:** Todas las decisiones de diseño deben favorecer la automatización de pipelines, documentación de reglas de negocio y trazabilidad campo a campo (STM).

---

## Fuentes de datos

| Sistema | Tecnología | Tablas principales |
|---|---|---|
| SANTO | Softek on-premise | COTIZACIONES, SOLICITUDES, CERTIFICADOS, BD_REFERIDOS2, BD Lista Ejecutivos, GUIA_STATUS, BD Estado Solicitud, BD Alianzas, BD_Tipo, Dias_Utiles |
| Agenda Comercial Digital | Microsoft Lists (SharePoint) | Prospectos canal digital |
| Agenda Comercial Cluster | Microsoft Lists (SharePoint) | Prospectos canal cluster |

Archivo Excel de trabajo: `20260228_SFC_Dashboard.xlsx` (24 hojas). BD_REFERIDOS2 unifica ambas Agendas con campo AGENDA. Total referidos al 19/03/2026: **8,209**.

---

## Arquitectura destino

Medallion en Databricks:
- **Bronze:** Delta tables, ingesta raw con metadata de carga
- **Silver:** Dedup, jerarquías resueltas, fechas estandarizadas, flags hitos
- **Gold:** Tablas para dashboards (Resumen Certificados, Total Referidos SCP)
- **Visualización:** Power BI (por confirmar) sobre capa Gold
- **Gobernanza:** Unity Catalog
- **Orquestación:** Databricks Workflows o Azure Data Factory
- **Storage:** Delta Lake sobre Azure Data Lake Storage Gen2
- **Control de versiones:** GitHub (`carloshac7/Claude_SFC`)

---

## Reglas de negocio documentadas

| ID | Nombre | Aplica a |
|---|---|---|
| RN-001 | Clasificación TIPO (REAL / PRUEBA) | Cotizaciones, Solicitudes, Certificados |
| RN-002 | Clave de deduplicación (FILTRO / NOMENCLATURA) | Cotizaciones, Solicitudes |
| RN-003 | Resolución de jerarquía de ejecutivos | Cotizaciones, Solicitudes, Certificados |
| RN-004 | Clasificación de estados (Estado2 / Estado3 / Estado4) | Solicitudes |
| RN-005 | Flags de hitos del proceso (7 flags binarios) | Solicitudes |
| RN-006 | Certificado válido y comisión CIA | Certificados |
| RN-007 | Dominio FRENTE y clasificación de referidos | Agenda Comercial, Certificados |

Detalle completo planeado en `03_Modelado/Reglas_de_Negocio.md` y el BRD.

---

## Métricas actuales de dashboards a replicar

- **Resumen Certificados:** 458 ventas acumuladas (Mar 2024 – Feb 2026), 402 activos
- **Total Referidos SCP:** 8,209 referidos — SI_CONTACTO 63%, NO_CONTACTO 26%, SIN_GESTIÓN 10%
- **Dashboards por grupo:** G10, G11, G12 — solicitudes, rankings, días para cierre de asamblea

---

## Pendientes críticos (al 2026-04-23)

| # | Acción | Prioridad |
|---|---|---|
| P-01 | Negociar acceso automatizado a SANTO con Softek | Crítica |
| P-02 | Sesión levantamiento BD Lista Ejecutivos | Alta |
| P-03 | Documentar mapeo Estado → Estado2/3/4 | Alta |
| P-04 | Sesión levantamiento fuente REFERIDOS completa | Alta |
| P-05 | Confirmar herramienta BI destino | Alta |
| P-06 | Elaborar STM campo a campo | Alta |
| P-07 | Levantar tablas TD, GUIA_CAMP y STATUS de SANTO | Media |
| P-08 | Diseñar modelo lógico ERD en Databricks | Media |
| P-09 | Project Charter formal con cronograma y RACI | Alta |

---

## Stack tecnológico

- Databricks (Delta Live Tables o notebooks PySpark)
- Delta Lake sobre Azure Data Lake Storage Gen2
- Unity Catalog, Databricks Workflows / Azure Data Factory
- Power BI (por confirmar), GitHub
