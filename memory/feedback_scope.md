---
name: Alcance del proyecto — solo documentación, sin Databricks
description: El proyecto documenta fuentes y procesos as-is; NO incluye diseño ni referencias a implementación Databricks
type: feedback
---

No incluir nada relacionado con Databricks en los documentos ni en las respuestas: ni como destino de implementación, ni como idea, ni como recomendación, ni como código PySpark, ni como DDL Delta, ni como referencias a capas Medallion (Bronze/Silver/Gold), Unity Catalog, Databricks Workflows, etc.

**Why:** El proyecto está en fase de levantamiento y documentación pura. El alcance es documentar lo que existe: archivos Excel, sistema SANTO, Microsoft Lists, cómo se estructuran los datos y cómo son consumidos como tablas, reportes y dashboards. La plataforma destino no está en el alcance de lo que se documenta aquí.

**How to apply:** Cuando se analice cualquier fuente (Excel, SANTO, Microsoft Lists), producir únicamente documentación descriptiva: campos, reglas de negocio implícitas en fórmulas/pivots, flujos de consumo, trazabilidad de datos entre hojas y gráficos. No proponer ni mencionar ninguna implementación tecnológica futura.
