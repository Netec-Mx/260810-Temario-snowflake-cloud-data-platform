# Seguridad con Roles, Privilegios y RBAC en Snowflake

## Metadatos del Laboratorio

| Campo | Valor |
|-------|-------|
| **Duración** | 50 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Edición Snowflake** | Enterprise Trial |
| **Rol inicial requerido** | ACCOUNTADMIN |

---

## Descripción General

En este laboratorio diseñarás e implementarás un modelo completo de control de acceso basado en roles (RBAC) sobre la base de datos `LAB_DB` creada en laboratorios anteriores. Crearás una jerarquía de cinco roles funcionales, tres usuarios de prueba, y ejecutarás pruebas de acceso positivas y negativas para validar la correcta segregación de privilegios. Finalmente, consultarás vistas de auditoría en `ACCOUNT_USAGE`, implementarás una política de enmascaramiento dinámico y documentarás una matriz de permisos completa.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear una jerarquía de roles personalizada con cinco roles funcionales siguiendo el modelo RBAC de Snowflake
- [ ] Asignar privilegios granulares (SELECT, INSERT, USAGE, MONITOR) a roles sobre objetos específicos de `LAB_DB`
- [ ] Crear usuarios de prueba, asignarles roles y validar accesos permitidos y denegados mediante pruebas explícitas
- [ ] Consultar vistas de auditoría `QUERY_HISTORY` y `LOGIN_HISTORY` en `ACCOUNT_USAGE` para rastrear actividad
- [ ] Implementar una política de enmascaramiento dinámico (Dynamic Data Masking) sobre una columna sensible
- [ ] Documentar una matriz de permisos completa que relacione roles, objetos y privilegios

---

## Prerrequisitos

### Conocimientos Previos

| Concepto | Nivel |
|----------|-------|
| Modelo de herencia de roles en Snowflake | Comprensión básica |
| Sentencias GRANT / REVOKE | Familiaridad |
| SQL DDL y DML básico | Dominio |
| Navegación en Snowsight | Práctica previa |

### Acceso y Objetos Requeridos

| Recurso | Detalle |
|---------|---------|
| Cuenta Snowflake | Enterprise Trial, región us-east-1 |
| Rol ACCOUNTADMIN | Disponible para crear roles y usuarios |
| Base de datos `LAB_DB` | Con schemas `INGEST_SCHEMA` y `ADMIN_SCHEMA` |
| Tabla `LAB_DB.INGEST_SCHEMA.SALES_RAW` | Creada en lab 08-02-01 |
| Tabla `LAB_DB.ADMIN_SCHEMA.QUERY_BENCHMARK` | Creada en lab 08-02-01 |
| Warehouses `LAB_WH_XS`, `LAB_WH_S` | Creados en lab 07-02-01 |
| Acceso a `SNOWFLAKE.ACCOUNT_USAGE` | Latencia de hasta 45 minutos |

---

## Entorno del Laboratorio

### Configuración de Software

| Software | Versión |
|----------|---------|
| Snowflake | Release 8.x Enterprise Edition Trial |
| Snowsight | 2024.4 (interfaz web) |
| Navegador | Chrome 124+, Firefox 125+ o Edge 124+ |

### Arquitectura de Roles a Implementar

```
ACCOUNTADMIN
    └── SYSADMIN
            └── LAB_ADMIN_ROLE
                    ├── LAB_ANALYST_ROLE
                    │       └── LAB_READONLY_ROLE
                    ├── LAB_INGEST_ROLE
                    └── LAB_MONITOR_ROLE
```

### Worksheet Inicial

Crea una nueva worksheet en Snowsight con el nombre **LAB09_Seguridad_RBAC** antes de comenzar.

---

## Paso 1: Crear la Jerarquía de Roles

### Objetivo

Crear cinco roles funcionales y establecer la jerarquía de herencia entre ellos según el modelo RBAC diseñado.

### Instrucciones

1. Abre Snowsight y selecciona la worksheet **LAB09_Seguridad_RBAC**.

2. Establece el contexto de sesión con ACCOUNTADMIN:

```sql
-- ============================================================
-- PASO 1: Creación de la jerarquía de roles RBAC
-- ============================================================
USE ROLE ACCOUNTADMIN;

-- 1.1 Crear los cinco roles funcionales
CREATE OR REPLACE ROLE LAB_ADMIN_ROLE
    COMMENT = 'Administración total de LAB_DB - Lab 09';

CREATE OR REPLACE ROLE LAB_ANALYST_ROLE
    COMMENT = 'Lectura de todos los schemas en LAB_DB';

CREATE OR REPLACE ROLE LAB_INGEST_ROLE
    COMMENT = 'Escritura exclusiva en INGEST_SCHEMA';

CREATE OR REPLACE ROLE LAB_READONLY_ROLE
    COMMENT = 'SELECT solo en vistas, sin acceso a tablas base';

CREATE OR REPLACE ROLE LAB_MONITOR_ROLE
    COMMENT = 'Acceso a ACCOUNT_USAGE para monitoreo';
```

3. Establece la jerarquía de herencia otorgando roles a roles:

```sql
-- 1.2 Establecer jerarquía de herencia
-- LAB_ADMIN_ROLE hereda de SYSADMIN
GRANT ROLE LAB_ADMIN_ROLE TO ROLE SYSADMIN;

-- LAB_ANALYST_ROLE y LAB_INGEST_ROLE heredan hacia LAB_ADMIN_ROLE
GRANT ROLE LAB_ANALYST_ROLE TO ROLE LAB_ADMIN_ROLE;
GRANT ROLE LAB_INGEST_ROLE TO ROLE LAB_ADMIN_ROLE;
GRANT ROLE LAB_MONITOR_ROLE TO ROLE LAB_ADMIN_ROLE;

-- LAB_READONLY_ROLE hereda hacia LAB_ANALYST_ROLE
GRANT ROLE LAB_READONLY_ROLE TO ROLE LAB_ANALYST_ROLE;
```

4. Verifica la jerarquía creada:

```sql
-- 1.3 Verificar jerarquía
SHOW GRANTS OF ROLE LAB_ADMIN_ROLE;
SHOW GRANTS OF ROLE LAB_ANALYST_ROLE;
SHOW GRANTS OF ROLE LAB_READONLY_ROLE;
```

### Resultado Esperado

Cada sentencia `SHOW GRANTS OF ROLE` debe mostrar la relación de herencia. Por ejemplo, `LAB_ANALYST_ROLE` debe aparecer como otorgado a `LAB_ADMIN_ROLE`.

### Verificación

```sql
-- Confirmar que SYSADMIN puede activar LAB_ADMIN_ROLE
USE ROLE SYSADMIN;
USE ROLE LAB_ADMIN_ROLE;  -- Debe funcionar sin error
USE ROLE ACCOUNTADMIN;    -- Volver al contexto administrativo
```

---

## Paso 2: Asignar Privilegios Granulares a Cada Rol

### Objetivo

Otorgar privilegios específicos sobre objetos de `LAB_DB` a cada rol, siguiendo el principio de mínimo privilegio.

### Instrucciones

1. Otorga privilegios al rol `LAB_ADMIN_ROLE` (administración total de LAB_DB):

```sql
-- ============================================================
-- PASO 2: Asignación de privilegios granulares
-- ============================================================
USE ROLE ACCOUNTADMIN;

-- 2.1 LAB_ADMIN_ROLE: Control total sobre LAB_DB
GRANT USAGE ON DATABASE LAB_DB TO ROLE LAB_ADMIN_ROLE;
GRANT ALL PRIVILEGES ON SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_ADMIN_ROLE;
GRANT ALL PRIVILEGES ON SCHEMA LAB_DB.ADMIN_SCHEMA TO ROLE LAB_ADMIN_ROLE;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_ADMIN_ROLE;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA LAB_DB.ADMIN_SCHEMA TO ROLE LAB_ADMIN_ROLE;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_ADMIN_ROLE;
GRANT ALL PRIVILEGES ON FUTURE TABLES IN SCHEMA LAB_DB.ADMIN_SCHEMA TO ROLE LAB_ADMIN_ROLE;

-- Warehouse para LAB_ADMIN_ROLE
GRANT USAGE, OPERATE, MONITOR ON WAREHOUSE LAB_WH_XS TO ROLE LAB_ADMIN_ROLE;
GRANT USAGE, OPERATE, MONITOR ON WAREHOUSE LAB_WH_S TO ROLE LAB_ADMIN_ROLE;
```

2. Otorga privilegios al rol `LAB_ANALYST_ROLE` (lectura en todos los schemas):

```sql
-- 2.2 LAB_ANALYST_ROLE: SELECT en todos los schemas
GRANT USAGE ON DATABASE LAB_DB TO ROLE LAB_ANALYST_ROLE;
GRANT USAGE ON SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_ANALYST_ROLE;
GRANT USAGE ON SCHEMA LAB_DB.ADMIN_SCHEMA TO ROLE LAB_ANALYST_ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_ANALYST_ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA LAB_DB.ADMIN_SCHEMA TO ROLE LAB_ANALYST_ROLE;
GRANT SELECT ON FUTURE TABLES IN SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_ANALYST_ROLE;
GRANT SELECT ON FUTURE TABLES IN SCHEMA LAB_DB.ADMIN_SCHEMA TO ROLE LAB_ANALYST_ROLE;

-- Warehouse para LAB_ANALYST_ROLE
GRANT USAGE ON WAREHOUSE LAB_WH_XS TO ROLE LAB_ANALYST_ROLE;
```

3. Otorga privilegios al rol `LAB_INGEST_ROLE` (escritura solo en INGEST_SCHEMA):

```sql
-- 2.3 LAB_INGEST_ROLE: INSERT en INGEST_SCHEMA únicamente
GRANT USAGE ON DATABASE LAB_DB TO ROLE LAB_INGEST_ROLE;
GRANT USAGE ON SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_INGEST_ROLE;
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_INGEST_ROLE;
GRANT SELECT, INSERT ON FUTURE TABLES IN SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_INGEST_ROLE;

-- Warehouse para ingesta
GRANT USAGE ON WAREHOUSE LAB_WH_XS TO ROLE LAB_INGEST_ROLE;
```

4. Crea una vista para el rol `LAB_READONLY_ROLE` y otorga acceso solo a vistas:

```sql
-- 2.4 LAB_READONLY_ROLE: Solo vistas, sin tablas base
-- Primero crear una vista en INGEST_SCHEMA
USE ROLE ACCOUNTADMIN;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

CREATE OR REPLACE SECURE VIEW VW_SALES_SUMMARY AS
SELECT
    region,
    product_category,
    COUNT(*) AS total_transactions,
    SUM(amount) AS total_amount,
    AVG(amount) AS avg_amount
FROM SALES_RAW
GROUP BY region, product_category;

CREATE OR REPLACE SECURE VIEW VW_SALES_RECENT AS
SELECT
    transaction_id,
    sale_date,
    region,
    product_category,
    amount
FROM SALES_RAW
WHERE sale_date >= DATEADD('day', -30, CURRENT_DATE());

-- Privilegios para LAB_READONLY_ROLE
GRANT USAGE ON DATABASE LAB_DB TO ROLE LAB_READONLY_ROLE;
GRANT USAGE ON SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_READONLY_ROLE;
GRANT SELECT ON VIEW LAB_DB.INGEST_SCHEMA.VW_SALES_SUMMARY TO ROLE LAB_READONLY_ROLE;
GRANT SELECT ON VIEW LAB_DB.INGEST_SCHEMA.VW_SALES_RECENT TO ROLE LAB_READONLY_ROLE;

-- Warehouse mínimo para READONLY
GRANT USAGE ON WAREHOUSE LAB_WH_XS TO ROLE LAB_READONLY_ROLE;
```

> **Nota:** Si la tabla `SALES_RAW` no tiene las columnas exactas mencionadas (por ejemplo, si tu tabla usa nombres diferentes), ajusta los nombres de columna en las vistas para que coincidan con tu estructura. Las columnas mínimas esperadas son: un identificador de transacción, fecha, región, categoría de producto y monto.

5. Otorga privilegios al rol `LAB_MONITOR_ROLE`:

```sql
-- 2.5 LAB_MONITOR_ROLE: Acceso a ACCOUNT_USAGE
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE LAB_MONITOR_ROLE;
GRANT USAGE ON WAREHOUSE LAB_WH_XS TO ROLE LAB_MONITOR_ROLE;
```

### Resultado Esperado

Todas las sentencias GRANT deben ejecutarse con el mensaje `Statement executed successfully`.

### Verificación

```sql
-- Verificar privilegios otorgados a LAB_ANALYST_ROLE
SHOW GRANTS TO ROLE LAB_ANALYST_ROLE;

-- Verificar privilegios otorgados a LAB_INGEST_ROLE
SHOW GRANTS TO ROLE LAB_INGEST_ROLE;

-- Verificar privilegios otorgados a LAB_READONLY_ROLE
SHOW GRANTS TO ROLE LAB_READONLY_ROLE;
```

Confirma que cada rol tiene únicamente los privilegios asignados y no más.

---

## Paso 3: Crear Usuarios de Prueba y Asignar Roles

### Objetivo

Crear tres usuarios de prueba con contraseñas temporales y asignarles los roles correspondientes para ejecutar pruebas de acceso.

### Instrucciones

1. Crea los tres usuarios de prueba:

```sql
-- ============================================================
-- PASO 3: Creación de usuarios de prueba
-- ============================================================
USE ROLE ACCOUNTADMIN;

-- 3.1 Usuario analista
CREATE OR REPLACE USER USR_ANALYST
    PASSWORD = 'Lab09_Analyst_2024!'
    LOGIN_NAME = 'USR_ANALYST'
    DISPLAY_NAME = 'Usuario Analista Lab09'
    EMAIL = 'usr_analyst@lab.local'
    DEFAULT_ROLE = 'LAB_ANALYST_ROLE'
    DEFAULT_WAREHOUSE = 'LAB_WH_XS'
    MUST_CHANGE_PASSWORD = FALSE
    COMMENT = 'Usuario de prueba para validación RBAC - Analista';

-- 3.2 Usuario de ingesta
CREATE OR REPLACE USER USR_INGEST
    PASSWORD = 'Lab09_Ingest_2024!'
    LOGIN_NAME = 'USR_INGEST'
    DISPLAY_NAME = 'Usuario Ingesta Lab09'
    EMAIL = 'usr_ingest@lab.local'
    DEFAULT_ROLE = 'LAB_INGEST_ROLE'
    DEFAULT_WAREHOUSE = 'LAB_WH_XS'
    MUST_CHANGE_PASSWORD = FALSE
    COMMENT = 'Usuario de prueba para validación RBAC - Ingesta';

-- 3.3 Usuario de solo lectura
CREATE OR REPLACE USER USR_READONLY
    PASSWORD = 'Lab09_Readonly_2024!'
    LOGIN_NAME = 'USR_READONLY'
    DISPLAY_NAME = 'Usuario Solo Lectura Lab09'
    EMAIL = 'usr_readonly@lab.local'
    DEFAULT_ROLE = 'LAB_READONLY_ROLE'
    DEFAULT_WAREHOUSE = 'LAB_WH_XS'
    MUST_CHANGE_PASSWORD = FALSE
    COMMENT = 'Usuario de prueba para validación RBAC - Solo Lectura';
```

2. Asigna roles a los usuarios:

```sql
-- 3.4 Asignar roles a usuarios
GRANT ROLE LAB_ANALYST_ROLE TO USER USR_ANALYST;
GRANT ROLE LAB_INGEST_ROLE TO USER USR_INGEST;
GRANT ROLE LAB_READONLY_ROLE TO USER USR_READONLY;

-- Asignar también LAB_MONITOR_ROLE al analista para consultas de auditoría
GRANT ROLE LAB_MONITOR_ROLE TO USER USR_ANALYST;
```

3. Verifica la asignación:

```sql
-- 3.5 Verificar asignaciones
SHOW GRANTS TO USER USR_ANALYST;
SHOW GRANTS TO USER USR_INGEST;
SHOW GRANTS TO USER USR_READONLY;
```

### Resultado Esperado

| Usuario | Roles Asignados |
|---------|----------------|
| USR_ANALYST | LAB_ANALYST_ROLE, LAB_MONITOR_ROLE |
| USR_INGEST | LAB_INGEST_ROLE |
| USR_READONLY | LAB_READONLY_ROLE |

### Verificación

Cada `SHOW GRANTS TO USER` debe listar exactamente los roles asignados en la tabla anterior.

---

## Paso 4: Ejecutar Pruebas de Acceso Positivas y Negativas

### Objetivo

Validar que cada rol permite únicamente las operaciones autorizadas y deniega las no autorizadas, documentando los resultados.

### Instrucciones

1. **Pruebas con LAB_ANALYST_ROLE** (acceso de lectura a todos los schemas):

```sql
-- ============================================================
-- PASO 4: Pruebas de acceso
-- ============================================================

-- 4.1 PRUEBAS POSITIVAS: LAB_ANALYST_ROLE
USE ROLE LAB_ANALYST_ROLE;
USE WAREHOUSE LAB_WH_XS;

-- ✅ Debe funcionar: SELECT en SALES_RAW
SELECT COUNT(*) AS total_filas FROM LAB_DB.INGEST_SCHEMA.SALES_RAW;

-- ✅ Debe funcionar: SELECT en QUERY_BENCHMARK
SELECT COUNT(*) AS total_filas FROM LAB_DB.ADMIN_SCHEMA.QUERY_BENCHMARK;

-- ✅ Debe funcionar: SELECT en vista
SELECT * FROM LAB_DB.INGEST_SCHEMA.VW_SALES_SUMMARY LIMIT 5;
```

2. **Pruebas negativas con LAB_ANALYST_ROLE** (escritura denegada):

```sql
-- 4.2 PRUEBAS NEGATIVAS: LAB_ANALYST_ROLE
-- ❌ Debe fallar: INSERT en SALES_RAW (solo tiene SELECT)
INSERT INTO LAB_DB.INGEST_SCHEMA.SALES_RAW (transaction_id, sale_date, region, product_category, amount)
VALUES ('TEST-001', CURRENT_DATE(), 'TEST', 'TEST', 0.00);
-- Error esperado: Insufficient privileges to operate on table 'SALES_RAW'

-- ❌ Debe fallar: DROP TABLE
DROP TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW;
-- Error esperado: Insufficient privileges
```

3. **Pruebas con LAB_INGEST_ROLE** (escritura solo en INGEST_SCHEMA):

```sql
-- 4.3 PRUEBAS POSITIVAS: LAB_INGEST_ROLE
USE ROLE LAB_INGEST_ROLE;

-- ✅ Debe funcionar: SELECT en INGEST_SCHEMA
SELECT COUNT(*) FROM LAB_DB.INGEST_SCHEMA.SALES_RAW;

-- ✅ Debe funcionar: INSERT en INGEST_SCHEMA
INSERT INTO LAB_DB.INGEST_SCHEMA.SALES_RAW (transaction_id, sale_date, region, product_category, amount)
VALUES ('TEST-INGEST-001', CURRENT_DATE(), 'NORTH', 'ELECTRONICS', 150.00);

-- Verificar la inserción
SELECT * FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
WHERE transaction_id = 'TEST-INGEST-001';
```

4. **Pruebas negativas con LAB_INGEST_ROLE**:

```sql
-- 4.4 PRUEBAS NEGATIVAS: LAB_INGEST_ROLE
-- ❌ Debe fallar: Acceso a ADMIN_SCHEMA
SELECT COUNT(*) FROM LAB_DB.ADMIN_SCHEMA.QUERY_BENCHMARK;
-- Error esperado: Object does not exist or not authorized

-- ❌ Debe fallar: DELETE en INGEST_SCHEMA (solo tiene SELECT, INSERT)
DELETE FROM LAB_DB.INGEST_SCHEMA.SALES_RAW WHERE transaction_id = 'TEST-INGEST-001';
-- Error esperado: Insufficient privileges
```

5. **Pruebas con LAB_READONLY_ROLE** (solo vistas):

```sql
-- 4.5 PRUEBAS POSITIVAS: LAB_READONLY_ROLE
USE ROLE LAB_READONLY_ROLE;

-- ✅ Debe funcionar: SELECT en vistas
SELECT * FROM LAB_DB.INGEST_SCHEMA.VW_SALES_SUMMARY LIMIT 5;
SELECT * FROM LAB_DB.INGEST_SCHEMA.VW_SALES_RECENT LIMIT 5;
```

6. **Pruebas negativas con LAB_READONLY_ROLE**:

```sql
-- 4.6 PRUEBAS NEGATIVAS: LAB_READONLY_ROLE
-- ❌ Debe fallar: Acceso directo a tabla base
SELECT COUNT(*) FROM LAB_DB.INGEST_SCHEMA.SALES_RAW;
-- Error esperado: Insufficient privileges to operate on table 'SALES_RAW'

-- ❌ Debe fallar: INSERT en cualquier objeto
INSERT INTO LAB_DB.INGEST_SCHEMA.SALES_RAW (transaction_id)
VALUES ('HACK-ATTEMPT');
-- Error esperado: Insufficient privileges
```

7. **Prueba de herencia de privilegios**:

```sql
-- 4.7 Validar herencia: LAB_ADMIN_ROLE hereda todo
USE ROLE LAB_ADMIN_ROLE;

-- ✅ Hereda SELECT de LAB_ANALYST_ROLE
SELECT COUNT(*) FROM LAB_DB.INGEST_SCHEMA.SALES_RAW;

-- ✅ Hereda INSERT de LAB_INGEST_ROLE
INSERT INTO LAB_DB.INGEST_SCHEMA.SALES_RAW (transaction_id, sale_date, region, product_category, amount)
VALUES ('TEST-ADMIN-001', CURRENT_DATE(), 'SOUTH', 'FOOD', 75.00);

-- ✅ Hereda acceso a vistas de LAB_READONLY_ROLE (vía LAB_ANALYST_ROLE)
SELECT * FROM LAB_DB.INGEST_SCHEMA.VW_SALES_SUMMARY LIMIT 3;
```

### Resultado Esperado

Documenta los resultados en la siguiente tabla:

| # | Rol | Operación | Objeto | Resultado Esperado | Resultado Real |
|---|-----|-----------|--------|-------------------|----------------|
| 1 | LAB_ANALYST_ROLE | SELECT | SALES_RAW | ✅ Permitido | |
| 2 | LAB_ANALYST_ROLE | INSERT | SALES_RAW | ❌ Denegado | |
| 3 | LAB_INGEST_ROLE | INSERT | SALES_RAW | ✅ Permitido | |
| 4 | LAB_INGEST_ROLE | SELECT | QUERY_BENCHMARK | ❌ Denegado | |
| 5 | LAB_READONLY_ROLE | SELECT | VW_SALES_SUMMARY | ✅ Permitido | |
| 6 | LAB_READONLY_ROLE | SELECT | SALES_RAW | ❌ Denegado | |
| 7 | LAB_ADMIN_ROLE | INSERT | SALES_RAW | ✅ Permitido (herencia) | |

### Verificación

Todas las pruebas positivas deben retornar datos o el mensaje `1 Row(s) inserted`. Todas las pruebas negativas deben retornar un error de privilegios insuficientes.

---

## Paso 5: Consultar Vistas de Auditoría en ACCOUNT_USAGE

### Objetivo

Utilizar `QUERY_HISTORY` y `LOGIN_HISTORY` para rastrear la actividad generada por las pruebas de acceso realizadas.

### Instrucciones

> **Importante:** Las vistas de `ACCOUNT_USAGE` tienen una latencia de 45 minutos a 2 horas. Si ejecutas estas consultas inmediatamente después del Paso 4, es posible que no veas todos los registros. En ese caso, puedes usar las funciones de `INFORMATION_SCHEMA` como alternativa en tiempo real.

1. Consulta el historial de consultas de los usuarios de prueba:

```sql
-- ============================================================
-- PASO 5: Consultas de auditoría
-- ============================================================
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE LAB_WH_XS;

-- 5.1 Historial de consultas ejecutadas con roles de laboratorio
-- (Opción A: ACCOUNT_USAGE - latencia 45 min)
SELECT
    query_id,
    user_name,
    role_name,
    query_text,
    start_time,
    execution_status,
    error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE role_name IN ('LAB_ANALYST_ROLE', 'LAB_INGEST_ROLE', 'LAB_READONLY_ROLE', 'LAB_ADMIN_ROLE')
  AND start_time >= DATEADD('hour', -2, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 30;
```

2. Alternativa en tiempo real usando `INFORMATION_SCHEMA`:

```sql
-- 5.2 Alternativa en tiempo real (sin latencia)
SELECT
    query_id,
    user_name,
    role_name,
    query_text,
    start_time,
    execution_status,
    error_message
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
    DATEADD('hour', -1, CURRENT_TIMESTAMP()),
    CURRENT_TIMESTAMP()
))
WHERE role_name IN ('LAB_ANALYST_ROLE', 'LAB_INGEST_ROLE', 'LAB_READONLY_ROLE', 'LAB_ADMIN_ROLE')
ORDER BY start_time DESC;
```

3. Consulta intentos de inicio de sesión:

```sql
-- 5.3 Historial de login (latencia ~2 horas)
SELECT
    event_timestamp,
    user_name,
    client_ip,
    reported_client_type,
    is_success,
    error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY event_timestamp DESC
LIMIT 20;
```

4. Identifica consultas fallidas por privilegios insuficientes:

```sql
-- 5.4 Consultas fallidas por permisos (evidencia de pruebas negativas)
SELECT
    query_id,
    user_name,
    role_name,
    SUBSTR(query_text, 1, 100) AS query_preview,
    start_time,
    error_message
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
    DATEADD('hour', -1, CURRENT_TIMESTAMP()),
    CURRENT_TIMESTAMP()
))
WHERE execution_status = 'FAIL'
  AND error_message ILIKE '%privilege%'
ORDER BY start_time DESC;
```

### Resultado Esperado

La consulta 5.4 debe mostrar las consultas que fallaron durante las pruebas negativas del Paso 4, confirmando que el modelo RBAC deniega correctamente los accesos no autorizados.

### Verificación

Confirma que al menos aparecen registros con `execution_status = 'FAIL'` correspondientes a las operaciones INSERT y DROP intentadas con `LAB_ANALYST_ROLE` y las operaciones SELECT sobre tablas base intentadas con `LAB_READONLY_ROLE`.

---

## Paso 6: Implementar Dynamic Data Masking Policy

### Objetivo

Crear una política de enmascaramiento dinámico sobre la columna `customer_id` de `SALES_RAW` que muestre el valor real solo al rol `LAB_ADMIN_ROLE` y enmascara el dato para todos los demás roles.

### Instrucciones

1. Crea la política de enmascaramiento:

```sql
-- ============================================================
-- PASO 6: Dynamic Data Masking
-- ============================================================
USE ROLE ACCOUNTADMIN;
USE DATABASE LAB_DB;
USE SCHEMA INGEST_SCHEMA;

-- 6.1 Crear la masking policy
CREATE OR REPLACE MASKING POLICY MASK_CUSTOMER_ID
    AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('LAB_ADMIN_ROLE', 'ACCOUNTADMIN') THEN val
        WHEN CURRENT_ROLE() = 'LAB_ANALYST_ROLE' THEN 'CUST-****-' || RIGHT(val, 4)
        ELSE '**ENMASCARADO**'
    END
    COMMENT = 'Enmascara customer_id para roles sin privilegio administrativo';
```

2. Aplica la política a la columna `customer_id` de `SALES_RAW`:

```sql
-- 6.2 Aplicar la política a la columna
ALTER TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW
    ALTER COLUMN customer_id
    SET MASKING POLICY MASK_CUSTOMER_ID;
```

> **Nota:** Si tu tabla `SALES_RAW` no tiene una columna `customer_id`, sustitúyela por la columna que contenga el identificador del cliente (por ejemplo, `id_cliente` o `client_id`). Ajusta el nombre en la sentencia ALTER TABLE.

3. Valida el enmascaramiento con diferentes roles:

```sql
-- 6.3 Validar con LAB_ADMIN_ROLE (ve dato real)
USE ROLE LAB_ADMIN_ROLE;
SELECT customer_id, region, amount
FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
LIMIT 5;
-- Resultado: customer_id muestra el valor original

-- 6.4 Validar con LAB_ANALYST_ROLE (ve dato parcialmente enmascarado)
USE ROLE LAB_ANALYST_ROLE;
SELECT customer_id, region, amount
FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
LIMIT 5;
-- Resultado: customer_id muestra 'CUST-****-XXXX'

-- 6.5 Validar con LAB_INGEST_ROLE (ve dato completamente enmascarado)
USE ROLE LAB_INGEST_ROLE;
SELECT customer_id, region, amount
FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
LIMIT 5;
-- Resultado: customer_id muestra '**ENMASCARADO**'
```

4. Verifica las políticas activas:

```sql
-- 6.6 Listar masking policies activas
USE ROLE ACCOUNTADMIN;
SHOW MASKING POLICIES IN DATABASE LAB_DB;

-- Ver referencias de la política
SELECT *
FROM TABLE(INFORMATION_SCHEMA.POLICY_REFERENCES(
    POLICY_NAME => 'MASK_CUSTOMER_ID'
));
```

### Resultado Esperado

| Rol | Valor mostrado en customer_id |
|-----|-------------------------------|
| LAB_ADMIN_ROLE | `CUST-2024-0158` (valor real) |
| LAB_ANALYST_ROLE | `CUST-****-0158` (parcial) |
| LAB_INGEST_ROLE | `**ENMASCARADO**` (total) |

### Verificación

Ejecuta las tres consultas SELECT con los tres roles y confirma que cada uno ve un nivel diferente de detalle en la columna `customer_id`.

---

## Paso 7: Documentar la Matriz de Permisos

### Objetivo

Construir una matriz de permisos completa que documente qué rol tiene qué privilegio sobre qué objeto, generándola tanto manualmente como con consultas SQL.

### Instrucciones

1. Genera la matriz de permisos con SQL:

```sql
-- ============================================================
-- PASO 7: Documentación - Matriz de Permisos
-- ============================================================
USE ROLE ACCOUNTADMIN;

-- 7.1 Extraer todos los privilegios otorgados a los roles del laboratorio
SELECT
    grantee_name AS rol,
    privilege,
    granted_on AS tipo_objeto,
    name AS nombre_objeto,
    grant_option
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE grantee_name IN (
    'LAB_ADMIN_ROLE',
    'LAB_ANALYST_ROLE',
    'LAB_INGEST_ROLE',
    'LAB_READONLY_ROLE',
    'LAB_MONITOR_ROLE'
)
  AND deleted_on IS NULL
ORDER BY grantee_name, granted_on, name;
```

2. Alternativa con `SHOW GRANTS` para cada rol:

```sql
-- 7.2 Vista consolidada por rol
SHOW GRANTS TO ROLE LAB_ADMIN_ROLE;
SHOW GRANTS TO ROLE LAB_ANALYST_ROLE;
SHOW GRANTS TO ROLE LAB_INGEST_ROLE;
SHOW GRANTS TO ROLE LAB_READONLY_ROLE;
SHOW GRANTS TO ROLE LAB_MONITOR_ROLE;
```

3. Documenta la matriz completa en el siguiente formato:

```sql
-- 7.3 Crear tabla de documentación de la matriz
USE ROLE LAB_ADMIN_ROLE;
USE DATABASE LAB_DB;
USE SCHEMA ADMIN_SCHEMA;

CREATE OR REPLACE TABLE PERMISSION_MATRIX (
    rol              STRING,
    objeto           STRING,
    tipo_objeto      STRING,
    privilegio       STRING,
    otorgado_por     STRING,
    fecha_registro   TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Insertar la matriz documentada
INSERT INTO PERMISSION_MATRIX (rol, objeto, tipo_objeto, privilegio, otorgado_por) VALUES
-- LAB_ADMIN_ROLE
('LAB_ADMIN_ROLE', 'LAB_DB', 'DATABASE', 'USAGE', 'ACCOUNTADMIN'),
('LAB_ADMIN_ROLE', 'INGEST_SCHEMA', 'SCHEMA', 'ALL PRIVILEGES', 'ACCOUNTADMIN'),
('LAB_ADMIN_ROLE', 'ADMIN_SCHEMA', 'SCHEMA', 'ALL PRIVILEGES', 'ACCOUNTADMIN'),
('LAB_ADMIN_ROLE', 'SALES_RAW', 'TABLE', 'ALL PRIVILEGES', 'ACCOUNTADMIN'),
('LAB_ADMIN_ROLE', 'LAB_WH_XS', 'WAREHOUSE', 'USAGE, OPERATE, MONITOR', 'ACCOUNTADMIN'),
('LAB_ADMIN_ROLE', 'LAB_WH_S', 'WAREHOUSE', 'USAGE, OPERATE, MONITOR', 'ACCOUNTADMIN'),
-- LAB_ANALYST_ROLE
('LAB_ANALYST_ROLE', 'LAB_DB', 'DATABASE', 'USAGE', 'ACCOUNTADMIN'),
('LAB_ANALYST_ROLE', 'INGEST_SCHEMA', 'SCHEMA', 'USAGE', 'ACCOUNTADMIN'),
('LAB_ANALYST_ROLE', 'ADMIN_SCHEMA', 'SCHEMA', 'USAGE', 'ACCOUNTADMIN'),
('LAB_ANALYST_ROLE', 'SALES_RAW', 'TABLE', 'SELECT', 'ACCOUNTADMIN'),
('LAB_ANALYST_ROLE', 'QUERY_BENCHMARK', 'TABLE', 'SELECT', 'ACCOUNTADMIN'),
('LAB_ANALYST_ROLE', 'LAB_WH_XS', 'WAREHOUSE', 'USAGE', 'ACCOUNTADMIN'),
-- LAB_INGEST_ROLE
('LAB_INGEST_ROLE', 'LAB_DB', 'DATABASE', 'USAGE', 'ACCOUNTADMIN'),
('LAB_INGEST_ROLE', 'INGEST_SCHEMA', 'SCHEMA', 'USAGE', 'ACCOUNTADMIN'),
('LAB_INGEST_ROLE', 'SALES_RAW', 'TABLE', 'SELECT, INSERT', 'ACCOUNTADMIN'),
('LAB_INGEST_ROLE', 'LAB_WH_XS', 'WAREHOUSE', 'USAGE', 'ACCOUNTADMIN'),
-- LAB_READONLY_ROLE
('LAB_READONLY_ROLE', 'LAB_DB', 'DATABASE', 'USAGE', 'ACCOUNTADMIN'),
('LAB_READONLY_ROLE', 'INGEST_SCHEMA', 'SCHEMA', 'USAGE', 'ACCOUNTADMIN'),
('LAB_READONLY_ROLE', 'VW_SALES_SUMMARY', 'VIEW', 'SELECT', 'ACCOUNTADMIN'),
('LAB_READONLY_ROLE', 'VW_SALES_RECENT', 'VIEW', 'SELECT', 'ACCOUNTADMIN'),
('LAB_READONLY_ROLE', 'LAB_WH_XS', 'WAREHOUSE', 'USAGE', 'ACCOUNTADMIN'),
-- LAB_MONITOR_ROLE
('LAB_MONITOR_ROLE', 'SNOWFLAKE', 'DATABASE', 'IMPORTED PRIVILEGES', 'ACCOUNTADMIN'),
('LAB_MONITOR_ROLE', 'LAB_WH_XS', 'WAREHOUSE', 'USAGE', 'ACCOUNTADMIN');

-- Consultar la matriz completa
SELECT rol, tipo_objeto, objeto, privilegio
FROM PERMISSION_MATRIX
ORDER BY rol, tipo_objeto, objeto;
```

### Resultado Esperado

La consulta final debe mostrar 22 filas organizadas por rol, tipo de objeto y nombre de objeto, representando la matriz completa de permisos del modelo RBAC implementado.

### Verificación

```sql
-- Verificar conteo por rol
SELECT rol, COUNT(*) AS total_privilegios
FROM LAB_DB.ADMIN_SCHEMA.PERMISSION_MATRIX
GROUP BY rol
ORDER BY total_privilegios DESC;
```

Resultado esperado:

| rol | total_privilegios |
|-----|-------------------|
| LAB_ADMIN_ROLE | 6 |
| LAB_ANALYST_ROLE | 6 |
| LAB_INGEST_ROLE | 4 |
| LAB_READONLY_ROLE | 4 |
| LAB_MONITOR_ROLE | 2 |

---

## Validación y Pruebas Finales

Ejecuta el siguiente bloque de validación integral para confirmar que todo el laboratorio se completó correctamente:

```sql
-- ============================================================
-- VALIDACIÓN INTEGRAL DEL LABORATORIO
-- ============================================================
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE LAB_WH_XS;

-- V1: Verificar existencia de los 5 roles
SELECT COUNT(*) AS roles_creados
FROM (
    SHOW ROLES LIKE 'LAB_%_ROLE'
);
-- Esperado: 5

-- V2: Verificar existencia de los 3 usuarios
SHOW USERS LIKE 'USR_%';
-- Esperado: 3 usuarios (USR_ANALYST, USR_INGEST, USR_READONLY)

-- V3: Verificar jerarquía - LAB_ADMIN_ROLE tiene roles hijo
SHOW GRANTS OF ROLE LAB_ANALYST_ROLE;
-- Debe mostrar: granted_to ROLE LAB_ADMIN_ROLE

-- V4: Verificar masking policy activa
SHOW MASKING POLICIES IN SCHEMA LAB_DB.INGEST_SCHEMA;
-- Debe mostrar: MASK_CUSTOMER_ID

-- V5: Verificar vistas seguras
SHOW VIEWS IN SCHEMA LAB_DB.INGEST_SCHEMA;
-- Debe mostrar: VW_SALES_SUMMARY, VW_SALES_RECENT (con is_secure = true)

-- V6: Verificar tabla de documentación
SELECT COUNT(*) AS registros_matriz
FROM LAB_DB.ADMIN_SCHEMA.PERMISSION_MATRIX;
-- Esperado: 22

-- V7: Prueba final de enmascaramiento
USE ROLE LAB_ANALYST_ROLE;
SELECT customer_id FROM LAB_DB.INGEST_SCHEMA.SALES_RAW LIMIT 1;
-- El valor debe aparecer parcialmente enmascarado
```

---

## Solución de Problemas

### Problema 1: Error "Object does not exist" al cambiar de rol

**Síntomas:** Al ejecutar `USE ROLE LAB_ANALYST_ROLE` seguido de un `SELECT`, Snowflake devuelve `SQL compilation error: Object 'LAB_DB.INGEST_SCHEMA.SALES_RAW' does not exist or not authorized`.

**Causa:** El rol no tiene el privilegio `USAGE` en la base de datos o en el schema. Snowflake oculta la existencia de objetos cuando el rol no tiene acceso, por lo que el mensaje es ambiguo intencionalmente.

**Solución:**

```sql
-- Verificar privilegios actuales del rol
USE ROLE ACCOUNTADMIN;
SHOW GRANTS TO ROLE LAB_ANALYST_ROLE;

-- Si falta USAGE en la base de datos:
GRANT USAGE ON DATABASE LAB_DB TO ROLE LAB_ANALYST_ROLE;

-- Si falta USAGE en el schema:
GRANT USAGE ON SCHEMA LAB_DB.INGEST_SCHEMA TO ROLE LAB_ANALYST_ROLE;

-- Si falta SELECT en la tabla:
GRANT SELECT ON TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW TO ROLE LAB_ANALYST_ROLE;
```

---

### Problema 2: La masking policy no se puede aplicar - "Column data type mismatch"

**Síntomas:** Al ejecutar `ALTER TABLE ... ALTER COLUMN customer_id SET MASKING POLICY MASK_CUSTOMER_ID`, Snowflake devuelve un error indicando que el tipo de dato de la columna no coincide con el de la política.

**Causa:** La política fue creada con `AS (val STRING) RETURNS STRING` pero la columna `customer_id` tiene un tipo de dato diferente (por ejemplo, `NUMBER` o `INTEGER`).

**Solución:**

```sql
-- Verificar el tipo de dato de la columna
DESCRIBE TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW;

-- Si customer_id es NUMBER, recrear la política con el tipo correcto:
CREATE OR REPLACE MASKING POLICY MASK_CUSTOMER_ID
    AS (val NUMBER) RETURNS NUMBER ->
    CASE
        WHEN CURRENT_ROLE() IN ('LAB_ADMIN_ROLE', 'ACCOUNTADMIN') THEN val
        WHEN CURRENT_ROLE() = 'LAB_ANALYST_ROLE' THEN val * -1  -- Ofuscar
        ELSE 0  -- Enmascarar completamente
    END
    COMMENT = 'Enmascara customer_id numérico';

-- Aplicar nuevamente
ALTER TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW
    ALTER COLUMN customer_id
    SET MASKING POLICY MASK_CUSTOMER_ID;
```

---

## Limpieza

> **⚠️ NO ejecutes esta limpieza si planeas continuar con el lab 10-02-01**, donde se reutilizarán los roles `LAB_INGEST_ROLE` y `LAB_ADMIN_ROLE` para asegurar el pipeline incremental.

Si necesitas limpiar el entorno al finalizar el curso completo:

```sql
-- ============================================================
-- LIMPIEZA (ejecutar solo al final del curso)
-- ============================================================
USE ROLE ACCOUNTADMIN;

-- Remover masking policy
ALTER TABLE LAB_DB.INGEST_SCHEMA.SALES_RAW
    ALTER COLUMN customer_id UNSET MASKING POLICY;
DROP MASKING POLICY IF EXISTS LAB_DB.INGEST_SCHEMA.MASK_CUSTOMER_ID;

-- Eliminar vistas
DROP VIEW IF EXISTS LAB_DB.INGEST_SCHEMA.VW_SALES_SUMMARY;
DROP VIEW IF EXISTS LAB_DB.INGEST_SCHEMA.VW_SALES_RECENT;

-- Eliminar tabla de documentación
DROP TABLE IF EXISTS LAB_DB.ADMIN_SCHEMA.PERMISSION_MATRIX;

-- Eliminar datos de prueba
DELETE FROM LAB_DB.INGEST_SCHEMA.SALES_RAW
WHERE transaction_id IN ('TEST-INGEST-001', 'TEST-ADMIN-001');

-- Eliminar usuarios de prueba
DROP USER IF EXISTS USR_ANALYST;
DROP USER IF EXISTS USR_INGEST;
DROP USER IF EXISTS USR_READONLY;

-- Eliminar roles (de abajo hacia arriba en la jerarquía)
DROP ROLE IF EXISTS LAB_READONLY_ROLE;
DROP ROLE IF EXISTS LAB_MONITOR_ROLE;
DROP ROLE IF EXISTS LAB_INGEST_ROLE;
DROP ROLE IF EXISTS LAB_ANALYST_ROLE;
DROP ROLE IF EXISTS LAB_ADMIN_ROLE;
```

---

## Resumen

En este laboratorio implementaste un modelo RBAC completo en Snowflake que incluye:

| Componente | Cantidad | Detalle |
|------------|----------|---------|
| Roles funcionales | 5 | ADMIN, ANALYST, INGEST, READONLY, MONITOR |
| Usuarios de prueba | 3 | USR_ANALYST, USR_INGEST, USR_READONLY |
| Jerarquía de herencia | 5 niveles | ACCOUNTADMIN → SYSADMIN → LAB_ADMIN → hijos |
| Pruebas de acceso | 7+ | Positivas y negativas documentadas |
| Masking policy | 1 | Sobre customer_id con 3 niveles |
| Vistas seguras | 2 | VW_SALES_SUMMARY, VW_SALES_RECENT |
| Matriz documentada | 22 registros | En tabla PERMISSION_MATRIX |

**Conceptos clave reforzados:**

- El principio de **mínimo privilegio** se implementa otorgando solo los permisos estrictamente necesarios para cada función.
- La **herencia de roles** permite que roles superiores acumulen los privilegios de sus roles hijos sin duplicar GRANTs.
- Las **vistas seguras** combinadas con restricciones de rol proporcionan una capa adicional de aislamiento de datos.
- El **enmascaramiento dinámico** protege datos sensibles sin necesidad de crear copias filtradas de las tablas.
- Las **vistas de auditoría** en `ACCOUNT_USAGE` proporcionan evidencia para cumplimiento normativo.

### Recursos Adicionales

- [Documentación oficial: Access Control Framework](https://docs.snowflake.com/en/user-guide/security-access-control-overview)
- [Documentación oficial: Dynamic Data Masking](https://docs.snowflake.com/en/user-guide/security-column-ddm-intro)
- [Documentación oficial: ACCOUNT_USAGE Schema](https://docs.snowflake.com/en/sql-reference/account-usage)
- [Best Practices: Role Hierarchy Design](https://docs.snowflake.com/en/user-guide/security-access-control-considerations)
