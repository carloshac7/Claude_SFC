# Source-to-Target Mapping (STM)
## Módulo: DASHBOARD VENTAS — Capa Gold

| Campo | Valor |
|---|---|
| Documento | STM Gold — Dashboard Ventas (Certificados) |
| Proyecto | Migración Fondos Colectivos — Banco Santander Consumer Perú |
| Fuentes origen | `silver.certificados`, `silver.ejecutivos` |
| Destino | `gold.*` (4 tablas — Unity Catalog: `scp_fondos.gold`) |
| Gráficos habilitados | Gráfico 5, 7, 44, 89, 131 (DASHBOARD) |
| Fecha | Abril 2026 |
| Versión | 1.0 |
| Estado | BORRADOR |

---

## 1. Diagrama de flujo Silver → Gold

```
silver.certificados (TIPO=REAL, CERTIFICADO_VALIDO=SI)
  │
  ├─ JOIN silver.ejecutivos ON usuario_actual = cod_ejecutivo
  │         (resuelve EJECUTIVO, JEFATURA, CANAL — RN-003)
  │
  ├─ GROUP BY periodo, mes, año
  │   └─► gold.resumen_certificados        [Gráfico 89, 7]
  │
  ├─ GROUP BY periodo, canal
  │   └─► gold.ventas_por_canal            [Gráfico 5]
  │
  ├─ GROUP BY periodo, ejecutivo, jefatura
  │   └─► gold.ventas_por_ejecutivo        [Gráfico 44]
  │
  └─ LAG(ventas, 1) OVER (ORDER BY periodo)
      └─► gold.ventas_comparativo          [Gráfico 131]
```

---

## 2. Tabla destino: `gold.resumen_certificados`

**Propósito:** Resumen mensual de ventas y estado de cartera. Replica la tabla manual `TD!$A$267:$N$281` del Excel.  
**Frecuencia de actualización:** Mensual (primer día hábil del mes siguiente)  
**Gráficos habilitados:** Gráfico 89 (barras mensuales), Gráfico 7 (línea tendencia)

### 2.1 Definición de campos

| # | Campo destino | Tipo | Campo(s) origen | Tabla origen | Transformación / Regla |
|---|---|---|---|---|---|
| 1 | `periodo` | STRING(6) | `PERIODO` | `silver.certificados` | YYYYMM — ya calculado en Silver |
| 2 | `año` | INTEGER | `AÑO_FEC_ORIG` | `silver.certificados` | `YEAR(fecha_contrato_norm)` |
| 3 | `mes` | INTEGER | `MES_FEC_ORIG` | `silver.certificados` | `MONTH(fecha_contrato_norm)` |
| 4 | `mes_nombre` | STRING | calculado | — | `date_format(fecha_contrato_norm, 'MMMM')` |
| 5 | `ventas` | INTEGER | COUNT(*) | `silver.certificados` | COUNT donde `es_real=true AND certificado_valido=true AND situacion_contrato != 'Retirado'` |
| 6 | `certif_activos` | INTEGER | `Situación del Contrato` | `silver.certificados` | COUNT donde `situacion_contrato = 'Activo' AND es_real=true` |
| 7 | `retiros_total` | INTEGER | `Situación del Contrato` | `silver.certificados` | COUNT donde `situacion_contrato = 'Retirado' AND es_real=true` |
| 8 | `retiro_a_solicitud_asociado` | INTEGER | `Modalidad Resolución` | `silver.certificados` | COUNT donde `modalidad_resolucion = 'Retiro a Solicitud del Asociado'` |
| 9 | `resol_x_deuda` | INTEGER | `Modalidad Resolución` | `silver.certificados` | COUNT donde `modalidad_resolucion LIKE '%Deuda%' OR modalidad_resolucion LIKE '%Incumplimiento%'` |
| 10 | `cupos_libres` | INTEGER | calculado | — | `retiros_total` (cada retiro libera un cupo reasignable — RN-010 pendiente confirmar) |
| 11 | `monto_total_ventas` | DECIMAL(18,2) | `Monto del Certificado` | `silver.certificados` | SUM donde `es_real=true AND certificado_valido=true` |
| 12 | `fecha_carga` | TIMESTAMP | — | — | `current_timestamp()` — metadata de ingesta |
| 13 | `periodo_origen` | STRING | — | — | Periodo YYYYMM del archivo fuente original |

### 2.2 Filtros aplicados (WHERE clause en Silver)

```sql
WHERE es_real = true                          -- RN-001: excluye PRUEBA
  AND certificado_valido = true               -- RN-006: solo certificados válidos
  AND codigo_programa = 'SAN001'              -- Fondo Tradicional únicamente
```

### 2.3 DDL Delta (Unity Catalog)

```sql
CREATE OR REPLACE TABLE scp_fondos.gold.resumen_certificados (
    periodo                      STRING        NOT NULL COMMENT 'Período YYYYMM',
    año                          INT           NOT NULL,
    mes                          INT           NOT NULL,
    mes_nombre                   STRING,
    ventas                       INT           COMMENT 'Certificados nuevos contratados en el periodo',
    certif_activos               INT           COMMENT 'Certificados activos al cierre del periodo',
    retiros_total                INT           COMMENT 'Total de retiros en el periodo',
    retiro_a_solicitud_asociado  INT           COMMENT 'Retiros por solicitud del propio asociado',
    resol_x_deuda                INT           COMMENT 'Resoluciones por incumplimiento de pago',
    cupos_libres                 INT           COMMENT 'Cupos liberados por retiro — potencialmente reasignables (RN-010)',
    monto_total_ventas           DECIMAL(18,2) COMMENT 'Suma de montos de certificados vendidos',
    fecha_carga                  TIMESTAMP     COMMENT 'Timestamp de la última actualización de la tabla',
    periodo_origen               STRING        COMMENT 'Periodo del archivo fuente que generó la carga'
)
USING DELTA
PARTITIONED BY (año)
COMMENT 'Resumen mensual de ventas y estado de cartera de Fondos Colectivos — replica lógica de hoja TD rows 267-281'
TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.autoOptimize.autoCompact'   = 'true'
);
```

---

## 3. Tabla destino: `gold.ventas_por_canal`

**Propósito:** Ventas agregadas por canal comercial y período. Habilita el análisis de mix de canal.  
**Gráficos habilitados:** Gráfico 5 (torta/barras por canal)

### 3.1 Definición de campos

| # | Campo destino | Tipo | Campo(s) origen | Tabla origen | Transformación / Regla |
|---|---|---|---|---|---|
| 1 | `periodo` | STRING(6) | `PERIODO` | `silver.certificados` | YYYYMM |
| 2 | `año` | INTEGER | `AÑO_FEC_ORIG` | `silver.certificados` | `YEAR(fecha_contrato_norm)` |
| 3 | `mes` | INTEGER | `MES_FEC_ORIG` | `silver.certificados` | `MONTH(fecha_contrato_norm)` |
| 4 | `canal` | STRING | `CANAL` | `silver.certificados` | Resuelto por RN-003 — valores: SCP DIRECTO, CLUSTER, DIGITAL, ALIANZA |
| 5 | `ventas` | INTEGER | COUNT(*) | `silver.certificados` | COUNT certificados válidos por canal |
| 6 | `pct_canal` | DECIMAL(5,2) | calculado | — | `ventas / SUM(ventas) OVER (PARTITION BY periodo) * 100` |
| 7 | `monto_ventas` | DECIMAL(18,2) | `Monto del Certificado` | `silver.certificados` | SUM por canal y período |
| 8 | `fecha_carga` | TIMESTAMP | — | — | `current_timestamp()` |

### 3.2 Filtros aplicados

```sql
WHERE es_real = true
  AND certificado_valido = true
  AND codigo_programa = 'SAN001'
  AND canal IS NOT NULL
```

### 3.3 DDL Delta

```sql
CREATE OR REPLACE TABLE scp_fondos.gold.ventas_por_canal (
    periodo         STRING        NOT NULL,
    año             INT           NOT NULL,
    mes             INT           NOT NULL,
    canal           STRING        NOT NULL COMMENT 'Canal comercial — RN-003',
    ventas          INT,
    pct_canal       DECIMAL(5,2)  COMMENT '% del total de ventas del periodo correspondiente a este canal',
    monto_ventas    DECIMAL(18,2),
    fecha_carga     TIMESTAMP
)
USING DELTA
PARTITIONED BY (año)
COMMENT 'Ventas de Fondos Colectivos agregadas por canal y periodo — habilita Gráfico 5 del DASHBOARD';
```

---

## 4. Tabla destino: `gold.ventas_comparativo`

**Propósito:** Comparativo mes actual vs mes anterior con variación absoluta y porcentual. Replica la lógica del Gráfico 131 que usa fórmulas Excel de resta entre filas adyacentes.  
**Gráficos habilitados:** Gráfico 131

### 4.1 Definición de campos

| # | Campo destino | Tipo | Campo(s) origen | Tabla origen | Transformación / Regla |
|---|---|---|---|---|---|
| 1 | `periodo` | STRING(6) | `periodo` | `gold.resumen_certificados` | YYYYMM actual |
| 2 | `año` | INTEGER | `año` | `gold.resumen_certificados` | — |
| 3 | `mes` | INTEGER | `mes` | `gold.resumen_certificados` | — |
| 4 | `ventas` | INTEGER | `ventas` | `gold.resumen_certificados` | Ventas del período actual |
| 5 | `ventas_prev` | INTEGER | LAG | `gold.resumen_certificados` | `LAG(ventas, 1) OVER (ORDER BY periodo)` |
| 6 | `variacion_abs` | INTEGER | calculado | — | `ventas - ventas_prev` |
| 7 | `variacion_pct` | DECIMAL(5,1) | calculado | — | `ROUND((ventas - ventas_prev) / ventas_prev * 100, 1)` |
| 8 | `certif_activos` | INTEGER | `certif_activos` | `gold.resumen_certificados` | Stock actual |
| 9 | `certif_activos_prev` | INTEGER | LAG | `gold.resumen_certificados` | `LAG(certif_activos, 1) OVER (ORDER BY periodo)` |
| 10 | `var_activos_abs` | INTEGER | calculado | — | `certif_activos - certif_activos_prev` |
| 11 | `retiros_total` | INTEGER | `retiros_total` | `gold.resumen_certificados` | Retiros del período |
| 12 | `retiros_prev` | INTEGER | LAG | `gold.resumen_certificados` | `LAG(retiros_total, 1) OVER (ORDER BY periodo)` |
| 13 | `fecha_carga` | TIMESTAMP | — | — | `current_timestamp()` |

### 4.2 PySpark — implementación completa

```python
from pyspark.sql import Window
import pyspark.sql.functions as F

# Leer desde Gold base
df_base = spark.table("scp_fondos.gold.resumen_certificados")

# Window para LAG — ordenado por periodo (YYYYMM ordena correctamente como string)
w = Window.orderBy("periodo")

df_comp = (
    df_base
    .withColumn("ventas_prev",        F.lag("ventas", 1).over(w))
    .withColumn("certif_activos_prev", F.lag("certif_activos", 1).over(w))
    .withColumn("retiros_prev",        F.lag("retiros_total", 1).over(w))
    .withColumn("variacion_abs",  F.col("ventas") - F.col("ventas_prev"))
    .withColumn("variacion_pct",
        F.round(
            (F.col("ventas") - F.col("ventas_prev")) / F.col("ventas_prev") * 100,
            1
        )
    )
    .withColumn("var_activos_abs", F.col("certif_activos") - F.col("certif_activos_prev"))
    .withColumn("fecha_carga", F.current_timestamp())
    .select(
        "periodo", "año", "mes",
        "ventas", "ventas_prev", "variacion_abs", "variacion_pct",
        "certif_activos", "certif_activos_prev", "var_activos_abs",
        "retiros_total", "retiros_prev",
        "fecha_carga"
    )
)

df_comp.write.format("delta").mode("overwrite").saveAsTable("scp_fondos.gold.ventas_comparativo")
```

### 4.3 DDL Delta

```sql
CREATE OR REPLACE TABLE scp_fondos.gold.ventas_comparativo (
    periodo               STRING        NOT NULL,
    año                   INT,
    mes                   INT,
    ventas                INT,
    ventas_prev           INT           COMMENT 'Ventas del período anterior',
    variacion_abs         INT           COMMENT 'Diferencia absoluta ventas actual - anterior',
    variacion_pct         DECIMAL(5,1)  COMMENT '% de variación vs período anterior',
    certif_activos        INT,
    certif_activos_prev   INT,
    var_activos_abs       INT,
    retiros_total         INT,
    retiros_prev          INT,
    fecha_carga           TIMESTAMP
)
USING DELTA
COMMENT 'Comparativo mensual de ventas — habilita Gráfico 131 del DASHBOARD (replica lógica LAG de fórmulas Excel)';
```

---

## 5. Tabla destino: `gold.ventas_por_ejecutivo`

**Propósito:** Ranking de ventas por ejecutivo, jefatura y período. Replica el pivot del Gráfico 44.  
**Gráficos habilitados:** Gráfico 44

### 5.1 Definición de campos

| # | Campo destino | Tipo | Campo(s) origen | Tabla origen | Transformación / Regla |
|---|---|---|---|---|---|
| 1 | `periodo` | STRING(6) | `PERIODO` | `silver.certificados` | YYYYMM |
| 2 | `año` | INTEGER | `AÑO_FEC_ORIG` | `silver.certificados` | — |
| 3 | `mes` | INTEGER | `MES_FEC_ORIG` | `silver.certificados` | — |
| 4 | `canal` | STRING | `CANAL` | `silver.certificados` | RN-003 |
| 5 | `usuario_ejecutivo` | STRING | `USUARIO_EJECUTIVO` | `silver.certificados` | Código de usuario SANTO |
| 6 | `ejecutivo` | STRING | `EJECUTIVO` | `silver.certificados` | Nombre resuelto — join RN-003 |
| 7 | `usuario_jefatura` | STRING | `USUARIO_JEFATURA` | `silver.certificados` | — |
| 8 | `jefatura` | STRING | `JEFATURA` | `silver.certificados` | Nombre jefatura — RN-003 |
| 9 | `coach` | STRING | `COACH` | `silver.certificados` | Coach asignado — RN-003 |
| 10 | `ventas` | INTEGER | COUNT(*) | `silver.certificados` | Certificados vendidos por ejecutivo en el período |
| 11 | `monto_ventas` | DECIMAL(18,2) | `Monto del Certificado` | `silver.certificados` | SUM por ejecutivo |
| 12 | `ranking_canal` | INTEGER | calculado | — | `RANK() OVER (PARTITION BY periodo, canal ORDER BY ventas DESC)` |
| 13 | `ranking_global` | INTEGER | calculado | — | `RANK() OVER (PARTITION BY periodo ORDER BY ventas DESC)` |
| 14 | `fecha_carga` | TIMESTAMP | — | — | `current_timestamp()` |

### 5.2 DDL Delta

```sql
CREATE OR REPLACE TABLE scp_fondos.gold.ventas_por_ejecutivo (
    periodo         STRING        NOT NULL,
    año             INT,
    mes             INT,
    canal           STRING        COMMENT 'Canal comercial — RN-003',
    usuario_ejecutivo STRING,
    ejecutivo       STRING,
    usuario_jefatura STRING,
    jefatura        STRING,
    coach           STRING,
    ventas          INT,
    monto_ventas    DECIMAL(18,2),
    ranking_canal   INT           COMMENT 'Ranking dentro del canal para el periodo',
    ranking_global  INT           COMMENT 'Ranking global para el periodo',
    fecha_carga     TIMESTAMP
)
USING DELTA
PARTITIONED BY (año)
COMMENT 'Ventas por ejecutivo y período — habilita Gráfico 44 y rankings del DASHBOARD';
```

---

## 6. Dependencias entre tablas Silver

```
bronze.certificados_raw
  └─ silver.certificados          ← transformaciones RN-001 a RN-006
        │   campos clave: es_real, certificado_valido, periodo,
        │                 canal, ejecutivo, jefatura, coach,
        │                 situacion_contrato, modalidad_resolucion,
        │                 monto_certificado, fecha_contrato_norm
        │
bronze.ejecutivos_raw
  └─ silver.ejecutivos            ← dimensión maestro RN-003
        │   campos clave: cod_ejecutivo, nombre_ejecutivo,
        │                 jefatura, coach, canal
        │
        └─ gold.resumen_certificados
        └─ gold.ventas_por_canal
        └─ gold.ventas_por_ejecutivo
              └─ gold.ventas_comparativo    ← consume gold.resumen_certificados
```

---

## 7. Reglas de negocio aplicadas — referencia cruzada

| RN | Nombre | Tabla(s) Gold afectada | Capa de aplicación |
|---|---|---|---|
| RN-001 | Clasificación TIPO (REAL/PRUEBA) | Todas | Silver — campo `es_real BOOLEAN` |
| RN-003 | Resolución jerarquía ejecutivos | `ventas_por_canal`, `ventas_por_ejecutivo` | Silver — JOIN con `silver.ejecutivos` |
| RN-006 | Certificado válido y comisión CIA | `resumen_certificados`, `ventas_por_canal` | Silver — campo `certificado_valido BOOLEAN` |
| RN-010 | Cupos Libres — reasignación por retiro | `resumen_certificados` | Gold — **pendiente confirmar lógica exacta con negocio** |

---

## 8. Matriz de trazabilidad completa

| Gráfico DASHBOARD | Tabla Gold | Pivote | Métrica | RNs |
|---|---|---|---|---|
| Gráfico 89 | `gold.resumen_certificados` | `periodo` | `ventas` | RN-001, RN-006 |
| Gráfico 7 | `gold.resumen_certificados` | `periodo` | `ventas` | RN-001, RN-006 |
| Gráfico 131 | `gold.ventas_comparativo` | `periodo` | `variacion_abs`, `variacion_pct` | RN-001, RN-006 |
| Gráfico 5 | `gold.ventas_por_canal` | `canal` | `ventas`, `pct_canal` | RN-001, RN-003, RN-006 |
| Gráfico 44 | `gold.ventas_por_ejecutivo` | `ejecutivo` | `ventas`, `ranking_canal` | RN-001, RN-003, RN-006 |

---

## 9. Validaciones de calidad (DQ)

| Check | Expresión | Acción si falla |
|---|---|---|
| No nulos en `periodo` | `periodo IS NOT NULL` | Rechazar registros — log en tabla `dq.certificados_rejected` |
| `ventas >= 0` | `ventas >= 0` | Alerta crítica — parar pipeline |
| `certif_activos >= ventas` acumulado | No directamente verificable en Gold | Verificar en Silver con COUNT DISTINCT |
| `variacion_pct` no infinito | `ventas_prev IS NULL OR ventas_prev != 0` | Tratar como NULL cuando prev = 0 |
| Total `ventas_por_canal.ventas` == `resumen_certificados.ventas` por periodo | Reconciliación cruzada | Alerta — posible doble conteo o filtro inconsistente |

---

## 10. Pendientes críticos del STM

| # | Pendiente | Prioridad |
|---|---|---|
| P-STM-01 | Confirmar columnas exactas de TD rows 267–281 en Excel (re-verificar con Carlos) | Alta |
| P-STM-02 | Definir lógica exacta de `cupos_libres` — ¿es siempre = `retiros_total`? (RN-010) | Alta |
| P-STM-03 | Confirmar valores del dominio `modalidad_resolucion` para clasificar `retiro_a_solicitud_asociado` vs `resol_x_deuda` | Alta |
| P-STM-04 | Validar que `codigo_programa = 'SAN001'` es el único filtro por programa o si hay múltiples | Media |
| P-STM-05 | Confirmar frecuencia de actualización: ¿mensual o diaria? (impacta diseño del pipeline Databricks Workflows) | Media |
| P-STM-06 | Definir esquema Unity Catalog: `scp_fondos.gold.*` — requiere permisos y aprobación TI | Media |
