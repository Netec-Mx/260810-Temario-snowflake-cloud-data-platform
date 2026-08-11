# Administración de Warehouses y Control de Consumo

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Virtual Warehouses (XS, S, M), Auto-Suspend/Auto-Resume, Resource Monitors, WAREHOUSE_METERING_HISTORY, QUERY_HISTORY, Object Tags, Snowsight |

---

## Descripción General

En este laboratorio crearás tres virtual warehouses de diferentes tamaños (X-Small, Small y Medium), configurarás políticas de auto-suspend y auto-resume diferenciadas, ejecutarás un conjunto idéntico de consultas analíticas sobre la tabla `SALES_RAW` con cada warehouse para comparar rendimiento, analizarás el consumo de créditos mediante vistas de `ACCOUNT_USAGE`, configurarás un Resource Monitor con umbrales de alerta y suspensión, y aplicarás tags de departamento para asignación de costos. Los warehouses creados aquí se reutilizarán en laboratorios posteriores.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear múltiples virtual warehouses con diferentes tamaños y configuraciones de auto-suspend/auto-resume
- [ ] Ejecutar consultas analíticas idénticas en warehouses de distinto tamaño y comparar tiempos de ejecución
- [ ] Consultar `WAREHOUSE_METERING_HISTORY` y `QUERY_HISTORY` para analizar consumo de créditos
- [ ] Configurar un Resource Monitor con cuota mensual, notificaciones y acciones de suspensión
- [ ] Aplicar tags de objeto a warehouses para facilitar la asignación de costos por departamento

---

## Prerrequisitos

### Conocimiento Previo

- Comprensión del modelo de créditos de Snowflake (facturación por segundo, tamaños de warehouse)
- Experiencia ejecutando consultas SQL en Snowsight
- Familiaridad con el esquema `SNOWFLAKE.ACCOUNT_USAGE`

### Acceso y Recursos

- Lab 06-02-01 completado: tabla `LAB_DB.INGEST_SCHEMA.SALES_RAW` con mínimo 300 filas
- Rol `ACCOUNTADMIN` disponible (necesario para Resource Monitors y ACCOUNT_USAGE)
- Rol `SYSADMIN` disponible (creación de warehouses y objetos)
- Cuenta Snowflake Enterprise Edition Trial activa en AWS us-east-1

---

## Entorno del Laboratorio

### Software Requerido

| Componente | Versión | Propósito |
|------------|---------|-----------|
| Snowflake Trial Account | 8.x Enterprise Edition | Plataforma principal |
| Snowsight (Web UI) | 2024.4 | Interfaz de ejecución y monitoreo |
| SnowSQL CLI | 1.2.32 | Ejecución alternativa de comandos |
| Navegador web | Chrome 124+ / Firefox 125+ / Edge 124+ | Acceso a Snowsight |

### Configuración Inicial

Abre una nueva worksheet en Snowsight con el nombre **LAB07_Warehouses_Consumo** y ejecuta la verificación de prerrequisitos:

```sql
-- Verificar existencia de la tabla SALES_RAW y contar filas
USE ROLE SYSADMIN;
SELECT COUNT(*) AS total_filas FROM LAB_DB.INGEST_SCHEMA.SALES_RAW;
```

**Resultado esperado:** `total_filas` ≥ 300.

---

## Paso 1: Crear Virtual Warehouses con Diferentes Tamaños y Políticas

### Objetivo

Crear tres warehouses (`LAB_WH_XS`, `LAB_WH_S`, `LAB_WH_M`) con tamaños X-Small, Small y Medium respectivamente, cada uno con una política de auto-suspend diferenciada.

### Instrucciones

1. Cambia al rol `SYSADMIN` que tiene privilegios para crear warehouses:

```sql
USE ROLE SYSADMIN;
```

2. Crea el warehouse **LAB_WH_XS** (X-Small, auto-suspend 60 segundos):

```sql
CREATE OR REPLACE WAREHOUSE LAB_WH_XS
    WITH
    WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 1
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'Warehouse X-Small para laboratorio 07 - auto-suspend 60s';
```

3. Crea el warehouse **LAB_WH_S** (Small, auto-suspend 120 segundos):

```sql
CREATE OR REPLACE WAREHOUSE LAB_WH_S
    WITH
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 1
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'Warehouse Small para laboratorio 07 - auto-suspend 120s';
```

4. Crea el warehouse **LAB_WH_M** (Medium, auto-suspend 300 segundos):

```sql
CREATE OR REPLACE WAREHOUSE LAB_WH_M
    WITH
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 1
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'Warehouse Medium para laboratorio 07 - auto-suspend 300s';
```

5. Verifica la creación de los tres warehouses:

```sql
SHOW WAREHOUSES LIKE 'LAB_WH_%';
```

### Resultado Esperado

La consulta `SHOW WAREHOUSES` debe mostrar tres filas con los siguientes valores clave:

| name | size | auto_suspend | auto_resume | state |
|------|------|--------------|-------------|-------|
| LAB_WH_XS | X-Small | 60 | true | SUSPENDED |
| LAB_WH_S | Small | 120 | true | SUSPENDED |
| LAB_WH_M | Medium | 300 | true | SUSPENDED |

### Verificación

```sql
-- Confirmar configuraciones específicas
SELECT "name", "size", "auto_suspend", "auto_resume"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "name" IN ('LAB_WH_XS', 'LAB_WH_S', 'LAB_WH_M')
ORDER BY "size";
```

---

## Paso 2: Crear Tabla de Registro de Tiempos de Ejecución

### Objetivo

Crear una tabla auxiliar donde se registrarán los tiempos de ejecución de cada consulta por warehouse para facilitar la comparación posterior.

### Instrucciones

1. Establece el contexto de base de datos y esquema:

```sql
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;
USE WAREHOUSE LAB_WH_XS;
```

2. Crea la tabla de registro de benchmarks:

```sql
CREATE OR REPLACE TABLE LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (
    test_id         NUMBER AUTOINCREMENT,
    warehouse_name  VARCHAR(50),
    warehouse_size  VARCHAR(20),
    query_number    NUMBER,
    query_desc      VARCHAR(200),
    execution_time_ms NUMBER,
    rows_produced   NUMBER,
    bytes_scanned   NUMBER,
    timestamp_exec  TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### Resultado Esperado

Tabla `BENCHMARK_RESULTS` creada sin errores. Mensaje: `Table BENCHMARK_RESULTS successfully created.`

### Verificación

```sql
DESC TABLE LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS;
```

---

## Paso 3: Ejecutar Consultas Analíticas con Cada Warehouse

### Objetivo

Ejecutar 5 consultas analíticas predefinidas (agregaciones, funciones de ventana, subconsultas) sobre `SALES_RAW` utilizando cada uno de los tres warehouses y registrar los tiempos de ejecución.

### Instrucciones

Las siguientes 5 consultas serán ejecutadas con cada warehouse. Primero se definen las consultas y luego se ejecuta el ciclo completo.

#### Definición de las 5 Consultas de Benchmark

**Consulta 1 — Agregación simple por región:**

```sql
-- Q1: Ventas totales y promedio por región
SELECT
    region,
    COUNT(*)            AS total_transacciones,
    SUM(cantidad * precio_venta) AS ingreso_total,
    AVG(precio_venta)   AS precio_promedio,
    AVG(descuento)      AS descuento_promedio
FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
GROUP BY region
ORDER BY ingreso_total DESC;
```

**Consulta 2 — Agregación con múltiples dimensiones:**

```sql
-- Q2: Ventas por canal y región con métricas derivadas
SELECT
    canal_venta,
    region,
    COUNT(DISTINCT id_cliente) AS clientes_unicos,
    SUM(cantidad)              AS unidades_vendidas,
    SUM(cantidad * precio_venta * (1 - descuento)) AS ingreso_neto,
    ROUND(AVG(cantidad * precio_venta * (1 - descuento)), 2) AS ticket_promedio
FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
GROUP BY canal_venta, region
ORDER BY ingreso_neto DESC;
```

**Consulta 3 — Función de ventana (ranking):**

```sql
-- Q3: Top productos por ingreso con ranking por región
SELECT
    id_producto,
    region,
    SUM(cantidad * precio_venta) AS ingreso_producto,
    RANK() OVER (PARTITION BY region ORDER BY SUM(cantidad * precio_venta) DESC) AS ranking_regional,
    RATIO_TO_REPORT(SUM(cantidad * precio_venta)) OVER (PARTITION BY region) AS pct_region
FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
GROUP BY id_producto, region
QUALIFY ranking_regional <= 5
ORDER BY region, ranking_regional;
```

**Consulta 4 — Función de ventana (acumulado temporal):**

```sql
-- Q4: Ventas acumuladas por fecha con media móvil
SELECT
    fecha_venta,
    SUM(cantidad * precio_venta) AS ingreso_diario,
    SUM(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS UNBOUNDED PRECEDING) AS ingreso_acumulado,
    AVG(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_movil_7d
FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
GROUP BY fecha_venta
ORDER BY fecha_venta;
```

**Consulta 5 — Subconsulta correlacionada y filtrado avanzado:**

```sql
-- Q5: Clientes cuyo gasto total supera el promedio general
WITH resumen_cliente AS (
    SELECT
        id_cliente,
        COUNT(*)                     AS num_compras,
        SUM(cantidad * precio_venta) AS gasto_total,
        MIN(fecha_venta)             AS primera_compra,
        MAX(fecha_venta)             AS ultima_compra
    FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
    GROUP BY id_cliente
),
promedio_general AS (
    SELECT AVG(gasto_total) AS avg_gasto FROM resumen_cliente
)
SELECT
    rc.id_cliente,
    rc.num_compras,
    rc.gasto_total,
    rc.primera_compra,
    rc.ultima_compra,
    DATEDIFF('day', rc.primera_compra, rc.ultima_compra) AS dias_activo
FROM resumen_cliente rc, promedio_general pg
WHERE rc.gasto_total > pg.avg_gasto
ORDER BY rc.gasto_total DESC;
```

#### Ejecución con LAB_WH_XS

3. Ejecuta las 5 consultas usando `LAB_WH_XS` y registra los resultados:

```sql
USE WAREHOUSE LAB_WH_XS;

-- Ejecutar Q1 y registrar
-- (Ejecuta la consulta Q1 arriba, luego inserta el resultado)
INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS
    (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT
    'LAB_WH_XS', 'X-SMALL', 1, 'Agregacion simple por region',
    TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
    RESULT_LIMIT => 1,
    END_TIME_RANGE_START => DATEADD('minute', -5, CURRENT_TIMESTAMP())
))
WHERE QUERY_TEXT ILIKE '%Q1%Ventas totales y promedio por region%'
ORDER BY START_TIME DESC
LIMIT 1;
```

> **Nota:** Dado que el método anterior puede ser complejo de sincronizar, se recomienda el siguiente enfoque simplificado. Ejecuta cada consulta individualmente y luego usa `LAST_QUERY_ID()`:

```sql
USE WAREHOUSE LAB_WH_XS;

-- Ejecutar Q1
SELECT region, COUNT(*) AS total_transacciones, SUM(cantidad * precio_venta) AS ingreso_total, AVG(precio_venta) AS precio_promedio, AVG(descuento) AS descuento_promedio FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY region ORDER BY ingreso_total DESC;

-- Registrar resultado Q1
INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_XS', 'X-SMALL', 1, 'Agregacion simple por region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = LAST_QUERY_ID(-2)
LIMIT 1;
```

4. Repite el patrón para las consultas Q2 a Q5. Para agilizar, usa el siguiente bloque que ejecuta las 5 consultas y registra automáticamente (ejecuta cada par de sentencias secuencialmente):

```sql
-- Q2 con LAB_WH_XS
SELECT canal_venta, region, COUNT(DISTINCT id_cliente) AS clientes_unicos, SUM(cantidad) AS unidades_vendidas, SUM(cantidad * precio_venta * (1 - descuento)) AS ingreso_neto, ROUND(AVG(cantidad * precio_venta * (1 - descuento)), 2) AS ticket_promedio FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY canal_venta, region ORDER BY ingreso_neto DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_XS', 'X-SMALL', 2, 'Agregacion multi-dimension canal y region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q3 con LAB_WH_XS
SELECT id_producto, region, SUM(cantidad * precio_venta) AS ingreso_producto, RANK() OVER (PARTITION BY region ORDER BY SUM(cantidad * precio_venta) DESC) AS ranking_regional, RATIO_TO_REPORT(SUM(cantidad * precio_venta)) OVER (PARTITION BY region) AS pct_region FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY id_producto, region QUALIFY ranking_regional <= 5 ORDER BY region, ranking_regional;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_XS', 'X-SMALL', 3, 'Window function ranking por region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q4 con LAB_WH_XS
SELECT fecha_venta, SUM(cantidad * precio_venta) AS ingreso_diario, SUM(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS UNBOUNDED PRECEDING) AS ingreso_acumulado, AVG(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_movil_7d FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY fecha_venta ORDER BY fecha_venta;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_XS', 'X-SMALL', 4, 'Window function acumulado temporal', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q5 con LAB_WH_XS
WITH resumen_cliente AS (SELECT id_cliente, COUNT(*) AS num_compras, SUM(cantidad * precio_venta) AS gasto_total, MIN(fecha_venta) AS primera_compra, MAX(fecha_venta) AS ultima_compra FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY id_cliente), promedio_general AS (SELECT AVG(gasto_total) AS avg_gasto FROM resumen_cliente) SELECT rc.id_cliente, rc.num_compras, rc.gasto_total, rc.primera_compra, rc.ultima_compra, DATEDIFF('day', rc.primera_compra, rc.ultima_compra) AS dias_activo FROM resumen_cliente rc, promedio_general pg WHERE rc.gasto_total > pg.avg_gasto ORDER BY rc.gasto_total DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_XS', 'X-SMALL', 5, 'CTE con subconsulta correlacionada', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;
```

#### Ejecución con LAB_WH_S

5. Cambia al warehouse Small y repite las 5 consultas:

```sql
USE WAREHOUSE LAB_WH_S;

-- Q1 con LAB_WH_S
SELECT region, COUNT(*) AS total_transacciones, SUM(cantidad * precio_venta) AS ingreso_total, AVG(precio_venta) AS precio_promedio, AVG(descuento) AS descuento_promedio FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY region ORDER BY ingreso_total DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_S', 'SMALL', 1, 'Agregacion simple por region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q2 con LAB_WH_S
SELECT canal_venta, region, COUNT(DISTINCT id_cliente) AS clientes_unicos, SUM(cantidad) AS unidades_vendidas, SUM(cantidad * precio_venta * (1 - descuento)) AS ingreso_neto, ROUND(AVG(cantidad * precio_venta * (1 - descuento)), 2) AS ticket_promedio FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY canal_venta, region ORDER BY ingreso_neto DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_S', 'SMALL', 2, 'Agregacion multi-dimension canal y region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q3 con LAB_WH_S
SELECT id_producto, region, SUM(cantidad * precio_venta) AS ingreso_producto, RANK() OVER (PARTITION BY region ORDER BY SUM(cantidad * precio_venta) DESC) AS ranking_regional, RATIO_TO_REPORT(SUM(cantidad * precio_venta)) OVER (PARTITION BY region) AS pct_region FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY id_producto, region QUALIFY ranking_regional <= 5 ORDER BY region, ranking_regional;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_S', 'SMALL', 3, 'Window function ranking por region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q4 con LAB_WH_S
SELECT fecha_venta, SUM(cantidad * precio_venta) AS ingreso_diario, SUM(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS UNBOUNDED PRECEDING) AS ingreso_acumulado, AVG(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_movil_7d FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY fecha_venta ORDER BY fecha_venta;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_S', 'SMALL', 4, 'Window function acumulado temporal', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q5 con LAB_WH_S
WITH resumen_cliente AS (SELECT id_cliente, COUNT(*) AS num_compras, SUM(cantidad * precio_venta) AS gasto_total, MIN(fecha_venta) AS primera_compra, MAX(fecha_venta) AS ultima_compra FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY id_cliente), promedio_general AS (SELECT AVG(gasto_total) AS avg_gasto FROM resumen_cliente) SELECT rc.id_cliente, rc.num_compras, rc.gasto_total, rc.primera_compra, rc.ultima_compra, DATEDIFF('day', rc.primera_compra, rc.ultima_compra) AS dias_activo FROM resumen_cliente rc, promedio_general pg WHERE rc.gasto_total > pg.avg_gasto ORDER BY rc.gasto_total DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_S', 'SMALL', 5, 'CTE con subconsulta correlacionada', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;
```

#### Ejecución con LAB_WH_M

6. Cambia al warehouse Medium y repite las 5 consultas:

```sql
USE WAREHOUSE LAB_WH_M;

-- Q1 con LAB_WH_M
SELECT region, COUNT(*) AS total_transacciones, SUM(cantidad * precio_venta) AS ingreso_total, AVG(precio_venta) AS precio_promedio, AVG(descuento) AS descuento_promedio FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY region ORDER BY ingreso_total DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_M', 'MEDIUM', 1, 'Agregacion simple por region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q2 con LAB_WH_M
SELECT canal_venta, region, COUNT(DISTINCT id_cliente) AS clientes_unicos, SUM(cantidad) AS unidades_vendidas, SUM(cantidad * precio_venta * (1 - descuento)) AS ingreso_neto, ROUND(AVG(cantidad * precio_venta * (1 - descuento)), 2) AS ticket_promedio FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY canal_venta, region ORDER BY ingreso_neto DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_M', 'MEDIUM', 2, 'Agregacion multi-dimension canal y region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q3 con LAB_WH_M
SELECT id_producto, region, SUM(cantidad * precio_venta) AS ingreso_producto, RANK() OVER (PARTITION BY region ORDER BY SUM(cantidad * precio_venta) DESC) AS ranking_regional, RATIO_TO_REPORT(SUM(cantidad * precio_venta)) OVER (PARTITION BY region) AS pct_region FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY id_producto, region QUALIFY ranking_regional <= 5 ORDER BY region, ranking_regional;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_M', 'MEDIUM', 3, 'Window function ranking por region', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q4 con LAB_WH_M
SELECT fecha_venta, SUM(cantidad * precio_venta) AS ingreso_diario, SUM(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS UNBOUNDED PRECEDING) AS ingreso_acumulado, AVG(SUM(cantidad * precio_venta)) OVER (ORDER BY fecha_venta ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS media_movil_7d FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY fecha_venta ORDER BY fecha_venta;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_M', 'MEDIUM', 4, 'Window function acumulado temporal', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;

-- Q5 con LAB_WH_M
WITH resumen_cliente AS (SELECT id_cliente, COUNT(*) AS num_compras, SUM(cantidad * precio_venta) AS gasto_total, MIN(fecha_venta) AS primera_compra, MAX(fecha_venta) AS ultima_compra FROM LAB_DB.INGEST_SCHEMA.SALES_RAW GROUP BY id_cliente), promedio_general AS (SELECT AVG(gasto_total) AS avg_gasto FROM resumen_cliente) SELECT rc.id_cliente, rc.num_compras, rc.gasto_total, rc.primera_compra, rc.ultima_compra, DATEDIFF('day', rc.primera_compra, rc.ultima_compra) AS dias_activo FROM resumen_cliente rc, promedio_general pg WHERE rc.gasto_total > pg.avg_gasto ORDER BY rc.gasto_total DESC;

INSERT INTO LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS (warehouse_name, warehouse_size, query_number, query_desc, execution_time_ms, rows_produced, bytes_scanned)
SELECT 'LAB_WH_M', 'MEDIUM', 5, 'CTE con subconsulta correlacionada', TOTAL_ELAPSED_TIME, ROWS_PRODUCED, BYTES_SCANNED
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()) WHERE QUERY_ID = LAST_QUERY_ID(-2) LIMIT 1;
```

### Resultado Esperado

La tabla `BENCHMARK_RESULTS` debe contener 15 filas (5 consultas × 3 warehouses).

### Verificación

```sql
SELECT warehouse_name, warehouse_size, query_number, execution_time_ms
FROM LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS
ORDER BY query_number, warehouse_size;
```

Debes ver 15 filas con tiempos de ejecución registrados. Los valores de `execution_time_ms` variarán, pero en general el warehouse Medium debería tener tiempos iguales o menores que el X-Small para consultas con volumen significativo de datos.

---

## Paso 4: Comparar Rendimiento entre Warehouses

### Objetivo

Generar un reporte comparativo que muestre la diferencia de rendimiento entre los tres tamaños de warehouse.

### Instrucciones

1. Ejecuta la consulta de comparación pivotada:

```sql
USE WAREHOUSE LAB_WH_XS;

-- Comparación de tiempos de ejecución por consulta y warehouse
SELECT
    query_number,
    query_desc,
    MAX(CASE WHEN warehouse_size = 'X-SMALL' THEN execution_time_ms END) AS tiempo_xs_ms,
    MAX(CASE WHEN warehouse_size = 'SMALL'   THEN execution_time_ms END) AS tiempo_s_ms,
    MAX(CASE WHEN warehouse_size = 'MEDIUM'  THEN execution_time_ms END) AS tiempo_m_ms,
    ROUND(
        (MAX(CASE WHEN warehouse_size = 'X-SMALL' THEN execution_time_ms END) -
         MAX(CASE WHEN warehouse_size = 'MEDIUM'  THEN execution_time_ms END)) * 100.0 /
        NULLIF(MAX(CASE WHEN warehouse_size = 'X-SMALL' THEN execution_time_ms END), 0), 1
    ) AS pct_mejora_m_vs_xs
FROM LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS
GROUP BY query_number, query_desc
ORDER BY query_number;
```

2. Calcula el resumen por warehouse:

```sql
-- Resumen total por warehouse
SELECT
    warehouse_name,
    warehouse_size,
    COUNT(*)                AS num_consultas,
    SUM(execution_time_ms)  AS tiempo_total_ms,
    AVG(execution_time_ms)  AS tiempo_promedio_ms,
    SUM(bytes_scanned)      AS bytes_total_scanned
FROM LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS
GROUP BY warehouse_name, warehouse_size
ORDER BY warehouse_size;
```

### Resultado Esperado

Un reporte con 5 filas mostrando los tiempos de cada consulta en los tres warehouses y el porcentaje de mejora. El resumen mostrará 3 filas con totales y promedios por warehouse.

> **Observación pedagógica:** Con solo 300 filas, las diferencias de tiempo serán mínimas (posiblemente todas bajo 1 segundo). Esto demuestra que para datasets pequeños, usar un warehouse más grande no aporta beneficio significativo pero sí consume más créditos por segundo activo. La lección clave es: **dimensionar el warehouse según el volumen real de datos**.

### Verificación

Confirma que la columna `pct_mejora_m_vs_xs` muestra valores (positivos o negativos). Valores cercanos a 0 o negativos con datasets pequeños son normales y refuerzan la importancia del dimensionamiento correcto.

---

## Paso 5: Analizar Consumo de Créditos con ACCOUNT_USAGE

### Objetivo

Consultar las vistas de `SNOWFLAKE.ACCOUNT_USAGE` para analizar los créditos consumidos por los warehouses del laboratorio.

### Instrucciones

1. Cambia al rol `ACCOUNTADMIN` (necesario para acceder a ACCOUNT_USAGE):

```sql
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE LAB_WH_XS;
```

2. Consulta el consumo de créditos por warehouse en la última hora:

```sql
-- Consumo de créditos por warehouse (últimas 2 horas)
-- Nota: ACCOUNT_USAGE tiene latencia de ~45 min a 3 horas
SELECT
    WAREHOUSE_NAME,
    SUM(CREDITS_USED_COMPUTE)        AS creditos_compute,
    SUM(CREDITS_USED_CLOUD_SERVICES) AS creditos_cloud,
    SUM(CREDITS_USED)                AS creditos_totales,
    ROUND(SUM(CREDITS_USED) * 3, 4)  AS costo_estimado_usd
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME >= DATEADD('hour', -2, CURRENT_TIMESTAMP())
  AND WAREHOUSE_NAME IN ('LAB_WH_XS', 'LAB_WH_S', 'LAB_WH_M')
GROUP BY WAREHOUSE_NAME
ORDER BY creditos_totales DESC;
```

3. Si la consulta anterior no retorna datos (por la latencia), usa la vista en tiempo real de `INFORMATION_SCHEMA`:

```sql
-- Alternativa en tiempo real: consultar historial de la sesión
SELECT
    WAREHOUSE_NAME,
    WAREHOUSE_SIZE,
    COUNT(*) AS consultas_ejecutadas,
    SUM(TOTAL_ELAPSED_TIME) / 1000 AS tiempo_total_seg,
    SUM(BYTES_SCANNED) / (1024*1024) AS mb_escaneados
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 100
))
WHERE WAREHOUSE_NAME IN ('LAB_WH_XS', 'LAB_WH_S', 'LAB_WH_M')
  AND QUERY_TYPE = 'SELECT'
GROUP BY WAREHOUSE_NAME, WAREHOUSE_SIZE
ORDER BY tiempo_total_seg DESC;
```

4. Identifica las consultas más costosas de la sesión:

```sql
-- Top 5 consultas más largas de esta sesión
SELECT
    QUERY_ID,
    LEFT(QUERY_TEXT, 80) AS query_preview,
    WAREHOUSE_NAME,
    WAREHOUSE_SIZE,
    TOTAL_ELAPSED_TIME / 1000 AS duracion_seg,
    BYTES_SCANNED / (1024*1024) AS mb_escaneados
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 50
))
WHERE QUERY_TYPE = 'SELECT'
  AND WAREHOUSE_NAME IN ('LAB_WH_XS', 'LAB_WH_S', 'LAB_WH_M')
ORDER BY TOTAL_ELAPSED_TIME DESC
LIMIT 5;
```

### Resultado Esperado

- La consulta de `WAREHOUSE_METERING_HISTORY` puede mostrar datos vacíos si no ha pasado suficiente tiempo (latencia normal).
- La consulta de `INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION` mostrará las consultas ejecutadas en la sesión actual con métricas en tiempo real.
- El consumo estimado para este laboratorio con 300 filas será muy bajo (fracciones de crédito).

### Verificación

Al menos una de las dos consultas (ACCOUNT_USAGE o INFORMATION_SCHEMA) debe retornar filas con los nombres de los tres warehouses.

---

## Paso 6: Configurar un Resource Monitor

### Objetivo

Crear un Resource Monitor con límite mensual de 10 créditos, notificación al 75% y suspensión al 100%, y asignarlo a los warehouses del laboratorio.

### Instrucciones

1. Asegúrate de estar con el rol `ACCOUNTADMIN` (obligatorio para Resource Monitors):

```sql
USE ROLE ACCOUNTADMIN;
```

2. Crea el Resource Monitor:

```sql
CREATE OR REPLACE RESOURCE MONITOR LAB_RESOURCE_MONITOR
    WITH
    CREDIT_QUOTA = 10
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND;
```

3. Asigna el Resource Monitor a los tres warehouses:

```sql
ALTER WAREHOUSE LAB_WH_XS SET RESOURCE_MONITOR = LAB_RESOURCE_MONITOR;
ALTER WAREHOUSE LAB_WH_S  SET RESOURCE_MONITOR = LAB_RESOURCE_MONITOR;
ALTER WAREHOUSE LAB_WH_M  SET RESOURCE_MONITOR = LAB_RESOURCE_MONITOR;
```

4. Verifica la configuración del monitor:

```sql
SHOW RESOURCE MONITORS;
```

5. Consulta los detalles del monitor creado:

```sql
-- Ver estado actual del Resource Monitor
SELECT *
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "name" = 'LAB_RESOURCE_MONITOR';
```

### Resultado Esperado

`SHOW RESOURCE MONITORS` debe mostrar una fila con:

| name | credit_quota | frequency | level | warehouses |
|------|-------------|-----------|-------|------------|
| LAB_RESOURCE_MONITOR | 10.00 | MONTHLY | (vacío o ACCOUNT) | LAB_WH_XS, LAB_WH_S, LAB_WH_M |

### Verificación

```sql
-- Verificar que los warehouses tienen el monitor asignado
SHOW WAREHOUSES LIKE 'LAB_WH_%';
```

En la columna `resource_monitor` de cada warehouse debe aparecer `LAB_RESOURCE_MONITOR`.

---

## Paso 7: Aplicar Tags de Departamento para Asignación de Costos

### Objetivo

Crear un tag de departamento y aplicarlo a cada warehouse para facilitar la atribución de costos por área organizacional.

### Instrucciones

1. Crea el tag en la base de datos del laboratorio:

```sql
USE ROLE SYSADMIN;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

CREATE OR REPLACE TAG DEPT_TAG
    ALLOWED_VALUES 'ANALYTICS', 'DATA_ENGINEERING', 'FINANCE', 'DEVELOPMENT'
    COMMENT = 'Tag para asignacion de costos por departamento';
```

2. Aplica el tag a cada warehouse con un valor diferente:

```sql
-- LAB_WH_XS asignado a DEVELOPMENT (desarrollo y pruebas)
ALTER WAREHOUSE LAB_WH_XS SET TAG LAB_DB.INGEST_SCHEMA.DEPT_TAG = 'DEVELOPMENT';

-- LAB_WH_S asignado a ANALYTICS (consultas analíticas)
ALTER WAREHOUSE LAB_WH_S SET TAG LAB_DB.INGEST_SCHEMA.DEPT_TAG = 'ANALYTICS';

-- LAB_WH_M asignado a DATA_ENGINEERING (cargas pesadas)
ALTER WAREHOUSE LAB_WH_M SET TAG LAB_DB.INGEST_SCHEMA.DEPT_TAG = 'DATA_ENGINEERING';
```

3. Verifica los tags aplicados:

```sql
-- Consultar tags asignados a los warehouses
SELECT
    OBJECT_NAME,
    TAG_NAME,
    TAG_VALUE
FROM TABLE(
    LAB_DB.INFORMATION_SCHEMA.TAG_REFERENCES('LAB_WH_XS', 'WAREHOUSE')
)
UNION ALL
SELECT
    OBJECT_NAME,
    TAG_NAME,
    TAG_VALUE
FROM TABLE(
    LAB_DB.INFORMATION_SCHEMA.TAG_REFERENCES('LAB_WH_S', 'WAREHOUSE')
)
UNION ALL
SELECT
    OBJECT_NAME,
    TAG_NAME,
    TAG_VALUE
FROM TABLE(
    LAB_DB.INFORMATION_SCHEMA.TAG_REFERENCES('LAB_WH_M', 'WAREHOUSE')
);
```

4. Alternativamente, usa `SYSTEM$GET_TAG` para verificar individualmente:

```sql
SELECT SYSTEM$GET_TAG('LAB_DB.INGEST_SCHEMA.DEPT_TAG', 'LAB_WH_XS', 'WAREHOUSE') AS tag_xs;
SELECT SYSTEM$GET_TAG('LAB_DB.INGEST_SCHEMA.DEPT_TAG', 'LAB_WH_S', 'WAREHOUSE')  AS tag_s;
SELECT SYSTEM$GET_TAG('LAB_DB.INGEST_SCHEMA.DEPT_TAG', 'LAB_WH_M', 'WAREHOUSE')  AS tag_m;
```

### Resultado Esperado

| OBJECT_NAME | TAG_NAME | TAG_VALUE |
|-------------|----------|-----------|
| LAB_WH_XS | DEPT_TAG | DEVELOPMENT |
| LAB_WH_S | DEPT_TAG | ANALYTICS |
| LAB_WH_M | DEPT_TAG | DATA_ENGINEERING |

### Verificación

Los tres comandos `SYSTEM$GET_TAG` deben retornar `DEVELOPMENT`, `ANALYTICS` y `DATA_ENGINEERING` respectivamente.

---

## Validación y Pruebas Finales

Ejecuta las siguientes consultas para confirmar que todos los componentes del laboratorio están correctamente configurados:

```sql
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE LAB_WH_XS;

-- 1. Verificar los 3 warehouses existen con configuración correcta
SELECT "name", "size", "auto_suspend", "auto_resume", "resource_monitor"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "name" LIKE 'LAB_WH_%';

-- (Primero ejecutar SHOW WAREHOUSES LIKE 'LAB_WH_%')
SHOW WAREHOUSES LIKE 'LAB_WH_%';

-- 2. Verificar tabla de benchmarks tiene 15 registros
SELECT COUNT(*) AS total_registros,
       COUNT(DISTINCT warehouse_name) AS warehouses_distintos,
       COUNT(DISTINCT query_number) AS consultas_distintas
FROM LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS;

-- 3. Verificar Resource Monitor activo
SHOW RESOURCE MONITORS LIKE 'LAB_RESOURCE_MONITOR';

-- 4. Verificar tags aplicados
SELECT SYSTEM$GET_TAG('LAB_DB.INGEST_SCHEMA.DEPT_TAG', 'LAB_WH_S', 'WAREHOUSE') AS tag_wh_s;
```

**Criterios de éxito:**

| Verificación | Resultado esperado |
|---|---|
| Warehouses creados | 3 warehouses (XS, S, M) visibles |
| Auto-suspend configurado | 60s, 120s, 300s respectivamente |
| Benchmark registrados | 15 filas en BENCHMARK_RESULTS |
| Resource Monitor | LAB_RESOURCE_MONITOR con cuota 10 créditos |
| Tags aplicados | 3 warehouses con DEPT_TAG asignado |

---

## Solución de Problemas

### Problema 1: La consulta a WAREHOUSE_METERING_HISTORY retorna 0 filas

**Síntomas:** La consulta al esquema `SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY` no retorna datos aunque se han ejecutado consultas en los warehouses.

**Causa:** Las vistas de `ACCOUNT_USAGE` tienen una latencia de entre 45 minutos y hasta 3 horas. Los datos de consumo no están disponibles de forma inmediata tras la ejecución de las consultas.

**Solución:**
- Utiliza la alternativa en tiempo real con `INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION()` como se mostró en el Paso 5.
- Si necesitas los datos de `ACCOUNT_USAGE`, espera al menos 45 minutos y vuelve a ejecutar la consulta.
- Amplía el rango temporal del filtro `WHERE START_TIME >= DATEADD('hour', -24, CURRENT_TIMESTAMP())` para capturar actividad previa.

### Problema 2: Error "Insufficient privileges" al crear Resource Monitor

**Síntomas:** Al ejecutar `CREATE RESOURCE MONITOR`, se recibe el error `SQL access control error: Insufficient privileges to operate on account`.

**Causa:** Los Resource Monitors solo pueden ser creados por el rol `ACCOUNTADMIN`. Si se está usando `SYSADMIN` u otro rol, la operación será denegada.

**Solución:**
```sql
-- Cambiar al rol ACCOUNTADMIN antes de crear el Resource Monitor
USE ROLE ACCOUNTADMIN;

-- Ahora crear el Resource Monitor
CREATE OR REPLACE RESOURCE MONITOR LAB_RESOURCE_MONITOR
    WITH CREDIT_QUOTA = 10
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND;
```
- Verifica que tu usuario tiene el rol `ACCOUNTADMIN` asignado: `SHOW GRANTS TO USER <tu_usuario>;`
- Si no tienes acceso a `ACCOUNTADMIN`, solicita al administrador de la cuenta que ejecute la creación del monitor o te otorgue el rol.

---

## Limpieza

> **Importante:** Los warehouses `LAB_WH_XS`, `LAB_WH_S` y `LAB_WH_M` se utilizan en los laboratorios 08-02-01, 09-02-01 y 10-02-01. **NO los elimines** si planeas continuar con el curso. Solo ejecuta la limpieza parcial a continuación.

### Limpieza parcial (recomendada si continúas el curso):

```sql
USE ROLE SYSADMIN;

-- Suspender warehouses para no consumir créditos innecesarios
ALTER WAREHOUSE LAB_WH_XS SUSPEND;
ALTER WAREHOUSE LAB_WH_S SUSPEND;
ALTER WAREHOUSE LAB_WH_M SUSPEND;

-- La tabla de benchmarks puede mantenerse para referencia
-- La tabla SALES_RAW se mantiene para labs posteriores
```

### Limpieza completa (solo si NO continúas el curso):

```sql
USE ROLE ACCOUNTADMIN;

-- Eliminar Resource Monitor
DROP RESOURCE MONITOR IF EXISTS LAB_RESOURCE_MONITOR;

-- Eliminar warehouses
DROP WAREHOUSE IF EXISTS LAB_WH_XS;
DROP WAREHOUSE IF EXISTS LAB_WH_S;
DROP WAREHOUSE IF EXISTS LAB_WH_M;

-- Eliminar tag y tabla de benchmarks
USE ROLE SYSADMIN;
DROP TAG IF EXISTS LAB_DB.INGEST_SCHEMA.DEPT_TAG;
DROP TABLE IF EXISTS LAB_DB.INGEST_SCHEMA.BENCHMARK_RESULTS;
```

---

## Resumen

En este laboratorio has aplicado las siguientes competencias clave de administración de warehouses en Snowflake:

| Competencia | Lo que realizaste |
|---|---|
| **Creación de warehouses** | 3 warehouses con tamaños XS, S y M con configuraciones diferenciadas |
| **Auto-suspend/Auto-resume** | Políticas de 60s, 120s y 300s según tipo de uso |
| **Benchmarking** | 5 consultas analíticas ejecutadas en 3 warehouses con registro de tiempos |
| **Monitoreo de consumo** | Consultas a ACCOUNT_USAGE y INFORMATION_SCHEMA para análisis de créditos |
| **Resource Monitors** | Monitor mensual con cuota de 10 créditos y triggers al 75% y 100% |
| **Tags de costo** | DEPT_TAG aplicado a warehouses para atribución por departamento |

### Lecciones clave aprendidas

1. **El tamaño no siempre importa:** Para datasets pequeños (<1000 filas), un warehouse X-Small ofrece rendimiento similar a uno Medium pero a menor costo por segundo.
2. **Auto-suspend es la primera línea de defensa:** Configurar correctamente este parámetro previene el consumo innecesario de créditos.
3. **Resource Monitors son preventivos:** Establecer presupuestos con acciones automáticas evita sorpresas en la facturación.
4. **Los tags habilitan gobernanza:** Etiquetar warehouses por departamento permite reportes de chargeback precisos.

### Recursos Adicionales

- [Documentación: Virtual Warehouses](https://docs.snowflake.com/en/user-guide/warehouses)
- [Documentación: Resource Monitors](https://docs.snowflake.com/en/user-guide/resource-monitors)
- [Documentación: ACCOUNT_USAGE Schema](https://docs.snowflake.com/en/sql-reference/account-usage)
- [Documentación: Object Tagging](https://docs.snowflake.com/en/user-guide/object-tagging)
