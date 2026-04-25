# Análisis Técnico — Hoja DASHBOARD
## Archivo: `20260228_SFC_Dashboard.xlsx`

| Campo | Valor |
|---|---|
| Documento | Análisis DASHBOARD — Trazabilidad de Gráficos y Tablas Dinámicas |
| Fuente analizada | `06_Referencia/Excel_Source/20260228_SFC_Dashboard.xlsx` → hoja DASHBOARD |
| Fecha análisis | Abril 2026 |
| Versión | 1.0 |
| Autor | Equipo de Datos — Proyecto Migración |
| Clasificación | CONFIDENCIAL |

---

## 1. Resumen ejecutivo

La hoja **DASHBOARD** actúa como la capa de visualización del libro Excel. Contiene **10 objetos gráficos** y **11 tablas dinámicas** que consumen datos de dos fuentes principales dentro del mismo libro:

| Fuente interna | Hoja TD | Contenido |
|---|---|---|
| Ventas / Certificados | `TD` | Datos provenientes de SANTO — tabla CERTIFICADOS (filas 267–281 = tabla resumen mensual) |
| Referidos / Prospectos | `TD_REFERIDOS2` | Datos de BD_REFERIDOS2 + Agendas Comerciales |

**Criterio de exclusión aplicado:** Los 5 gráficos vinculados a `TD_REFERIDOS2` quedan fuera del alcance de este análisis (corresponden al módulo Referidos / Prospectos).

**Hallazgo clave:** Todos los gráficos de Ventas aplican el filtro `TIPO = "REAL"` (RN-001) a nivel de tabla dinámica. El campo `CANAL` (RN-003) es el pivote universal de segmentación comercial.

---

## 2. Inventario completo de gráficos — DASHBOARD

| # | Gráfico ID | Fuente pivot | Módulo | Estado en análisis |
|---|---|---|---|---|
| 1 | Gráfico 131 | `TD` (tabla manual comparativo) | VENTAS | **Analizado** |
| 2 | Gráfico 5 | `TD` | VENTAS | **Analizado** |
| 3 | Gráfico 7 | `TD` | VENTAS | **Analizado** |
| 4 | Gráfico 44 | `TD` | VENTAS | **Analizado** |
| 5 | Gráfico 89 | `TD` (filas 267–281, col. M = VENTAS) | VENTAS | **Analizado** |
| 6 | Gráfico 28 | `TD_REFERIDOS2` | REFERIDOS | Excluido |
| 7 | Gráfico 29 | `TD_REFERIDOS2` | REFERIDOS | Excluido |
| 8 | Gráfico 45 | `TD_REFERIDOS2` | REFERIDOS | Excluido |
| 9 | Gráfico 60 | `TD_REFERIDOS2` | REFERIDOS | Excluido |
| 10 | Gráfico 108 | `TD_REFERIDOS2` | REFERIDOS | Excluido |

> **Método de trazabilidad:** Extracción del OOXML interno del libro Excel vía `drawing1.xml` → `drawing1.xml.rels` → `chartN.xml` → campo `<c:pivotSource>` → `pivotTableN.xml` → `pivotCacheDefinitionN.xml`

---

## 3. Análisis detallado — Gráficos VENTAS

### 3.1 Gráfico 89 — Resumen Mensual de Ventas

**Tipo:** Gráfico de barras / columnas agrupadas  
**Fuente de datos:** Tabla de resumen `TD!$A$267:$N$281` — tabla calculada por fórmulas (no es tabla dinámica)  
**Columna clave:** Columna M = `VENTAS` (conteo de certificados vendidos por mes)

**Campos identificados en la tabla origen (filas 267–281):**

| Columna | Campo | Descripción |
|---|---|---|
| A | PERIODO / MES | Mes de referencia |
| M | VENTAS | Conteo de ventas del período |
| — | CERTIF ACTIVOS | Total de certificados activos en el período |
| — | CUPOS LIBRES | Cupos liberados por retiro — potencialmente reasignables (ver RN-010) |
| — | RETIRO A SOLICITUD ASOCIADO | Retiros iniciados por el propio asociado |
| — | RESOL x DEUDA | Resoluciones de contrato por incumplimiento de pago |

**Reglas de negocio aplicadas implícitamente en las fórmulas:**
- `TIPO = "REAL"` (RN-001) — excluye registros de prueba
- `CERTIFICADO VALIDO? = "SI"` (RN-006) — excluye certificados inválidos
- Campo `Situación del Contrato` discrimina entre ACTIVOS, RETIRADOS y RESUELTOS

---

### 3.2 Gráfico 131 — Comparativo de Períodos

**Tipo:** Gráfico de barras comparativas (período actual vs período anterior)  
**Fuente de datos:** Tabla manual de comparativo — calcula diferencia entre filas adyacentes de la tabla TD (filas 267–281)  
**Lógica Excel:** `= VALOR_MES_N - VALOR_MES_(N-1)` — referencia directa a dos filas del resumen mensual

**Campos que compara:**
- Ventas período actual vs período anterior
- Variación absoluta y porcentual
- Certificados activos actual vs anterior

**Observación:** Esta lógica de resta entre filas consecutivas es la representación en Excel de una comparación temporal. Requiere que la tabla TD rows 267–281 esté ordenada cronológicamente por PERIODO.

---

### 3.3 Gráfico 5 — Ventas por Canal

**Tipo:** Gráfico de torta o barras apiladas  
**Fuente de datos:** Tabla dinámica en DASHBOARD → pivot cache → hoja `TD` → campo `CANAL`  
**Filtros de tabla dinámica:** `TIPO = "REAL"` (filtro de informe)  
**Dimensión de análisis:** Campo `CANAL` (RN-003)

**Valores de CANAL identificados:**
- SCP DIRECTO
- CLUSTER
- DIGITAL
- ALIANZA (concesionarios: Autoland, Astara, Derco, Interamericana, Revo Motors)

**Métrica:** Conteo de certificados agrupados por CANAL y PERIODO

---

### 3.4 Gráfico 7 — Evolución Mensual (Tendencia)

**Tipo:** Gráfico de líneas  
**Fuente de datos:** Tabla dinámica → hoja `TD` → dimensión temporal PERIODO / MES  
**Filtros aplicados:** `TIPO = "REAL"`, filtro por AÑO activo  
**Dimensión de análisis:** PERIODO (YYYYMM) en eje X — conteo de certificados en eje Y  

**Uso de negocio:** Visualiza la tendencia mensual acumulada y permite identificar estacionalidad y comparación contra períodos anteriores.

---

### 3.5 Gráfico 44 — Ranking por Ejecutivo / Grupo

**Tipo:** Gráfico de barras horizontales  
**Fuente de datos:** Tabla dinámica → hoja `TD` → campo `EJECUTIVO` o `CODIGO_GRUPO`  
**Filtros aplicados:** `TIPO = "REAL"`, selección de CANAL  
**Dimensión de análisis:** Ejecutivo o Jefatura con conteo de ventas

**Observación:** El nombre del ejecutivo en este gráfico es el resultado de aplicar la regla de resolución de jerarquía (RN-003), que traduce el código `USUARIO_ACTUAL` de SANTO al nombre real del ejecutivo usando la tabla `BD Lista Ejecutivos`.

---

## 4. Inventario de tablas dinámicas — DASHBOARD

Total identificadas: **11 tablas dinámicas** en la hoja DASHBOARD.

| # | Pivot Source | Datos origen | Campo Fila | Campo Columna | Métrica | Gráfico vinculado |
|---|---|---|---|---|---|---|
| 1 | TD | CERTIFICADOS (SANTO) | CANAL | PERIODO | COUNT | Gráfico 5 |
| 2 | TD | CERTIFICADOS (SANTO) | PERIODO | — | COUNT | Gráfico 7 |
| 3 | TD | CERTIFICADOS (SANTO) | EJECUTIVO | PERIODO | COUNT | Gráfico 44 |
| 4 | TD (fórmulas) | Tabla resumen filas 267–281 | PERIODO | — | Col. M: VENTAS | Gráfico 89 |
| 5 | TD (fórmulas) | Tabla comparativo | PERIODO | — | Variación vs período prev. | Gráfico 131 |
| 6–11 | TD_REFERIDOS2 | BD_REFERIDOS2 + Agendas | Varios | Varios | COUNT | Gráficos excluidos (módulo Referidos) |

> **Nota:** Las filas 6–11 corresponden a los 5 gráficos de REFERIDOS excluidos del análisis. El conteo de 11 puede incluir tablas dinámicas adicionales sin gráfico vinculado que alimentan KPI cards del dashboard.

---

## 5. Hallazgos clave

| # | Hallazgo | Implicación |
|---|---|---|
| H-01 | `TIPO = "REAL"` es filtro universal en todos los pivots de VENTAS | Es una regla de negocio implícita (RN-001) que debe documentarse como requisito de cualquier reporte de ventas |
| H-02 | `CANAL` es la dimensión de corte universal para ventas | Cualquier reporte de ventas requiere este campo resuelto; depende de RN-003 (jerarquía de ejecutivos) |
| H-03 | La tabla TD filas 267–281 es una tabla de fórmulas, no una tabla dinámica | Su lógica de cálculo está embebida en fórmulas Excel; es la fuente directa de los KPIs de ventas del dashboard |
| H-04 | Gráfico 131 usa resta entre filas consecutivas (comparativo temporal) | La tabla TD rows 267–281 debe estar ordenada cronológicamente; el comparativo depende del orden de filas |
| H-05 | `CUPOS LIBRES` representa cupos liberados por retiro, potencialmente reasignables | Regla de negocio no documentada formalmente — pendiente confirmar con el área (RN-010) |
| H-06 | `RESOL x DEUDA` y `RETIRO A SOLICITUD ASOCIADO` son subcategorías del campo `Modalidad Resolución` de SANTO | La clasificación de retiros en el dashboard depende de los valores exactos de este campo en el sistema fuente |
| H-07 | Los 5 gráficos de REFERIDOS usan `TD_REFERIDOS2` como pivot source | La hoja TD_REFERIDOS2 unifica Agenda Digital + Cluster; su análisis es independiente del módulo VENTAS |
| H-08 | Todos los gráficos del DASHBOARD están en una única capa de dibujo (`drawing1.xml`) | El DASHBOARD es una sola hoja con múltiples objetos superpuestos; no existen sub-hojas ni pestañas separadas por módulo |

---

## 6. Cadena de trazabilidad — Módulo VENTAS

```
SANTO (on-premise)
  └─ Tabla CERTIFICADOS → exportación manual CSV
        └─ Hoja CERTIFICADOS del libro Excel
              └─ Columnas calculadas: TIPO, CANAL, EJECUTIVO, JEFATURA, PERIODO…
                    └─ Hoja TD (consolidación y tablas dinámicas)
                          ├─ Filas 267–281 (tabla resumen mensual por fórmulas)
                          │     ├─ Col. M: VENTAS ──────────────── Gráfico 89
                          │     └─ Comparativo entre filas ──────── Gráfico 131
                          └─ Tablas dinámicas
                                ├─ Pivot por CANAL ─────────────── Gráfico 5
                                ├─ Pivot por PERIODO ────────────── Gráfico 7
                                └─ Pivot por EJECUTIVO ──────────── Gráfico 44
```

---

## 7. Pendientes de levantamiento

| # | Acción | Prioridad |
|---|---|---|
| P-01 | Confirmar columnas exactas de la tabla TD filas 267–281 (re-verificar en Excel) | Alta |
| P-02 | Documentar RN-010: lógica de CUPOS LIBRES y condiciones de reasignación | Alta |
| P-03 | Confirmar valores del dominio `Modalidad Resolución` en SANTO para clasificar subcategorías de retiro | Alta |
| P-04 | Analizar gráficos REFERIDOS del DASHBOARD (Gráfico 28, 29, 45, 60, 108) — módulo separado | Media |
| P-05 | Levantar KPI cards del DASHBOARD (valores numéricos sin gráfico) y sus fuentes | Media |
