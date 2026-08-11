# Optimización de Consultas con Query Profile y Caching

## Metadatos del Laboratorio

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Analizar |
| **Tecnologías clave** | Query Profile, Result Cache, Local/Remote Disk Cache, Partition Pruning, GENERATOR(), QUERY_HISTORY |

## Descripción General

En este laboratorio ejecutarás seis consultas predefinidas sobre un dataset expandido de 500,000+ filas, diagnosticarás problemas de rendimiento utilizando el Query Profile de Snowsight, observarás el efecto de los tres niveles de caching de Snowflake y aplicarás técnicas de optimización documentando las mejoras obtenidas. Al finalizar, habrás desarrollado un flujo de trabajo sistemático de diagnóstico y optimización reproducible en cualquier proyecto Snowflake.

## Objetivos de Aprendizaje

- [ ] Ejecutar consultas de referencia (baseline) y registrar métricas iniciales de rendimiento en una tabla de benchmark
- [ ] Navegar e interpretar el Query Profile identificando nodos costosos, bytes escaneados y particiones leídas
- [ ] Identificar patrones de bajo rendimiento: full table scans, data spilling y productos cartesianos accidentales
- [ ] Medir el efecto del result cache ejecutando consultas idénticas consecutivamente
- [ ] Aplicar técnicas de optimización (filtros tempranos, proyección selectiva, warehouse sizing) y comparar métricas antes/después

## Prerrequisitos

### Conocimientos Previos
- Comprensión de planes de ejecución SQL y operaciones de join/agregación
- Familiaridad con micro-particiones y partition pruning (lección 8.1)
- Experiencia navegando Snowsight y ejecutando consultas en worksheets

### Acceso y Recursos
- Lab 06-02-01 completado: tabla `LAB_DB.INGEST_SCHEMA.SALES_RAW` existente con datos
- Lab 07-02-01 completado: warehouse `LAB_WH_S` (Small) disponible y tabla `LAB_DB.ADMIN_SCHEMA.QUERY_BENCHMARK` creada
- Rol `SYSADMIN` o superior con permisos para ver Query Profile y consultar `SNOWFLAKE.ACCOUNT_USAGE`

## Entorno del Laboratorio

| Componente | Especificación |
|------------|---------------|
| Cuenta Snowflake | Enterprise Edition Trial, AWS us-east-1 |
| Warehouse | `LAB_WH_S` (Small, AUTO_SUSPEND=60, AUTO_RESUME=TRUE) |
| Base de datos | `LAB_DB` |
| Esquemas | `INGEST_SCHEMA`, `ADMIN_SCHEMA` |
| Tabla principal | `LAB_DB.INGEST_SCHEMA.SALES_RAW` |
| Tabla de métricas | `LAB_DB.ADMIN_SCHEMA.QUERY_BENCHMARK` |
| Navegador | Chrome 124+, Firefox 125+ o Edge 124+ |
| Resolución mínima | 1366×768 px |

### Configuración Inicial

Abre Snowsight, crea una nueva worksheet con el nombre **LAB08_OptimizacionQueryProfile** y ejecuta:

```sql
-- Configurar contexto de sesión
USE ROLE SYSADMIN;
USE WAREHOUSE LAB_WH_S;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

-- Verificar que la tabla SALES_RAW existe y tiene datos
SELECT COUNT(*) AS filas_actuales FROM SALES_RAW;
```

> **Resultado esperado:** Un conteo mayor a 0 (los datos cargados en el lab 06-02-01).

---

## Paso 1: Expansión del Dataset con GENERATOR()

### Objetivo
Insertar 500,000 filas adicionales en `SALES_RAW` para garantizar un volumen de datos suficiente que permita observar diferencias de rendimiento y efectos de caching.

### Instrucciones

1. Verifica la estructura actual de `SALES_RAW`:

```sql
DESCRIBE TABLE SALES_RAW;
```

2. Ejecuta el script de expansión de datos. Este inserta 500,000 filas sintéticas con distribución realista:

```sql
INSERT INTO SALES_RAW
SELECT
    -- Generar ID único basado en secuencia + offset
    UUID_STRING()                                           AS sale_id,
    -- Cliente aleatorio entre 1 y 5000
    'CUST-' || LPAD(UNIFORM(1, 5000, RANDOM())::VARCHAR, 5, '0') AS customer_id,
    -- Producto aleatorio entre 1 y 200
    'PROD-' || LPAD(UNIFORM(1, 200, RANDOM())::VARCHAR, 4, '0')  AS product_id,
    -- Fecha aleatoria en los últimos 3 años
    DATEADD('day', -UNIFORM(1, 1095, RANDOM()), CURRENT_DATE())  AS sale_date,
    -- Cantidad entre 1 y 50
    UNIFORM(1, 50, RANDOM())                               AS quantity,
    -- Precio unitario entre 5.00 y 500.00
    ROUND(UNIFORM(5, 500, RANDOM()) + UNIFORM(0, 99, RANDOM()) / 100.0, 2) AS unit_price,
    -- Descuento entre 0 y 0.30
    ROUND(UNIFORM(0, 30, RANDOM()) / 100.0, 2)            AS discount,
    -- Canal de venta
    CASE UNIFORM(1, 4, RANDOM())
        WHEN 1 THEN 'ONLINE'
        WHEN 2 THEN 'TIENDA'
        WHEN 3 THEN 'DISTRIBUIDOR'
        ELSE 'TELEFONO'
    END                                                     AS sales_channel,
    -- Región
    CASE UNIFORM(1, 5, RANDOM())
        WHEN 1 THEN 'NORTE'
        WHEN 2 THEN 'SUR'
        WHEN 3 THEN 'ESTE'
        WHEN 4 THEN 'OESTE'
        ELSE 'CENTRO'
    END                                                     AS region,
    -- Timestamp de ingesta
    CURRENT_TIMESTAMP()                                     AS ingested_at
FROM TABLE(GENERATOR(ROWCOUNT => 500000));
```

3. Confirma el nuevo volumen total:

```sql
SELECT
    COUNT(*)           AS total_filas,
    COUNT(DISTINCT customer_id) AS clientes_unicos,
    COUNT(DISTINCT product_id)  AS productos_unicos,
    MIN(sale_date)     AS fecha_min,
    MAX(sale_date)     AS fecha_max
FROM SALES_RAW;
```

### Resultado Esperado

| total_filas | clientes_unicos | productos_unicos | fecha_min | fecha_max |
|-------------|-----------------|------------------|-----------|-----------|
| ≥ 500,000 | ~5,000 | ~200 | ~3 años atrás | ~hoy |

### Verificación
- El conteo total debe ser al menos 500,000 filas
- La inserción debe completarse en menos de 30 segundos con warehouse Small

---

## Paso 2: Preparación de la Tabla de Benchmark

### Objetivo
Asegurar que la tabla `QUERY_BENCHMARK` está lista para registrar las métricas de cada consulta ejecutada en este laboratorio.

### Instrucciones

1. Cambia al esquema de administración y verifica/recrea la tabla de benchmark:

```sql
USE SCHEMA ADMIN_SCHEMA;

-- Verificar si la tabla existe; si no, crearla
CREATE TABLE IF NOT EXISTS QUERY_BENCHMARK (
    benchmark_id        NUMBER AUTOINCREMENT,
    query_label         VARCHAR(100)   NOT NULL,
    query_id            VARCHAR(50),
    execution_time_ms   NUMBER,
    bytes_scanned       NUMBER,
    rows_produced       NUMBER,
    partitions_scanned  NUMBER,
    partitions_total    NUMBER,
    bytes_spilled_local NUMBER DEFAULT 0,
    bytes_spilled_remote NUMBER DEFAULT 0,
    result_from_cache   BOOLEAN DEFAULT FALSE,
    optimization_notes  VARCHAR(500),
    executed_at         TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Limpiar registros previos de este lab (si se re-ejecuta)
DELETE FROM QUERY_BENCHMARK WHERE query_label LIKE 'LAB08%';
```

2. Vuelve al esquema de ingesta para las consultas:

```sql
USE SCHEMA INGEST_SCHEMA;
```

### Resultado Esperado
- Tabla `QUERY_BENCHMARK` disponible y vacía de registros `LAB08*`

### Verificación

```sql
SELECT COUNT(*) FROM ADMIN_SCHEMA.QUERY_BENCHMARK WHERE query_label LIKE 'LAB08%';
-- Debe retornar 0
```

---

## Paso 3: Ejecución de Consultas Baseline (Problemas de Rendimiento)

### Objetivo
Ejecutar 3 consultas intencionalmente ineficientes para generar perfiles de ejecución con patrones de bajo rendimiento identificables.

### Instrucciones

**Consulta 1: Full Table Scan (SELECT * sin filtros)**

1. Desactiva el result cache para obtener métricas reales:

```sql
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
```

2. Ejecuta la consulta de escaneo completo:

```sql
-- CONSULTA 1: Full scan sin filtros ni proyección
SELECT *
FROM SALES_RAW;
```

3. Inmediatamente después, captura el QUERY_ID y registra las métricas:

```sql
SET qid1 = LAST_QUERY_ID();

INSERT INTO ADMIN_SCHEMA.QUERY_BENCHMARK
    (query_label, query_id, execution_time_ms, bytes_scanned, rows_produced,
     partitions_scanned, partitions_total, bytes_spilled_local, bytes_spilled_remote,
     result_from_cache, optimization_notes)
SELECT
    'LAB08_Q1_FULL_SCAN',
    $qid1,
    EXECUTION_TIME,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE,
    FALSE,
    'SELECT * sin filtros - full table scan esperado'
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = $qid1;
```

4. **Navega al Query Profile**: En el panel de resultados, haz clic en la pestaña **"Query Profile"**. Observa:
   - Nodo `TableScan`: ¿`PARTITIONS_SCANNED` = `PARTITIONS_TOTAL`?
   - Porcentaje de tiempo en el nodo de escaneo
   - Bytes totales escaneados

**Consulta 2: Agregación sin GROUP BY adecuado (explosión de datos)**

5. Ejecuta una consulta con agregación ineficiente:

```sql
-- CONSULTA 2: Agregación con muchas columnas en GROUP BY innecesarias
SELECT
    customer_id,
    product_id,
    sale_date,
    sales_channel,
    region,
    sale_id,  -- Columna única por fila (anula la agregación)
    SUM(quantity * unit_price) AS total_venta,
    AVG(discount) AS descuento_promedio
FROM SALES_RAW
GROUP BY 1, 2, 3, 4, 5, 6
ORDER BY total_venta DESC;
```

6. Registra métricas:

```sql
SET qid2 = LAST_QUERY_ID();

INSERT INTO ADMIN_SCHEMA.QUERY_BENCHMARK
    (query_label, query_id, execution_time_ms, bytes_scanned, rows_produced,
     partitions_scanned, partitions_total, bytes_spilled_local, bytes_spilled_remote,
     result_from_cache, optimization_notes)
SELECT
    'LAB08_Q2_BAD_AGGREGATION',
    $qid2,
    EXECUTION_TIME,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE,
    FALSE,
    'GROUP BY incluye sale_id (PK) - agregación inútil, posible spill'
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = $qid2;
```

7. **Revisa el Query Profile**: Busca nodos `Aggregate` y `Sort` con alto porcentaje de tiempo. Verifica si hay spilling.

**Consulta 3: Producto Cartesiano Intencional**

8. Ejecuta un cross join limitado (para no colapsar el warehouse):

```sql
-- CONSULTA 3: JOIN sin condición (producto cartesiano) - LIMITADO para seguridad
SELECT
    a.sale_id,
    a.customer_id,
    b.product_id,
    b.unit_price
FROM (SELECT * FROM SALES_RAW LIMIT 1000) a,
     (SELECT * FROM SALES_RAW LIMIT 1000) b;
-- Esto genera 1,000 x 1,000 = 1,000,000 filas
```

9. Registra métricas:

```sql
SET qid3 = LAST_QUERY_ID();

INSERT INTO ADMIN_SCHEMA.QUERY_BENCHMARK
    (query_label, query_id, execution_time_ms, bytes_scanned, rows_produced,
     partitions_scanned, partitions_total, bytes_spilled_local, bytes_spilled_remote,
     result_from_cache, optimization_notes)
SELECT
    'LAB08_Q3_CARTESIAN_PRODUCT',
    $qid3,
    EXECUTION_TIME,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE,
    FALSE,
    'Cross join intencional - 1M filas de 1K x 1K'
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = $qid3;
```

10. **Revisa el Query Profile**: El nodo `Join` debe mostrar filas de salida = 1,000,000 con entradas de 1,000 cada una.

### Resultado Esperado

| Consulta | Patrón | Señal esperada en Query Profile |
|----------|--------|---------------------------------|
| Q1 | Full Table Scan | partitions_scanned ≈ partitions_total, >90% tiempo en TableScan |
| Q2 | Agregación ineficiente | rows_produced ≈ rows_input, posible spill |
| Q3 | Producto cartesiano | 1,000,000 filas de salida en nodo Join |

### Verificación

```sql
SELECT query_label, execution_time_ms, bytes_scanned, rows_produced,
       partitions_scanned, partitions_total
FROM ADMIN_SCHEMA.QUERY_BENCHMARK
WHERE query_label LIKE 'LAB08_Q%'
ORDER BY query_label;
```

Debes ver 3 registros con métricas pobladas.

---

## Paso 4: Observación del Efecto de Caching

### Objetivo
Demostrar el funcionamiento del result cache ejecutando una consulta idéntica dos veces y comparando los tiempos de ejecución.

### Instrucciones

1. Reactiva el result cache:

```sql
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
```

2. Ejecuta una consulta analítica significativa (primera ejecución — sin cache):

```sql
-- CONSULTA 4: Primera ejecución - debe ir a almacenamiento
SELECT
    region,
    sales_channel,
    DATE_TRUNC('month', sale_date) AS mes,
    COUNT(*) AS num_ventas,
    SUM(quantity * unit_price * (1 - discount)) AS revenue_neto,
    AVG(quantity) AS cantidad_promedio
FROM SALES_RAW
WHERE sale_date >= DATEADD('year', -1, CURRENT_DATE())
GROUP BY 1, 2, 3
ORDER BY mes DESC, revenue_neto DESC;
```

3. Registra métricas de la primera ejecución:

```sql
SET qid4a = LAST_QUERY_ID();

INSERT INTO ADMIN_SCHEMA.QUERY_BENCHMARK
    (query_label, query_id, execution_time_ms, bytes_scanned, rows_produced,
     partitions_scanned, partitions_total, bytes_spilled_local, bytes_spilled_remote,
     result_from_cache, optimization_notes)
SELECT
    'LAB08_Q4A_FIRST_EXEC',
    $qid4a,
    EXECUTION_TIME,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE,
    FALSE,
    'Primera ejecución - sin result cache'
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = $qid4a;
```

4. Ejecuta **exactamente la misma consulta** por segunda vez (copia/pega sin modificar ni un espacio):

```sql
-- CONSULTA 4 (repetición): Segunda ejecución - debe usar result cache
SELECT
    region,
    sales_channel,
    DATE_TRUNC('month', sale_date) AS mes,
    COUNT(*) AS num_ventas,
    SUM(quantity * unit_price * (1 - discount)) AS revenue_neto,
    AVG(quantity) AS cantidad_promedio
FROM SALES_RAW
WHERE sale_date >= DATEADD('year', -1, CURRENT_DATE())
GROUP BY 1, 2, 3
ORDER BY mes DESC, revenue_neto DESC;
```

5. Registra métricas de la segunda ejecución:

```sql
SET qid4b = LAST_QUERY_ID();

INSERT INTO ADMIN_SCHEMA.QUERY_BENCHMARK
    (query_label, query_id, execution_time_ms, bytes_scanned, rows_produced,
     partitions_scanned, partitions_total, bytes_spilled_local, bytes_spilled_remote,
     result_from_cache, optimization_notes)
SELECT
    'LAB08_Q4B_CACHED_EXEC',
    $qid4b,
    EXECUTION_TIME,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE,
    TRUE,
    'Segunda ejecución - result cache esperado (bytes_scanned = 0)'
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = $qid4b;
```

6. **Verifica en el Query Profile** de la segunda ejecución: Deberías ver el mensaje **"Query result reused"** y el tiempo de ejecución cercano a 0 ms.

### Resultado Esperado

| Ejecución | execution_time_ms | bytes_scanned | Observación |
|-----------|-------------------|---------------|-------------|
| Q4A (primera) | > 500 ms (típico) | > 0 | Lectura completa de particiones filtradas |
| Q4B (segunda) | < 50 ms | 0 | Result cache activo |

La reducción de tiempo debe ser **>90%** en la segunda ejecución.

### Verificación

```sql
SELECT query_label, execution_time_ms, bytes_scanned, result_from_cache
FROM ADMIN_SCHEMA.QUERY_BENCHMARK
WHERE query_label LIKE 'LAB08_Q4%'
ORDER BY query_label;
```

---

## Paso 5: Aplicación de Técnicas de Optimización

### Objetivo
Reescribir las consultas problemáticas del Paso 3 aplicando filtros tempranos, proyección selectiva y predicate pushdown, luego comparar métricas.

### Instrucciones

1. Desactiva el cache nuevamente para medir rendimiento real:

```sql
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
```

**Consulta 5: Versión optimizada de Q1 (filtros tempranos + proyección selectiva)**

2. Ejecuta la versión optimizada del full scan:

```sql
-- CONSULTA 5: Optimización de Q1 - filtros tempranos + columnas específicas
SELECT
    customer_id,
    product_id,
    sale_date,
    quantity,
    unit_price,
    (quantity * unit_price * (1 - discount)) AS revenue_neto
FROM SALES_RAW
WHERE sale_date >= DATEADD('month', -3, CURRENT_DATE())
  AND region = 'NORTE'
  AND sales_channel = 'ONLINE';
```

3. Registra métricas:

```sql
SET qid5 = LAST_QUERY_ID();

INSERT INTO ADMIN_SCHEMA.QUERY_BENCHMARK
    (query_label, query_id, execution_time_ms, bytes_scanned, rows_produced,
     partitions_scanned, partitions_total, bytes_spilled_local, bytes_spilled_remote,
     result_from_cache, optimization_notes)
SELECT
    'LAB08_Q5_OPTIMIZED_FILTERS',
    $qid5,
    EXECUTION_TIME,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE,
    FALSE,
    'Filtros tempranos (fecha+region+canal) + proyección de 6 columnas vs SELECT *'
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = $qid5;
```

4. **Revisa el Query Profile**: Compara `partitions_scanned` con Q1. Debe ser significativamente menor.

**Consulta 6: Versión optimizada de Q2 (agregación correcta)**

5. Ejecuta la agregación corregida:

```sql
-- CONSULTA 6: Optimización de Q2 - GROUP BY significativo sin PK
SELECT
    customer_id,
    region,
    sales_channel,
    DATE_TRUNC('month', sale_date) AS mes,
    COUNT(*) AS num_transacciones,
    SUM(quantity * unit_price) AS total_venta,
    AVG(discount) AS descuento_promedio
FROM SALES_RAW
WHERE sale_date >= DATEADD('year', -1, CURRENT_DATE())
GROUP BY 1, 2, 3, 4
HAVING COUNT(*) > 1
ORDER BY total_venta DESC
LIMIT 100;
```

6. Registra métricas:

```sql
SET qid6 = LAST_QUERY_ID();

INSERT INTO ADMIN_SCHEMA.QUERY_BENCHMARK
    (query_label, query_id, execution_time_ms, bytes_scanned, rows_produced,
     partitions_scanned, partitions_total, bytes_spilled_local, bytes_spilled_remote,
     result_from_cache, optimization_notes)
SELECT
    'LAB08_Q6_OPTIMIZED_AGGREGATION',
    $qid6,
    EXECUTION_TIME,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE,
    FALSE,
    'GROUP BY sin PK + filtro fecha + LIMIT 100 - reduce filas de salida drásticamente'
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
WHERE QUERY_ID = $qid6;
```

7. **Revisa el Query Profile**: Compara `rows_produced` con Q2. El nodo `Aggregate` debe reducir filas significativamente.

### Resultado Esperado

| Comparación | Métrica | Q1 (baseline) | Q5 (optimizada) | Mejora esperada |
|-------------|---------|---------------|-----------------|-----------------|
| Bytes escaneados | bytes_scanned | Alto (100% tabla) | Reducido | >50% menos |
| Particiones | partitions_scanned | ≈ total | < total | Pruning activo |
| Filas producidas | rows_produced | 500K+ | Miles | >90% menos |

| Comparación | Métrica | Q2 (baseline) | Q6 (optimizada) | Mejora esperada |
|-------------|---------|---------------|-----------------|-----------------|
| Filas producidas | rows_produced | ≈500K | 100 (LIMIT) | >99% menos |
| Spilling | bytes_spilled | Posible >0 | 0 | Eliminado |
| Tiempo | execution_time_ms | Alto | Bajo | >60% menos |

### Verificación

```sql
-- Comparación lado a lado
SELECT
    query_label,
    execution_time_ms,
    ROUND(bytes_scanned / (1024*1024), 2) AS mb_escaneados,
    rows_produced,
    partitions_scanned,
    partitions_total,
    bytes_spilled_local,
    optimization_notes
FROM ADMIN_SCHEMA.QUERY_BENCHMARK
WHERE query_label LIKE 'LAB08%'
ORDER BY query_label;
```

---

## Paso 6: Análisis Comparativo y Documentación Final

### Objetivo
Generar un reporte consolidado que compare todas las métricas y documente las mejoras obtenidas.

### Instrucciones

1. Genera el reporte comparativo final:

```sql
-- Reporte final de benchmark
SELECT
    query_label,
    execution_time_ms,
    ROUND(bytes_scanned / (1024.0 * 1024), 2) AS mb_escaneados,
    rows_produced,
    partitions_scanned || ' / ' || partitions_total AS particiones,
    ROUND(partitions_scanned * 100.0 / NULLIF(partitions_total, 0), 1) AS pct_particiones,
    CASE
        WHEN bytes_spilled_local > 0 THEN 'SPILL LOCAL: ' || ROUND(bytes_spilled_local/(1024*1024),1) || ' MB'
        WHEN bytes_spilled_remote > 0 THEN 'SPILL REMOTE: ' || ROUND(bytes_spilled_remote/(1024*1024),1) || ' MB'
        ELSE 'Sin spilling'
    END AS spill_status,
    result_from_cache,
    optimization_notes
FROM ADMIN_SCHEMA.QUERY_BENCHMARK
WHERE query_label LIKE 'LAB08%'
ORDER BY query_label;
```

2. Calcula las mejoras porcentuales entre baseline y optimizada:

```sql
-- Cálculo de mejoras Q1 vs Q5
SELECT
    'Full Scan → Filtros' AS comparacion,
    b.execution_time_ms AS tiempo_baseline,
    o.execution_time_ms AS tiempo_optimizado,
    ROUND((1 - o.execution_time_ms::FLOAT / NULLIF(b.execution_time_ms, 0)) * 100, 1) AS pct_mejora_tiempo,
    ROUND((1 - o.bytes_scanned::FLOAT / NULLIF(b.bytes_scanned, 0)) * 100, 1) AS pct_mejora_bytes
FROM ADMIN_SCHEMA.QUERY_BENCHMARK b
JOIN ADMIN_SCHEMA.QUERY_BENCHMARK o
    ON b.query_label = 'LAB08_Q1_FULL_SCAN'
   AND o.query_label = 'LAB08_Q5_OPTIMIZED_FILTERS'

UNION ALL

SELECT
    'Agregación mala → Correcta',
    b.execution_time_ms,
    o.execution_time_ms,
    ROUND((1 - o.execution_time_ms::FLOAT / NULLIF(b.execution_time_ms, 0)) * 100, 1),
    ROUND((1 - o.bytes_scanned::FLOAT / NULLIF(b.bytes_scanned, 0)) * 100, 1)
FROM ADMIN_SCHEMA.QUERY_BENCHMARK b
JOIN ADMIN_SCHEMA.QUERY_BENCHMARK o
    ON b.query_label = 'LAB08_Q2_BAD_AGGREGATION'
   AND o.query_label = 'LAB08_Q6_OPTIMIZED_AGGREGATION'

UNION ALL

SELECT
    'Sin cache → Con cache',
    b.execution_time_ms,
    o.execution_time_ms,
    ROUND((1 - o.execution_time_ms::FLOAT / NULLIF(b.execution_time_ms, 0)) * 100, 1),
    ROUND((1 - o.bytes_scanned::FLOAT / NULLIF(b.bytes_scanned, 0)) * 100, 1)
FROM ADMIN_SCHEMA.QUERY_BENCHMARK b
JOIN ADMIN_SCHEMA.QUERY_BENCHMARK o
    ON b.query_label = 'LAB08_Q4A_FIRST_EXEC'
   AND o.query_label = 'LAB08_Q4B_CACHED_EXEC';
```

3. Restaura la configuración de sesión:

```sql
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
```

### Resultado Esperado

La consulta de mejoras debe mostrar porcentajes positivos significativos:

| Comparación | % Mejora Tiempo | % Mejora Bytes |
|-------------|-----------------|----------------|
| Full Scan → Filtros | >50% | >50% |
| Agregación mala → Correcta | >60% | >30% |
| Sin cache → Con cache | >90% | ~100% |

### Verificación
- Todas las filas del reporte tienen valores no nulos
- Los porcentajes de mejora son positivos (indicando reducción)
- El resultado del cache muestra ~100% de mejora en bytes (0 bytes escaneados)

---

## Validación y Testing

Ejecuta la siguiente consulta de validación integral para confirmar que el laboratorio se completó correctamente:

```sql
-- Validación integral del laboratorio
WITH lab_stats AS (
    SELECT
        COUNT(*) AS total_benchmarks,
        COUNT(CASE WHEN query_label LIKE 'LAB08_Q1%' THEN 1 END) AS q1_registros,
        COUNT(CASE WHEN query_label LIKE 'LAB08_Q2%' THEN 1 END) AS q2_registros,
        COUNT(CASE WHEN query_label LIKE 'LAB08_Q3%' THEN 1 END) AS q3_registros,
        COUNT(CASE WHEN query_label LIKE 'LAB08_Q4%' THEN 1 END) AS q4_registros,
        COUNT(CASE WHEN query_label LIKE 'LAB08_Q5%' THEN 1 END) AS q5_registros,
        COUNT(CASE WHEN query_label LIKE 'LAB08_Q6%' THEN 1 END) AS q6_registros,
        COUNT(CASE WHEN result_from_cache = TRUE THEN 1 END) AS cache_hits,
        COUNT(CASE WHEN execution_time_ms IS NOT NULL THEN 1 END) AS con_metricas
    FROM ADMIN_SCHEMA.QUERY_BENCHMARK
    WHERE query_label LIKE 'LAB08%'
)
SELECT
    CASE WHEN total_benchmarks >= 7 THEN '✅' ELSE '❌' END || ' Total registros: ' || total_benchmarks AS check_registros,
    CASE WHEN q1_registros >= 1 THEN '✅' ELSE '❌' END || ' Q1 Full Scan' AS check_q1,
    CASE WHEN q2_registros >= 1 THEN '✅' ELSE '❌' END || ' Q2 Bad Aggregation' AS check_q2,
    CASE WHEN q3_registros >= 1 THEN '✅' ELSE '❌' END || ' Q3 Cartesian' AS check_q3,
    CASE WHEN q4_registros >= 2 THEN '✅' ELSE '❌' END || ' Q4 Cache Test (2 ejecuciones)' AS check_q4,
    CASE WHEN q5_registros >= 1 THEN '✅' ELSE '❌' END || ' Q5 Optimized Filters' AS check_q5,
    CASE WHEN q6_registros >= 1 THEN '✅' ELSE '❌' END || ' Q6 Optimized Aggregation' AS check_q6,
    CASE WHEN cache_hits >= 1 THEN '✅' ELSE '❌' END || ' Cache hit detectado' AS check_cache,
    CASE WHEN con_metricas = total_benchmarks THEN '✅' ELSE '❌' END || ' Todas las métricas pobladas' AS check_metricas
FROM lab_stats;
```

**Criterio de éxito:** Todos los checks deben mostrar ✅.

Adicionalmente, verifica el volumen de datos en SALES_RAW:

```sql
SELECT
    CASE WHEN COUNT(*) >= 500000 THEN '✅ Dataset expandido correctamente: ' || COUNT(*) || ' filas'
         ELSE '❌ Dataset insuficiente: ' || COUNT(*) || ' filas'
    END AS check_dataset
FROM INGEST_SCHEMA.SALES_RAW;
```

---

## Solución de Problemas

### Problema 1: La segunda ejecución de Q4 no muestra result cache (bytes_scanned > 0)

**Síntomas:**
- La consulta Q4B muestra `bytes_scanned > 0` y tiempo similar a Q4A
- En el Query Profile no aparece el mensaje "Query result reused"

**Causa:**
El result cache se invalida si: (1) la sesión tiene `USE_CACHED_RESULT = FALSE`, (2) los datos subyacentes cambiaron entre ejecuciones (un INSERT/UPDATE/DELETE en SALES_RAW), (3) la consulta tiene funciones no determinísticas como `CURRENT_TIMESTAMP()` en el SELECT, o (4) el texto de la consulta difiere en un solo carácter (incluyendo comentarios).

**Solución:**
```sql
-- 1. Verificar que el cache está activo
SHOW PARAMETERS LIKE 'USE_CACHED_RESULT' IN SESSION;

-- 2. Asegurarse de que no hubo DML entre las dos ejecuciones
-- Si insertaste datos, el cache se invalidó. Re-ejecuta ambas consultas sin DML intermedio.

-- 3. Verificar que la consulta es idéntica (copiar/pegar sin modificar)
-- Los comentarios SQL (--) forman parte del texto y cambian el hash

-- 4. Si usas CURRENT_DATE() en el WHERE, es determinístico dentro del mismo día
-- pero CURRENT_TIMESTAMP() cambia en cada ejecución y puede invalidar el cache
```

### Problema 2: El INSERT con GENERATOR() falla con error de columnas

**Síntomas:**
- Error: `Insert value list does not match column list expecting X columns but got Y`
- O error de tipos incompatibles en alguna columna

**Causa:**
La estructura de `SALES_RAW` creada en el lab 06-02-01 puede diferir del esquema asumido en el script de expansión (nombres de columnas diferentes, columnas adicionales o faltantes).

**Solución:**
```sql
-- 1. Verificar la estructura real de la tabla
DESCRIBE TABLE INGEST_SCHEMA.SALES_RAW;

-- 2. Adaptar el INSERT para que coincida con las columnas existentes
-- Si la tabla tiene columnas diferentes, ajusta el SELECT del GENERATOR()
-- para producir exactamente las mismas columnas en el mismo orden.

-- 3. Alternativa: especificar columnas explícitamente en el INSERT
INSERT INTO SALES_RAW (col1, col2, col3, ...)  -- listar columnas reales
SELECT
    valor1, valor2, valor3, ...  -- en el mismo orden
FROM TABLE(GENERATOR(ROWCOUNT => 500000));

-- 4. Si la tabla no existe, créala primero:
CREATE TABLE IF NOT EXISTS INGEST_SCHEMA.SALES_RAW (
    sale_id         VARCHAR(50),
    customer_id     VARCHAR(20),
    product_id      VARCHAR(20),
    sale_date       DATE,
    quantity        NUMBER,
    unit_price      NUMBER(10,2),
    discount        NUMBER(5,2),
    sales_channel   VARCHAR(20),
    region          VARCHAR(20),
    ingested_at     TIMESTAMP_NTZ
);
```

---

## Limpieza

Los datos y la tabla de benchmark se mantienen para uso en laboratorios posteriores (lab 10-02-01). Si deseas liberar recursos del warehouse:

```sql
-- Suspender el warehouse para detener el consumo de créditos
ALTER WAREHOUSE LAB_WH_S SUSPEND;

-- (OPCIONAL) Si deseas eliminar solo los datos expandidos de este lab:
-- DELETE FROM INGEST_SCHEMA.SALES_RAW WHERE ingested_at >= '<timestamp_inicio_lab>';
-- No recomendado: los datos se necesitan en labs posteriores.
```

> **Nota importante:** NO elimines la tabla `SALES_RAW` ni los registros de `QUERY_BENCHMARK` con prefijo `LAB08`. Estos serán reutilizados como lógica dentro de stored procedures en el lab 10-02-01.

---

## Resumen

En este laboratorio has completado un ciclo completo de diagnóstico y optimización de consultas en Snowflake:

| Actividad | Resultado clave |
|-----------|----------------|
| Expansión de datos con GENERATOR() | Dataset de 500K+ filas para análisis significativo |
| Ejecución de consultas problemáticas | Identificación de full scan, agregación ineficiente y producto cartesiano |
| Análisis de Query Profile | Lectura de nodos, bytes escaneados, particiones y spilling |
| Experimento de caching | Reducción >90% en tiempo con result cache activo |
| Aplicación de optimizaciones | Mejoras documentadas >50% en bytes y tiempo |
| Documentación en QUERY_BENCHMARK | Métricas trazables para análisis posterior |

### Conceptos Clave Reforzados

- **Partition pruning** se activa con filtros sobre rangos de fechas (no funciones sobre columnas)
- **Result cache** tiene TTL de 24 horas y se invalida con cualquier DML en las tablas subyacentes
- **SELECT *** y **GROUP BY con columnas únicas** son antipatrones que siempre degradan el rendimiento
- El Query Profile es la herramienta primaria para identificar la causa raíz de consultas lentas

### Recursos Adicionales

- [Documentación oficial: Query Profile](https://docs.snowflake.com/en/user-guide/ui-query-profile)
- [Documentación oficial: Understanding Query Processing](https://docs.snowflake.com/en/user-guide/ui-query-profile-understanding)
- [Best Practices: Query Performance](https://docs.snowflake.com/en/user-guide/performance-query)
- [QUERY_HISTORY View](https://docs.snowflake.com/en/sql-reference/account-usage/query_history)
