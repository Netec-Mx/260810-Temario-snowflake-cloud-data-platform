# Exploración Inicial de Snowflake y Ejecución de Consultas

## 1. Metadatos del Laboratorio

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |
| **Edición Snowflake** | Enterprise Trial |
| **Región** | AWS us-east-1 |

---

## 2. Descripción General

En este laboratorio accederás por primera vez a tu cuenta Snowflake Trial, explorarás sistemáticamente cada sección de la interfaz Snowsight y ejecutarás consultas SQL progresivas sobre la base de datos de muestra `SNOWFLAKE_SAMPLE_DATA`. Aprenderás a configurar el contexto de sesión mediante la interfaz gráfica y comandos SQL `USE`, y finalizarás revisando el historial de ejecución para analizar métricas básicas de rendimiento. Este laboratorio establece la base operativa para todos los laboratorios posteriores del curso.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Acceder exitosamente a la cuenta Snowflake Trial y navegar las secciones principales de Snowsight (Home, Worksheets, Data, Activity, Admin)
- [ ] Identificar y diferenciar los objetos fundamentales de Snowflake: roles del sistema, warehouses, bases de datos, esquemas y tablas
- [ ] Configurar el contexto de sesión (rol, warehouse, base de datos y esquema) usando tanto la UI como comandos SQL `USE`
- [ ] Crear y ejecutar consultas SQL progresivas (SELECT, WHERE, ORDER BY, COUNT, GROUP BY) sobre `SNOWFLAKE_SAMPLE_DATA`
- [ ] Revisar el historial de ejecución de consultas en Activity → Query History y analizar métricas de duración y bytes escaneados

---

## 4. Prerrequisitos

### Conocimientos Previos

| Conocimiento | Nivel Requerido |
|---|---|
| SQL básico (SELECT, WHERE, ORDER BY, GROUP BY) | Intermedio |
| Navegación en interfaces web | Básico |
| Conceptos de bases de datos relacionales | Básico |

### Acceso y Cuentas

- Cuenta Snowflake Trial activa y verificada por email (AWS, us-east-1, Enterprise Edition)
- URL de acceso en formato: `https://<account_identifier>.snowflakecomputing.com`
- Credenciales de administrador (usuario ADMIN o el definido durante el registro)
- Navegador web moderno: Chrome 124.0+, Firefox 125.0+ o Edge 124.0+

---

## 5. Entorno del Laboratorio

### Requisitos de Hardware

| Componente | Especificación Mínima |
|---|---|
| RAM disponible para navegador | 4 GB |
| Resolución de pantalla | 1366 × 768 píxeles |
| Conexión a internet | 10 Mbps descarga |

### Software Utilizado

| Herramienta | Versión | Propósito |
|---|---|---|
| Snowsight (Snowflake Web UI) | 2024.4 | Interfaz principal de trabajo |
| Snowflake Engine | Release 8.x | Motor de procesamiento |
| Navegador Chrome/Firefox/Edge | 124.0+ / 125.0+ / 124.0+ | Acceso a Snowsight |

### Objetos Snowflake Utilizados (preexistentes)

| Objeto | Nombre | Notas |
|---|---|---|
| Warehouse | `COMPUTE_WH` | X-Small, incluido con la cuenta Trial |
| Base de datos | `SNOWFLAKE_SAMPLE_DATA` | Precargada en toda cuenta Snowflake |
| Esquema | `TPCH_SF1` | Dataset TPC-H escala 1 (~1 GB) |
| Roles | `ACCOUNTADMIN`, `SYSADMIN`, `PUBLIC` | Roles del sistema predeterminados |

---

## 6. Instrucciones Paso a Paso

### Paso 1: Acceder a Snowflake y Verificar la Cuenta

**Objetivo:** Iniciar sesión en Snowsight y confirmar que la cuenta Trial está activa en la edición y región correctas.

**Instrucciones:**

1. Abre tu navegador web y navega a la URL de tu cuenta Snowflake:
   ```
   https://<tu_account_identifier>.snowflakecomputing.com
   ```
2. Ingresa tu nombre de usuario y contraseña definidos durante el registro.
3. Una vez dentro, observa la pantalla **Home** de Snowsight. Identifica:
   - El panel de navegación lateral izquierdo con las secciones principales.
   - El nombre de usuario y rol activo en la esquina inferior izquierda.
4. Haz clic en tu nombre de usuario (esquina inferior izquierda) y verifica que aparezca el rol `ACCOUNTADMIN` como opción disponible.
5. Crea una nueva worksheet haciendo clic en el botón **"+"** en la sección **Worksheets** del panel lateral, luego selecciona **"SQL Worksheet"**.
6. Renombra la worksheet haciendo clic en su nombre por defecto y escribe: `LAB01_ExploracionInicial`.
7. Ejecuta la siguiente consulta para verificar la información de tu cuenta:

```sql
-- Verificar información de la cuenta, región y versión
SELECT
    CURRENT_ACCOUNT()   AS cuenta,
    CURRENT_REGION()    AS region,
    CURRENT_VERSION()   AS version_snowflake;
```

**Resultado Esperado:**

| cuenta | region | version_snowflake |
|--------|--------|-------------------|
| `<tu_identificador>` | `AWS_US_EAST_1` | `8.x.x` (versión actual) |

**Verificación:**
- La columna `region` debe mostrar `AWS_US_EAST_1` confirmando el despliegue en AWS us-east-1.
- La versión debe comenzar con `8.` indicando Snowflake Release 8.x.

---

### Paso 2: Explorar las Secciones Principales de Snowsight

**Objetivo:** Familiarizarse con la estructura de navegación de Snowsight identificando el propósito de cada sección.

**Instrucciones:**

1. **Sección Home (Casa):** Haz clic en el ícono de casa. Observa los accesos rápidos a worksheets recientes y recursos de aprendizaje.

2. **Sección Data (Base de datos):** Haz clic en **Data** en el panel lateral.
   - Expande la sección **Databases** y localiza `SNOWFLAKE_SAMPLE_DATA`.
   - Haz clic en `SNOWFLAKE_SAMPLE_DATA` → `TPCH_SF1` → **Tables**.
   - Identifica las tablas disponibles: `CUSTOMER`, `LINEITEM`, `NATION`, `ORDERS`, `PART`, `PARTSUPP`, `REGION`, `SUPPLIER`.
   - Haz clic en la tabla `CUSTOMER` y observa las pestañas: **Columns**, **Data Preview**, **Details**.

3. **Sección Compute (Warehouses):** Navega a **Admin** → **Warehouses**.
   - Identifica el warehouse `COMPUTE_WH` (creado automáticamente con la cuenta Trial).
   - Observa su tamaño (X-Small), estado (Started/Suspended) y configuración de auto-suspend.

4. **Sección Activity (Historial):** Haz clic en **Activity** → **Query History**.
   - Observa que ya aparece la consulta ejecutada en el Paso 1.
   - Nota las columnas: Status, SQL Text, Duration, Warehouse, User.

5. **Sección Admin (Administración):** Navega a **Admin** → **Users & Roles**.
   - Haz clic en la pestaña **Roles** y localiza los roles del sistema: `ACCOUNTADMIN`, `SYSADMIN`, `SECURITYADMIN`, `USERADMIN`, `PUBLIC`.
   - Observa la jerarquía: `ACCOUNTADMIN` es el rol de mayor privilegio que hereda todos los demás.

**Resultado Esperado:**
- Puedes ver al menos 8 tablas dentro de `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1`.
- El warehouse `COMPUTE_WH` aparece listado con tamaño X-Small.
- Los 5 roles del sistema son visibles en la sección Admin → Users & Roles.

**Verificación:**
- Confirma que puedes navegar entre secciones sin errores.
- Verifica que la tabla `CUSTOMER` muestra columnas como `C_CUSTKEY`, `C_NAME`, `C_ADDRESS`, etc.

---

### Paso 3: Identificar Roles del Sistema y Jerarquía de Privilegios

**Objetivo:** Comprender los roles predeterminados del sistema y su jerarquía mediante consultas SQL.

**Instrucciones:**

1. Regresa a tu worksheet `LAB01_ExploracionInicial`.
2. Ejecuta la siguiente consulta para listar los roles disponibles:

```sql
-- Listar todos los roles disponibles en la cuenta
SHOW ROLES;
```

3. Observa en los resultados las columnas `name`, `owner` y `granted_roles`. Identifica los 5 roles del sistema.

4. Ejecuta la siguiente consulta para ver la jerarquía de roles:

```sql
-- Ver qué roles están otorgados a ACCOUNTADMIN
SHOW GRANTS TO ROLE ACCOUNTADMIN;
```

5. Ahora verifica tu rol actual y los roles que tienes asignados:

```sql
-- Ver el rol activo actual
SELECT CURRENT_ROLE() AS rol_activo;

-- Ver los roles otorgados a tu usuario
SHOW GRANTS TO USER ADMIN;
```

> **Nota:** Reemplaza `ADMIN` con tu nombre de usuario si es diferente.

**Resultado Esperado:**

La consulta `SHOW ROLES` debe mostrar al menos estos roles:

| name | owner |
|------|-------|
| ACCOUNTADMIN | (vacío - rol del sistema) |
| SYSADMIN | (vacío - rol del sistema) |
| SECURITYADMIN | (vacío - rol del sistema) |
| USERADMIN | (vacío - rol del sistema) |
| PUBLIC | (vacío - rol del sistema) |

**Verificación:**
- Confirma que `ACCOUNTADMIN` aparece como el rol de mayor nivel.
- Verifica que tu usuario tiene asignado al menos el rol `ACCOUNTADMIN`.

---

### Paso 4: Configurar el Contexto de Sesión con Comandos SQL USE

**Objetivo:** Aprender a establecer y cambiar el contexto de sesión (rol, warehouse, base de datos, esquema) mediante comandos SQL.

**Instrucciones:**

1. En tu worksheet `LAB01_ExploracionInicial`, ejecuta los siguientes comandos de contexto uno por uno:

```sql
-- Paso 4.1: Establecer el rol SYSADMIN (buena práctica para trabajo diario)
USE ROLE SYSADMIN;

-- Paso 4.2: Seleccionar el warehouse de cómputo
USE WAREHOUSE COMPUTE_WH;

-- Paso 4.3: Seleccionar la base de datos de muestra
USE DATABASE SNOWFLAKE_SAMPLE_DATA;

-- Paso 4.4: Seleccionar el esquema TPC-H escala 1
USE SCHEMA TPCH_SF1;
```

2. Verifica que todo el contexto está correctamente configurado:

```sql
-- Verificar el contexto completo de la sesión
SELECT
    CURRENT_USER()      AS usuario,
    CURRENT_ROLE()      AS rol,
    CURRENT_WAREHOUSE() AS warehouse,
    CURRENT_DATABASE()  AS base_datos,
    CURRENT_SCHEMA()    AS esquema,
    CURRENT_TIMESTAMP() AS timestamp_actual;
```

3. Ahora cambia el contexto usando la **interfaz gráfica**:
   - En la parte superior de la worksheet, observa los selectores de contexto.
   - Haz clic en el selector de rol y verifica que muestra `SYSADMIN`.
   - Haz clic en el selector de warehouse y verifica que muestra `COMPUTE_WH`.
   - Haz clic en el selector de base de datos/esquema y verifica `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1`.

4. Experimenta cambiando el rol temporalmente:

```sql
-- Cambiar a rol PUBLIC (menos privilegios)
USE ROLE PUBLIC;

-- Verificar el cambio
SELECT CURRENT_ROLE() AS rol_activo;

-- Volver a SYSADMIN para continuar el laboratorio
USE ROLE SYSADMIN;
```

**Resultado Esperado:**

```
| usuario | rol      | warehouse  | base_datos              | esquema  | timestamp_actual         |
|---------|----------|------------|-------------------------|----------|--------------------------|
| ADMIN   | SYSADMIN | COMPUTE_WH | SNOWFLAKE_SAMPLE_DATA  | TPCH_SF1 | 2024-xx-xx xx:xx:xx.xxx  |
```

**Verificación:**
- Los cuatro elementos del contexto (rol, warehouse, base de datos, esquema) están correctamente establecidos.
- Al cambiar a `PUBLIC` y volver a `SYSADMIN`, la consulta refleja el cambio correctamente.
- Los selectores gráficos en la parte superior de la worksheet coinciden con los valores SQL.

---

### Paso 5: Ejecutar Consultas SQL Básicas — SELECT y LIMIT

**Objetivo:** Ejecutar las primeras consultas de exploración sobre las tablas de muestra usando SELECT con LIMIT.

**Instrucciones:**

1. Asegúrate de que el contexto está configurado correctamente (rol `SYSADMIN`, warehouse `COMPUTE_WH`, base de datos `SNOWFLAKE_SAMPLE_DATA`, esquema `TPCH_SF1`).

2. Ejecuta las siguientes consultas exploratorias:

```sql
-- Consulta 1: Ver las primeras 10 filas de la tabla CUSTOMER
SELECT *
FROM CUSTOMER
LIMIT 10;
```

```sql
-- Consulta 2: Explorar la estructura de la tabla ORDERS (primeras 5 filas)
SELECT *
FROM ORDERS
LIMIT 5;
```

```sql
-- Consulta 3: Seleccionar columnas específicas de NATION
SELECT
    N_NATIONKEY   AS id_nacion,
    N_NAME        AS nombre_nacion,
    N_REGIONKEY   AS id_region
FROM NATION;
```

3. Observa en el panel de resultados:
   - El número de filas retornadas.
   - Los tipos de datos de cada columna (hover sobre el encabezado).
   - El tiempo de ejecución mostrado en la esquina inferior derecha del panel de resultados.

**Resultado Esperado:**

- **Consulta 1:** Retorna exactamente 10 filas con columnas como `C_CUSTKEY`, `C_NAME`, `C_ADDRESS`, `C_NATIONKEY`, `C_PHONE`, `C_ACCTBAL`, `C_MKTSEGMENT`, `C_COMMENT`.
- **Consulta 2:** Retorna 5 filas con columnas de órdenes incluyendo `O_ORDERKEY`, `O_CUSTKEY`, `O_ORDERSTATUS`, `O_TOTALPRICE`, `O_ORDERDATE`.
- **Consulta 3:** Retorna 25 filas (todas las naciones del dataset TPC-H).

**Verificación:**
- La tabla `NATION` debe tener exactamente 25 filas (sin LIMIT retorna todas).
- Los tiempos de ejecución deben ser menores a 2 segundos para estas consultas simples.

---

### Paso 6: Consultas con Filtros WHERE y Ordenamiento ORDER BY

**Objetivo:** Aplicar filtros y ordenamiento para refinar los resultados de las consultas.

**Instrucciones:**

1. Ejecuta consultas con condiciones de filtrado:

```sql
-- Consulta 4: Clientes del segmento AUTOMOBILE con saldo > 5000
SELECT
    C_CUSTKEY     AS id_cliente,
    C_NAME        AS nombre,
    C_MKTSEGMENT  AS segmento,
    C_ACCTBAL     AS saldo_cuenta,
    C_NATIONKEY   AS id_nacion
FROM CUSTOMER
WHERE C_MKTSEGMENT = 'AUTOMOBILE'
  AND C_ACCTBAL > 5000
LIMIT 15;
```

```sql
-- Consulta 5: Órdenes con estado 'F' (completadas) ordenadas por precio total descendente
SELECT
    O_ORDERKEY    AS id_orden,
    O_CUSTKEY     AS id_cliente,
    O_ORDERSTATUS AS estado,
    O_TOTALPRICE  AS precio_total,
    O_ORDERDATE   AS fecha_orden
FROM ORDERS
WHERE O_ORDERSTATUS = 'F'
ORDER BY O_TOTALPRICE DESC
LIMIT 10;
```

```sql
-- Consulta 6: Proveedores de la nación 24 (UNITED STATES), ordenados por nombre
SELECT
    S_SUPPKEY   AS id_proveedor,
    S_NAME      AS nombre_proveedor,
    S_ADDRESS   AS direccion,
    S_PHONE     AS telefono,
    S_NATIONKEY AS id_nacion
FROM SUPPLIER
WHERE S_NATIONKEY = 24
ORDER BY S_NAME ASC
LIMIT 10;
```

2. Para cada consulta, observa:
   - Cuántas filas cumplen el filtro (el resultado puede ser menor que el LIMIT).
   - Que el ordenamiento es correcto (descendente para precios, ascendente para nombres).

**Resultado Esperado:**

- **Consulta 4:** Retorna hasta 15 clientes del segmento AUTOMOBILE con saldo superior a 5000.
- **Consulta 5:** Las 10 órdenes completadas más caras, con `precio_total` en orden descendente (el valor más alto primero).
- **Consulta 6:** Hasta 10 proveedores de Estados Unidos (nación 24) ordenados alfabéticamente.

**Verificación:**
- En la Consulta 5, confirma que la primera fila tiene el `precio_total` más alto y la última el más bajo del grupo.
- En la Consulta 6, verifica que todos los registros tienen `id_nacion = 24`.

---

### Paso 7: Consultas con Agregaciones — COUNT y GROUP BY

**Objetivo:** Ejecutar consultas analíticas con funciones de agregación para obtener resúmenes estadísticos.

**Instrucciones:**

1. Ejecuta consultas con funciones de agregación:

```sql
-- Consulta 7: Contar clientes por segmento de mercado
SELECT
    C_MKTSEGMENT            AS segmento,
    COUNT(*)                AS total_clientes,
    ROUND(AVG(C_ACCTBAL), 2) AS saldo_promedio
FROM CUSTOMER
GROUP BY C_MKTSEGMENT
ORDER BY total_clientes DESC;
```

```sql
-- Consulta 8: Resumen de órdenes por estado con métricas
SELECT
    O_ORDERSTATUS                   AS estado_orden,
    COUNT(*)                        AS cantidad_ordenes,
    ROUND(SUM(O_TOTALPRICE), 2)     AS ingreso_total,
    ROUND(AVG(O_TOTALPRICE), 2)     AS precio_promedio,
    MIN(O_ORDERDATE)                AS primera_orden,
    MAX(O_ORDERDATE)                AS ultima_orden
FROM ORDERS
GROUP BY O_ORDERSTATUS
ORDER BY cantidad_ordenes DESC;
```

```sql
-- Consulta 9: Top 5 naciones con más proveedores
SELECT
    N.N_NAME        AS nacion,
    COUNT(S.S_SUPPKEY) AS total_proveedores
FROM SUPPLIER S
JOIN NATION N ON S.S_NATIONKEY = N.N_NATIONKEY
GROUP BY N.N_NAME
ORDER BY total_proveedores DESC
LIMIT 5;
```

```sql
-- Consulta 10: Distribución de órdenes por año
SELECT
    YEAR(O_ORDERDATE)   AS anio,
    COUNT(*)            AS total_ordenes,
    ROUND(SUM(O_TOTALPRICE), 2) AS ingreso_anual
FROM ORDERS
GROUP BY YEAR(O_ORDERDATE)
ORDER BY anio;
```

2. Analiza los resultados:
   - La Consulta 7 debe mostrar exactamente 5 segmentos de mercado.
   - La Consulta 8 debe mostrar 3 estados de orden: `F` (fulfilled), `O` (open), `P` (pending).
   - La Consulta 10 muestra la distribución temporal de las órdenes.

**Resultado Esperado:**

**Consulta 7** (valores aproximados para TPCH_SF1 con 150,000 clientes):

| segmento | total_clientes | saldo_promedio |
|----------|---------------|----------------|
| BUILDING | ~30,142 | ~4,500 |
| AUTOMOBILE | ~29,968 | ~4,500 |
| MACHINERY | ~30,027 | ~4,500 |
| HOUSEHOLD | ~29,947 | ~4,500 |
| FURNITURE | ~29,916 | ~4,500 |

**Consulta 8** (valores aproximados para 1,500,000 órdenes):

| estado_orden | cantidad_ordenes | ingreso_total |
|---|---|---|
| F | ~729,413 | ~alto |
| O | ~733,392 | ~alto |
| P | ~37,195 | ~bajo |

**Verificación:**
- La suma de `total_clientes` en la Consulta 7 debe ser exactamente 150,000 (escala SF1).
- La Consulta 8 debe mostrar exactamente 3 filas (tres estados posibles).
- La Consulta 9 debe retornar exactamente 5 filas.

---

### Paso 8: Revisar el Historial de Ejecución y Métricas de Rendimiento

**Objetivo:** Utilizar la sección Activity → Query History para analizar las consultas ejecutadas y sus métricas.

**Instrucciones:**

1. Navega a **Activity** → **Query History** en el panel lateral izquierdo.

2. En la vista del historial, aplica los siguientes filtros:
   - **User:** tu nombre de usuario.
   - **Status:** Succeeded.
   - **Time range:** Last hour (última hora).

3. Localiza las consultas ejecutadas durante este laboratorio. Para cada una, observa:
   - **Duration:** tiempo total de ejecución en milisegundos o segundos.
   - **Rows:** número de filas retornadas.
   - **Bytes Scanned:** volumen de datos leídos del almacenamiento.

4. Haz clic en una de las consultas de agregación (Consulta 7 u 8) para ver su detalle. Observa:
   - El texto SQL completo.
   - El **Query ID** (identificador único de la consulta).
   - Las pestañas **Results**, **Query Profile** y **Statistics**.

5. Haz clic en la pestaña **Query Profile** de la consulta seleccionada:
   - Observa el diagrama de nodos que muestra los pasos de ejecución.
   - Identifica los nodos principales: `TableScan`, `Aggregate`, `Sort`, `Result`.
   - Nota el porcentaje de tiempo dedicado a cada operación.

6. Regresa a tu worksheet y ejecuta una consulta para acceder al historial programáticamente:

```sql
-- Consultar el historial de consultas de la última hora
-- (requiere rol ACCOUNTADMIN o privilegios específicos)
USE ROLE ACCOUNTADMIN;

SELECT
    QUERY_ID,
    QUERY_TEXT,
    USER_NAME,
    WAREHOUSE_NAME,
    EXECUTION_STATUS,
    TOTAL_ELAPSED_TIME / 1000 AS duracion_segundos,
    BYTES_SCANNED,
    ROWS_PRODUCED,
    START_TIME
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
    DATEADD('hours', -1, CURRENT_TIMESTAMP()),
    CURRENT_TIMESTAMP()
))
WHERE USER_NAME = CURRENT_USER()
ORDER BY START_TIME DESC
LIMIT 15;
```

7. Vuelve al rol `SYSADMIN` para mantener buenas prácticas:

```sql
-- Regresar al rol SYSADMIN
USE ROLE SYSADMIN;
```

**Resultado Esperado:**

La consulta del historial debe retornar las últimas 15 consultas ejecutadas con información como:

| QUERY_ID | QUERY_TEXT (truncado) | duracion_segundos | BYTES_SCANNED | ROWS_PRODUCED |
|---|---|---|---|---|
| `01b...` | `SELECT C_MKTSEGMENT...` | 0.5–3.0 | ~1,000,000+ | 5 |
| `01b...` | `SELECT O_ORDERSTATUS...` | 0.5–3.0 | ~5,000,000+ | 3 |

**Verificación:**
- Todas las consultas deben mostrar `EXECUTION_STATUS = 'SUCCESS'`.
- Las consultas sobre tablas más grandes (`ORDERS` con 1.5M filas) deben mostrar más `BYTES_SCANNED` que las consultas sobre tablas pequeñas (`NATION` con 25 filas).
- La duración típica para estas consultas en un warehouse XS debe ser inferior a 5 segundos.

---

## 7. Validación y Pruebas Finales

Ejecuta el siguiente bloque de validación como comprobación final de que todos los objetivos del laboratorio se cumplieron:

```sql
-- ============================================
-- VALIDACIÓN FINAL - LAB01
-- ============================================
USE ROLE SYSADMIN;
USE WAREHOUSE COMPUTE_WH;
USE DATABASE SNOWFLAKE_SAMPLE_DATA;
USE SCHEMA TPCH_SF1;

-- Test 1: Contexto correctamente configurado
SELECT
    CASE WHEN CURRENT_ROLE() = 'SYSADMIN' THEN '✓ PASS' ELSE '✗ FAIL' END AS test_rol,
    CASE WHEN CURRENT_WAREHOUSE() = 'COMPUTE_WH' THEN '✓ PASS' ELSE '✗ FAIL' END AS test_warehouse,
    CASE WHEN CURRENT_DATABASE() = 'SNOWFLAKE_SAMPLE_DATA' THEN '✓ PASS' ELSE '✗ FAIL' END AS test_database,
    CASE WHEN CURRENT_SCHEMA() = 'TPCH_SF1' THEN '✓ PASS' ELSE '✗ FAIL' END AS test_schema;

-- Test 2: Acceso a tablas de muestra
SELECT
    CASE WHEN COUNT(*) = 150000 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_customer_count
FROM CUSTOMER;

-- Test 3: Acceso a tabla ORDERS
SELECT
    CASE WHEN COUNT(*) = 1500000 THEN '✓ PASS' ELSE '✗ FAIL' END AS test_orders_count
FROM ORDERS;

-- Test 4: Verificar región de la cuenta
SELECT
    CASE WHEN CURRENT_REGION() = 'AWS_US_EAST_1' THEN '✓ PASS' ELSE '✗ FAIL' END AS test_region;
```

**Criterios de Éxito:**
- Los 4 tests deben mostrar `✓ PASS`.
- Si alguno muestra `✗ FAIL`, revisa el paso correspondiente antes de continuar al siguiente laboratorio.

---

## 8. Solución de Problemas

### Problema 1: Error "No active warehouse selected" al ejecutar consultas

**Síntomas:**
- Al ejecutar cualquier consulta SELECT, aparece el error:
  ```
  SQL execution error: No active warehouse selected in the current session. Select an active warehouse with the 'use warehouse' command.
  ```

**Causa:**
El warehouse no está seleccionado en el contexto de la sesión. Esto ocurre cuando se crea una nueva worksheet sin configurar el contexto, o cuando el warehouse se suspendió y la sesión perdió la referencia.

**Solución:**
1. Ejecuta el comando:
   ```sql
   USE WAREHOUSE COMPUTE_WH;
   ```
2. Alternativamente, selecciona `COMPUTE_WH` en el selector de warehouse ubicado en la parte superior derecha de la worksheet.
3. Si el warehouse no aparece en la lista, verifica que tu rol activo tiene privilegio `USAGE` sobre el warehouse:
   ```sql
   USE ROLE ACCOUNTADMIN;
   SHOW GRANTS ON WAREHOUSE COMPUTE_WH;
   ```

---

### Problema 2: La base de datos SNOWFLAKE_SAMPLE_DATA no aparece o las tablas están vacías

**Síntomas:**
- Al navegar en Data → Databases, no se ve `SNOWFLAKE_SAMPLE_DATA`.
- O bien, al ejecutar `SELECT * FROM CUSTOMER LIMIT 10;` se obtiene el error:
  ```
  Object 'CUSTOMER' does not exist or not authorized.
  ```

**Causa:**
El rol activo no tiene privilegios sobre la base de datos de muestra. Esto sucede cuando se está usando el rol `PUBLIC` o un rol personalizado sin los grants necesarios. La base de datos `SNOWFLAKE_SAMPLE_DATA` es compartida (share) y requiere el privilegio `IMPORTED PRIVILEGES`.

**Solución:**
1. Cambia al rol `ACCOUNTADMIN`:
   ```sql
   USE ROLE ACCOUNTADMIN;
   ```
2. Verifica que la base de datos existe:
   ```sql
   SHOW DATABASES LIKE 'SNOWFLAKE_SAMPLE_DATA';
   ```
3. Si existe pero no es accesible desde `SYSADMIN`, otorga los privilegios:
   ```sql
   GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE_SAMPLE_DATA TO ROLE SYSADMIN;
   ```
4. Cambia de vuelta a `SYSADMIN` y verifica el acceso:
   ```sql
   USE ROLE SYSADMIN;
   USE DATABASE SNOWFLAKE_SAMPLE_DATA;
   USE SCHEMA TPCH_SF1;
   SELECT COUNT(*) FROM CUSTOMER;
   ```

---

## 9. Limpieza

Este laboratorio **no crea objetos permanentes** en la cuenta Snowflake. No se requiere limpieza de bases de datos, tablas o warehouses.

Las únicas acciones opcionales de limpieza son:

```sql
-- Opcional: Si deseas mantener orden en tus worksheets,
-- puedes conservar LAB01_ExploracionInicial para referencia futura.
-- No es necesario eliminarla.

-- Suspender el warehouse manualmente si no se usará inmediatamente
-- (normalmente AUTO_SUSPEND lo hará automáticamente tras 300 segundos)
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

> **Nota:** A partir del Laboratorio 02, se crearán warehouses dedicados (`WH_CURSO_XS`) y la base de datos del curso (`DB_CURSO_SNOWFLAKE`). El warehouse `COMPUTE_WH` no se utilizará en laboratorios posteriores.

---

## 10. Resumen

En este laboratorio completaste los siguientes logros:

| Actividad | Estado |
|---|---|
| Acceso a cuenta Snowflake Trial y verificación de región/edición | ✓ |
| Exploración completa de secciones de Snowsight | ✓ |
| Identificación de roles del sistema y jerarquía | ✓ |
| Configuración de contexto de sesión (UI + SQL USE) | ✓ |
| Ejecución de 10 consultas SQL progresivas | ✓ |
| Revisión del historial de consultas y Query Profile | ✓ |

### Conceptos Clave Aprendidos

- **Contexto de sesión:** Los cuatro elementos (rol, warehouse, base de datos, esquema) deben estar configurados antes de ejecutar consultas.
- **Roles del sistema:** `ACCOUNTADMIN` > `SYSADMIN` > `PUBLIC` en jerarquía de privilegios. Usa `SYSADMIN` para trabajo diario.
- **Snowsight:** Las worksheets son el entorno principal de trabajo; Activity → Query History es la herramienta de diagnóstico.
- **Buenas prácticas:** Nombra tus worksheets descriptivamente, usa `SYSADMIN` en lugar de `ACCOUNTADMIN` para operaciones rutinarias.

### Preparación para el Siguiente Laboratorio

En el **Laboratorio 02** crearás:
- La base de datos principal del curso: `DB_CURSO_SNOWFLAKE`
- Los esquemas por capa: `SCH_STAGING`, `SCH_CORE`, `SCH_ANALYTICS`
- Los warehouses dedicados: `WH_CURSO_XS` y `WH_CURSO_LOAD_XS`

Asegúrate de que tu cuenta Trial está funcionando correctamente antes de continuar.

### Recursos Adicionales

- [Documentación oficial de Snowsight](https://docs.snowflake.com/en/user-guide/ui-snowsight)
- [Referencia de comandos USE](https://docs.snowflake.com/en/sql-reference/sql/use)
- [Guía de Query History](https://docs.snowflake.com/en/user-guide/ui-snowsight-activity)
- [Dataset TPC-H en Snowflake](https://docs.snowflake.com/en/user-guide/sample-data-tpch)

---
