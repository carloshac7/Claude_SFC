# Claude_SFC — Proyecto de Migración de Datos · Fondos Colectivos · Banco Santander Consumer Perú

> **Estado:** Fase 1 — Levantamiento y análisis as-is  
> **Última actualización:** Abril 2026  
> **Clasificación:** Confidencial

---

## Descripción del proyecto

Proyecto de migración, orquestación y modelado de bases de datos en la nube para el área de **Fondos Colectivos** de **Banco Santander Consumer Perú (SCP)**.

Las fuentes de datos actuales están gobernadas por un proveedor externo (Softek) y residen on-premise. El objetivo es diseñar e implementar una plataforma de datos moderna sobre **Databricks** (arquitectura Medallion: Bronze / Silver / Gold), eliminando la dependencia de procesos manuales en Excel y estableciendo una capa de gobernanza de datos con Unity Catalog.

---

## Contexto de negocio

**Fondos Colectivos** es un producto de ahorro colectivo vehicular. Un grupo de clientes (asociados) realiza aportes mensuales a un fondo común; periódicamente, mediante sorteo o remate, uno recibe el vehículo financiado. El producto se llama **Fondo Tradicional (programa SAN001)**.

Los canales de venta son: Ejecutivos SCP directos, Clusters de agentes externos, Canal Digital (web/redes sociales) y alianzas con concesionarios (Autoland, Astara, Derco, Interamericana, Revo Motors, entre otros).

---

## Problema actual

- Las fuentes (SANTO, Agenda Comercial) son administradas por Softek (tercero) y residen on-premise.
- Los usuarios descargan CSV manualmente desde SANTO y procesan en Excel con fórmulas propias.
- La lógica de negocio (clasificación, deduplicación, jerarquías, hitos del proceso) vive en columnas calculadas de Excel, sin versionar ni documentar.
- Los dashboards se alimentan de exportaciones manuales sin frecuencia definida.
- No existe una capa de datos centralizada, integrada ni gobernada.

---

## Fuentes de datos identificadas

| Sistema | Tecnología | Tablas principales | Extracción actual |
|---|---|---|---|
| SANTO | Softek on-premise | COTIZACIONES, SOLICITUDES, CERTIFICADOS, BD_REFERIDOS2, BD Lista Ejecutivos, GUIA_STATUS, BD Estado Solicitud, BD Alianzas, BD_Tipo, Dias_Utiles | Descarga manual CSV |
| Agenda Comercial Digital | Microsoft Lists (SharePoint) | Prospectos canal digital | Exportación manual CSV |
| Agenda Comercial Cluster | Microsoft Lists (SharePoint) | Prospectos canal cluster | Exportación manual CSV |

> **Nota:** En el libro Excel de trabajo (`20260228_SFC_Dashboard.xlsx`), las dos Agendas Comerciales están unificadas en la hoja `BD_REFERIDOS2` con el campo `AGENDA` distinguiendo la fuente. Al 19/03/2026 suman **8,209 referidos totales**.

---

## Arquitectura destino

```
Fuentes on-premise / SharePoint
        │
        ▼ (pipeline automatizado)
┌─────────────────────────────────┐
│  BRONZE — ingesta raw           │  Delta tables, sin transformación,
│  (Databricks / Delta Lake)      │  con metadata de carga
└─────────────────────────────────┘
        │
        ▼ (transformaciones Silver)
┌─────────────────────────────────┐
│  SILVER — datos limpios         │  Dedup, jerarquías resueltas,
│  (Databricks)                   │  fechas estandarizadas, flags hitos
└─────────────────────────────────┘
        │
        ▼ (agregaciones Gold)
┌─────────────────────────────────┐
│  GOLD — datos para consumo      │  Tablas para dashboards:
│  (Databricks)                   │  Resumen Certificados,
└─────────────────────────────────┘  Total Referidos SCP
        │
        ▼
   Dashboards / Power BI (por confirmar)
```

**Plataformas evaluadas:** Databricks (principal), Cosmos DB, AWS (fase 2)

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

> Detalle completo en `03_Modelado/Reglas_de_Negocio.md` y en el BRD.

---

## Dashboards actuales a replicar

### Resumen Certificados
- 458 ventas totales acumuladas (Mar 2024 – Feb 2026)
- 402 certificados activos
- Desglose diario / mensual por equipo (SCP / Digital) y tipo (Directo / Referidos)

### Total Referidos SCP
- 8,209 referidos totales (19/03/2026)
- Métricas: SI_CONTACTO (63%), NO_CONTACTO (26%), SIN_GESTIÓN (10%)
- Breakdown: SC_ASOCIADO, SC_EN_GESTIÓN, SC_VOLVER_A_LLAMAR, SC_NO_CALIFICA, SC_NO_DESEA

### Dashboards por grupo (G10, G11, G12)
- Solicitudes en trámite por canal y status
- Ranking de certificados activos por ejecutivo y jefatura
- Días para cierre de asamblea

---

## Estructura del repositorio

```
Claude_SFC/
│
├── 01_Levantamiento/               # BRD y diccionarios de datos
│   ├── BRD_Fondos_Colectivos_v1.docx
│   ├── Diccionario_Campos_SANTO_v1.xlsx
│   └── assets/capturas/            # Capturas de sesiones con usuarios
│
├── 02_Arquitectura/                # Diagramas as-is y arquitectura destino
│   ├── As-Is_Data_Landscape.md
│   ├── Medallion_Target_Architecture.md
│   └── diagrams/
│
├── 03_Modelado/                    # STM, reglas de negocio, ERD
│   ├── Source_to_Target_Mapping_SANTO.xlsx
│   ├── Reglas_de_Negocio.md
│   └── ERD_Fondos_Colectivos.md
│
├── 04_Diccionarios/                # Diccionarios por tabla
│   ├── Diccionario_SANTO_Cotizaciones.xlsx
│   ├── Diccionario_SANTO_Solicitudes.xlsx
│   ├── Diccionario_SANTO_Certificados.xlsx
│   ├── Diccionario_BD_REFERIDOS2.xlsx
│   └── Diccionario_BD_Lista_Ejecutivos.xlsx
│
├── 05_Databricks/                  # Notebooks y scripts de transformación
│   ├── bronze/ingest_scripts/
│   ├── silver/transform_scripts/
│   └── gold/aggregations/
│
├── 06_Referencia/                  # Archivos fuente y plantillas
│   ├── Excel_Source/               # 20260228_SFC_Dashboard.xlsx
│   └── Templates/
│
└── README.md
```

---

## Entregables completados

| # | Entregable | Archivo | Estado |
|---|---|---|---|
| 1 | Diagrama as-is de fuentes | (widget interactivo en sesión) | ✅ |
| 2 | Inventario de campos con categorías | (widget interactivo en sesión) | ✅ |
| 3 | Diccionario de datos SANTO | `04_Diccionarios/Diccionario_Campos_SANTO_v1.xlsx` | ✅ |
| 4 | BRD — Documento de Requerimientos de Negocio | `01_Levantamiento/BRD_Fondos_Colectivos_v1.docx` | ✅ borrador |
| 5 | Análisis completo del libro Excel SFC Dashboard | (documentado en esta sesión) | ✅ |
| 6 | Mapa interactivo de las 24 hojas del libro Excel | (widget interactivo en sesión) | ✅ |

---

## Pendientes críticos

| # | Acción | Responsable | Prioridad |
|---|---|---|---|
| P-01 | Negociar acceso automatizado a SANTO con Softek | TI SCP + Softek | Crítica |
| P-02 | Sesión de levantamiento tabla BD Lista Ejecutivos (estructura completa) | Equipo + Softek | Alta |
| P-03 | Documentar mapeo Estado → Estado2/3/4 con negocio | Equipo + Usuarios | Alta |
| P-04 | Sesión de levantamiento fuente REFERIDOS completa | Equipo + Usuarios | Alta |
| P-05 | Confirmar herramienta BI destino (Power BI u otra) | TI SCP | Alta |
| P-06 | Elaborar Source-to-Target Mapping (STM) campo a campo | Equipo | Alta |
| P-07 | Levantar tablas TD, GUIA_CAMP y STATUS de SANTO | Equipo + Softek | Media |
| P-08 | Diseñar modelo lógico ERD en Databricks | Equipo | Media |
| P-09 | Elaborar Project Charter formal con cronograma y RACI | PM | Alta |

---

## Glosario rápido

| Término | Definición |
|---|---|
| Certificado | Contrato de fondo colectivo vehicular suscrito por un asociado |
| Asociado | Cliente que ha contratado un certificado |
| Prospecto | Persona con interés en el producto, aún sin contratar |
| FRENTE | Campo que clasifica el origen comercial (REFERIDO SCP, BD ASTARA, CARTERA PROPIA, etc.) |
| TIPO | Clasificación REAL / PRUEBA de cada registro |
| BD_REFERIDOS2 | Hoja unificada de Agenda Comercial Digital + Cluster |
| AGENDA | Campo en BD_REFERIDOS2 que distingue entre Digital y Cluster |
| SCP | Santander Consumer Perú |
| SANTO | Sistema core de Fondos Colectivos (Softek, on-premise) |
| Bronze / Silver / Gold | Capas de la arquitectura Medallion en Databricks |
| STM | Source-to-Target Mapping — plantilla técnica de migración campo a campo |
| CIA | Comisión de Incentivo Anual — porcentaje variable según campaña |
| WC | Welcome Call — llamada de bienvenida al asociado (estado 38 del flujo) |

---

## Stack tecnológico objetivo

- **Ingesta y transformación:** Databricks (Delta Live Tables o notebooks PySpark)
- **Almacenamiento:** Delta Lake sobre Azure Data Lake Storage Gen2
- **Gobernanza:** Unity Catalog (Databricks)
- **Orquestación:** Databricks Workflows o Azure Data Factory
- **Visualización:** Power BI (por confirmar) conectado a capa Gold
- **Control de versiones:** GitHub (`carloshac7/Claude_SFC`)
- **Base de datos adicional (fase 2):** Cosmos DB / AWS (en evaluación)

---

## Cómo contribuir a este repositorio

1. Clonar: `git clone https://github.com/carloshac7/Claude_SFC.git`
2. Crear rama: `git checkout -b feature/nombre-entregable`
3. Agregar archivos en la carpeta correspondiente
4. Commit con mensaje descriptivo: `git commit -m "feat: agrega STM Cotizaciones v1"`
5. Push y Pull Request hacia `main`

### Convención de commits

```
feat:     nuevo entregable o funcionalidad
docs:     actualización de documentación
fix:      corrección de error en documento o script
data:     actualización de datos o diccionario
refactor: reorganización sin cambio de contenido
```

---

*Proyecto iniciado en Abril 2026. Desarrollado con apoyo de Claude (Anthropic) como asistente de análisis y documentación.*
