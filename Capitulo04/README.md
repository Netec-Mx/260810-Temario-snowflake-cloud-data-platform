# Buenas Prácticas de Nomenclatura y Ambientes — Modelado Dimensional y Vistas Analíticas

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Worksheet** | `LAB04_Arquitectura_Capas` |

## Descripción General

En este laboratorio construirás un modelo dimensional completo sobre la base de datos `DB_CURSO_SNOWFLAKE`, moviendo datos desde la capa de staging (`SCH_STAGING`) hacia la capa core (`SCH_CORE`) con tablas de hechos y dimensiones, y finalmente exponiendo la información a través de vistas estándar y vistas seguras en la capa analytics (`SCH_ANALYTICS`). Aplicarás convenciones de nomenclatura DWH estándar y verificarás que las vistas responden preguntas de negocio sin exponer lógica interna a roles no propietarios.

## Objetivos de Aprendizaje

- [ ] Crear tablas de hechos (`FACT_VENTAS`) y dimensiones (`DIM_CLIENTES`, `DIM_PRODUCTOS`) en `SCH_CORE` siguiendo convenciones de nomenclatura estándar.
- [ ] Poblar las tablas de `SCH_CORE` transformando y limpiando datos desde `SCH_STAGING` con `INSERT INTO ... SELECT`.
- [ ] Crear vistas estándar en `SCH_ANALYTICS` que simplifiquen el acceso al modelo dimensional.
- [ ] Implementar vistas seguras (`SECURE VIEW`) que protejan lógica de negocio sensible y controlen acceso a datos.
- [ ] Validar que las vistas responden correctamente preguntas de negocio sobre ventas, clientes y productos.

## Prerrequisitos

### Conocimientos

- Comprensión de modelos dimensionales básicos (tablas de hechos y dimensiones).
- Familiaridad con sentencias DDL y DML en Snowflake (`CREATE TABLE`, `INSERT INTO ... SELECT`).
- Experiencia con JOINs y funciones de agregación SQL.

### Acceso y Entorno

- Laboratorio 03-02-01 completado: tablas `STG_CLIENTES` (50 registros), `STG_PRODUCTOS` (30 registros) y `STG_VENTAS` (200 registros) pobladas en `SCH_STAGING`.
- Esquemas `SCH_STAGING`, `SCH_CORE` y `SCH_ANALYTICS` existentes en `DB_CURSO_SNOWFLAKE`.
- Warehouse `WH_CURSO_XS` operativo.
- Rol `SYSADMIN` activo.

## Entorno del Laboratorio

| Componente | Detalle |
|---|---|
| Cuenta Snowflake | Enterprise Trial, AWS us-east-1 |
| Base de datos | `DB_CURSO_SNOWFLAKE` |
| Esquemas | `SCH_STAGING`, `SCH_CORE`, `SCH_ANALYTICS` |
| Warehouse | `WH_CURSO_XS` (X-Small, AUTO_SUSPEND=60) |
| Interfaz | Snowsight (Worksheets) |
| Rol | `SYSADMIN` |

### Configuración Inicial

Abre Snowsight y crea una nueva worksheet llamada `LAB04_Arquitectura_Capas`. Ejecuta el siguiente bloque para establecer el contexto:

```sql
-- Configuración de contexto
USE ROLE SYSADMIN;
USE WAREHOUSE WH_CURSO_XS;
USE DATABASE DB_CURSO_SNOWFLAKE;
```

Verifica que las tablas de staging contienen datos:

```sql
SELECT 'STG_CLIENTES' AS tabla, COUNT(*) AS registros FROM SCH_STAGING.STG_CLIENTES
UNION ALL
SELECT 'STG_PRODUCTOS', COUNT(*) FROM SCH_STAGING.STG_PRODUCTOS
UNION ALL
SELECT 'STG_VENTAS', COUNT(*) FROM SCH_STAGING.STG_VENTAS;
```

**Resultado esperado:**

| TABLA | REGISTROS |
|---|---|
| STG_CLIENTES | 50 |
| STG_PRODUCTOS | 30 |
| STG_VENTAS | 200 |

> ⚠️ Si los conteos no coinciden, regresa al laboratorio 03-02-01 y verifica la inserción de datos antes de continuar.

---

## Paso 1: Crear Tablas Dimensionales en SCH_CORE

### Objetivo

Definir las tablas `DIM_CLIENTES` y `DIM_PRODUCTOS` con columnas de metadata adicionales (fecha de creación, fecha de actualización, flag activo) siguiendo las convenciones de nomenclatura del curso.

### Instrucciones

1. Cambia al esquema `SCH_CORE`:

```sql
USE SCHEMA SCH_CORE;
```

2. Crea la tabla `DIM_CLIENTES`:

```sql
CREATE OR REPLACE TABLE SCH_CORE.DIM_CLIENTES (
    sk_cliente          INT AUTOINCREMENT START 1 INCREMENT 1,  -- Surrogate key
    id_cliente          INT             NOT NULL,               -- Business key
    nombre              VARCHAR(100)    NOT NULL,
    apellido            VARCHAR(100)    NOT NULL,
    email               VARCHAR(200),
    telefono            VARCHAR(50),
    ciudad              VARCHAR(100),
    pais                VARCHAR(100),
    segmento            VARCHAR(50),
    fecha_registro      DATE,
    activo              BOOLEAN         DEFAULT TRUE,
    -- Columnas de metadata
    fecha_creacion      TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    fecha_actualizacion TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    registro_activo     BOOLEAN         DEFAULT TRUE,
    CONSTRAINT pk_dim_clientes PRIMARY KEY (sk_cliente),
    CONSTRAINT uk_dim_clientes_bk UNIQUE (id_cliente)
)
COMMENT = 'Dimensión de clientes. Fuente: SCH_STAGING.STG_CLIENTES. Capa: Core.';
```

3. Crea la tabla `DIM_PRODUCTOS`:

```sql
CREATE OR REPLACE TABLE SCH_CORE.DIM_PRODUCTOS (
    sk_producto         INT AUTOINCREMENT START 1 INCREMENT 1,  -- Surrogate key
    id_producto         INT             NOT NULL,               -- Business key
    nombre_producto     VARCHAR(200)    NOT NULL,
    categoria           VARCHAR(100),
    subcategoria        VARCHAR(100),
    precio_unitario     NUMBER(12,2),
    costo_unitario      NUMBER(12,2),
    margen_unitario     NUMBER(12,2),                           -- Columna calculada
    unidad_medida       VARCHAR(50),
    activo              BOOLEAN         DEFAULT TRUE,
    -- Columnas de metadata
    fecha_creacion      TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    fecha_actualizacion TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    registro_activo     BOOLEAN         DEFAULT TRUE,
    CONSTRAINT pk_dim_productos PRIMARY KEY (sk_producto),
    CONSTRAINT uk_dim_productos_bk UNIQUE (id_producto)
)
COMMENT = 'Dimensión de productos con margen calculado. Fuente: SCH_STAGING.STG_PRODUCTOS. Capa: Core.';
```

### Resultado Esperado

Ambas sentencias deben retornar: `Table DIM_CLIENTES successfully created.` y `Table DIM_PRODUCTOS successfully created.`

### Verificación

```sql
SHOW TABLES IN SCHEMA SCH_CORE;
```

Debes ver `DIM_CLIENTES` y `DIM_PRODUCTOS` en la lista de tablas.

---

## Paso 2: Crear la Tabla de Hechos FACT_VENTAS

### Objetivo

Definir la tabla de hechos `FACT_VENTAS` con claves foráneas a las dimensiones y métricas calculadas (monto total, monto con descuento).

### Instrucciones

1. Crea la tabla de hechos:

```sql
CREATE OR REPLACE TABLE SCH_CORE.FACT_VENTAS (
    sk_venta            INT AUTOINCREMENT START 1 INCREMENT 1,
    id_venta            INT             NOT NULL,
    id_cliente          INT             NOT NULL,
    id_producto         INT             NOT NULL,
    fecha_venta         DATE            NOT NULL,
    cantidad            INT             NOT NULL,
    precio_venta        NUMBER(12,2)    NOT NULL,
    descuento           NUMBER(5,2)     DEFAULT 0,
    canal_venta         VARCHAR(50),
    region              VARCHAR(100),
    -- Métricas calculadas
    monto_bruto         NUMBER(14,2),
    monto_descuento     NUMBER(14,2),
    monto_neto          NUMBER(14,2),
    -- Metadata
    fecha_creacion      TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    CONSTRAINT pk_fact_ventas PRIMARY KEY (sk_venta)
)
COMMENT = 'Tabla de hechos de ventas. Métricas: monto_bruto, monto_descuento, monto_neto. Capa: Core.';
```

### Resultado Esperado

`Table FACT_VENTAS successfully created.`

### Verificación

```sql
SELECT table_name, comment
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'SCH_CORE'
ORDER BY table_name;
```

| TABLE_NAME | COMMENT |
|---|---|
| DIM_CLIENTES | Dimensión de clientes... |
| DIM_PRODUCTOS | Dimensión de productos... |
| FACT_VENTAS | Tabla de hechos de ventas... |

---

## Paso 3: Poblar Tablas de SCH_CORE desde SCH_STAGING

### Objetivo

Mover y transformar datos desde staging hacia core aplicando limpieza: `TRIM` en strings, `COALESCE` para nulos y cálculo de métricas derivadas.

### Instrucciones

1. Pobla `DIM_CLIENTES` con transformaciones:

```sql
INSERT INTO SCH_CORE.DIM_CLIENTES (
    id_cliente, nombre, apellido, email, telefono,
    ciudad, pais, segmento, fecha_registro, activo
)
SELECT
    id_cliente,
    TRIM(nombre),
    TRIM(apellido),
    LOWER(TRIM(email)),
    COALESCE(TRIM(telefono), 'N/A'),
    COALESCE(TRIM(ciudad), 'Sin ciudad'),
    COALESCE(TRIM(pais), 'Sin país'),
    COALESCE(TRIM(segmento), 'General'),
    fecha_registro,
    COALESCE(activo, TRUE)
FROM SCH_STAGING.STG_CLIENTES;
```

2. Pobla `DIM_PRODUCTOS` con cálculo de margen:

```sql
INSERT INTO SCH_CORE.DIM_PRODUCTOS (
    id_producto, nombre_producto, categoria, subcategoria,
    precio_unitario, costo_unitario, margen_unitario,
    unidad_medida, activo
)
SELECT
    id_producto,
    TRIM(nombre_producto),
    COALESCE(TRIM(categoria), 'Sin categoría'),
    COALESCE(TRIM(subcategoria), 'Sin subcategoría'),
    precio_unitario,
    costo_unitario,
    ROUND(precio_unitario - costo_unitario, 2) AS margen_unitario,
    COALESCE(TRIM(unidad_medida), 'Unidad'),
    COALESCE(activo, TRUE)
FROM SCH_STAGING.STG_PRODUCTOS;
```

3. Pobla `FACT_VENTAS` con métricas calculadas:

```sql
INSERT INTO SCH_CORE.FACT_VENTAS (
    id_venta, id_cliente, id_producto, fecha_venta,
    cantidad, precio_venta, descuento, canal_venta, region,
    monto_bruto, monto_descuento, monto_neto
)
SELECT
    id_venta,
    id_cliente,
    id_producto,
    fecha_venta,
    cantidad,
    precio_venta,
    COALESCE(descuento, 0),
    COALESCE(TRIM(canal_venta), 'Desconocido'),
    COALESCE(TRIM(region), 'Sin región'),
    ROUND(cantidad * precio_venta, 2)                                    AS monto_bruto,
    ROUND(cantidad * precio_venta * COALESCE(descuento, 0) / 100, 2)    AS monto_descuento,
    ROUND(cantidad * precio_venta * (1 - COALESCE(descuento, 0) / 100), 2) AS monto_neto
FROM SCH_STAGING.STG_VENTAS;
```

### Resultado Esperado

- `DIM_CLIENTES`: 50 filas insertadas.
- `DIM_PRODUCTOS`: 30 filas insertadas.
- `FACT_VENTAS`: 200 filas insertadas.

### Verificación

```sql
SELECT 'DIM_CLIENTES' AS tabla, COUNT(*) AS registros FROM SCH_CORE.DIM_CLIENTES
UNION ALL
SELECT 'DIM_PRODUCTOS', COUNT(*) FROM SCH_CORE.DIM_PRODUCTOS
UNION ALL
SELECT 'FACT_VENTAS', COUNT(*) FROM SCH_CORE.FACT_VENTAS;
```

Confirma que los conteos coinciden con los de staging. Además verifica la limpieza:

```sql
-- Verificar que no hay espacios en blanco residuales
SELECT COUNT(*) AS registros_con_espacios
FROM SCH_CORE.DIM_CLIENTES
WHERE nombre != TRIM(nombre) OR apellido != TRIM(apellido);
-- Debe retornar 0
```

---

## Paso 4: Crear Vistas Estándar en SCH_ANALYTICS

### Objetivo

Crear 4 vistas estándar que encapsulen la lógica de negocio y simplifiquen el acceso analítico al modelo dimensional.

### Instrucciones

1. Cambia al esquema analytics:

```sql
USE SCHEMA SCH_ANALYTICS;
```

2. **VW_VENTAS_POR_CLIENTE** — Análisis de ventas agrupado por cliente:

```sql
CREATE OR REPLACE VIEW SCH_ANALYTICS.VW_VENTAS_POR_CLIENTE
    COMMENT = 'Resumen de ventas por cliente con métricas agregadas. JOIN entre FACT_VENTAS y DIM_CLIENTES.'
AS
SELECT
    c.id_cliente,
    c.nombre || ' ' || c.apellido       AS nombre_completo,
    c.ciudad,
    c.pais,
    c.segmento,
    COUNT(f.sk_venta)                    AS total_transacciones,
    SUM(f.cantidad)                      AS total_unidades,
    SUM(f.monto_bruto)                   AS total_bruto,
    SUM(f.monto_neto)                    AS total_neto,
    ROUND(AVG(f.monto_neto), 2)          AS ticket_promedio,
    MIN(f.fecha_venta)                   AS primera_compra,
    MAX(f.fecha_venta)                   AS ultima_compra
FROM SCH_CORE.FACT_VENTAS       f
JOIN SCH_CORE.DIM_CLIENTES      c ON f.id_cliente = c.id_cliente
GROUP BY c.id_cliente, c.nombre, c.apellido, c.ciudad, c.pais, c.segmento;
```

3. **VW_VENTAS_POR_PRODUCTO** — Análisis por producto y categoría:

```sql
CREATE OR REPLACE VIEW SCH_ANALYTICS.VW_VENTAS_POR_PRODUCTO
    COMMENT = 'Análisis de ventas por producto y categoría con margen calculado.'
AS
SELECT
    p.id_producto,
    p.nombre_producto,
    p.categoria,
    p.subcategoria,
    p.precio_unitario,
    p.costo_unitario,
    p.margen_unitario,
    COUNT(f.sk_venta)                    AS total_transacciones,
    SUM(f.cantidad)                      AS total_unidades_vendidas,
    SUM(f.monto_bruto)                   AS ingreso_bruto,
    SUM(f.monto_neto)                    AS ingreso_neto,
    SUM(f.cantidad * p.costo_unitario)   AS costo_total,
    SUM(f.monto_neto) - SUM(f.cantidad * p.costo_unitario) AS margen_total,
    ROUND(
        (SUM(f.monto_neto) - SUM(f.cantidad * p.costo_unitario))
        / NULLIF(SUM(f.monto_neto), 0) * 100, 2
    )                                    AS margen_pct
FROM SCH_CORE.FACT_VENTAS       f
JOIN SCH_CORE.DIM_PRODUCTOS     p ON f.id_producto = p.id_producto
GROUP BY p.id_producto, p.nombre_producto, p.categoria, p.subcategoria,
         p.precio_unitario, p.costo_unitario, p.margen_unitario;
```

4. **VW_VENTAS_MENSUALES** — Serie temporal mensual:

```sql
CREATE OR REPLACE VIEW SCH_ANALYTICS.VW_VENTAS_MENSUALES
    COMMENT = 'Serie temporal de ventas mensuales con totales y promedios.'
AS
SELECT
    DATE_TRUNC('MONTH', f.fecha_venta)   AS mes,
    COUNT(DISTINCT f.id_venta)           AS num_ventas,
    COUNT(DISTINCT f.id_cliente)         AS clientes_unicos,
    SUM(f.cantidad)                      AS unidades_vendidas,
    SUM(f.monto_bruto)                   AS ingreso_bruto,
    SUM(f.monto_descuento)               AS total_descuentos,
    SUM(f.monto_neto)                    AS ingreso_neto,
    ROUND(AVG(f.monto_neto), 2)          AS ticket_promedio
FROM SCH_CORE.FACT_VENTAS f
GROUP BY DATE_TRUNC('MONTH', f.fecha_venta)
ORDER BY mes;
```

5. **VW_RESUMEN_EJECUTIVO** — Métricas consolidadas de alto nivel:

```sql
CREATE OR REPLACE VIEW SCH_ANALYTICS.VW_RESUMEN_EJECUTIVO
    COMMENT = 'Resumen ejecutivo con KPIs principales del negocio.'
AS
SELECT
    COUNT(DISTINCT f.id_venta)           AS total_ventas,
    COUNT(DISTINCT f.id_cliente)         AS total_clientes_activos,
    COUNT(DISTINCT f.id_producto)        AS total_productos_vendidos,
    SUM(f.monto_bruto)                   AS ingreso_bruto_total,
    SUM(f.monto_neto)                    AS ingreso_neto_total,
    SUM(f.monto_descuento)               AS descuentos_otorgados,
    ROUND(AVG(f.monto_neto), 2)          AS ticket_promedio_global,
    ROUND(SUM(f.monto_descuento) / NULLIF(SUM(f.monto_bruto), 0) * 100, 2) AS pct_descuento_promedio,
    MIN(f.fecha_venta)                   AS fecha_primera_venta,
    MAX(f.fecha_venta)                   AS fecha_ultima_venta
FROM SCH_CORE.FACT_VENTAS f;
```

### Resultado Esperado

Cada sentencia retorna: `View VW_... successfully created.`

### Verificación

```sql
SHOW VIEWS IN SCHEMA SCH_ANALYTICS;
```

Debes ver 4 vistas listadas con `is_secure = false`.

---

## Paso 5: Crear Vistas Seguras en SCH_ANALYTICS

### Objetivo

Implementar 2 vistas seguras que protejan datos sensibles (PII de clientes) y lógica de negocio confidencial (cálculo de márgenes).

### Instrucciones

1. **SVW_DATOS_CLIENTES_SENSIBLES** — Controla acceso a PII según el rol:

```sql
CREATE OR REPLACE SECURE VIEW SCH_ANALYTICS.SVW_DATOS_CLIENTES_SENSIBLES
    COMMENT = 'Vista segura de clientes. Expone PII completo solo a ACCOUNTADMIN y SYSADMIN.'
AS
SELECT
    c.id_cliente,
    c.nombre || ' ' || c.apellido       AS nombre_completo,
    c.ciudad,
    c.pais,
    c.segmento,
    -- Lógica condicional basada en rol
    CASE
        WHEN CURRENT_ROLE() IN ('ACCOUNTADMIN', 'SYSADMIN')
            THEN c.email
        ELSE '***PROTEGIDO***'
    END                                  AS email,
    CASE
        WHEN CURRENT_ROLE() IN ('ACCOUNTADMIN', 'SYSADMIN')
            THEN c.telefono
        ELSE '***PROTEGIDO***'
    END                                  AS telefono,
    c.fecha_registro,
    c.activo
FROM SCH_CORE.DIM_CLIENTES c
WHERE c.registro_activo = TRUE;
```

2. **SVW_MARGENES_PRODUCTOS** — Protege la lógica de cálculo de márgenes:

```sql
CREATE OR REPLACE SECURE VIEW SCH_ANALYTICS.SVW_MARGENES_PRODUCTOS
    COMMENT = 'Vista segura con márgenes de productos. Lógica de cálculo protegida.'
AS
SELECT
    p.id_producto,
    p.nombre_producto,
    p.categoria,
    p.subcategoria,
    SUM(f.cantidad)                                              AS unidades_vendidas,
    SUM(f.monto_neto)                                            AS ingreso_neto,
    SUM(f.cantidad * p.costo_unitario)                           AS costo_total,
    SUM(f.monto_neto) - SUM(f.cantidad * p.costo_unitario)      AS margen_absoluto,
    ROUND(
        (SUM(f.monto_neto) - SUM(f.cantidad * p.costo_unitario))
        / NULLIF(SUM(f.monto_neto), 0) * 100, 2
    )                                                            AS margen_pct,
    CASE
        WHEN (SUM(f.monto_neto) - SUM(f.cantidad * p.costo_unitario))
             / NULLIF(SUM(f.monto_neto), 0) * 100 > 40 THEN 'Alto'
        WHEN (SUM(f.monto_neto) - SUM(f.cantidad * p.costo_unitario))
             / NULLIF(SUM(f.monto_neto), 0) * 100 > 20 THEN 'Medio'
        ELSE 'Bajo'
    END                                                          AS clasificacion_margen
FROM SCH_CORE.FACT_VENTAS       f
JOIN SCH_CORE.DIM_PRODUCTOS     p ON f.id_producto = p.id_producto
GROUP BY p.id_producto, p.nombre_producto, p.categoria, p.subcategoria;
```

### Resultado Esperado

Cada sentencia retorna: `View SVW_... successfully created.`

### Verificación

```sql
-- Confirmar que las vistas son seguras
SELECT table_name, is_secure
FROM INFORMATION_SCHEMA.VIEWS
WHERE table_schema = 'SCH_ANALYTICS'
ORDER BY table_name;
```

| TABLE_NAME | IS_SECURE |
|---|---|
| SVW_DATOS_CLIENTES_SENSIBLES | YES |
| SVW_MARGENES_PRODUCTOS | YES |
| VW_RESUMEN_EJECUTIVO | NO |
| VW_VENTAS_MENSUALES | NO |
| VW_VENTAS_POR_CLIENTE | NO |
| VW_VENTAS_POR_PRODUCTO | NO |

---

## Paso 6: Verificar Protección de Vistas Seguras

### Objetivo

Demostrar que la definición SQL de las vistas seguras no es visible para roles no propietarios y que `GET_DDL` respeta esta restricción.

### Instrucciones

1. Verifica la definición como propietario (`SYSADMIN`):

```sql
-- Como SYSADMIN (propietario), sí podemos ver la definición
SELECT GET_DDL('VIEW', 'DB_CURSO_SNOWFLAKE.SCH_ANALYTICS.SVW_MARGENES_PRODUCTOS');
```

Esto debe retornar la definición SQL completa.

2. Verifica la visibilidad en `SHOW VIEWS`:

```sql
SHOW VIEWS LIKE 'SVW_%' IN SCHEMA SCH_ANALYTICS;
```

Observa la columna `text`: para vistas seguras, si consultas desde un rol no propietario, el campo estará vacío. Como `SYSADMIN` es el propietario, verás el texto completo.

3. Verifica el comportamiento de la vista con datos PII:

```sql
-- Como SYSADMIN, debemos ver los datos completos
SELECT * FROM SCH_ANALYTICS.SVW_DATOS_CLIENTES_SENSIBLES LIMIT 5;
```

Los campos `email` y `telefono` deben mostrar valores reales (no enmascarados) porque estamos usando `SYSADMIN`.

4. (Opcional) Si deseas simular un rol sin privilegios para ver el enmascaramiento, puedes crear un rol de prueba:

```sql
-- Simulación rápida (opcional)
USE ROLE ACCOUNTADMIN;
CREATE ROLE IF NOT EXISTS ROLE_ANALISTA_BASICO;
GRANT USAGE ON DATABASE DB_CURSO_SNOWFLAKE TO ROLE ROLE_ANALISTA_BASICO;
GRANT USAGE ON SCHEMA DB_CURSO_SNOWFLAKE.SCH_ANALYTICS TO ROLE ROLE_ANALISTA_BASICO;
GRANT SELECT ON ALL VIEWS IN SCHEMA DB_CURSO_SNOWFLAKE.SCH_ANALYTICS TO ROLE ROLE_ANALISTA_BASICO;
GRANT USAGE ON WAREHOUSE WH_CURSO_XS TO ROLE ROLE_ANALISTA_BASICO;
GRANT ROLE ROLE_ANALISTA_BASICO TO USER ADMIN;

-- Cambiar al rol de prueba
USE ROLE ROLE_ANALISTA_BASICO;
USE WAREHOUSE WH_CURSO_XS;
SELECT * FROM DB_CURSO_SNOWFLAKE.SCH_ANALYTICS.SVW_DATOS_CLIENTES_SENSIBLES LIMIT 5;
-- email y telefono deben mostrar '***PROTEGIDO***'

-- Intentar ver la definición (debe fallar o retornar vacío)
SELECT GET_DDL('VIEW', 'DB_CURSO_SNOWFLAKE.SCH_ANALYTICS.SVW_MARGENES_PRODUCTOS');
-- Error: Insufficient privileges o definición no visible

-- Regresar al rol principal
USE ROLE SYSADMIN;
```

### Resultado Esperado

- Como `SYSADMIN`: datos PII visibles, definición SQL accesible.
- Como `ROLE_ANALISTA_BASICO`: campos PII enmascarados, definición SQL no disponible.

---

## Paso 7: Responder Preguntas de Negocio Usando las Vistas

### Objetivo

Demostrar la ventaja de abstracción ejecutando consultas de negocio exclusivamente sobre las vistas de `SCH_ANALYTICS`, sin referenciar tablas base.

### Instrucciones

1. **¿Cuáles son los 5 clientes con mayor ingreso neto?**

```sql
SELECT nombre_completo, segmento, total_neto, total_transacciones
FROM SCH_ANALYTICS.VW_VENTAS_POR_CLIENTE
ORDER BY total_neto DESC
LIMIT 5;
```

2. **¿Cuál es la categoría de producto más rentable?**

```sql
SELECT categoria, SUM(ingreso_neto) AS ingreso_total, SUM(margen_total) AS margen_total, 
       ROUND(SUM(margen_total) / NULLIF(SUM(ingreso_neto), 0) * 100, 2) AS margen_pct
FROM SCH_ANALYTICS.VW_VENTAS_POR_PRODUCTO
GROUP BY categoria
ORDER BY margen_total DESC;
```

3. **¿Cuál es la tendencia mensual de ventas?**

```sql
SELECT mes, num_ventas, ingreso_neto, clientes_unicos
FROM SCH_ANALYTICS.VW_VENTAS_MENSUALES
ORDER BY mes;
```

4. **¿Cuál es el resumen ejecutivo general?**

```sql
SELECT * FROM SCH_ANALYTICS.VW_RESUMEN_EJECUTIVO;
```

5. **¿Qué productos tienen margen alto según la vista segura?**

```sql
SELECT nombre_producto, categoria, margen_pct, clasificacion_margen
FROM SCH_ANALYTICS.SVW_MARGENES_PRODUCTOS
WHERE clasificacion_margen = 'Alto'
ORDER BY margen_pct DESC;
```

### Resultado Esperado

Cada consulta retorna resultados coherentes sin necesidad de escribir JOINs ni conocer la estructura interna del modelo. Las consultas son simples (2-5 líneas) y autoexplicativas.

### Verificación

Confirma que la consulta 4 (`VW_RESUMEN_EJECUTIVO`) retorna exactamente 1 fila con `total_ventas = 200` (o el número esperado según tus datos).

---

## Validación y Testing

Ejecuta el siguiente script de validación completo para confirmar que todo el laboratorio está correctamente implementado:

```sql
-- ============================================================
-- SCRIPT DE VALIDACIÓN - LAB04
-- ============================================================
USE ROLE SYSADMIN;
USE DATABASE DB_CURSO_SNOWFLAKE;

-- 1. Validar existencia de tablas en SCH_CORE
SELECT 'TABLAS_CORE' AS validacion,
       COUNT(*) AS total,
       CASE WHEN COUNT(*) = 3 THEN '✅ PASS' ELSE '❌ FAIL' END AS resultado
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'SCH_CORE' AND table_type = 'BASE TABLE'
  AND table_name IN ('DIM_CLIENTES', 'DIM_PRODUCTOS', 'FACT_VENTAS');

-- 2. Validar conteo de registros en SCH_CORE
SELECT 'REGISTROS_CORE' AS validacion,
       (SELECT COUNT(*) FROM SCH_CORE.DIM_CLIENTES) AS dim_clientes,
       (SELECT COUNT(*) FROM SCH_CORE.DIM_PRODUCTOS) AS dim_productos,
       (SELECT COUNT(*) FROM SCH_CORE.FACT_VENTAS) AS fact_ventas,
       CASE 
           WHEN (SELECT COUNT(*) FROM SCH_CORE.DIM_CLIENTES) = 50
            AND (SELECT COUNT(*) FROM SCH_CORE.DIM_PRODUCTOS) = 30
            AND (SELECT COUNT(*) FROM SCH_CORE.FACT_VENTAS) = 200
           THEN '✅ PASS' ELSE '❌ FAIL' 
       END AS resultado;

-- 3. Validar existencia de vistas en SCH_ANALYTICS
SELECT 'VISTAS_ANALYTICS' AS validacion,
       COUNT(*) AS total,
       CASE WHEN COUNT(*) = 6 THEN '✅ PASS' ELSE '❌ FAIL' END AS resultado
FROM INFORMATION_SCHEMA.VIEWS
WHERE table_schema = 'SCH_ANALYTICS';

-- 4. Validar vistas seguras
SELECT 'VISTAS_SEGURAS' AS validacion,
       COUNT(*) AS total_seguras,
       CASE WHEN COUNT(*) = 2 THEN '✅ PASS' ELSE '❌ FAIL' END AS resultado
FROM INFORMATION_SCHEMA.VIEWS
WHERE table_schema = 'SCH_ANALYTICS' AND is_secure = 'YES';

-- 5. Validar que las vistas retornan datos
SELECT 'VW_RESUMEN_EJECUTIVO' AS validacion,
       total_ventas,
       CASE WHEN total_ventas > 0 THEN '✅ PASS' ELSE '❌ FAIL' END AS resultado
FROM SCH_ANALYTICS.VW_RESUMEN_EJECUTIVO;

-- 6. Validar limpieza de datos (sin NULLs en campos clave)
SELECT 'LIMPIEZA_DATOS' AS validacion,
       COUNT(*) AS nulos_encontrados,
       CASE WHEN COUNT(*) = 0 THEN '✅ PASS' ELSE '❌ FAIL' END AS resultado
FROM SCH_CORE.DIM_CLIENTES
WHERE nombre IS NULL OR apellido IS NULL;
```

**Criterio de éxito:** Todas las validaciones deben retornar `✅ PASS`.

---

## Solución de Problemas

### Problema 1: Error "Object does not exist" al crear vistas en SCH_ANALYTICS

**Síntomas:** Al ejecutar `CREATE VIEW` en `SCH_ANALYTICS`, Snowflake retorna error indicando que `SCH_CORE.FACT_VENTAS` o `SCH_CORE.DIM_CLIENTES` no existe.

**Causa:** El contexto de esquema activo es `SCH_ANALYTICS` y las referencias a tablas en `SCH_CORE` no incluyen el nombre completo del esquema, o el esquema `SCH_CORE` no existe.

**Solución:**
1. Verifica que usas nombres completamente calificados en la definición de la vista (por ejemplo, `SCH_CORE.FACT_VENTAS` en lugar de solo `FACT_VENTAS`).
2. Confirma la existencia del esquema: `SHOW SCHEMAS IN DATABASE DB_CURSO_SNOWFLAKE;`
3. Si el esquema no existe, créalo: `CREATE SCHEMA IF NOT EXISTS SCH_CORE;` y repite desde el Paso 1.

---

### Problema 2: La vista segura SVW_DATOS_CLIENTES_SENSIBLES muestra '***PROTEGIDO***' incluso con SYSADMIN

**Síntomas:** Al consultar la vista segura como `SYSADMIN`, los campos `email` y `telefono` muestran `***PROTEGIDO***` en lugar de los valores reales.

**Causa:** La función `CURRENT_ROLE()` retorna el nombre del rol exactamente como está definido. Si el rol activo es `SYSADMIN` pero la condición en la vista compara contra una cadena diferente (por ejemplo, con espacios o minúsculas), la condición no se cumple.

**Solución:**
1. Verifica tu rol activo: `SELECT CURRENT_ROLE();` — debe retornar exactamente `SYSADMIN`.
2. Confirma que la condición en la vista usa la cadena correcta: `CURRENT_ROLE() IN ('ACCOUNTADMIN', 'SYSADMIN')`.
3. Si usaste un rol diferente (por ejemplo, `sysadmin` en minúsculas), corrige la vista o cambia al rol correcto con `USE ROLE SYSADMIN;`.

---

## Limpieza

> **Nota:** Las tablas y vistas creadas en este laboratorio se utilizan en laboratorios posteriores del curso. **No las elimines** a menos que el instructor lo indique.

Si creaste el rol de prueba opcional en el Paso 6, puedes limpiarlo:

```sql
USE ROLE ACCOUNTADMIN;
DROP ROLE IF EXISTS ROLE_ANALISTA_BASICO;
```

Si necesitas reiniciar completamente este laboratorio (solo si es necesario):

```sql
USE ROLE SYSADMIN;
USE DATABASE DB_CURSO_SNOWFLAKE;

-- Eliminar vistas
DROP VIEW IF EXISTS SCH_ANALYTICS.VW_VENTAS_POR_CLIENTE;
DROP VIEW IF EXISTS SCH_ANALYTICS.VW_VENTAS_POR_PRODUCTO;
DROP VIEW IF EXISTS SCH_ANALYTICS.VW_VENTAS_MENSUALES;
DROP VIEW IF EXISTS SCH_ANALYTICS.VW_RESUMEN_EJECUTIVO;
DROP VIEW IF EXISTS SCH_ANALYTICS.SVW_DATOS_CLIENTES_SENSIBLES;
DROP VIEW IF EXISTS SCH_ANALYTICS.SVW_MARGENES_PRODUCTOS;

-- Eliminar tablas core
DROP TABLE IF EXISTS SCH_CORE.FACT_VENTAS;
DROP TABLE IF EXISTS SCH_CORE.DIM_PRODUCTOS;
DROP TABLE IF EXISTS SCH_CORE.DIM_CLIENTES;
```

---

## Resumen

En este laboratorio completaste la implementación de una arquitectura de datos en tres capas:

| Capa | Esquema | Objetos Creados | Propósito |
|---|---|---|---|
| **Staging** | `SCH_STAGING` | `STG_CLIENTES`, `STG_PRODUCTOS`, `STG_VENTAS` | Datos crudos (lab anterior) |
| **Core** | `SCH_CORE` | `DIM_CLIENTES`, `DIM_PRODUCTOS`, `FACT_VENTAS` | Modelo dimensional limpio |
| **Analytics** | `SCH_ANALYTICS` | 4 vistas estándar + 2 vistas seguras | Consumo analítico |

**Conceptos clave aplicados:**

- **Convenciones de nomenclatura**: prefijos `DIM_`, `FACT_`, `VW_`, `SVW_` en mayúsculas con guiones bajos.
- **Transformación ETL**: `TRIM`, `COALESCE`, `LOWER`, cálculos derivados durante la carga a core.
- **Vistas como contrato semántico**: los consumidores acceden a datos limpios sin conocer la complejidad interna.
- **Vistas seguras**: protección de lógica de negocio y datos PII mediante `CREATE SECURE VIEW` y `CURRENT_ROLE()`.
- **Abstracción por capas**: las preguntas de negocio se responden con consultas simples sobre vistas, no sobre tablas base.

### Recursos Adicionales

- [Documentación oficial: CREATE VIEW](https://docs.snowflake.com/en/sql-reference/sql/create-view)
- [Documentación oficial: Secure Views](https://docs.snowflake.com/en/user-guide/views-secure)
- [Best Practices: Data Modeling in Snowflake](https://docs.snowflake.com/en/user-guide/data-modeling)
