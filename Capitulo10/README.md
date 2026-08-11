# Pipeline Incremental con Time Travel, Cloning, Streams, Tasks y Lógica Procedural

## Metadatos del Laboratorio

| Campo | Valor |
|-------|-------|
| **Duración** | 55 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar (Apply) |
| **Rol principal** | ACCOUNTADMIN / SYSADMIN |
| **Warehouse** | LAB_WH_XS |
| **Base de datos** | LAB_DB / LAB_DB_DEV |

---

## Descripción General

En este laboratorio capstone construirás un pipeline CDC (Change Data Capture) completamente funcional y automatizado dentro de Snowflake. Partirás de la base de datos `LAB_DB` con la tabla `SALES_RAW` (500,000+ filas), simularás un error de eliminación accidental que recuperarás con Time Travel, crearás un clon zero-copy para desarrollo, desarrollarás una UDF en JavaScript y un Stored Procedure en Snowflake Scripting, configurarás un Stream estándar para captura de cambios y programarás una Task CRON que ejecute el pipeline de forma automática cada 5 minutos.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Recuperar datos eliminados accidentalmente usando Time Travel con cláusulas `AT` y `BEFORE`
- [ ] Crear un clon zero-copy de una base de datos completa para ambiente de desarrollo
- [ ] Definir una UDF en JavaScript y un Stored Procedure en Snowflake Scripting para lógica de transformación incremental
- [ ] Configurar un Stream estándar sobre una tabla y consumir sus columnas de metadatos CDC
- [ ] Programar una Task con scheduling CRON que procese automáticamente el Stream y pueble una tabla de destino
- [ ] Monitorear la ejecución del pipeline usando `TASK_HISTORY` y verificar el estado del Stream

---

## Prerrequisitos

### Conocimiento Previo

| Requisito | Descripción |
|-----------|-------------|
| Labs anteriores | Labs 06-02-01 a 09-02-01 completados exitosamente |
| SQL intermedio | Sentencias MERGE, CTE, funciones de ventana |
| CDC | Comprensión básica de Change Data Capture |
| JavaScript básico | Sintaxis de funciones y condicionales |
| RBAC | Familiaridad con roles y privilegios en Snowflake |

### Acceso Requerido

| Recurso | Detalle |
|---------|---------|
| Cuenta Snowflake | Enterprise Trial, AWS us-east-1 |
| Rol | ACCOUNTADMIN para configuración inicial, SYSADMIN para desarrollo |
| Base de datos | `LAB_DB` con esquema `INGEST_SCHEMA` y tabla `SALES_RAW` (500K+ filas) |
| Warehouse | `LAB_WH_XS` (X-Small, AUTO_SUSPEND=60, AUTO_RESUME=TRUE) |
| Rol RBAC | `LAB_INGEST_ROLE` creado en lab 09-02-01 |

---

## Entorno del Laboratorio

### Software

| Herramienta | Versión | Uso |
|-------------|---------|-----|
| Snowflake | 8.x Enterprise Trial | Plataforma principal |
| Snowsight | 2024.4 | Interfaz web para ejecución de consultas |
| Navegador | Chrome 124+ / Firefox 125+ / Edge 124+ | Acceso a Snowsight |

### Configuración Inicial

Abre Snowsight y crea una nueva Worksheet con el nombre **LAB10_Pipeline_Incremental**. Establece el contexto de sesión:

```sql
-- Establecer contexto de sesión
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE LAB_WH_XS;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;
```

Verifica que la tabla `SALES_RAW` existe y tiene datos:

```sql
SELECT COUNT(*) AS total_filas FROM SALES_RAW;
-- Esperado: 500,000+ filas
```

---

## Paso 1: Simular Error y Recuperar Datos con Time Travel

### Objetivo
Simular una eliminación accidental de 50 filas en `SALES_RAW` y recuperarlas usando Time Travel con la cláusula `AT(TIMESTAMP => ...)`.

### Instrucciones

1. **Registra el timestamp actual** antes de la eliminación:

```sql
-- Guardar el timestamp previo al error
SET ts_antes_error = CURRENT_TIMESTAMP();
SELECT $ts_antes_error AS timestamp_referencia;
```

2. **Identifica las filas que serán eliminadas** (las primeras 50 por `ID_VENTA`):

```sql
-- Identificar las 50 filas a eliminar (simulación de error)
SELECT MIN(ID_VENTA) AS id_min, MAX(ID_VENTA) AS id_max
FROM (
    SELECT ID_VENTA
    FROM SALES_RAW
    ORDER BY ID_VENTA ASC
    LIMIT 50
);
```

3. **Ejecuta la eliminación accidental**:

```sql
-- SIMULAR ERROR: eliminar 50 filas accidentalmente
DELETE FROM SALES_RAW
WHERE ID_VENTA IN (
    SELECT ID_VENTA
    FROM SALES_RAW
    ORDER BY ID_VENTA ASC
    LIMIT 50
);
```

4. **Verifica que las filas fueron eliminadas**:

```sql
-- Confirmar la eliminación
SELECT COUNT(*) AS filas_post_delete FROM SALES_RAW;
-- Debería ser 50 menos que el conteo original
```

5. **Recupera las filas usando Time Travel con AT(TIMESTAMP)**:

```sql
-- Recuperar las filas eliminadas usando Time Travel
INSERT INTO SALES_RAW
SELECT *
FROM SALES_RAW AT(TIMESTAMP => $ts_antes_error)
WHERE ID_VENTA NOT IN (SELECT ID_VENTA FROM SALES_RAW);
```

6. **Verifica la recuperación completa**:

```sql
-- Verificar que se restauraron las 50 filas
SELECT COUNT(*) AS filas_restauradas FROM SALES_RAW;
```

### Resultado Esperado

```
+--------------------+
| FILAS_RESTAURADAS  |
|--------------------|
| 500000             | -- (o el número original antes del DELETE)
+--------------------+
```

### Verificación

```sql
-- Verificación adicional: no debe haber diferencia con el estado previo
SELECT COUNT(*) FROM SALES_RAW
MINUS
SELECT COUNT(*) FROM SALES_RAW AT(TIMESTAMP => $ts_antes_error);
-- Resultado: 0 filas (sin diferencia)
```

---

## Paso 2: Crear Zero-Copy Clone para Desarrollo

### Objetivo
Crear un clon zero-copy de `LAB_DB` como `LAB_DB_DEV` para desarrollar y probar la lógica del pipeline sin afectar el ambiente de producción.

### Instrucciones

1. **Crea el clon de la base de datos completa**:

```sql
-- Crear clon zero-copy de LAB_DB para desarrollo
CREATE OR REPLACE DATABASE LAB_DB_DEV
  CLONE LAB_DB
  COMMENT = 'Ambiente de desarrollo - clon zero-copy de LAB_DB para lab 10';
```

2. **Verifica la creación del clon**:

```sql
-- Verificar que el clon existe con todos los esquemas
SHOW SCHEMAS IN DATABASE LAB_DB_DEV;
```

3. **Confirma que los datos están presentes en el clon**:

```sql
-- Cambiar al contexto del clon
USE DATABASE LAB_DB_DEV;
USE SCHEMA INGEST_SCHEMA;

-- Verificar datos en la tabla clonada
SELECT COUNT(*) AS filas_en_clon FROM SALES_RAW;
```

4. **Verifica que el clon no consume almacenamiento adicional**:

```sql
-- Consultar almacenamiento (zero-copy no duplica datos)
SELECT TABLE_CATALOG, TABLE_SCHEMA, TABLE_NAME,
       BYTES, ROW_COUNT
FROM LAB_DB_DEV.INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME = 'SALES_RAW';
```

### Resultado Esperado

```
+---------------+--------------+------------+-----------+-----------+
| TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | BYTES     | ROW_COUNT |
|---------------+--------------+------------+-----------+-----------|
| LAB_DB_DEV    | INGEST_SCHEMA| SALES_RAW  | <valor>   | 500000    |
+---------------+--------------+------------+-----------+-----------+
```

### Verificación

```sql
-- El clon debe tener los mismos objetos que el original
SELECT COUNT(*) AS total_tablas
FROM LAB_DB_DEV.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'INGEST_SCHEMA';
```

---

## Paso 3: Desarrollar UDF en JavaScript en el Ambiente de Desarrollo

### Objetivo
Crear una UDF en JavaScript llamada `CALCULATE_DISCOUNT` que aplique descuentos diferenciados por región, desarrollándola primero en `LAB_DB_DEV`.

### Instrucciones

1. **Establece el contexto en el ambiente de desarrollo**:

```sql
USE DATABASE LAB_DB_DEV;
USE SCHEMA INGEST_SCHEMA;
```

2. **Crea la UDF en JavaScript**:

```sql
-- UDF que calcula descuento basado en monto y región
CREATE OR REPLACE FUNCTION CALCULATE_DISCOUNT(AMOUNT FLOAT, REGION STRING)
  RETURNS FLOAT
  LANGUAGE JAVASCRIPT
  COMMENT = 'Calcula descuento porcentual basado en monto y región de venta'
AS
$$
  var discount = 0.0;

  // Reglas de descuento por región
  switch(REGION) {
    case 'NORTH':
      discount = AMOUNT > 1000 ? 0.15 : 0.05;
      break;
    case 'SOUTH':
      discount = AMOUNT > 1000 ? 0.12 : 0.04;
      break;
    case 'EAST':
      discount = AMOUNT > 1000 ? 0.10 : 0.03;
      break;
    case 'WEST':
      discount = AMOUNT > 1000 ? 0.18 : 0.06;
      break;
    default:
      discount = 0.02;  // Descuento mínimo para regiones no mapeadas
  }

  return AMOUNT * discount;
$$;
```

3. **Prueba la UDF con valores conocidos**:

```sql
-- Pruebas unitarias de la UDF
SELECT
    CALCULATE_DISCOUNT(1500, 'NORTH') AS desc_north_alto,   -- Esperado: 225.0
    CALCULATE_DISCOUNT(500, 'NORTH')  AS desc_north_bajo,   -- Esperado: 25.0
    CALCULATE_DISCOUNT(2000, 'WEST')  AS desc_west_alto,    -- Esperado: 360.0
    CALCULATE_DISCOUNT(800, 'OTHER')  AS desc_default;      -- Esperado: 16.0
```

4. **Prueba la UDF contra datos reales de SALES_RAW**:

```sql
-- Aplicar la UDF sobre una muestra de datos reales
SELECT
    ID_VENTA,
    PRECIO_VENTA,
    REGION,
    CALCULATE_DISCOUNT(PRECIO_VENTA, REGION) AS descuento_calculado,
    PRECIO_VENTA - CALCULATE_DISCOUNT(PRECIO_VENTA, REGION) AS precio_final
FROM SALES_RAW
LIMIT 10;
```

### Resultado Esperado

```
+------------------+------------------+-------------------+
| DESC_NORTH_ALTO  | DESC_NORTH_BAJO  | DESC_WEST_ALTO    |
|------------------+------------------+-------------------|
| 225.0            | 25.0             | 360.0             |
+------------------+------------------+-------------------+
```

### Verificación

```sql
-- Verificar que la UDF está registrada
SHOW USER FUNCTIONS LIKE 'CALCULATE_DISCOUNT%' IN SCHEMA LAB_DB_DEV.INGEST_SCHEMA;
```

---

## Paso 4: Crear Stored Procedure para Procesamiento Incremental

### Objetivo
Desarrollar un Stored Procedure `SP_PROCESS_INCREMENTAL` en Snowflake Scripting que lea el Stream, aplique la UDF y ejecute un MERGE en la tabla de destino `SALES_PROCESSED`.

### Instrucciones

1. **Crea la tabla de destino `SALES_PROCESSED` en el ambiente de desarrollo**:

```sql
-- Tabla de destino para datos procesados
CREATE OR REPLACE TABLE LAB_DB_DEV.INGEST_SCHEMA.SALES_PROCESSED (
    ID_VENTA        NUMBER,
    ID_CLIENTE      NUMBER,
    ID_PRODUCTO     NUMBER,
    FECHA_VENTA     DATE,
    CANTIDAD        NUMBER,
    PRECIO_VENTA    FLOAT,
    REGION          VARCHAR(50),
    CANAL_VENTA     VARCHAR(50),
    DESCUENTO_CALC  FLOAT,
    PRECIO_FINAL    FLOAT,
    ACCION_CDC      VARCHAR(10),
    PROCESADO_EN    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_SALES_PROCESSED PRIMARY KEY (ID_VENTA)
);
```

2. **Crea el Stored Procedure en Snowflake Scripting**:

```sql
-- Stored Procedure para procesamiento incremental
CREATE OR REPLACE PROCEDURE LAB_DB_DEV.INGEST_SCHEMA.SP_PROCESS_INCREMENTAL()
  RETURNS VARCHAR
  LANGUAGE SQL
  COMMENT = 'Procedimiento que procesa datos incrementales del Stream SALES_STREAM aplicando UDF de descuento y MERGE en SALES_PROCESSED'
AS
$$
DECLARE
    filas_procesadas INTEGER DEFAULT 0;
    msg_resultado VARCHAR;
BEGIN
    -- Verificar si el stream tiene datos
    IF (SYSTEM$STREAM_HAS_DATA('INGEST_SCHEMA.SALES_STREAM')) THEN

        -- MERGE: insertar nuevos, actualizar existentes
        MERGE INTO INGEST_SCHEMA.SALES_PROCESSED AS destino
        USING (
            SELECT
                ID_VENTA,
                ID_CLIENTE,
                ID_PRODUCTO,
                FECHA_VENTA,
                CANTIDAD,
                PRECIO_VENTA,
                REGION,
                CANAL_VENTA,
                CALCULATE_DISCOUNT(PRECIO_VENTA, REGION) AS DESCUENTO_CALC,
                PRECIO_VENTA - CALCULATE_DISCOUNT(PRECIO_VENTA, REGION) AS PRECIO_FINAL,
                METADATA$ACTION AS ACCION_CDC
            FROM INGEST_SCHEMA.SALES_STREAM
            WHERE METADATA$ACTION = 'INSERT'
        ) AS origen
        ON destino.ID_VENTA = origen.ID_VENTA
        WHEN MATCHED THEN
            UPDATE SET
                destino.ID_CLIENTE     = origen.ID_CLIENTE,
                destino.ID_PRODUCTO    = origen.ID_PRODUCTO,
                destino.FECHA_VENTA    = origen.FECHA_VENTA,
                destino.CANTIDAD       = origen.CANTIDAD,
                destino.PRECIO_VENTA   = origen.PRECIO_VENTA,
                destino.REGION         = origen.REGION,
                destino.CANAL_VENTA    = origen.CANAL_VENTA,
                destino.DESCUENTO_CALC = origen.DESCUENTO_CALC,
                destino.PRECIO_FINAL   = origen.PRECIO_FINAL,
                destino.ACCION_CDC     = 'UPDATE',
                destino.PROCESADO_EN   = CURRENT_TIMESTAMP()
        WHEN NOT MATCHED THEN
            INSERT (ID_VENTA, ID_CLIENTE, ID_PRODUCTO, FECHA_VENTA, CANTIDAD,
                    PRECIO_VENTA, REGION, CANAL_VENTA, DESCUENTO_CALC, PRECIO_FINAL,
                    ACCION_CDC, PROCESADO_EN)
            VALUES (origen.ID_VENTA, origen.ID_CLIENTE, origen.ID_PRODUCTO,
                    origen.FECHA_VENTA, origen.CANTIDAD, origen.PRECIO_VENTA,
                    origen.REGION, origen.CANAL_VENTA, origen.DESCUENTO_CALC,
                    origen.PRECIO_FINAL, origen.ACCION_CDC, CURRENT_TIMESTAMP());

        -- Obtener filas afectadas
        filas_procesadas := SQLROWCOUNT;
        msg_resultado := 'Pipeline ejecutado exitosamente. Filas procesadas: ' || filas_procesadas::VARCHAR;

    ELSE
        msg_resultado := 'Sin datos nuevos en el Stream. No se requiere procesamiento.';
    END IF;

    RETURN msg_resultado;
END;
$$;
```

3. **Prueba el Stored Procedure en el ambiente de desarrollo** (requiere que exista un Stream; lo crearemos temporalmente para validar):

```sql
-- Crear un stream temporal en DEV para probar el SP
CREATE OR REPLACE STREAM LAB_DB_DEV.INGEST_SCHEMA.SALES_STREAM
  ON TABLE LAB_DB_DEV.INGEST_SCHEMA.SALES_RAW
  COMMENT = 'Stream de prueba en ambiente DEV';

-- Insertar datos de prueba para activar el stream
INSERT INTO LAB_DB_DEV.INGEST_SCHEMA.SALES_RAW
    (ID_VENTA, ID_CLIENTE, ID_PRODUCTO, FECHA_VENTA, CANTIDAD, PRECIO_VENTA, DESCUENTO, CANAL_VENTA, REGION)
VALUES
    (999901, 101, 201, '2024-11-01', 3, 1500.00, 0.10, 'ONLINE', 'NORTH'),
    (999902, 102, 202, '2024-11-02', 1, 800.00, 0.05, 'STORE', 'SOUTH'),
    (999903, 103, 203, '2024-11-03', 5, 2200.00, 0.15, 'ONLINE', 'WEST');

-- Ejecutar el SP
CALL LAB_DB_DEV.INGEST_SCHEMA.SP_PROCESS_INCREMENTAL();
```

4. **Verifica los resultados en la tabla de destino**:

```sql
-- Verificar datos procesados
SELECT * FROM LAB_DB_DEV.INGEST_SCHEMA.SALES_PROCESSED
WHERE ID_VENTA >= 999901
ORDER BY ID_VENTA;
```

### Resultado Esperado

```
+----------+------------+-------------+-------------+----------+--------------+--------+-------------+----------------+--------------+------------+---------------------+
| ID_VENTA | ID_CLIENTE | ID_PRODUCTO | FECHA_VENTA | CANTIDAD | PRECIO_VENTA | REGION | CANAL_VENTA | DESCUENTO_CALC | PRECIO_FINAL | ACCION_CDC | PROCESADO_EN        |
|----------+------------+-------------+-------------+----------+--------------+--------+-------------+----------------+--------------+------------+---------------------|
| 999901   | 101        | 201         | 2024-11-01  | 3        | 1500.00      | NORTH  | ONLINE      | 225.00         | 1275.00      | INSERT     | 2024-xx-xx ...      |
| 999902   | 102        | 202         | 2024-11-02  | 1        | 800.00       | SOUTH  | STORE       | 32.00          | 768.00       | INSERT     | 2024-xx-xx ...      |
| 999903   | 103        | 203         | 2024-11-03  | 5        | 2200.00      | WEST   | ONLINE      | 396.00         | 1804.00      | INSERT     | 2024-xx-xx ...      |
+----------+------------+-------------+-------------+----------+--------------+--------+-------------+----------------+--------------+------------+---------------------+
```

### Verificación

```sql
-- El SP debe retornar mensaje de éxito
-- Esperado: 'Pipeline ejecutado exitosamente. Filas procesadas: 3'
```

---

## Paso 5: Replicar Objetos en Producción (LAB_DB)

### Objetivo
Una vez validada la lógica en el ambiente de desarrollo, replicar la UDF, la tabla de destino y el Stored Procedure en `LAB_DB.INGEST_SCHEMA`.

### Instrucciones

1. **Cambiar al contexto de producción**:

```sql
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;
```

2. **Crear la UDF en producción**:

```sql
-- Replicar UDF en producción
CREATE OR REPLACE FUNCTION CALCULATE_DISCOUNT(AMOUNT FLOAT, REGION STRING)
  RETURNS FLOAT
  LANGUAGE JAVASCRIPT
  COMMENT = 'Calcula descuento porcentual basado en monto y región de venta'
AS
$$
  var discount = 0.0;
  switch(REGION) {
    case 'NORTH':
      discount = AMOUNT > 1000 ? 0.15 : 0.05;
      break;
    case 'SOUTH':
      discount = AMOUNT > 1000 ? 0.12 : 0.04;
      break;
    case 'EAST':
      discount = AMOUNT > 1000 ? 0.10 : 0.03;
      break;
    case 'WEST':
      discount = AMOUNT > 1000 ? 0.18 : 0.06;
      break;
    default:
      discount = 0.02;
  }
  return AMOUNT * discount;
$$;
```

3. **Crear la tabla de destino en producción**:

```sql
-- Tabla de destino en producción
CREATE OR REPLACE TABLE SALES_PROCESSED (
    ID_VENTA        NUMBER,
    ID_CLIENTE      NUMBER,
    ID_PRODUCTO     NUMBER,
    FECHA_VENTA     DATE,
    CANTIDAD        NUMBER,
    PRECIO_VENTA    FLOAT,
    REGION          VARCHAR(50),
    CANAL_VENTA     VARCHAR(50),
    DESCUENTO_CALC  FLOAT,
    PRECIO_FINAL    FLOAT,
    ACCION_CDC      VARCHAR(10),
    PROCESADO_EN    TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT PK_SALES_PROCESSED PRIMARY KEY (ID_VENTA)
);
```

4. **Crear el Stored Procedure en producción**:

```sql
-- Replicar SP en producción
CREATE OR REPLACE PROCEDURE SP_PROCESS_INCREMENTAL()
  RETURNS VARCHAR
  LANGUAGE SQL
  COMMENT = 'Procedimiento de producción: procesa datos incrementales del Stream SALES_STREAM'
AS
$$
DECLARE
    filas_procesadas INTEGER DEFAULT 0;
    msg_resultado VARCHAR;
BEGIN
    IF (SYSTEM$STREAM_HAS_DATA('INGEST_SCHEMA.SALES_STREAM')) THEN
        MERGE INTO INGEST_SCHEMA.SALES_PROCESSED AS destino
        USING (
            SELECT
                ID_VENTA, ID_CLIENTE, ID_PRODUCTO, FECHA_VENTA,
                CANTIDAD, PRECIO_VENTA, REGION, CANAL_VENTA,
                CALCULATE_DISCOUNT(PRECIO_VENTA, REGION) AS DESCUENTO_CALC,
                PRECIO_VENTA - CALCULATE_DISCOUNT(PRECIO_VENTA, REGION) AS PRECIO_FINAL,
                METADATA$ACTION AS ACCION_CDC
            FROM INGEST_SCHEMA.SALES_STREAM
            WHERE METADATA$ACTION = 'INSERT'
        ) AS origen
        ON destino.ID_VENTA = origen.ID_VENTA
        WHEN MATCHED THEN
            UPDATE SET
                destino.ID_CLIENTE     = origen.ID_CLIENTE,
                destino.ID_PRODUCTO    = origen.ID_PRODUCTO,
                destino.FECHA_VENTA    = origen.FECHA_VENTA,
                destino.CANTIDAD       = origen.CANTIDAD,
                destino.PRECIO_VENTA   = origen.PRECIO_VENTA,
                destino.REGION         = origen.REGION,
                destino.CANAL_VENTA    = origen.CANAL_VENTA,
                destino.DESCUENTO_CALC = origen.DESCUENTO_CALC,
                destino.PRECIO_FINAL   = origen.PRECIO_FINAL,
                destino.ACCION_CDC     = 'UPDATE',
                destino.PROCESADO_EN   = CURRENT_TIMESTAMP()
        WHEN NOT MATCHED THEN
            INSERT (ID_VENTA, ID_CLIENTE, ID_PRODUCTO, FECHA_VENTA, CANTIDAD,
                    PRECIO_VENTA, REGION, CANAL_VENTA, DESCUENTO_CALC, PRECIO_FINAL,
                    ACCION_CDC, PROCESADO_EN)
            VALUES (origen.ID_VENTA, origen.ID_CLIENTE, origen.ID_PRODUCTO,
                    origen.FECHA_VENTA, origen.CANTIDAD, origen.PRECIO_VENTA,
                    origen.REGION, origen.CANAL_VENTA, origen.DESCUENTO_CALC,
                    origen.PRECIO_FINAL, origen.ACCION_CDC, CURRENT_TIMESTAMP());

        filas_procesadas := SQLROWCOUNT;
        msg_resultado := 'Pipeline ejecutado exitosamente. Filas procesadas: ' || filas_procesadas::VARCHAR;
    ELSE
        msg_resultado := 'Sin datos nuevos en el Stream. No se requiere procesamiento.';
    END IF;
    RETURN msg_resultado;
END;
$$;
```

### Resultado Esperado

Cada sentencia `CREATE OR REPLACE` devuelve:
```
Statement executed successfully.
```

### Verificación

```sql
-- Confirmar objetos creados en producción
SHOW USER FUNCTIONS LIKE 'CALCULATE_DISCOUNT%' IN SCHEMA LAB_DB.INGEST_SCHEMA;
SHOW PROCEDURES LIKE 'SP_PROCESS_INCREMENTAL%' IN SCHEMA LAB_DB.INGEST_SCHEMA;
SHOW TABLES LIKE 'SALES_PROCESSED' IN SCHEMA LAB_DB.INGEST_SCHEMA;
```

---

## Paso 6: Configurar Stream en Producción

### Objetivo
Crear un Stream estándar sobre `SALES_RAW` en `LAB_DB` para capturar todos los cambios DML (INSERT, UPDATE, DELETE).

### Instrucciones

1. **Crea el Stream estándar**:

```sql
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

-- Crear Stream estándar sobre SALES_RAW
CREATE OR REPLACE STREAM SALES_STREAM
  ON TABLE SALES_RAW
  COMMENT = 'Stream estándar CDC sobre SALES_RAW - captura INSERT, UPDATE y DELETE';
```

2. **Verifica la configuración del Stream**:

```sql
-- Ver propiedades del Stream
SHOW STREAMS LIKE 'SALES_STREAM' IN SCHEMA INGEST_SCHEMA;
```

3. **Confirma que el Stream está vacío inicialmente**:

```sql
-- El stream no debe tener datos pendientes tras su creación
SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') AS tiene_datos;
-- Esperado: FALSE
```

4. **Inserta datos de prueba para activar el Stream**:

```sql
-- Insertar 5 filas de prueba para generar cambios en el Stream
INSERT INTO SALES_RAW
    (ID_VENTA, ID_CLIENTE, ID_PRODUCTO, FECHA_VENTA, CANTIDAD, PRECIO_VENTA, DESCUENTO, CANAL_VENTA, REGION)
VALUES
    (900001, 1001, 2001, '2024-12-01', 2, 1200.00, 0.10, 'ONLINE', 'NORTH'),
    (900002, 1002, 2002, '2024-12-01', 4, 750.00, 0.05, 'STORE', 'SOUTH'),
    (900003, 1003, 2003, '2024-12-02', 1, 3500.00, 0.20, 'ONLINE', 'WEST'),
    (900004, 1004, 2004, '2024-12-02', 3, 450.00, 0.00, 'DISTRIBUTOR', 'EAST'),
    (900005, 1005, 2005, '2024-12-03', 6, 1800.00, 0.12, 'ONLINE', 'NORTH');
```

5. **Verifica que el Stream capturó los cambios**:

```sql
-- Verificar datos en el Stream
SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') AS tiene_datos;
-- Esperado: TRUE

-- Consultar el contenido del Stream
SELECT
    ID_VENTA,
    PRECIO_VENTA,
    REGION,
    METADATA$ACTION,
    METADATA$ISUPDATE,
    METADATA$ROW_ID
FROM SALES_STREAM
ORDER BY ID_VENTA;
```

### Resultado Esperado

```
+----------+--------------+--------+-----------------+-------------------+------------------+
| ID_VENTA | PRECIO_VENTA | REGION | METADATA$ACTION | METADATA$ISUPDATE | METADATA$ROW_ID  |
|----------+--------------+--------+-----------------+-------------------+------------------|
| 900001   | 1200.00      | NORTH  | INSERT          | FALSE             | <hash_value>     |
| 900002   | 750.00       | SOUTH  | INSERT          | FALSE             | <hash_value>     |
| 900003   | 3500.00      | WEST   | INSERT          | FALSE             | <hash_value>     |
| 900004   | 450.00       | EAST   | INSERT          | FALSE             | <hash_value>     |
| 900005   | 1800.00      | NORTH  | INSERT          | FALSE             | <hash_value>     |
+----------+--------------+--------+-----------------+-------------------+------------------+
```

### Verificación

```sql
-- Contar filas pendientes en el Stream
SELECT COUNT(*) AS filas_pendientes FROM SALES_STREAM;
-- Esperado: 5
```

---

## Paso 7: Programar Task con Scheduling CRON

### Objetivo
Crear y activar una Task `SALES_PIPELINE_TASK` que ejecute `SP_PROCESS_INCREMENTAL()` cada 5 minutos, condicionada a que el Stream tenga datos pendientes.

### Instrucciones

1. **Otorga privilegio EXECUTE TASK al rol LAB_INGEST_ROLE**:

```sql
USE ROLE ACCOUNTADMIN;

-- Otorgar privilegio para ejecutar tasks
GRANT EXECUTE TASK ON ACCOUNT TO ROLE LAB_INGEST_ROLE;
GRANT EXECUTE MANAGED TASK ON ACCOUNT TO ROLE LAB_INGEST_ROLE;

-- Otorgar ownership de los objetos necesarios
GRANT USAGE ON DATABASE LAB_DB TO ROLE LAB_INGEST_ROLE;
GRANT USAGE ON SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_INGEST_ROLE;
GRANT USAGE ON WAREHOUSE LAB_WH_XS TO ROLE LAB_INGEST_ROLE;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW TO ROLE LAB_INGEST_ROLE;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE LAB_DB.INGEST_SCHEMA.SALES_PROCESSED TO ROLE LAB_INGEST_ROLE;
GRANT SELECT ON STREAM LAB_DB.INGEST_SCHEMA.SALES_STREAM TO ROLE LAB_INGEST_ROLE;
GRANT USAGE ON FUNCTION LAB_DB.INGEST_SCHEMA.CALCULATE_DISCOUNT(FLOAT, STRING) TO ROLE LAB_INGEST_ROLE;
GRANT USAGE ON PROCEDURE LAB_DB.INGEST_SCHEMA.SP_PROCESS_INCREMENTAL() TO ROLE LAB_INGEST_ROLE;
```

2. **Crea la Task con scheduling CRON**:

```sql
USE ROLE SYSADMIN;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

-- Crear la Task programada cada 5 minutos
CREATE OR REPLACE TASK SALES_PIPELINE_TASK
  WAREHOUSE = LAB_WH_XS
  SCHEDULE  = 'USING CRON */5 * * * * America/Mexico_City'
  COMMENT   = 'Task que procesa datos incrementales de SALES_STREAM cada 5 minutos'
  WHEN SYSTEM$STREAM_HAS_DATA('INGEST_SCHEMA.SALES_STREAM')
AS
  CALL SP_PROCESS_INCREMENTAL();
```

3. **Verifica que la Task fue creada en estado SUSPENDED**:

```sql
-- Verificar estado de la Task
SHOW TASKS LIKE 'SALES_PIPELINE_TASK' IN SCHEMA INGEST_SCHEMA;
```

4. **Activa (resume) la Task**:

```sql
-- Activar la Task
ALTER TASK SALES_PIPELINE_TASK RESUME;

-- Confirmar que está activa
SHOW TASKS LIKE 'SALES_PIPELINE_TASK' IN SCHEMA INGEST_SCHEMA;
-- La columna "state" debe mostrar "started"
```

5. **Ejecuta manualmente la Task para validar inmediatamente** (sin esperar al schedule):

```sql
-- Ejecución manual para validación inmediata
EXECUTE TASK SALES_PIPELINE_TASK;
```

6. **Espera 10-15 segundos y verifica los resultados**:

```sql
-- Verificar que SALES_PROCESSED tiene los datos procesados
SELECT COUNT(*) AS filas_procesadas FROM SALES_PROCESSED;
-- Esperado: 5

-- Ver detalle de los datos procesados
SELECT
    ID_VENTA,
    PRECIO_VENTA,
    REGION,
    DESCUENTO_CALC,
    PRECIO_FINAL,
    ACCION_CDC,
    PROCESADO_EN
FROM SALES_PROCESSED
ORDER BY ID_VENTA;
```

### Resultado Esperado

```
+----------+--------------+--------+----------------+--------------+------------+---------------------+
| ID_VENTA | PRECIO_VENTA | REGION | DESCUENTO_CALC | PRECIO_FINAL | ACCION_CDC | PROCESADO_EN        |
|----------+--------------+--------+----------------+--------------+------------+---------------------|
| 900001   | 1200.00      | NORTH  | 180.00         | 1020.00      | INSERT     | 2024-xx-xx ...      |
| 900002   | 750.00       | SOUTH  | 30.00          | 720.00       | INSERT     | 2024-xx-xx ...      |
| 900003   | 3500.00      | WEST   | 630.00         | 2870.00      | INSERT     | 2024-xx-xx ...      |
| 900004   | 450.00       | EAST   | 13.50          | 436.50       | INSERT     | 2024-xx-xx ...      |
| 900005   | 1800.00      | NORTH  | 270.00         | 1530.00      | INSERT     | 2024-xx-xx ...      |
+----------+--------------+--------+----------------+--------------+------------+---------------------+
```

### Verificación

```sql
-- El Stream debe estar vacío después del consumo exitoso
SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') AS tiene_datos;
-- Esperado: FALSE
```

---

## Paso 8: Monitorear el Pipeline con TASK_HISTORY

### Objetivo
Consultar el historial de ejecución de la Task y verificar el estado del pipeline usando vistas del sistema.

### Instrucciones

1. **Consulta el historial de ejecución de la Task**:

```sql
-- Historial de ejecuciones de la Task
SELECT
    NAME,
    STATE,
    SCHEDULED_TIME,
    COMPLETED_TIME,
    ERROR_CODE,
    ERROR_MESSAGE,
    RETURN_VALUE
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'SALES_PIPELINE_TASK',
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP()),
    RESULT_LIMIT => 10
))
ORDER BY SCHEDULED_TIME DESC;
```

2. **Inserta más datos para probar una segunda ejecución del pipeline**:

```sql
-- Insertar nuevos datos para activar el Stream nuevamente
INSERT INTO SALES_RAW
    (ID_VENTA, ID_CLIENTE, ID_PRODUCTO, FECHA_VENTA, CANTIDAD, PRECIO_VENTA, DESCUENTO, CANAL_VENTA, REGION)
VALUES
    (900006, 1006, 2006, '2024-12-04', 2, 950.00, 0.08, 'ONLINE', 'EAST'),
    (900007, 1007, 2007, '2024-12-04', 1, 4200.00, 0.15, 'STORE', 'WEST');

-- Verificar que el Stream detectó los cambios
SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') AS tiene_datos;
-- Esperado: TRUE
```

3. **Ejecuta la Task manualmente de nuevo**:

```sql
-- Segunda ejecución manual
EXECUTE TASK SALES_PIPELINE_TASK;
```

4. **Espera 10-15 segundos y verifica el historial actualizado**:

```sql
-- Verificar historial con ambas ejecuciones
SELECT
    NAME,
    STATE,
    SCHEDULED_TIME,
    COMPLETED_TIME,
    RETURN_VALUE
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'SALES_PIPELINE_TASK',
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP()),
    RESULT_LIMIT => 10
))
ORDER BY SCHEDULED_TIME DESC;
```

5. **Verifica el total de datos procesados**:

```sql
-- Total acumulado en SALES_PROCESSED
SELECT COUNT(*) AS total_procesadas FROM SALES_PROCESSED;
-- Esperado: 7

-- Resumen por región
SELECT
    REGION,
    COUNT(*) AS ventas,
    ROUND(SUM(DESCUENTO_CALC), 2) AS total_descuentos,
    ROUND(SUM(PRECIO_FINAL), 2) AS total_ingresos_netos
FROM SALES_PROCESSED
GROUP BY REGION
ORDER BY total_ingresos_netos DESC;
```

6. **Consulta el estado del Stream**:

```sql
-- Información del Stream
SHOW STREAMS LIKE 'SALES_STREAM' IN SCHEMA INGEST_SCHEMA;

-- Verificar que está vacío tras el consumo
SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') AS tiene_datos;
-- Esperado: FALSE
```

### Resultado Esperado

El `TASK_HISTORY` debe mostrar al menos 2 ejecuciones con `STATE = 'SUCCEEDED'`:

```
+------------------------+-----------+---------------------+---------------------+--------------------------------------------------+
| NAME                   | STATE     | SCHEDULED_TIME      | COMPLETED_TIME      | RETURN_VALUE                                     |
|------------------------+-----------+---------------------+---------------------+--------------------------------------------------|
| SALES_PIPELINE_TASK    | SUCCEEDED | 2024-xx-xx ...      | 2024-xx-xx ...      | Pipeline ejecutado exitosamente. Filas proc...   |
| SALES_PIPELINE_TASK    | SUCCEEDED | 2024-xx-xx ...      | 2024-xx-xx ...      | Pipeline ejecutado exitosamente. Filas proc...   |
+------------------------+-----------+---------------------+---------------------+--------------------------------------------------+
```

### Verificación

```sql
-- No debe haber ejecuciones fallidas
SELECT COUNT(*) AS ejecuciones_fallidas
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'SALES_PIPELINE_TASK',
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP())
))
WHERE STATE = 'FAILED';
-- Esperado: 0
```

---

## Validación y Testing Final

Ejecuta el siguiente bloque de validación integral para confirmar que todos los componentes del pipeline funcionan correctamente:

```sql
-- ============================================
-- VALIDACIÓN INTEGRAL DEL PIPELINE
-- ============================================

USE ROLE SYSADMIN;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

-- 1. Verificar que Time Travel funcionó (tabla tiene filas originales)
SELECT
    CASE WHEN COUNT(*) >= 500000 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_time_travel
FROM SALES_RAW;

-- 2. Verificar existencia del clon
SELECT
    CASE WHEN COUNT(*) > 0 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_clone
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES
WHERE DATABASE_NAME = 'LAB_DB_DEV' AND DELETED IS NULL;

-- 3. Verificar UDF
SELECT
    CASE WHEN CALCULATE_DISCOUNT(1500, 'NORTH') = 225 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_udf;

-- 4. Verificar Stream existe
SELECT
    CASE WHEN COUNT(*) > 0 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_stream
FROM INFORMATION_SCHEMA.TABLES  -- Usamos SHOW como alternativa
WHERE 1=0; -- Placeholder

SHOW STREAMS LIKE 'SALES_STREAM' IN SCHEMA INGEST_SCHEMA;

-- 5. Verificar Task activa
SHOW TASKS LIKE 'SALES_PIPELINE_TASK' IN SCHEMA INGEST_SCHEMA;

-- 6. Verificar datos procesados
SELECT
    CASE WHEN COUNT(*) >= 7 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_sales_processed
FROM SALES_PROCESSED;

-- 7. Verificar ejecuciones exitosas
SELECT
    CASE WHEN COUNT(*) >= 2 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_task_history
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    TASK_NAME => 'SALES_PIPELINE_TASK',
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP())
))
WHERE STATE = 'SUCCEEDED';
```

### Criterios de Éxito

| Componente | Criterio | Estado |
|------------|----------|--------|
| Time Travel | SALES_RAW tiene ≥ 500,000 filas restauradas | ✓ |
| Clone | LAB_DB_DEV existe y es accesible | ✓ |
| UDF | CALCULATE_DISCOUNT retorna valores correctos | ✓ |
| Stored Procedure | SP_PROCESS_INCREMENTAL ejecuta sin errores | ✓ |
| Stream | SALES_STREAM existe y captura cambios | ✓ |
| Task | SALES_PIPELINE_TASK en estado 'started' | ✓ |
| Pipeline E2E | SALES_PROCESSED tiene ≥ 7 filas procesadas | ✓ |
| Monitoreo | TASK_HISTORY muestra ≥ 2 ejecuciones exitosas | ✓ |

---

## Solución de Problemas

### Problema 1: La Task no se ejecuta — estado SKIPPED en TASK_HISTORY

**Síntomas:**
- `EXECUTE TASK` no produce resultados en `SALES_PROCESSED`
- `TASK_HISTORY` muestra `STATE = 'SKIPPED'` con `CONDITION_TEXT` indicando que la condición WHEN fue FALSE

**Causa:**
La cláusula `WHEN SYSTEM$STREAM_HAS_DATA(...)` evalúa a FALSE porque el Stream no tiene datos pendientes. Esto ocurre cuando: (a) el Stream fue consumido previamente por una consulta SELECT en una transacción auto-commit, o (b) los INSERT se realizaron antes de crear el Stream.

**Solución:**

```sql
-- Verificar si el Stream tiene datos
SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') AS tiene_datos;

-- Si es FALSE, insertar nuevos datos DESPUÉS de que el Stream existe
INSERT INTO SALES_RAW
    (ID_VENTA, ID_CLIENTE, ID_PRODUCTO, FECHA_VENTA, CANTIDAD, PRECIO_VENTA, DESCUENTO, CANAL_VENTA, REGION)
VALUES
    (900099, 1099, 2099, '2024-12-05', 1, 500.00, 0.05, 'ONLINE', 'NORTH');

-- Verificar de nuevo
SELECT SYSTEM$STREAM_HAS_DATA('SALES_STREAM') AS tiene_datos;
-- Ahora debe ser TRUE

-- Ejecutar manualmente
EXECUTE TASK SALES_PIPELINE_TASK;
```

---

### Problema 2: Error "Insufficient privileges" al ejecutar la Task o el Stored Procedure

**Síntomas:**
- `TASK_HISTORY` muestra `STATE = 'FAILED'` con `ERROR_MESSAGE` conteniendo "SQL access control error: Insufficient privileges to operate on..."
- La ejecución manual con `CALL SP_PROCESS_INCREMENTAL()` falla con error de permisos

**Causa:**
El rol owner de la Task no tiene privilegios suficientes sobre todos los objetos referenciados: tabla de origen, tabla de destino, Stream, UDF o warehouse. Esto es común cuando la Task fue creada con un rol diferente al que tiene los grants.

**Solución:**

```sql
USE ROLE ACCOUNTADMIN;

-- Verificar el owner actual de la Task
SHOW TASKS LIKE 'SALES_PIPELINE_TASK' IN SCHEMA LAB_DB.INGEST_SCHEMA;
-- Revisar la columna "owner"

-- Otorgar todos los privilegios necesarios al rol owner
GRANT ALL PRIVILEGES ON TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW TO ROLE SYSADMIN;
GRANT ALL PRIVILEGES ON TABLE LAB_DB.INGEST_SCHEMA.SALES_PROCESSED TO ROLE SYSADMIN;
GRANT SELECT ON STREAM LAB_DB.INGEST_SCHEMA.SALES_STREAM TO ROLE SYSADMIN;
GRANT USAGE ON FUNCTION LAB_DB.INGEST_SCHEMA.CALCULATE_DISCOUNT(FLOAT, STRING) TO ROLE SYSADMIN;
GRANT USAGE ON WAREHOUSE LAB_WH_XS TO ROLE SYSADMIN;

-- Alternativamente, transferir ownership de la Task al rol con privilegios
ALTER TASK LAB_DB.INGEST_SCHEMA.SALES_PIPELINE_TASK SET OWNER = SYSADMIN;

-- Reactivar la Task después de cambios
USE ROLE SYSADMIN;
ALTER TASK SALES_PIPELINE_TASK RESUME;
```

---

## Limpieza

Una vez completado el laboratorio y validados todos los resultados, ejecuta la siguiente limpieza para suspender la Task y evitar consumo innecesario de créditos:

```sql
USE ROLE SYSADMIN;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

-- 1. IMPORTANTE: Suspender la Task para detener ejecuciones automáticas
ALTER TASK SALES_PIPELINE_TASK SUSPEND;

-- Verificar que está suspendida
SHOW TASKS LIKE 'SALES_PIPELINE_TASK' IN SCHEMA INGEST_SCHEMA;
-- state debe ser "suspended"

-- 2. (OPCIONAL) Eliminar datos de prueba de SALES_RAW
DELETE FROM SALES_RAW WHERE ID_VENTA >= 900000;

-- 3. (OPCIONAL) Eliminar el clon de desarrollo si ya no se necesita
-- DROP DATABASE LAB_DB_DEV;

-- NOTA: NO eliminar la Task, Stream, SP ni UDF si planeas usarlos
-- en laboratorios posteriores o como referencia.
```

> ⚠️ **IMPORTANTE:** Siempre suspende las Tasks al finalizar un laboratorio. Las Tasks activas consumen créditos del warehouse cada vez que se ejecutan según el schedule CRON.

---

## Resumen

En este laboratorio has construido un pipeline CDC incremental de extremo a extremo en Snowflake, integrando cinco capacidades fundamentales de la plataforma:

| Componente | Función en el Pipeline |
|------------|----------------------|
| **Time Travel** | Recuperación de datos eliminados accidentalmente usando `AT(TIMESTAMP)` |
| **Zero-Copy Cloning** | Ambiente de desarrollo aislado sin duplicar almacenamiento |
| **UDF JavaScript** | Lógica de negocio encapsulada (`CALCULATE_DISCOUNT`) para descuentos por región |
| **Stored Procedure** | Orquestación del MERGE incremental con manejo de estado |
| **Stream** | Captura automática de cambios CDC sobre `SALES_RAW` |
| **Task CRON** | Automatización del pipeline cada 5 minutos con condición `WHEN` |
| **TASK_HISTORY** | Monitoreo y diagnóstico de ejecuciones del pipeline |

### Conceptos Clave Aprendidos

1. Los Streams no duplican datos; mantienen un offset que avanza solo tras un COMMIT exitoso
2. Las Tasks se crean en estado SUSPENDED y requieren `ALTER TASK ... RESUME` para activarse
3. La cláusula `WHEN SYSTEM$STREAM_HAS_DATA()` evita consumo innecesario de créditos
4. El patrón MERGE + Stream es el estándar para CDC incremental en Snowflake
5. El desarrollo en clones zero-copy permite probar sin riesgo para producción

### Recursos Adicionales

- [Documentación oficial: Streams](https://docs.snowflake.com/en/user-guide/streams)
- [Documentación oficial: Tasks](https://docs.snowflake.com/en/user-guide/tasks-intro)
- [Documentación oficial: Time Travel](https://docs.snowflake.com/en/user-guide/data-time-travel)
- [Documentación oficial: Zero-Copy Cloning](https://docs.snowflake.com/en/user-guide/tables-storage-considerations#label-cloning-tables)
- [Documentación oficial: JavaScript UDFs](https://docs.snowflake.com/en/developer-guide/udf/javascript/udf-javascript-introduction)
