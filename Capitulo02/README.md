# Creación de Warehouses, Bases de Datos, Esquemas y Tablas

## Metadata

| Campo | Detalle |
|-------|---------|
| **Duración** | 60 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |
| **Rol principal** | SYSADMIN |
| **Edición Snowflake** | Enterprise Trial |

---

## Descripción General

En este laboratorio construirás la infraestructura base que será utilizada en todos los laboratorios posteriores del curso. Crearás virtual warehouses dedicados con políticas de auto-suspend y auto-resume, la base de datos principal con sus tres esquemas arquitectónicos (staging, core, analytics), y las tablas iniciales del modelo de ventas. Además, observarás y documentarás el ciclo de vida de un warehouse (Started → Suspended → Resumed) en tiempo real.

---

## Objetivos de Aprendizaje

- [ ] Crear un virtual warehouse dedicado de tamaño X-Small con auto-suspend en 60 segundos y auto-resume habilitado
- [ ] Crear la base de datos principal `DB_CURSO_SNOWFLAKE` con los esquemas `SCH_STAGING`, `SCH_CORE` y `SCH_ANALYTICS`
- [ ] Crear tablas iniciales vacías con tipos de datos nativos de Snowflake aplicando convenciones de nomenclatura establecidas
- [ ] Observar y documentar el comportamiento de auto-suspend y auto-resume del warehouse mediante consultas controladas
- [ ] Comprender la diferencia entre warehouses de un solo cluster y multi-cluster, y entre tablas permanentes, temporales y transitorias

---

## Prerrequisitos

### Conocimiento previo
- Laboratorio 01-02-01 completado: cuenta Snowflake activa y navegación en Snowsight dominada
- Comprensión básica de DDL SQL: `CREATE`, `DROP`, `ALTER`
- Familiaridad con tipos de datos SQL estándar (VARCHAR, NUMBER, DATE, BOOLEAN)

### Acceso requerido
- Cuenta Snowflake Enterprise Trial activa (AWS, us-east-1)
- Rol `SYSADMIN` disponible (verificado en laboratorio anterior)
- Rol `ACCOUNTADMIN` disponible para verificaciones puntuales
- Navegador web moderno con acceso a Snowsight

---

## Entorno del Laboratorio

### Software requerido

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Snowflake Trial Account | Enterprise Edition 8.x | Plataforma principal |
| Snowsight (Web UI) | 2024.4 | Interfaz de ejecución |
| Navegador web | Chrome 124+ / Firefox 125+ / Edge 124+ | Acceso a Snowsight |

### Configuración inicial

Antes de comenzar, asegúrate de:
1. Iniciar sesión en Snowsight: `https://<tu_account_identifier>.snowflakecomputing.com`
2. Verificar que puedes cambiar al rol `SYSADMIN` desde el selector de roles en la esquina superior izquierda
3. Crear una nueva Worksheet nombrada **LAB02_Infraestructura** (menú `+` → SQL Worksheet → renombrar)

---

## Paso a Paso

### Paso 1: Configurar el contexto de sesión

**Objetivo:** Establecer el rol correcto y crear la worksheet de trabajo para el laboratorio.

**Instrucciones:**

1. En Snowsight, abre la worksheet **LAB02_Infraestructura** que acabas de crear.

2. Ejecuta los siguientes comandos para establecer el contexto de sesión:

```sql
-- Establecer el rol SYSADMIN para crear objetos
USE ROLE SYSADMIN;

-- Verificar el rol activo
SELECT CURRENT_ROLE();
```

3. Verifica que no existan objetos previos que puedan generar conflictos:

```sql
-- Listar warehouses existentes
SHOW WAREHOUSES;

-- Listar bases de datos existentes
SHOW DATABASES;
```

**Resultado esperado:**
- `CURRENT_ROLE()` devuelve `SYSADMIN`.
- La lista de warehouses muestra únicamente el warehouse `COMPUTE_WH` por defecto (si existe).
- La lista de bases de datos muestra solo las bases del sistema (`SNOWFLAKE`, `SNOWFLAKE_SAMPLE_DATA`).

**Verificación:**
- Si el rol no es `SYSADMIN`, verifica que tu usuario tenga el rol asignado desde `ACCOUNTADMIN` → Security → Users.

---

### Paso 2: Crear el Virtual Warehouse principal (WH_CURSO_XS)

**Objetivo:** Crear el warehouse de trabajo principal con auto-suspend de 60 segundos y auto-resume habilitado.

**Instrucciones:**

1. Ejecuta el comando de creación del warehouse:

```sql
-- Crear warehouse principal para consultas analíticas
CREATE WAREHOUSE WH_CURSO_XS
  WAREHOUSE_SIZE = 'X-SMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Warehouse principal del curso - consultas analíticas (labs 02-04)';
```

2. Verifica la creación y los parámetros configurados:

```sql
-- Mostrar detalles del warehouse recién creado
SHOW WAREHOUSES LIKE 'WH_CURSO_XS';
```

3. Examina las propiedades detalladas:

```sql
-- Ver todas las propiedades del warehouse
DESCRIBE WAREHOUSE WH_CURSO_XS;
```

4. Establece el warehouse como activo en la sesión:

```sql
-- Usar el warehouse creado
USE WAREHOUSE WH_CURSO_XS;
```

**Resultado esperado:**

| Propiedad | Valor esperado |
|-----------|---------------|
| name | WH_CURSO_XS |
| size | X-Small |
| state | SUSPENDED (inicialmente) |
| auto_suspend | 60 |
| auto_resume | true |
| min_cluster_count | 1 |
| max_cluster_count | 1 |

**Verificación:**
- El comando `SHOW WAREHOUSES LIKE 'WH_CURSO_XS'` devuelve exactamente una fila.
- La columna `state` muestra `SUSPENDED` porque se creó con `INITIALLY_SUSPENDED = TRUE`.
- Al ejecutar `USE WAREHOUSE WH_CURSO_XS`, el estado cambiará a `STARTED` (auto-resume en acción).

---

### Paso 3: Observar el comportamiento de Auto-Resume y Auto-Suspend

**Objetivo:** Documentar el ciclo de vida del warehouse observando las transiciones de estado.

**Instrucciones:**

1. Ejecuta una consulta simple para activar el warehouse (auto-resume):

```sql
-- Esta consulta activará el warehouse automáticamente
SELECT CURRENT_TIMESTAMP() AS momento_activacion, 'Warehouse activado por auto-resume' AS evento;
```

2. Verifica inmediatamente el estado del warehouse:

```sql
-- Verificar que el warehouse está activo
SHOW WAREHOUSES LIKE 'WH_CURSO_XS';
-- Observa la columna "state": debe ser "Started"
```

3. **Espera 70 segundos** sin ejecutar ninguna consulta. Puedes usar un temporizador. El auto-suspend está configurado en 60 segundos.

4. Después de esperar, verifica el estado del warehouse:

```sql
-- Verificar el estado después de 70 segundos de inactividad
SHOW WAREHOUSES LIKE 'WH_CURSO_XS';
-- Observa la columna "state": debe ser "Suspended"
```

5. Ejecuta otra consulta para observar el auto-resume:

```sql
-- Esta consulta disparará el auto-resume
SELECT CURRENT_TIMESTAMP() AS momento_resume, 'Warehouse reactivado por auto-resume' AS evento;
```

6. Consulta el historial de actividad del warehouse:

```sql
-- Consultar historial de uso del warehouse (últimos 30 minutos)
SELECT 
    START_TIME,
    END_TIME,
    WAREHOUSE_NAME,
    CREDITS_USED,
    CREDITS_USED_COMPUTE,
    CREDITS_USED_CLOUD_SERVICES
FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_METERING_HISTORY(
    DATE_RANGE_START => DATEADD('MINUTE', -30, CURRENT_TIMESTAMP()),
    WAREHOUSE_NAME => 'WH_CURSO_XS'
));
```

**Resultado esperado:**
- Primer `SHOW WAREHOUSES` después de la consulta: `state = Started`
- Segundo `SHOW WAREHOUSES` después de 70 segundos: `state = Suspended`
- La consulta de historial muestra al menos un registro con créditos consumidos (valor pequeño, cercano a 0.0167 créditos por 1 minuto de uso en X-Small)

**Verificación:**
- Si el warehouse no se suspende después de 70 segundos, espera unos segundos más (el timer no es exacto al milisegundo).
- Si la consulta de historial no devuelve resultados, es normal durante los primeros minutos de una cuenta nueva. Los datos de metering pueden tardar hasta 3 horas en aparecer.

> **Nota importante:** Documenta mentalmente los tiempos observados. En un entorno real, el auto-resume típicamente tarda 1-3 segundos para warehouses X-Small.

---

### Paso 4: Crear la Base de Datos y los Esquemas

**Objetivo:** Crear la base de datos principal del curso con la arquitectura de tres capas (staging, core, analytics).

**Instrucciones:**

1. Crea la base de datos principal:

```sql
-- Crear la base de datos principal del curso
CREATE DATABASE DB_CURSO_SNOWFLAKE
  COMMENT = 'Base de datos principal del curso Snowflake - modelo de ventas';
```

2. Verifica la creación:

```sql
-- Verificar la base de datos creada
SHOW DATABASES LIKE 'DB_CURSO_SNOWFLAKE';
```

3. Establece la base de datos como contexto activo:

```sql
-- Usar la base de datos
USE DATABASE DB_CURSO_SNOWFLAKE;
```

4. Crea los tres esquemas arquitectónicos:

```sql
-- Esquema para datos crudos (staging)
CREATE SCHEMA SCH_STAGING
  COMMENT = 'Capa de staging: datos crudos, stages y file formats';

-- Esquema para datos procesados (core)
CREATE SCHEMA SCH_CORE
  COMMENT = 'Capa core: tablas dimensionales y de hechos normalizadas';

-- Esquema para vistas y agregados (analytics)
CREATE SCHEMA SCH_ANALYTICS
  COMMENT = 'Capa analytics: vistas, vistas seguras y agregaciones';
```

5. Verifica los esquemas creados:

```sql
-- Listar todos los esquemas de la base de datos
SHOW SCHEMAS IN DATABASE DB_CURSO_SNOWFLAKE;
```

**Resultado esperado:**

La consulta `SHOW SCHEMAS` debe devolver 5 esquemas:
| name | comment |
|------|---------|
| INFORMATION_SCHEMA | (sistema - creado automáticamente) |
| PUBLIC | (esquema por defecto - creado automáticamente) |
| SCH_STAGING | Capa de staging: datos crudos, stages y file formats |
| SCH_CORE | Capa core: tablas dimensionales y de hechos normalizadas |
| SCH_ANALYTICS | Capa analytics: vistas, vistas seguras y agregaciones |

**Verificación:**
- Los tres esquemas personalizados aparecen en la lista.
- El esquema `PUBLIC` existe automáticamente pero no se usará en este curso.
- El esquema `INFORMATION_SCHEMA` es de solo lectura y contiene vistas del sistema.

---

### Paso 5: Crear las Tablas de Staging

**Objetivo:** Crear las tablas iniciales del modelo de ventas en el esquema `SCH_STAGING` con tipos de datos nativos de Snowflake.

**Instrucciones:**

1. Establece el esquema de trabajo:

```sql
-- Posicionarse en el esquema staging
USE SCHEMA DB_CURSO_SNOWFLAKE.SCH_STAGING;
```

2. Crea la tabla de clientes:

```sql
-- Tabla de staging para clientes
CREATE TABLE STG_CLIENTES (
    ID_CLIENTE      NUMBER(10,0)      NOT NULL,
    NOMBRE          VARCHAR(50)       NOT NULL,
    APELLIDO        VARCHAR(50)       NOT NULL,
    EMAIL           VARCHAR(100),
    TELEFONO        VARCHAR(20),
    CIUDAD          VARCHAR(50),
    PAIS            VARCHAR(50),
    SEGMENTO        VARCHAR(30),
    FECHA_REGISTRO  DATE,
    ACTIVO          BOOLEAN           DEFAULT TRUE,
    CONSTRAINT PK_STG_CLIENTES PRIMARY KEY (ID_CLIENTE)
)
COMMENT = 'Tabla staging de clientes - datos crudos de ingesta';
```

3. Crea la tabla de productos:

```sql
-- Tabla de staging para productos
CREATE TABLE STG_PRODUCTOS (
    ID_PRODUCTO      NUMBER(10,0)     NOT NULL,
    NOMBRE_PRODUCTO  VARCHAR(100)     NOT NULL,
    CATEGORIA        VARCHAR(50),
    SUBCATEGORIA     VARCHAR(50),
    PRECIO_UNITARIO  NUMBER(12,2),
    COSTO_UNITARIO   NUMBER(12,2),
    UNIDAD_MEDIDA    VARCHAR(20),
    ACTIVO           BOOLEAN          DEFAULT TRUE,
    CONSTRAINT PK_STG_PRODUCTOS PRIMARY KEY (ID_PRODUCTO)
)
COMMENT = 'Tabla staging de productos - catálogo de productos';
```

4. Crea la tabla de ventas:

```sql
-- Tabla de staging para ventas (tabla de hechos)
CREATE TABLE STG_VENTAS (
    ID_VENTA      NUMBER(15,0)       NOT NULL,
    ID_CLIENTE    NUMBER(10,0)       NOT NULL,
    ID_PRODUCTO   NUMBER(10,0)       NOT NULL,
    FECHA_VENTA   TIMESTAMP_NTZ      NOT NULL,
    CANTIDAD      NUMBER(10,0),
    PRECIO_VENTA  NUMBER(12,2),
    DESCUENTO     NUMBER(5,2)        DEFAULT 0.00,
    CANAL_VENTA   VARCHAR(30),
    REGION        VARCHAR(50),
    CONSTRAINT PK_STG_VENTAS PRIMARY KEY (ID_VENTA)
)
COMMENT = 'Tabla staging de ventas - transacciones de venta';
```

5. Verifica las tablas creadas:

```sql
-- Listar tablas en el esquema staging
SHOW TABLES IN SCHEMA DB_CURSO_SNOWFLAKE.SCH_STAGING;
```

6. Examina la estructura de cada tabla:

```sql
-- Describir la estructura de STG_CLIENTES
DESCRIBE TABLE STG_CLIENTES;

-- Describir la estructura de STG_PRODUCTOS
DESCRIBE TABLE STG_PRODUCTOS;

-- Describir la estructura de STG_VENTAS
DESCRIBE TABLE STG_VENTAS;
```

**Resultado esperado:**

`SHOW TABLES` devuelve 3 tablas:
| name | kind | comment |
|------|------|---------|
| STG_CLIENTES | TABLE | Tabla staging de clientes - datos crudos de ingesta |
| STG_PRODUCTOS | TABLE | Tabla staging de productos - catálogo de productos |
| STG_VENTAS | TABLE | Tabla staging de ventas - transacciones de venta |

`DESCRIBE TABLE STG_CLIENTES` muestra las 10 columnas con sus tipos de datos correctos.

**Verificación:**
- Todas las columnas `NOT NULL` están marcadas correctamente.
- Los tipos de datos coinciden con la especificación: `NUMBER`, `VARCHAR`, `DATE`, `TIMESTAMP_NTZ`, `BOOLEAN`.
- Las restricciones de clave primaria están presentes (columna `primary key` = `Y`).

---

### Paso 6: Explorar Tipos de Tablas — Permanentes, Temporales y Transitorias

**Objetivo:** Comprender las diferencias entre los tres tipos de tablas en Snowflake y cuándo usar cada uno.

**Instrucciones:**

1. Crea una tabla temporal para demostración:

```sql
-- Tabla temporal: solo existe durante la sesión activa
CREATE TEMPORARY TABLE TMP_TEST (
    ID          NUMBER(5,0),
    DESCRIPCION VARCHAR(100),
    FECHA_CARGA TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
)
COMMENT = 'Tabla temporal de prueba - se eliminará al cerrar sesión';
```

2. Crea una tabla transitoria para comparación:

```sql
-- Tabla transitoria: persiste entre sesiones pero sin Fail-safe (7 días menos de protección)
CREATE TRANSIENT TABLE STG_TRANSIENT_TEST (
    ID          NUMBER(5,0),
    VALOR       VARCHAR(50)
)
COMMENT = 'Tabla transitoria de prueba - sin periodo Fail-safe';
```

3. Inserta datos de prueba para verificar funcionalidad:

```sql
-- Insertar datos en la tabla temporal
INSERT INTO TMP_TEST (ID, DESCRIPCION) VALUES 
  (1, 'Registro temporal de prueba'),
  (2, 'Este dato desaparecerá al cerrar sesión');

-- Verificar
SELECT * FROM TMP_TEST;
```

4. Compara las propiedades de los tres tipos de tabla:

```sql
-- Consultar metadata de todas las tablas del esquema
SELECT 
    TABLE_NAME,
    TABLE_TYPE,
    IS_TRANSIENT,
    RETENTION_TIME,
    ROW_COUNT,
    COMMENT
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'SCH_STAGING'
ORDER BY TABLE_NAME;
```

**Resultado esperado:**

| TABLE_NAME | TABLE_TYPE | IS_TRANSIENT | RETENTION_TIME |
|------------|-----------|--------------|----------------|
| STG_CLIENTES | BASE TABLE | NO | 1 |
| STG_PRODUCTOS | BASE TABLE | NO | 1 |
| STG_TRANSIENT_TEST | BASE TABLE | YES | 0 o 1 |
| STG_VENTAS | BASE TABLE | NO | 1 |
| TMP_TEST | LOCAL TEMPORARY | NO | 0 |

**Verificación:**

Observa las diferencias clave:

| Característica | Permanente | Transitoria | Temporal |
|---------------|-----------|-------------|----------|
| Persiste entre sesiones | ✅ | ✅ | ❌ |
| Time Travel (hasta 90 días) | ✅ | ✅ (máx 1 día) | ✅ (máx 1 día) |
| Fail-safe (7 días) | ✅ | ❌ | ❌ |
| Visible para otros usuarios | ✅ | ✅ | ❌ |
| Costo de almacenamiento | Mayor | Menor | Mínimo |

5. Elimina la tabla transitoria de prueba (la temporal se eliminará sola):

```sql
-- Limpiar tabla transitoria de prueba
DROP TABLE STG_TRANSIENT_TEST;
```

---

### Paso 7: Consultar Metadata con INFORMATION_SCHEMA

**Objetivo:** Utilizar las vistas del sistema para verificar programáticamente todos los objetos creados.

**Instrucciones:**

1. Lista todas las columnas de las tablas de staging:

```sql
-- Consultar columnas de todas las tablas en SCH_STAGING
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH,
    NUMERIC_PRECISION,
    NUMERIC_SCALE,
    IS_NULLABLE,
    COLUMN_DEFAULT
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'SCH_STAGING'
  AND TABLE_NAME IN ('STG_CLIENTES', 'STG_PRODUCTOS', 'STG_VENTAS')
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

2. Verifica los esquemas creados:

```sql
-- Consultar esquemas desde INFORMATION_SCHEMA
SELECT 
    SCHEMA_NAME,
    SCHEMA_OWNER,
    CREATED,
    COMMENT
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME NOT IN ('INFORMATION_SCHEMA')
ORDER BY CREATED;
```

3. Cuenta los objetos creados como resumen:

```sql
-- Resumen de objetos creados en este laboratorio
SELECT 'Esquemas' AS tipo_objeto, COUNT(*) AS cantidad
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME NOT IN ('INFORMATION_SCHEMA', 'PUBLIC')
UNION ALL
SELECT 'Tablas permanentes', COUNT(*)
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'SCH_STAGING' AND TABLE_TYPE = 'BASE TABLE' AND IS_TRANSIENT = 'NO'
UNION ALL
SELECT 'Columnas totales', COUNT(*)
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'SCH_STAGING'
  AND TABLE_NAME IN ('STG_CLIENTES', 'STG_PRODUCTOS', 'STG_VENTAS');
```

**Resultado esperado:**

| tipo_objeto | cantidad |
|-------------|----------|
| Esquemas | 3 |
| Tablas permanentes | 3 |
| Columnas totales | 27 |

**Verificación:**
- 3 esquemas: `SCH_STAGING`, `SCH_CORE`, `SCH_ANALYTICS`
- 3 tablas permanentes: `STG_CLIENTES` (10 columnas), `STG_PRODUCTOS` (8 columnas), `STG_VENTAS` (9 columnas)
- Total de columnas: 10 + 8 + 9 = 27

---

### Paso 8: Crear el Segundo Warehouse para Cargas de Datos

**Objetivo:** Crear el warehouse `WH_CURSO_LOAD_XS` dedicado exclusivamente a operaciones de carga (preparación para Lab 05).

**Instrucciones:**

1. Crea el warehouse de carga:

```sql
-- Warehouse dedicado para cargas de datos (ETL/ELT)
CREATE WAREHOUSE WH_CURSO_LOAD_XS
  WAREHOUSE_SIZE = 'X-SMALL'
  AUTO_SUSPEND = 120
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Warehouse dedicado para cargas batch de datos (lab 05)';
```

2. Verifica ambos warehouses creados:

```sql
-- Listar todos los warehouses del curso
SHOW WAREHOUSES LIKE 'WH_CURSO%';
```

3. Compara las configuraciones:

```sql
-- Comparar parámetros de ambos warehouses
SELECT 
    "name" AS WAREHOUSE_NAME,
    "size" AS TAMANO,
    "state" AS ESTADO,
    "auto_suspend" AS AUTO_SUSPEND_SEG,
    "auto_resume" AS AUTO_RESUME,
    "min_cluster_count" AS MIN_CLUSTERS,
    "max_cluster_count" AS MAX_CLUSTERS,
    "comment" AS COMENTARIO
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

**Resultado esperado:**

| WAREHOUSE_NAME | TAMANO | ESTADO | AUTO_SUSPEND_SEG | AUTO_RESUME | MIN_CLUSTERS | MAX_CLUSTERS |
|---------------|--------|--------|-----------------|-------------|--------------|--------------|
| WH_CURSO_LOAD_XS | X-Small | SUSPENDED | 120 | true | 1 | 1 |
| WH_CURSO_XS | X-Small | STARTED | 60 | true | 1 | 1 |

**Verificación:**
- Ambos warehouses son X-Small (1 crédito/hora cuando activos).
- `WH_CURSO_XS` tiene auto-suspend de 60s (para consultas interactivas rápidas).
- `WH_CURSO_LOAD_XS` tiene auto-suspend de 120s (las cargas de datos pueden tener pausas breves entre archivos).
- Ambos son single-cluster (`min_cluster_count = 1`, `max_cluster_count = 1`).

---

### Paso 9: Comprender Single-Cluster vs. Multi-Cluster

**Objetivo:** Demostrar conceptualmente la diferencia entre warehouses de un solo cluster y multi-cluster, y cuándo aplicar cada tipo de escalabilidad.

**Instrucciones:**

1. Examina la configuración actual (single-cluster):

```sql
-- Nuestros warehouses son single-cluster (max_cluster_count = 1)
-- Esto es apropiado para desarrollo y cargas de baja concurrencia
SHOW WAREHOUSES LIKE 'WH_CURSO_XS';
```

2. Observa cómo se vería un warehouse multi-cluster (solo lectura, no lo crearemos permanentemente):

```sql
-- EJEMPLO CONCEPTUAL: Warehouse multi-cluster para alta concurrencia
-- (Ejecutar para ver la sintaxis, luego lo eliminaremos)
CREATE WAREHOUSE WH_EJEMPLO_MULTICLUSTER
  WAREHOUSE_SIZE = 'SMALL'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Ejemplo multi-cluster - solo para demostración';
```

3. Compara la configuración:

```sql
-- Ver la diferencia en cluster count
SHOW WAREHOUSES LIKE 'WH_%';
```

4. Observa que el warehouse multi-cluster muestra `max_cluster_count = 3`, lo que significa que puede escalar horizontalmente hasta 3 clústeres simultáneos bajo carga.

5. Elimina el warehouse de ejemplo:

```sql
-- Eliminar el warehouse de ejemplo (no lo necesitamos)
DROP WAREHOUSE WH_EJEMPLO_MULTICLUSTER;
```

6. Documenta la decisión arquitectónica ejecutando esta consulta informativa:

```sql
-- Resumen de la estrategia de warehouses del curso
SELECT 'WH_CURSO_XS' AS warehouse, 'Consultas analíticas' AS proposito, 
       'Single-cluster, XS, 60s suspend' AS configuracion,
       'Labs 02-04' AS uso
UNION ALL
SELECT 'WH_CURSO_LOAD_XS', 'Cargas de datos (ETL)', 
       'Single-cluster, XS, 120s suspend',
       'Lab 05'
UNION ALL
SELECT '(Futuro) Multi-cluster', 'Alta concurrencia BI', 
       'Multi-cluster, Medium, 3 max clusters',
       'Producción (no aplica al curso)';
```

**Resultado esperado:**

| warehouse | proposito | configuracion | uso |
|-----------|-----------|---------------|-----|
| WH_CURSO_XS | Consultas analíticas | Single-cluster, XS, 60s suspend | Labs 02-04 |
| WH_CURSO_LOAD_XS | Cargas de datos (ETL) | Single-cluster, XS, 120s suspend | Lab 05 |
| (Futuro) Multi-cluster | Alta concurrencia BI | Multi-cluster, Medium, 3 max clusters | Producción (no aplica al curso) |

**Verificación:**
- Comprendes que **escalabilidad vertical** = cambiar el tamaño (XS → Small → Medium) para consultas individuales más rápidas.
- Comprendes que **escalabilidad horizontal** = agregar más clústeres (multi-cluster) para más usuarios concurrentes.
- Para este curso, single-cluster X-Small es suficiente porque trabajas individualmente.

---

### Paso 10: Verificación Final y Establecer Contexto por Defecto

**Objetivo:** Confirmar que toda la infraestructura está correctamente creada y establecer el contexto de trabajo para futuros laboratorios.

**Instrucciones:**

1. Establece el contexto completo de trabajo:

```sql
-- Establecer contexto completo para los próximos laboratorios
USE ROLE SYSADMIN;
USE WAREHOUSE WH_CURSO_XS;
USE DATABASE DB_CURSO_SNOWFLAKE;
USE SCHEMA SCH_STAGING;
```

2. Ejecuta la verificación integral:

```sql
-- Verificación integral de todos los objetos creados
SELECT '✅ Rol activo' AS verificacion, CURRENT_ROLE() AS valor
UNION ALL
SELECT '✅ Warehouse activo', CURRENT_WAREHOUSE()
UNION ALL
SELECT '✅ Base de datos', CURRENT_DATABASE()
UNION ALL
SELECT '✅ Esquema activo', CURRENT_SCHEMA();
```

3. Verifica que las tablas están vacías y listas para recibir datos:

```sql
-- Confirmar que las tablas están vacías
SELECT 
    'STG_CLIENTES' AS tabla, COUNT(*) AS registros FROM STG_CLIENTES
UNION ALL
SELECT 'STG_PRODUCTOS', COUNT(*) FROM STG_PRODUCTOS
UNION ALL
SELECT 'STG_VENTAS', COUNT(*) FROM STG_VENTAS;
```

**Resultado esperado:**

Verificación de contexto:
| verificacion | valor |
|-------------|-------|
| ✅ Rol activo | SYSADMIN |
| ✅ Warehouse activo | WH_CURSO_XS |
| ✅ Base de datos | DB_CURSO_SNOWFLAKE |
| ✅ Esquema activo | SCH_STAGING |

Conteo de registros:
| tabla | registros |
|-------|-----------|
| STG_CLIENTES | 0 |
| STG_PRODUCTOS | 0 |
| STG_VENTAS | 0 |

**Verificación:**
- Los cuatro valores de contexto son correctos.
- Las tres tablas tienen 0 registros (se poblarán en laboratorios posteriores).

---

## Validación y Testing

Ejecuta el siguiente script de validación completo para confirmar que todos los objetivos del laboratorio se cumplieron:

```sql
-- ============================================
-- SCRIPT DE VALIDACIÓN - LAB 02-02-01
-- ============================================

USE ROLE SYSADMIN;

-- Test 1: Verificar warehouses
SELECT '1. Warehouses' AS test, 
       COUNT(*) AS resultado,
       CASE WHEN COUNT(*) = 2 THEN '✅ PASS' ELSE '❌ FAIL' END AS estado
FROM TABLE(RESULT_SCAN((SELECT LAST_QUERY_ID() FROM TABLE(RESULT_SCAN(
    (SHOW WAREHOUSES LIKE 'WH_CURSO%'))))))
WHERE "name" IN ('WH_CURSO_XS', 'WH_CURSO_LOAD_XS');
```

```sql
-- Test 2: Verificar base de datos
SHOW DATABASES LIKE 'DB_CURSO_SNOWFLAKE';
SELECT '2. Base de datos' AS test,
       CASE WHEN COUNT(*) = 1 THEN '✅ PASS' ELSE '❌ FAIL' END AS estado
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

```sql
-- Test 3: Verificar esquemas
SELECT '3. Esquemas' AS test,
       COUNT(*) AS esquemas_encontrados,
       CASE WHEN COUNT(*) = 3 THEN '✅ PASS' ELSE '❌ FAIL' END AS estado
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME IN ('SCH_STAGING', 'SCH_CORE', 'SCH_ANALYTICS');
```

```sql
-- Test 4: Verificar tablas y columnas
SELECT '4. Tablas staging' AS test,
       COUNT(DISTINCT TABLE_NAME) AS tablas,
       COUNT(*) AS columnas_totales,
       CASE WHEN COUNT(DISTINCT TABLE_NAME) = 3 AND COUNT(*) = 27 
            THEN '✅ PASS' ELSE '❌ FAIL' END AS estado
FROM DB_CURSO_SNOWFLAKE.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'SCH_STAGING'
  AND TABLE_NAME IN ('STG_CLIENTES', 'STG_PRODUCTOS', 'STG_VENTAS');
```

```sql
-- Test 5: Verificar auto-suspend del warehouse principal
SHOW WAREHOUSES LIKE 'WH_CURSO_XS';
SELECT '5. Auto-suspend WH_CURSO_XS' AS test,
       "auto_suspend" AS valor,
       CASE WHEN "auto_suspend" = 60 THEN '✅ PASS' ELSE '❌ FAIL' END AS estado
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "name" = 'WH_CURSO_XS';
```

```sql
-- Test 6: Verificar auto-suspend del warehouse de carga
SHOW WAREHOUSES LIKE 'WH_CURSO_LOAD_XS';
SELECT '6. Auto-suspend WH_CURSO_LOAD_XS' AS test,
       "auto_suspend" AS valor,
       CASE WHEN "auto_suspend" = 120 THEN '✅ PASS' ELSE '❌ FAIL' END AS estado
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "name" = 'WH_CURSO_LOAD_XS';
```

**Todos los tests deben mostrar ✅ PASS.**

---

## Solución de Problemas

### Problema 1: Error "Insufficient privileges" al crear el warehouse

**Síntomas:**
```
SQL access control error: Insufficient privileges to operate on warehouse 'WH_CURSO_XS'
```

**Causa:** El rol activo no tiene permisos para crear warehouses. Esto ocurre si estás usando un rol diferente a `SYSADMIN` o `ACCOUNTADMIN`, o si el rol `SYSADMIN` no fue correctamente asignado a tu usuario.

**Solución:**

```sql
-- Cambiar a ACCOUNTADMIN para verificar y reparar
USE ROLE ACCOUNTADMIN;

-- Verificar que SYSADMIN existe y tiene los grants necesarios
SHOW GRANTS TO ROLE SYSADMIN;

-- Asegurar que tu usuario tiene SYSADMIN
GRANT ROLE SYSADMIN TO USER <tu_usuario>;

-- Volver a SYSADMIN y reintentar
USE ROLE SYSADMIN;
CREATE WAREHOUSE WH_CURSO_XS ...;
```

---

### Problema 2: El warehouse no se suspende después de 60 segundos

**Síntomas:** Después de esperar más de 70 segundos sin ejecutar consultas, `SHOW WAREHOUSES` sigue mostrando `state = Started`.

**Causa:** El auto-suspend cuenta desde la última actividad del warehouse, incluyendo consultas internas del sistema o actividad de la pestaña de Snowsight que puede mantener el warehouse activo (por ejemplo, la función de autocompletado o la actualización de la vista de resultados). También puede haber otras worksheets abiertas usando el mismo warehouse.

**Solución:**

```sql
-- Opción 1: Suspender manualmente para verificar que funciona
ALTER WAREHOUSE WH_CURSO_XS SUSPEND;

-- Verificar estado
SHOW WAREHOUSES LIKE 'WH_CURSO_XS';
-- Debe mostrar state = "Suspended"

-- Opción 2: Verificar que el auto-suspend está correctamente configurado
ALTER WAREHOUSE WH_CURSO_XS SET AUTO_SUSPEND = 60;

-- Opción 3: Cerrar otras worksheets/pestañas que puedan estar usando el warehouse
-- y esperar nuevamente 70+ segundos
```

> **Consejo:** Para observar el auto-suspend de forma más confiable, cierra todas las demás pestañas de Snowsight y no hagas clic en ningún lugar de la interfaz durante el periodo de espera.

---

## Limpieza

> ⚠️ **NO ejecutes la limpieza de este laboratorio.** Los objetos creados aquí (`DB_CURSO_SNOWFLAKE`, `WH_CURSO_XS`, `WH_CURSO_LOAD_XS` y todas las tablas) son la base estructural para los laboratorios 03, 04 y 05.

Si necesitas recrear el laboratorio desde cero (por ejemplo, en caso de error grave), usa este script de limpieza:

```sql
-- ⚠️ SOLO USAR SI NECESITAS REINICIAR COMPLETAMENTE EL LABORATORIO
-- Esto eliminará TODOS los objetos creados

USE ROLE SYSADMIN;

DROP DATABASE IF EXISTS DB_CURSO_SNOWFLAKE;
DROP WAREHOUSE IF EXISTS WH_CURSO_XS;
DROP WAREHOUSE IF EXISTS WH_CURSO_LOAD_XS;
```

---

## Resumen

En este laboratorio completaste exitosamente las siguientes tareas fundamentales:

| Objetivo | Objeto creado | Estado |
|----------|--------------|--------|
| Warehouse para consultas | `WH_CURSO_XS` (XS, 60s suspend, auto-resume ON) | ✅ |
| Warehouse para cargas | `WH_CURSO_LOAD_XS` (XS, 120s suspend, auto-resume ON) | ✅ |
| Base de datos principal | `DB_CURSO_SNOWFLAKE` | ✅ |
| Esquema staging | `SCH_STAGING` | ✅ |
| Esquema core | `SCH_CORE` | ✅ |
| Esquema analytics | `SCH_ANALYTICS` | ✅ |
| Tabla clientes | `STG_CLIENTES` (10 columnas) | ✅ |
| Tabla productos | `STG_PRODUCTOS` (8 columnas) | ✅ |
| Tabla ventas | `STG_VENTAS` (9 columnas) | ✅ |
| Observación auto-suspend/resume | Ciclo de vida documentado | ✅ |

### Conceptos clave reforzados:
- **Separación de warehouses por carga de trabajo** previene interferencia entre consultas analíticas y procesos de carga.
- **Auto-suspend + auto-resume** optimizan costos automáticamente sin intervención manual.
- **Escalabilidad vertical** (resize) acelera consultas individuales; **escalabilidad horizontal** (multi-cluster) atiende más usuarios concurrentes.
- **Tablas permanentes** ofrecen máxima protección (Time Travel + Fail-safe); **temporales** y **transitorias** reducen costos de almacenamiento a costa de menor protección.
- **INFORMATION_SCHEMA** permite consultar metadata de forma programática para auditoría y validación.

### Recursos adicionales:
- [Documentación oficial: CREATE WAREHOUSE](https://docs.snowflake.com/en/sql-reference/sql/create-warehouse)
- [Documentación oficial: Multi-cluster Warehouses](https://docs.snowflake.com/en/user-guide/warehouses-multicluster)
- [Documentación oficial: Table Types](https://docs.snowflake.com/en/user-guide/tables-temp-transient)
- [Best Practices: Warehouse Considerations](https://docs.snowflake.com/en/user-guide/warehouses-considerations)

---
