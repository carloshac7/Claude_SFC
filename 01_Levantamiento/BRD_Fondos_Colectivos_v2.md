# DOCUMENTO DE REQUERIMIENTOS DE NEGOCIO
## Business Requirements Document (BRD)

| Campo | Valor |
|---|---|
| Nombre del proyecto | Migración y Modelado de Datos — Fondos Colectivos |
| Empresa / Cliente | Banco Santander Consumer Perú (SCP) |
| Área solicitante | Fondos Colectivos |
| Gerente de proyecto | Por definir |
| Autor del documento | Equipo de Datos — Proyecto Migración |
| Fecha de elaboración | Abril 2026 |
| Versión | 2.0 — Actualización con inventario completo de campos |
| Estado | BORRADOR |
| Clasificación | CONFIDENCIAL |

### Historial de versiones

| Versión | Descripción | Fecha | Responsable |
|---|---|---|---|
| 1.0 | Borrador inicial — levantamiento de fuentes SANTO y Agenda Comercial | Abril 2026 | Equipo Migración |
| 2.0 | Actualización con inventario completo de campos extraído del libro `20260228_SFC_Dashboard.xlsx` — se documentan BD_REFERIDOS2, BD Lista Ejecutivos y tablas de referencia que estaban pendientes | Abril 2026 | Equipo Migración |

---

## 1. Resumen ejecutivo

El presente documento recoge el levantamiento de información realizado con los usuarios finales del área de Fondos Colectivos del Banco Santander Consumer Perú (SCP), en el marco de un proyecto de migración, orquestación y modelado de bases de datos en la nube.

El objetivo es entender el estado actual (as-is) de las fuentes de datos, los sistemas que las alimentan, los procesos de negocio que los consumen, los indicadores que se calculan y las reglas de negocio implícitas que hoy residen principalmente en archivos Excel. Este entendimiento es el insumo fundamental para diseñar la arquitectura destino sobre Databricks (capas Bronze, Silver y Gold) y garantizar que los indicadores y reportes actuales puedan reproducirse y mejorarse en la plataforma objetivo.

### Alcance de este documento

- Este BRD cubre únicamente el área de Fondos Colectivos (SCP) en su primera etapa de levantamiento.
- Las fuentes documentadas son: SANTO (sistema Softek) y Agenda Comercial (Microsoft Lists / SharePoint).
- La versión 2.0 incorpora el inventario completo de campos de todas las tablas del libro Excel `20260228_SFC_Dashboard.xlsx`.
- La arquitectura destino (Databricks, Cosmos DB, AWS) se define en un documento técnico separado.

---

## 2. Descripción del proyecto

### 2.1 Contexto del negocio

Fondos Colectivos es una unidad de negocio de Banco Santander Consumer Perú que comercializa planes de ahorro colectivo vehicular. En este modelo, un grupo de clientes (asociados) realiza aportes mensuales a un fondo común; periódicamente, mediante sorteo o remate, uno de ellos recibe el bien (vehículo) financiado por el grupo. El producto se llama **Fondo Tradicional (programa SAN001)**.

El negocio opera a través de múltiples canales de venta: ejecutivos SCP directos, clusters de agentes, canal digital (web/redes sociales), y alianzas con concesionarios (Autoland, Astara, Derco, Interamericana, Revo Motors, entre otros). Cada canal genera prospectos que pasan por un proceso de cotización → solicitud → evaluación crediticia → contratación (certificado).

### 2.2 Problema identificado

La gestión y análisis de datos del área se realiza actualmente de forma manual y fragmentada:

- Las fuentes de datos (SANTO, Agenda Comercial) son gestionadas por una empresa tercera y residen on-premise, lo que limita el acceso y control del equipo interno.
- Los usuarios finales descargan archivos CSV desde SANTO y los procesan manualmente en Excel, construyendo tablas dinámicas y calculando indicadores con fórmulas propias.
- La lógica de negocio (clasificación de registros, jerarquías de ejecutivos, indicadores de hitos, reglas de deduplicación) vive en columnas calculadas de Excel y no está documentada ni versionada.
- Los dashboards de seguimiento (Resumen Certificados, Total Referidos SCP) se alimentan de exportaciones manuales, lo que introduce riesgo de error y limita la frecuencia de actualización.
- No existe una capa de datos centralizada, integrada y gobernada que permita análisis histórico, cruce entre fuentes o escalabilidad.

### 2.3 Objetivo del proyecto

Diseñar e implementar una plataforma de datos en la nube (Databricks como motor principal) que:

1. Ingiera automáticamente los datos de las fuentes identificadas (SANTO, Agenda Comercial, Referidos).
2. Aplique las transformaciones y reglas de negocio actualmente en Excel en pipelines versionados y auditables.
3. Exponga capas de datos limpias (Silver) y agregadas (Gold) que soporten los dashboards y análisis actuales.
4. Siente las bases de gobernanza de datos (Unity Catalog, diccionario de datos, linaje) requeridas por el banco.

---

## 3. Alcance del proyecto

### 3.1 Dentro del alcance (IN-SCOPE)

- Levantamiento de fuentes: Inventario completo de tablas, campos y reglas de SANTO (Cotizaciones, Solicitudes, Certificados) y Agenda Comercial (Microsoft Lists).
- Diccionario de datos: Documento estructurado con definición funcional, tipo de dato, ejemplo y criticidad de cada campo.
- Documentación de reglas de negocio: Traducción a pseudocódigo de toda la lógica actualmente en columnas calculadas de Excel.
- Diseño de arquitectura destino: Modelo Medallion (Bronze / Silver / Gold) sobre Databricks para las fuentes identificadas.
- Pipeline de ingesta: Extracción automatizada desde las fuentes on-premise hacia la capa Bronze.
- Transformaciones Silver: Limpieza, deduplicación, estandarización de fechas, resolución de jerarquías de ejecutivos.
- Capa Gold para dashboards: Tablas agregadas que soporten los reportes Resumen Certificados y Total Referidos SCP.
- Registro en Unity Catalog: Gobernanza de tablas, permisos y linaje en Databricks.

### 3.2 Fuera del alcance (OUT-OF-SCOPE)

- Migración de datos históricos anteriores a 2024: Solo se migra data activa y del periodo vigente en primera fase.
- Modificación de sistemas fuente: SANTO y la infraestructura Softek no se alteran; solo se consume lo que exponen.
- Desarrollo de nuevos dashboards: Se replican los existentes; nuevas vistas son un proyecto posterior.
- Integración con Cosmos DB / AWS: Se evalúa en fase 2; fuera del alcance inicial.
- Datos de otras áreas del banco: Solo Fondos Colectivos SCP.

---

## 4. Motores del negocio

| Driver | Descripción |
|---|---|
| Acceso y control de datos | Las fuentes actuales son gestionadas por un tercero (Softek) y residen on-premise. El banco requiere soberanía sobre sus datos en una plataforma propia y gobernada. |
| Eficiencia operativa | El proceso manual de descarga de CSV y cálculo en Excel consume tiempo de los equipos comerciales y de TI, e introduce errores. La automatización libera ese tiempo para análisis de valor. |
| Escalabilidad | El volumen de certificados activos crece mensualmente. Excel no es escalable como plataforma de análisis; se necesita una capa de datos que soporte el crecimiento. |
| Calidad de datos | Las reglas de negocio en Excel no están documentadas ni versionadas. Existe riesgo de inconsistencias entre reportes. Una capa Silver centralizada garantiza una única fuente de verdad. |
| Cumplimiento y gobernanza | Los estándares internos del banco exigen trazabilidad, linaje y control de acceso a los datos. Unity Catalog en Databricks satisface este requerimiento. |
| Habilitación analítica | Una plataforma de datos moderna permite incorporar en el futuro modelos predictivos (propensión de pago, riesgo de retiro) que hoy no son posibles con la infraestructura actual. |

---

## 5. Proceso actual (As-Is)

### 5.1 Mapa de fuentes de datos

| Sistema | Tecnología | Tablas / Módulos | Extracción actual | Propietario |
|---|---|---|---|---|
| SANTO | Softek (on-premise) | Cotizaciones, Solicitudes, Certificados, BD_REFERIDOS2, BD Lista Ejecutivos, BD_Tipo, BD Alianzas, GUIA_STATUS, STATUS, BD Estado Solicitud, Dias_Utiles | Descarga manual CSV | Tercero (Softek) |
| Agenda Comercial Digital | Microsoft Lists (SharePoint Online) | Prospectos canal digital | Exportación manual CSV | Banco SCP (TI) |
| Agenda Comercial Cluster | Microsoft Lists (SharePoint Online) | Prospectos canal cluster / agentes | Exportación manual CSV | Banco SCP (TI) |

> **Nota v2.0:** BD_REFERIDOS2 unifica en el libro Excel las dos Agendas Comerciales (Digital y Cluster) usando el campo `AGENDA` para distinguir la fuente. Al 19/03/2026 suman **8,209 referidos totales**.

### 5.2 Flujo de datos actual

| # | Paso | Descripción |
|---|---|---|
| 1 | Extracción manual | El usuario accede a SANTO y descarga archivos CSV para cada tabla (Cotizaciones, Solicitudes, Certificados) bajo demanda, sin periodicidad definida. |
| 2 | Procesamiento en Excel | Los CSV se abren en Excel. El usuario aplica columnas calculadas con fórmulas para clasificar registros (TIPO REAL/PRUEBA), detectar duplicados (FILTRO, SE REPITE?), descomponer fechas (AÑO, MES, DIA, PERIODO), resolver jerarquías de ejecutivos (USUARIO → JEFATURA → COACH) y calcular hitos del proceso. |
| 3 | Tablas dinámicas | Con los datos procesados se construyen tablas dinámicas manuales para los reportes de gestión. |
| 4 | Dashboard Referidos | Desde Agenda Comercial se exporta la lista filtrada. Con estos datos se alimenta manualmente el dashboard "Total Referidos SCP". |
| 5 | Actualización ad-hoc | No existe automatización. Los reportes se actualizan cuando el usuario lo decide, lo que puede generar versiones desactualizadas o inconsistentes entre áreas. |

---

## 6. Inventario de sistemas y tablas

### 6.1 SANTO — Sistema Fondos Colectivos (Softek)

SANTO es el sistema core de Fondos Colectivos, administrado por la empresa tercera Softek.

---

#### Tabla: COTIZACIONES

Registro de cada cotización generada para un prospecto. Contiene datos del prospecto, canal de venta, punto de venta, estado y fecha.

**Campos fuente (15):**

| # | Campo | Descripción |
|---|---|---|
| 1 | Nro. de Cotización | Identificador único de la cotización |
| 2 | Canal Venta | Canal por el que se generó la cotización |
| 3 | Tipo Canal | Clasificación del canal (ej: Tipo Canal 01) |
| 4 | Punto Venta | Punto de venta o agencia |
| 5 | Situación Consumer | Estado en el sistema consumer |
| 6 | Tipo de Documento | DNI u otro tipo de documento del prospecto |
| 7 | Nro. de Documento | Número de documento de identidad |
| 8 | Nombre Completo | Nombre del prospecto |
| 9 | Nro. de Celular | Teléfono de contacto |
| 10 | Correo Electrónico | Email de contacto |
| 11 | Estado | Estado de la cotización en SANTO |
| 12 | Área Actual | Área funcional asignada actualmente |
| 13 | Usuario Actual | Código de usuario/ejecutivo en SANTO |
| 14 | Comentario | Notas del ejecutivo |
| 15 | Fecha | Fecha de la cotización |

**Campos calculados en Excel (10) → Silver en Databricks:**

| # | Campo | Lógica / Regla |
|---|---|---|
| 1 | FECHA2 | Normalización/conversión de fecha |
| 2 | FILTRO | Clave de deduplicación: CONCAT(Usuario_Actual, Nro_Documento) — RN-002 |
| 3 | SE REPITE? | "SI" si FILTRO aparece más de una vez en el periodo — RN-002 |
| 4 | TIPO | "REAL" o "PRUEBA" según clasificación — RN-001 |
| 5 | USUARIO | Nombre del ejecutivo resuelto desde BD Lista Ejecutivos — RN-003 |
| 6 | JEFATURA | Jefatura resuelta desde BD Lista Ejecutivos — RN-003 |
| 7 | CANAL | EJECUTIVO SCP / CANAL DIGITAL / CLUSTER / ADM — RN-003 |
| 8 | TIENE COTIZACIÓN | Flag: el prospecto tiene cotización registrada |
| 9 | TIENE SOLICITUD | Flag: el prospecto avanzó a solicitud |
| 10 | ALIANZA | Concesionario aliado si aplica, desde BD Alianzas |

---

#### Tabla: SOLICITUDES

Registro de la solicitud formal de crédito. Contiene el estado detallado del flujo de aprobación (38 estados), datos del solicitante, programa y grupo, y flags de hitos del proceso.

**Campos fuente (16):**

| # | Campo | Descripción |
|---|---|---|
| 1 | Id Solicitud | Identificador único de la solicitud en SANTO |
| 2 | Fecha de Solicitud | Fecha de ingreso de la solicitud |
| 3 | Canal Venta | Canal comercial de origen |
| 4 | Tipo Canal | Clasificación del canal |
| 5 | Punto Venta | Punto de venta o agencia |
| 6 | Situación Consumer | Estado en el sistema consumer |
| 7 | Estado | Estado del flujo en SANTO (38 valores posibles) |
| 8 | Área Actual | Área funcional asignada |
| 9 | Usuario Actual | Código de usuario/ejecutivo en SANTO |
| 10 | Tipo Persona | Natural o jurídica |
| 11 | Tipo de Documento | Tipo de documento del solicitante |
| 12 | Nro. de Documento | Número de documento de identidad |
| 13 | Nombre Asociado | Nombre del solicitante |
| 14 | Tipo de Programa | Tipo de programa (ej: SAN001) |
| 15 | Tipo de Grupo | Grupo al que pertenece la solicitud |
| 16 | Monto de Certificado | Monto comprometido en el certificado |

**Campos calculados en Excel (30) → Silver/Gold en Databricks:**

| # | Campo | Lógica / Regla |
|---|---|---|
| 1 | Id Solicitud2 | ID de solicitud normalizado |
| 2 | FECHA | Fecha normalizada |
| 3 | Estado2 | Estado con prefijo numérico de ordenamiento — RN-004 |
| 4 | Estado3 | Agrupación intermedia de gestión — RN-004 |
| 5 | Estado4 | Agrupación macro para tableros — RN-004 |
| 6 | TIPO | "REAL" o "PRUEBA" — RN-001 |
| 7 | USUARIO | Nombre ejecutivo resuelto — RN-003 |
| 8 | JEFATURA | Jefatura resuelta — RN-003 |
| 9 | SE REPITE? | Flag de duplicado — RN-002 |
| 10 | CANAL | Clasificación de canal — RN-003 |
| 11 | TIENE SOLICITUD | Flag: tiene solicitud registrada |
| 12 | ALIANZA | Concesionario aliado si aplica |
| 13 | NOMENCLATURA | Clave de dedup: CONCAT(Nro_Documento, AÑO) — RN-002 |
| 14 | DUPLICADO | Marca la solicitud como duplicada si NOMENCLATURA se repite |
| 15 | AÑO | Año extraído de Fecha de Solicitud |
| 16 | MES | Mes extraído |
| 17 | DIA | Día extraído |
| 18 | PERIODO | YYYYMM |
| 19 | VALIDACIÓN REPETICIÓN | Validación de lógica de deduplicación |
| 20 | PRE REGISTRO | Flag hito: solicitud en pre-registro — RN-005 |
| 21 | REGISTRO | Flag hito: solicitud registrada — RN-005 |
| 22 | EVALUADA | Flag hito: solicitud evaluada — RN-005 |
| 23 | APROBADA | Flag hito: solicitud aprobada — RN-005 |
| 24 | DOCS FIRMADOS | Flag hito: documentos firmados — RN-005 |
| 25 | LLAMADA CALIDAD OK | Flag hito: llamada de calidad superada — RN-005 |
| 26 | PAGO REALIZADO | Flag hito: primer pago realizado — RN-005 |
| 27 | COD_BUSQ | CONCAT(Nro_Documento, "_", Usuario_Actual) — clave de cruce con Certificados |
| 28 | EJECUTIVO QUE PARTICIPA | Ejecutivo resuelto final que participa en la venta |
| 29 | SEMANA | Semana del año |
| 30 | FLAG_AVANCE | 1 si la solicitud avanzó más allá del pre-registro |

---

#### Tabla: CERTIFICADOS

Contrato activo de fondo colectivo. Es la tabla más completa e incluye datos del contrato, estado, montos, canal, ejecutivo, fechas, adjudicación, retiro y comisiones.

**Campos fuente (33):**

| # | Campo | Descripción |
|---|---|---|
| 1 | Id Solicitud | ID de la solicitud que originó el certificado |
| 2 | Nro. Operación | Número de operación del certificado |
| 3 | Fecha Contrato | Fecha de firma del contrato |
| 4 | Modalidad de Ingreso | Forma en que ingresó el asociado |
| 5 | Canal Venta | Canal comercial de origen |
| 6 | Tipo Canal | Clasificación del canal |
| 7 | Punto Venta | Punto de venta o agencia |
| 8 | Estado Contrato | Estado del contrato en SANTO |
| 9 | Estado Actual | Estado actual en el flujo |
| 10 | Área Actual | Área funcional asignada |
| 11 | Usuario Actual | Código de usuario/ejecutivo en SANTO |
| 12 | Tipo Documento | Tipo de documento del asociado |
| 13 | Nro. de Documento | Número de documento de identidad |
| 14 | Nombre Asociado | Nombre del asociado |
| 15 | Código Programa | Código del programa (ej: SAN001) |
| 16 | Código Grupo | Código del grupo de fondo |
| 17 | Marca | Marca del vehículo |
| 18 | Dealer | Concesionario |
| 19 | Monto del Certificado | Monto del plan contratado |
| 20 | Monto Pagado | Monto acumulado pagado |
| 21 | Días de atraso | Días de mora |
| 22 | Moneda | Moneda del contrato (PEN/USD) |
| 23 | Situación del Contrato | Situación: Activo, Retirado, Resuelto |
| 24 | Modalidad Resolución | Motivo de resolución si aplica |
| 25 | Condición Evaluación Adjudicación | Condición para ser evaluado en adjudicación |
| 26 | Condición Adjudicación | Resultado de evaluación de adjudicación |
| 27 | Modalidad Adjudicación | Sorteo o Remate |
| 28 | Condición de Pago | Estado de pago del asociado |
| 29 | Forma Pago Devengado | Modalidad de pago |
| 30 | Código Contrato Asociado | Código del contrato vinculado |
| 31 | ¿Participó en su 1era Asamblea? | SI/NO |
| 32 | Cantidad Asambleas que ha participado | Número de asambleas |
| 33 | Adjudicado | SI/NO — indica si el asociado recibió el vehículo |

**Campos calculados en Excel (35) → Silver/Gold en Databricks:**

| # | Campo | Lógica / Regla |
|---|---|---|
| 1 | TIPO | "REAL" o "PRUEBA" — RN-001 |
| 2 | CERTIFICADO VALIDO? | SI/NO según criterios de validez — RN-006 |
| 3 | FECHA | Fecha normalizada |
| 4 | MES2 | Mes de la fecha de contrato |
| 5 | FECHA ORIG | Fecha original del sistema |
| 6 | AÑO_FEC_ORIG | Año de la fecha original |
| 7 | MES_ FEC_ORIG | Mes de la fecha original |
| 8 | PERIODO | YYYYMM |
| 9 | PERIODO AGRUPADO | Agrupación de periodos para reportes |
| 10 | MES ORIG | Mes original |
| 11 | DIA ORIG | Día original |
| 12 | MONTO CERTIF 2 | Monto normalizado |
| 13 | CANAL | Clasificación de canal — RN-003 |
| 14 | USUARIO_EJECUTIVO | Código de usuario del ejecutivo |
| 15 | EJECUTIVO | Nombre del ejecutivo resuelto — RN-003 |
| 16 | USUARIO_JEFATURA | Código de usuario de la jefatura |
| 17 | JEFATURA | Nombre de la jefatura — RN-003 |
| 18 | USUARIO_JEFATURA2 | Código de segunda jefatura |
| 19 | COACH | Nombre del coach — RN-003 |
| 20 | REFERIDOS SCP | SI/NO — si el certificado viene de un referido SCP |
| 21 | FRENTE | Origen comercial: REFERIDO SCP, BD ASTARA, CARTERA PROPIA, etc. — RN-007 |
| 22 | USUARIO_EJ. REFIERE | Código del ejecutivo que refirió |
| 23 | EJECUTIVO QUE REFIERE | Nombre del ejecutivo que refirió |
| 24 | USUARIO_JEFEVTA | Código del jefe de ventas SCP |
| 25 | JEFE VTA SCP | Nombre del jefe de ventas SCP |
| 26 | EJECUTIVO SCP Q PARTICIPA | Ejecutivo SCP final que participa en la venta |
| 27 | MOTIVO RETIRO | Motivo si el certificado fue retirado |
| 28 | RETIRO_COMENTARIO | Comentario de retiro |
| 29 | FECHA RETIRO | Fecha de retiro |
| 30 | PERIODO_RET | Periodo YYYYMM del retiro |
| 31 | % CIA | Porcentaje de comisión de incentivo anual — RN-006 |
| 32 | Campaña CIA 0% | SI/NO — campaña con comisión reducida a 0% — RN-006 |
| 33 | DIAS_UTILES_RESTANTES | Días hábiles restantes al cierre del mes — cruce con Dias_Utiles |
| 34 | COD_BUSQUEDA | Código de búsqueda para cruce entre tablas |
| 35 | FLAG CIERRE DE MES | Indica si el certificado fue procesado en cierre oficial del mes |

---

#### Tabla: BD_REFERIDOS2

Base de datos unificada de referidos y prospectos de Agenda Comercial. En el libro Excel unifica la Agenda Comercial Digital y la Agenda Comercial Cluster mediante el campo `AGENDA`. Al 19/03/2026 contiene **8,209 registros totales**.

> **Actualización v2.0:** Esta tabla estaba marcada como "Por levantar" en v1.0. Se documenta aquí con los campos extraídos del libro Excel.

**Campos identificados (47):**

| # | Campo | Tipo | Descripción |
|---|---|---|---|
| 1 | Fecha de Registro | Fecha | Fecha en que se registró el prospecto en la Agenda |
| 2 | FRENTE | Texto | Origen comercial: REFERIDO SCP, BD ASTARA, BD ASTARA SURCO, BD AUTOLAND, CARTERA PROPIA, PILOTO AUTOLAND — RN-007 |
| 3 | Tipo Doc Identidad | Texto | DNI u otro |
| 4 | N° Doc Identidad | Texto | Número de documento del prospecto |
| 5 | Nombre Prospecto | Texto | Nombre completo del prospecto |
| 6 | Celular Prospecto | Texto | Teléfono de contacto |
| 7 | Mail de Prospecto | Texto | Correo electrónico |
| 8 | Estado Civil | Texto | Estado civil del prospecto |
| 9 | Tipo Doc Ident Cónyuge / Conviviente | Texto | Tipo de documento del cónyuge |
| 10 | N° Doc Cónyuge / Conviviente | Texto | Número de documento del cónyuge |
| 11 | Nombre Cónyuge / Conviviente | Texto | Nombre del cónyuge |
| 12 | Ingreso Mensual | Numérico | Ingreso mensual declarado |
| 13 | Categoría Renta del Prospecto | Texto | Clasificación de renta |
| 14 | VISOR Decisión Riesgos | Texto | Resultado del motor de riesgos VISOR |
| 15 | Status Operación | Texto | Estado de la operación comercial |
| 16 | Fecha de Gestión | Fecha | Última fecha de gestión comercial |
| 17 | Intentos Contacto | Numérico | Número de intentos de contacto realizados |
| 18 | Fecha Proximo contacto | Fecha | Fecha programada para próximo contacto |
| 19 | INTERES COMPRA | Texto | FRIO / TIBIO / DESISTE |
| 20 | Comentario / Seguimiento | Texto | Notas del ejecutivo sobre la gestión |
| 21 | Certificado | Texto | Número de certificado si el prospecto se convirtió en asociado |
| 22 | Ejecutivo SFC Gestión | Texto | Ejecutivo de fondos colectivos que gestiona el prospecto |
| 23 | Coach SFC | Texto | Coach de fondos colectivos asignado |
| 24 | Ejecutivo SCP q refiere | Texto | Ejecutivo SCP que refirió al prospecto |
| 25 | Jefe Ventas | Texto | Jefe de ventas SCP del ejecutivo que refirió |
| 26 | Concesionario | Texto | Concesionario si aplica |
| 27 | Sucursal - Concesionario | Texto | Sucursal del concesionario |
| 28 | MARCA | Texto | Marca del vehículo de interés |
| 29 | Tipo Doc Ident Vendedor | Texto | Tipo de documento del vendedor del concesionario |
| 30 | N° Doc Vendedor | Texto | Número de documento del vendedor |
| 31 | Vendedor | Texto | Nombre del vendedor del concesionario |
| 32 | LIDER CANAL | Texto | Líder del canal correspondiente |
| 33 | ID | Texto | Identificador interno de la lista |
| 34 | ID_VISOR | Texto | ID en el sistema VISOR de riesgos |
| 35 | Creado | Fecha | Fecha de creación del registro en Microsoft Lists |
| 36 | Creado por | Texto | Usuario que creó el registro |
| 37 | Modificado | Fecha | Última fecha de modificación |
| 38 | Modificado por | Texto | Último usuario que modificó el registro |
| 39 | AGENDA | Texto | Distingue la fuente: "Digital" o "Cluster" |
| 40 | STATUS SANTO | Texto | Estado verificado en el sistema SANTO |
| 41 | ES REFERIDO | Texto | SI/NO — confirma si es un referido |
| 42 | STATUS ACTUALIZADO | Texto | Status actualizado en la última gestión |
| 43 | STATUS 0 | Texto | Clasificación de primer nivel del status — ver tabla STATUS |
| 44 | STATUS 1 | Texto | Clasificación de segundo nivel — ver tabla STATUS |
| 45 | EN REVISIÓN INTERNA | Texto | Flag de revisión interna pendiente |
| 46 | USUARIO_EJ_FONDOS | Texto | Código de usuario del ejecutivo de fondos |
| 47 | COD_BUSCAR_REF | Texto | Código de búsqueda para cruce con CERTIFICADOS |
| 48 | PERIODO_CREACION | Texto | Periodo YYYYMM de creación del registro |

---

#### Tabla: BD Lista Ejecutivos

Tabla maestra de ejecutivos y jerarquías organizacionales. Es la dimensión central usada para resolver `USUARIO_ACTUAL → NOMBRE_EJECUTIVO → JEFATURA → COACH` en las tres tablas principales.

> **Actualización v2.0:** Esta tabla estaba marcada como "Por levantar" en v1.0. Se documenta aquí con los campos extraídos del libro Excel.

**Campos identificados (16):**

| # | Campo | Descripción |
|---|---|---|
| 1 | ORI_DES_EJECUTIVO | Nombre completo del ejecutivo (origen SANTO) |
| 2 | CAL_DES_JEFE_COMERCIAL | Nombre del jefe comercial asignado (origen Calendar/SCP) |
| 3 | CAL_DES_LIDER_DISTRIBUCION | Nombre del líder de distribución |
| 4 | DNI_EJECUTIVO | Documento de identidad del ejecutivo |
| 5 | CORREO_EJECUTIVO | Correo corporativo del ejecutivo |
| 6 | Celular | Teléfono de contacto |
| 7 | PLANILLA | Clasificación en planilla (ej: ADM) |
| 8 | USUARIO | Código de usuario en SANTO (clave de resolución) |
| 9 | GRUPO | Grupo al que pertenece el ejecutivo (G10, G11, G12) |
| 10 | ESTADO | Estado del contrato del ejecutivo |
| 11 | Columna1 | Campo adicional (por documentar) |
| 12 | Columna2 | Campo adicional (por documentar) |
| 13 | Jefatura 1 | Nombre de la primera jefatura |
| 14 | Jefatura 2 | Nombre de la segunda jefatura |
| 15 | CLUSTER | Nombre del cluster (ej: CLUSTER - LA MARINA SAN MIGUEL) |
| 16 | CLUSTER EQUIVALENCIA | Nombre simplificado del cluster (ej: LA MARINA) |

> **Pendiente (P-02):** Confirmar los campos `Columna1` y `Columna2`, la regla de asignación de CANAL y la jerarquía Coach/Jefe VTA SCP en sesión formal con Softek.

---

### 6.2 Agenda Comercial — Microsoft Lists

La Agenda Comercial es un formulario en Microsoft Lists (SharePoint) que funciona como CRM del equipo comercial. Registra prospectos, su gestión comercial y el resultado de cada contacto.

| Atributo | Detalle |
|---|---|
| Plataforma | Microsoft Lists sobre SharePoint Online |
| Registros totales | 8,209 registros (Digital + Cluster) al 19/03/2026 |
| Campo clave | `AGENDA`: distingue entre Digital y Cluster |
| Campo de filtro | `FRENTE`: REFERIDO SCP, BD ASTARA, BD ASTARA SURCO, BD AUTOLAND, CARTERA PROPIA, PILOTO AUTOLAND |

Ver estructura completa de campos en la tabla BD_REFERIDOS2 de la sección 6.1.

---

### 6.3 Tablas de referencia (dimensiones)

Estas tablas existen en el libro Excel como hojas auxiliares y deben modelarse como tablas de referencia en la capa Bronze/Silver de Databricks.

#### BD_Tipo

Tabla de clasificación de registros de prueba. Usada para implementar la regla RN-001.

| Columna | Valores ejemplo | Descripción |
|---|---|---|
| TIPO (Cotizaciones) | COTIZACIONES DE PRUEBA | Valores que clasifican una cotización como PRUEBA |
| TIPO (Solicitudes/Cert.) | SOLICITUDES / CERTIFICADOS DE PRUEBA | Valores que clasifican solicitudes/certificados como PRUEBA |
| SE REPITE | (flag) | Indica si el tipo se repite en el periodo |

#### BD Alianzas

Tabla de concesionarios aliados. Usada para enriquecer el campo `ALIANZA` en las tablas transaccionales.

| Columna | Descripción | Ejemplo |
|---|---|---|
| CONCESIONARIO BD COMERCIAL | Nombre en la BD Comercial | 1222 PERU S.A.C. |
| CONCESIONARIO BD SANTO | Nombre en SANTO | 1222 PERU S.A.C. |
| ALIANZA | Grupo aliado | INCHCAPE |
| MARCA | Marca del vehículo | BMW |

Alianzas identificadas: **INCHCAPE** (BMW, BMW Motorrad), **ASTARA**, **AUTOLAND**, **DERCO**, **INTERAMERICANA**, **REVO MOTORS**, entre otras.

#### STATUS

Tabla de mapeo de estados de la Agenda Comercial. Usada para normalizar los campos STATUS 0 y STATUS 1 en BD_REFERIDOS2.

| Columna | Descripción |
|---|---|
| STATUS AGENDA | Valor literal en la Agenda Comercial |
| STATUS 0 | Clasificación de primer nivel normalizada |
| STATUS 1 | Clasificación de segundo nivel normalizada |
| VALOR | Valor de referencia |
| EQUIVALENCIA | Equivalencia con el canal de venta |

Valores principales identificados:

| STATUS AGENDA | STATUS 0 | STATUS 1 |
|---|---|---|
| SIN GESTION | SIN_GESTION | SIN_GESTION |
| SI CONTACTO Desistió | SI_CONTACTO (SC) | SC_NO DESEA |

#### BD Estado Solicitud

Tabla de mapeo del Estado de SOLICITUDES hacia los tres niveles de agrupación. Usada para implementar la regla RN-004.

| Columna | Descripción | Ejemplo |
|---|---|---|
| Orden | Número de orden del estado | 00 |
| Estado Final | Nombre del estado en SANTO | Rechazada |
| Concatenar | Estado con prefijo (Estado2) | 00. RECHAZADA /BAJA |
| AGRUP 1 | Agrupación nivel 1 (Estado3) | 00. RECH /BAJA |
| AGRUP 2 | Agrupación nivel 2 (Estado4) | (por documentar) |
| AGRUP 3 | Agrupación nivel 3 | (por documentar) |
| N° Mes | Número de mes | (referencia calendario) |
| Nombre Mes | Nombre del mes | ENE |

> **Pendiente (P-03):** Documentar la tabla de mapeo completa con los 38 estados en sesión con el equipo de negocio.

#### GUIA_STATUS

Tabla de referencia para el cálculo de flags de hitos del proceso. Estructura pivote con los 7 hitos identificados:

`PRE REGISTRO | REGISTRO | EVALUADA | APROBADA | DOCS FIRMADOS | LLAMADA CALIDAD OK | PAGO REALIZADO`

#### Dias_Utiles

Tabla de calendario hábil. Usada para calcular el campo `DIAS_UTILES_RESTANTES` en Certificados.

| Columna | Descripción |
|---|---|
| FECHA | Fecha del día |
| NOMBRE_DIA | Nombre del día (Monday, Tuesday…) |
| FLAG_DIA_HABIL | 1 = hábil, 0 = no hábil |
| DIA_SEMANA | Número de día en la semana |
| DIA_MES | Número de día en el mes |
| SEMANA_ANUAL | Número de semana del año |
| DIA_ANIO | Número de día del año |
| NOMBRE_MES | Nombre del mes (January…) |
| ANIO | Año |
| SEMESTRE | Semestre (1 o 2) |
| PERIODO | YYYY-MM |
| NUMERO_DIA_HABIL_MES | Número de día hábil dentro del mes |
| NUMERO_DIAS_UTILES_MES | Total de días hábiles del mes |
| NUMEROS_DIAS_UTILES_RESTANTES | Días hábiles restantes en el mes |
| PERIODO ORIG | Periodo original |
| PERIODO AGRUPADO | Agrupación de periodos para reportes |

---

## 7. Indicadores y dashboards actuales

### 7.1 Dashboard: Resumen Certificados

Dashboard de seguimiento de ventas de certificados activos (fuente: hoja CERTIFICADOS).

| Indicador | Valor (Feb 2026) | Descripción / Lógica |
|---|---|---|
| Total Ventas | 458 | Certificados contratados acumulados (Mar 2024–Feb 2026) |
| Total Certificados Activos | 402 | Estado Actual = Activo |
| Total Retiro a solicitud Asociado | 39 | Modalidad Resolución = "Retiro a solicitud" |
| Total Resolución por Deuda | 17 | Modalidad Resolución = "Resuelto con Devolución" |
| Venta del día (SCP) | 1 | Certificados nuevos en día de corte, canal SCP |
| Venta del día (Digital) | 2 | Certificados nuevos en día de corte, canal Digital |
| Certificados activos por equipo | CLUSTER: 15 / DIGITAL: 13 | Desglose por equipo de venta |
| Certificados activos por tipo | SCP_CLUSTER_DIRECTO: 10 / SCP_REFERIDOS: 5 / DIG_DIRECTO: 13 | Desglose por tipo |

### 7.2 Dashboard: Total Referidos SCP

Dashboard de seguimiento de prospectos del frente REFERIDO SCP (fuente: BD_REFERIDOS2).

> **Actualización v2.0:** Datos actualizados al 19/03/2026 (v1.0 mostraba datos al 08/02/2026).

| Indicador | Valor (19/03/2026) | Descripción |
|---|---|---|
| Total Referidos SCP (acumulado) | **8,209** | Total de prospectos registrados (Digital + Cluster) |
| SI_CONTACTO (SC) | ~63% | Prospectos contactados |
| NO_CONTACTO (NC) | ~26% | Prospectos no contactados |
| SIN_GESTIÓN | ~10% | Prospectos sin ninguna gestión |
| SC_ASOCIADO | (por medir) | Contactados que se convirtieron en asociados |
| SC_EN GESTIÓN | (por medir) | En proceso de gestión activa |
| SC_VOLVER A LLAMAR | (por medir) | Expresaron interés, piden recontacto |
| SC_NO CALIFICA | (por medir) | No cumplen criterios de aprobación |
| SC_NO DESEA | (por medir) | No desean el producto |

### 7.3 Dashboards por grupo (G10, G11, G12)

Reportes de seguimiento por grupo de fondo (hojas G10, G11, G12 SANTO DASHBOARD):

- Solicitudes en trámite por canal y estado
- Ranking de certificados activos por ejecutivo y jefatura
- Días para cierre de asamblea

---

## 8. Reglas de negocio identificadas

### RN-001 | Clasificación TIPO (REAL vs PRUEBA)

**Aplica a:** Cotizaciones, Solicitudes, Certificados  
**Fuente:** BD_Tipo  
**Lógica:** Si el nombre del asociado, comentario o usuario contiene valores de prueba del sistema → `TIPO = "PRUEBA"`. En caso contrario → `TIPO = "REAL"`.  
**Impacto:** Los registros `TIPO = "PRUEBA"` se excluyen de todos los KPIs de negocio.

### RN-002 | Clave de deduplicación (FILTRO / NOMENCLATURA)

**Aplica a:** Cotizaciones (FILTRO), Solicitudes (NOMENCLATURA)  
**Lógica Cotizaciones:** `FILTRO = CONCAT(Usuario_Actual, Nro_Documento)`. Ej: "asakata15356181".  
**Lógica Solicitudes:** `NOMENCLATURA = CONCAT(Nro_Documento, AÑO)`. Ej: "1014525622025".  
`SE REPITE? = "SI"` si la clave aparece más de una vez en el periodo.  
`COD_BUSQ = CONCAT(Nro_Documento, "_", Usuario_Actual)` — clave de cruce con Certificados.

### RN-003 | Resolución de jerarquía de ejecutivos

**Aplica a:** Cotizaciones, Solicitudes, Certificados  
**Fuente de resolución:** BD Lista Ejecutivos (campo clave: `USUARIO`)  
**Cadena:** `USUARIO_ACTUAL → ORI_DES_EJECUTIVO → Jefatura 1 → Jefatura 2 → CLUSTER`  
**CANAL:** Se clasifica en EJECUTIVO SCP / CANAL DIGITAL / CLUSTER / ADM según canal de venta y tipo de usuario.

### RN-004 | Clasificación de estados (Estado2, Estado3, Estado4)

**Aplica a:** Solicitudes  
**Fuente:** BD Estado Solicitud  
**38 estados** en SANTO se agrupan en tres niveles:

| Campo | Descripción | Ejemplo |
|---|---|---|
| Estado2 | Estado con prefijo numérico (Concatenar) | "00. RECHAZADA /BAJA" |
| Estado3 | Agrupación AGRUP 1 | "00. RECH /BAJA" |
| Estado4 | Agrupación AGRUP 2 | (macro para tableros) |

> **Pendiente (P-03):** Documentar los 38 estados completos con el equipo de negocio.

### RN-005 | Flags de hitos del proceso de solicitud

**Aplica a:** Solicitudes  
**Fuente:** GUIA_STATUS  
**7 hitos binarios (0/1):** PRE_REGISTRO → REGISTRO → EVALUADA → APROBADA → DOCS_FIRMADOS → LLAMADA_CALIDAD_OK → PAGO_REALIZADO  
**FLAG_AVANCE = 1** si la solicitud avanzó más allá del pre-registro.  
**Uso:** Tasas de conversión entre etapas del funnel.

### RN-006 | Certificado válido y comisión CIA

**Aplica a:** Certificados  
**CERTIFICADO VALIDO? = NO** si: `TIPO = "PRUEBA"` OR (`Situación = "Retirado"` AND `¿Participó en su 1era Asamblea? = "NO"`).  
**% CIA:** Porcentaje de comisión de incentivo anual — cruce con GUIA_CAMP según periodo y canal.  
**DIAS_UTILES_RESTANTES:** Cruce con tabla Dias_Utiles para calcular días hábiles al cierre del mes.

### RN-007 | Dominio FRENTE

**Aplica a:** BD_REFERIDOS2 y Certificados  
**Valores controlados:** (vacío), BD ASTARA, BD ASTARA SURCO, BD AUTOLAND, CARTERA PROPIA, PILOTO AUTOLAND, REFERIDO SCP  
**En Certificados:** FRENTE se calcula por lookup desde BD_REFERIDOS2 usando Nro. de Documento como clave.  
**REFERIDOS SCP = "SI"** si `FRENTE = "REFERIDO SCP"`.

---

## 9. Requerimientos funcionales

### 9.1 Tabla de prioridades

| Valor | Prioridad |
|---|---|
| 1 | Inmediata — crítico para viabilidad del proyecto |
| 2 | Alta — esencial a corto plazo |
| 3 | Media — importante para valor completo |
| 4 | Baja — deseable |
| 5 | Prospectiva — fases futuras |

### 9.2 Requerimientos de ingesta (Bronze)

| ID | Requerimiento | Fuente | Prior. |
|---|---|---|---|
| RF-01 | Ingerir Cotizaciones, Solicitudes y Certificados de SANTO sin transformación | SANTO | 1 |
| RF-02 | Preservar todos los campos fuente tal como los entrega SANTO | SANTO | 1 |
| RF-03 | Agregar metadata: timestamp de ingesta, archivo fuente, batch_id | Sistema | 1 |
| RF-04 | Ingerir BD_REFERIDOS2 (Digital + Cluster) con todos los valores de FRENTE | Agenda Comercial | 1 |
| RF-05 | Ingerir tablas de referencia: BD Lista Ejecutivos, BD Alianzas, STATUS, BD Estado Solicitud, Dias_Utiles, BD_Tipo | SANTO | 2 |
| RF-06 | Frecuencia mínima de actualización: diaria para todas las fuentes | Todas | 2 |
| RF-07 | Manejo de reintentos automáticos ante fallos de conexión | Todas | 2 |

### 9.3 Requerimientos de transformación (Silver)

| ID | Requerimiento | Tabla | Prior. |
|---|---|---|---|
| RT-01 | Implementar clasificación TIPO (REAL/PRUEBA) — RN-001 | Todas | 1 |
| RT-02 | Implementar deduplicación FILTRO/NOMENCLATURA — RN-002 | Cotiz./Solic. | 1 |
| RT-03 | Resolver jerarquía ejecutivos por join con BD Lista Ejecutivos — RN-003 | Todas | 1 |
| RT-04 | Descomponer fechas: AÑO, MES, DIA, PERIODO (YYYYMM), SEMANA | Todas | 1 |
| RT-05 | Clasificar estados en tres niveles por join con BD Estado Solicitud — RN-004 | Solicitudes | 2 |
| RT-06 | Calcular 7 flags de hitos del proceso — RN-005 | Solicitudes | 2 |
| RT-07 | Estandarizar CANAL (EJECUTIVO SCP / CANAL DIGITAL / CLUSTER / ADM) | Todas | 2 |
| RT-08 | Implementar CERTIFICADO_VALIDO? — RN-006 | Certificados | 2 |
| RT-09 | Calcular DIAS_UTILES_RESTANTES por join con Dias_Utiles | Certificados | 2 |
| RT-10 | Normalizar STATUS 0 y STATUS 1 en BD_REFERIDOS2 por join con STATUS | Referidos | 2 |
| RT-11 | Calcular FRENTE en Certificados por lookup desde BD_REFERIDOS2 — RN-007 | Certificados | 3 |

### 9.4 Requerimientos de capa Gold (reportes)

| ID | Requerimiento | Dashboard destino | Prior. |
|---|---|---|---|
| RG-01 | Tabla de certificados activos por mes, equipo y tipo (DIRECTO/REFERIDO) | Resumen Certificados | 1 |
| RG-02 | Tabla de seguimiento diario de ventas (SCP/Digital, por subcanal) | Resumen Certificados | 1 |
| RG-03 | Tabla de referidos con métricas de contactabilidad (SI/NO CONTACTO, SIN GESTIÓN, breakdowns) | Total Referidos SCP | 1 |
| RG-04 | Serie temporal diaria de referidos con desglose por STATUS 0 / STATUS 1 | Total Referidos SCP | 2 |
| RG-05 | Dashboards por grupo G10, G11, G12: solicitudes, rankings, días cierre asamblea | G10/G11/G12 | 2 |
| RG-06 | Tasas de conversión del funnel: prospectos → cotizaciones → solicitudes → certificados | Nuevo | 3 |

---

## 10. Requerimientos no funcionales

| ID | Categoría | Requerimiento |
|---|---|---|
| RNF-01 | Rendimiento | Pipelines de ingesta dentro de ventana nocturna (máximo 4 horas) |
| RNF-02 | Disponibilidad | Datos en capa Gold actualizados antes de las 08:00 AM hora Lima cada día hábil |
| RNF-03 | Calidad de datos | Controles de calidad (Great Expectations o similar) en cada ejecución |
| RNF-04 | Gobernanza | Todas las tablas registradas en Unity Catalog con descripción, propietario y política de acceso |
| RNF-05 | Seguridad | Datos PII (DNI, nombre, celular, correo) con política de enmascaramiento o acceso restringido |
| RNF-06 | Trazabilidad | Cada registro Silver/Gold mantiene referencia al batch de origen en Bronze |
| RNF-07 | Mantenibilidad | Reglas de negocio como funciones documentadas y versionadas en Git |
| RNF-08 | Compatibilidad | Tablas Gold compatibles con la herramienta BI destino (confirmar Power BI u otra) |

---

## 11. Participantes clave

| Nombre / Rol | Área | Tipo | Responsabilidad |
|---|---|---|---|
| Por confirmar | Fondos Colectivos | Usuario Final | Valida reglas de negocio, indicadores y diccionario de datos |
| Por confirmar | TI / Datos SCP | Sponsor técnico | Aprueba arquitectura destino y accesos a fuentes |
| Por confirmar | Tercero (Softek) | Proveedor | Proporciona documentación técnica de SANTO y facilita acceso |
| Equipo Migración | Consultoría | Responsable | Levantamiento, diseño, implementación y entrega |

---

## 12. Restricciones del proyecto

| Restricción | Descripción |
|---|---|
| Acceso a fuentes | SANTO reside on-premise y es administrado por Softek. El acceso automatizado requiere negociación formal. |
| Datos gestionados por tercero | El equipo no puede modificar los sistemas fuente ni sus esquemas. |
| Documentación de SANTO | La documentación técnica de Softek es limitada. El levantamiento depende de disponibilidad del proveedor. |
| Disponibilidad de usuarios | Las sesiones de validación dependen de la agenda del equipo comercial. |
| Herramienta de BI destino | Por confirmar si es Power BI u otra, lo que puede afectar el formato de las tablas Gold. |

---

## 13. Glosario de términos

| Término | Definición |
|---|---|
| Certificado | Contrato de fondo colectivo vehicular suscrito por un asociado |
| Asociado | Cliente que ha contratado un certificado |
| Prospecto | Persona con interés en el producto, aún sin contratar |
| Adjudicado | El asociado ganó el sorteo/remate y recibió el vehículo |
| FRENTE | Campo de origen comercial: REFERIDO SCP, BD ASTARA, CARTERA PROPIA, etc. |
| Solicitud | Trámite formal de crédito. Flujo de 38 estados desde Pre-Registrada hasta WC realizado |
| Cotización | Paso previo a la solicitud. Registro del interés del prospecto |
| Canal Digital | Ventas originadas por medios digitales: web, redes sociales, app |
| EJECUTIVO SCP | Vendedor directo del banco Santander Consumer Perú |
| CLUSTER | Grupo de agentes externos bajo supervisión de un Jefe SCP |
| CIA (% CIA) | Comisión de Incentivo Anual |
| WC (Welcome Call) | Llamada de bienvenida al asociado. Estado 38 del flujo |
| SOFTEK / SANTO | Empresa y sistema core de fondos colectivos (on-premise) |
| SCP | Santander Consumer Perú |
| AGENDA | Campo en BD_REFERIDOS2 que distingue fuente: Digital o Cluster |
| Bronze / Silver / Gold | Capas de arquitectura Medallion en Databricks |
| STM | Source-to-Target Mapping — plantilla de transformación campo a campo |
| PII | Personally Identifiable Information — datos personales identificables |
| Unity Catalog | Módulo de gobernanza de Databricks |
| BRD | Business Requirements Document |
| As-Is | Estado actual antes de la transformación |
| KPI | Key Performance Indicator |

---

## 14. Pendientes y próximos pasos

| # | Acción pendiente | Responsable | Prioridad | Estado |
|---|---|---|---|---|
| P-01 | Negociar acceso automatizado a SANTO con Softek para el pipeline de ingesta | TI SCP + Softek | Crítica | Abierto |
| P-02 | Confirmar campos `Columna1` y `Columna2` de BD Lista Ejecutivos y reglas de asignación de CANAL/Coach | Equipo + Softek | Alta | Abierto |
| P-03 | Documentar tabla de mapeo completa: 38 estados → Estado2 → Estado3 → Estado4 | Equipo + Usuarios | Alta | Abierto |
| P-04 | Confirmar herramienta de visualización BI destino (Power BI u otra) | TI SCP | Alta | Abierto |
| P-05 | Completar diccionario de datos de Agenda Comercial (campos calculados y reglas de status) | Equipo | Media | Abierto |
| P-06 | Levantar tablas TD, TD_CERT y GUIA_CAMP de SANTO | Equipo + Softek | Media | Abierto |
| P-07 | Elaborar Source-to-Target Mapping (STM) campo a campo para las tres tablas principales | Equipo | Alta | Abierto |
| P-08 | Definir modelo lógico de datos (ERD) en Databricks | Equipo | Media | Abierto |
| P-09 | Elaborar Project Charter formal con cronograma, presupuesto y RACI | PM | Alta | Abierto |

---

## 15. Referencias

| Documento | Descripción |
|---|---|
| `04_Diccionarios/Diccionario_Campos_SANTO_Fondos_Colectivos.xlsx` | Diccionario de datos completo de las tablas Cotizaciones, Solicitudes y Certificados. Incluye campos fuente y calculados. Abril 2026. |
| `06_Referencia/Excel_Source/20260228_SFC_Dashboard.xlsx` | Libro Excel de trabajo con 24 hojas. Fuente principal para el inventario de campos documentado en este BRD v2.0. |
| `knowledge/README.md` | Resumen ejecutivo del proyecto, métricas actuales y estructura del repositorio. |
| Diagrama As-Is — Fondos Colectivos | Diagrama de arquitectura del estado actual elaborado en sesión de levantamiento. |

---

*Versión 2.0 — Actualización con inventario completo de campos — Abril 2026 — Confidencial*
