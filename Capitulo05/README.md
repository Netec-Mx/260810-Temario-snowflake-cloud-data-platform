# Carga Batch con Stages, File Formats y COPY INTO

## Metadatos del Laboratorio

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar (Apply) |
| **Worksheet** | LAB05_Carga_Batch |

---

## Descripción General

En este laboratorio implementarás el flujo completo de ingesta batch en Snowflake: desde la generación de archivos de datos sintéticos con Python/Faker, pasando por la creación de file formats y stages internos, hasta la ejecución de `COPY INTO` con validación previa, manejo de errores y auditoría posterior. Este es el laboratorio más complejo del curso y representa el cierre del flujo end-to-end de carga de datos.

---

## Objetivos de Aprendizaje

- [ ] Crear file formats para archivos CSV y JSON con configuraciones específicas de delimitadores, encabezados y manejo de nulos
- [ ] Configurar stages internos y cargar archivos usando el comando `PUT` desde SnowSQL CLI
- [ ] Ejecutar `COPY INTO` para cargar datos desde stages hacia tablas de staging
- [ ] Usar `VALIDATION_MODE` para validar archivos antes de cargar y detectar errores sin modificar datos
- [ ] Revisar `COPY_HISTORY` y manejar errores de carga con `ON_ERROR` y la función `VALIDATE()`

---

## Prerrequisitos

### Conocimientos Previos
- Laboratorio 02-02-01 completado (warehouse `WH_CURSO_LOAD_XS` y tablas `STG_CLIENTES`, `STG_PRODUCTOS`, `STG_VENTAS` existentes)
- Familiaridad con comandos SQL básicos en Snowsight
- Conocimiento básico de Python para ejecutar scripts

### Acceso y Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| SnowSQL CLI | 1.2.32 | Ejecutar comando `PUT` para subir archivos |
| Python | 3.12.3 | Generar archivos de datos sintéticos |
| faker (pip) | 24.11.0 | Librería para datos ficticios |
| Snowflake Trial | Enterprise Edition 8.x | Plataforma de datos |
| Snowsight | 2024.4 | Interfaz web para SQL |

---

## Entorno del Laboratorio

### Estructura de Directorios

```
# Windows
C:\curso_snowflake\lab05\
├── genera_datos.py
└── data\
    ├── clientes_nuevos.csv
    ├── productos_nuevos.csv
    ├── ventas_nuevas.csv
    ├── ventas_con_errores.csv
    └── eventos_web.json

# macOS/Linux
~/curso_snowflake/lab05/
├── genera_datos.py
└── data/
    ├── clientes_nuevos.csv
    ├── productos_nuevos.csv
    ├── ventas_nuevas.csv
    ├── ventas_con_errores.csv
    └── eventos_web.json
```

### Objetos Snowflake Utilizados

| Objeto | Nombre | Estado |
|--------|--------|--------|
| Base de datos | `DB_CURSO_SNOWFLAKE` | Existente (lab 02) |
| Esquema | `SCH_STAGING` | Existente (lab 02) |
| Warehouse | `WH_CURSO_LOAD_XS` | Existente (lab 02) |
| Tabla | `STG_CLIENTES` | Existente (lab 02) |
| Tabla | `STG_PRODUCTOS` | Existente (lab 02) |
| Tabla | `STG_VENTAS` | Existente (lab 02) |

---

## Paso 1: Generar Archivos de Datos con Python

### Objetivo
Ejecutar el script `genera_datos.py` para crear los archivos CSV y JSON que se utilizarán como fuente de datos para la carga batch.

### Instrucciones

1. Abre una terminal (Command Prompt en Windows, Terminal en macOS/Linux) y navega al directorio del laboratorio:

```bash
# Windows
cd C:\curso_snowflake\lab05

# macOS/Linux
cd ~/curso_snowflake/lab05
```

2. Verifica que Python y faker estén instalados:

```bash
python --version
pip show faker
```

3. Crea el archivo `genera_datos.py` con el siguiente contenido:

```python
"""
genera_datos.py - Generador de datos sintéticos para Lab 05
Curso Snowflake - Carga Batch
"""
import csv
import json
import os
import random
from datetime import datetime, timedelta
from faker import Faker

fake = Faker('es_ES')
Faker.seed(42)
random.seed(42)

# Crear directorio de salida
os.makedirs('data', exist_ok=True)

# ============================================================
# 1. clientes_nuevos.csv - 100 registros
# ============================================================
segmentos = ['Corporativo', 'Consumidor', 'Gobierno', 'PYME']
paises = ['México', 'Colombia', 'Argentina', 'Chile', 'Perú', 'España']

with open('data/clientes_nuevos.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow(['id_cliente', 'nombre', 'apellido', 'email', 'telefono',
                     'ciudad', 'pais', 'segmento', 'fecha_registro', 'activo'])
    for i in range(1, 101):
        writer.writerow([
            5000 + i,
            fake.first_name(),
            fake.last_name(),
            fake.email(),
            fake.phone_number()[:15],
            fake.city(),
            random.choice(paises),
            random.choice(segmentos),
            fake.date_between(start_date='-2y', end_date='today').isoformat(),
            random.choice([1, 1, 1, 0])  # 75% activos
        ])

print("✓ clientes_nuevos.csv generado (100 registros)")

# ============================================================
# 2. productos_nuevos.csv - 50 registros
# ============================================================
categorias = {
    'Electrónica': ['Smartphones', 'Laptops', 'Tablets', 'Accesorios'],
    'Hogar': ['Muebles', 'Decoración', 'Cocina', 'Iluminación'],
    'Oficina': ['Papelería', 'Impresoras', 'Sillas', 'Escritorios']
}
unidades = ['unidad', 'paquete', 'caja', 'set']

with open('data/productos_nuevos.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow(['id_producto', 'nombre_producto', 'categoria', 'subcategoria',
                     'precio_unitario', 'costo_unitario', 'unidad_medida', 'activo'])
    for i in range(1, 51):
        cat = random.choice(list(categorias.keys()))
        subcat = random.choice(categorias[cat])
        precio = round(random.uniform(15.0, 2500.0), 2)
        costo = round(precio * random.uniform(0.3, 0.7), 2)
        writer.writerow([
            3000 + i,
            f"{subcat} {fake.word().capitalize()} {fake.random_letter().upper()}{random.randint(10,99)}",
            cat,
            subcat,
            precio,
            costo,
            random.choice(unidades),
            random.choice([1, 1, 1, 0])
        ])

print("✓ productos_nuevos.csv generado (50 registros)")

# ============================================================
# 3. ventas_nuevas.csv - 500 registros
# ============================================================
canales = ['Online', 'Tienda', 'Distribuidor', 'Marketplace']
regiones = ['Norte', 'Sur', 'Centro', 'Este', 'Oeste']

with open('data/ventas_nuevas.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow(['id_venta', 'id_cliente', 'id_producto', 'fecha_venta',
                     'cantidad', 'precio_venta', 'descuento', 'canal_venta', 'region'])
    for i in range(1, 501):
        precio = round(random.uniform(15.0, 2500.0), 2)
        writer.writerow([
            10000 + i,
            random.randint(5001, 5100),
            random.randint(3001, 3050),
            fake.date_between(start_date='-6m', end_date='today').isoformat(),
            random.randint(1, 20),
            precio,
            round(random.uniform(0, 0.25), 2),
            random.choice(canales),
            random.choice(regiones)
        ])

print("✓ ventas_nuevas.csv generado (500 registros)")

# ============================================================
# 4. ventas_con_errores.csv - 500 registros + 5 con errores
# ============================================================
with open('data/ventas_con_errores.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow(['id_venta', 'id_cliente', 'id_producto', 'fecha_venta',
                     'cantidad', 'precio_venta', 'descuento', 'canal_venta', 'region'])
    for i in range(1, 501):
        precio = round(random.uniform(15.0, 2500.0), 2)
        writer.writerow([
            20000 + i,
            random.randint(5001, 5100),
            random.randint(3001, 3050),
            fake.date_between(start_date='-6m', end_date='today').isoformat(),
            random.randint(1, 20),
            precio,
            round(random.uniform(0, 0.25), 2),
            random.choice(canales),
            random.choice(regiones)
        ])
    # 5 registros con errores intencionales
    writer.writerow([20501, 'NO_ES_NUMERO', 3001, '2024-01-15', 5, 100.00, 0.1, 'Online', 'Norte'])
    writer.writerow([20502, 5001, 3001, 'FECHA_INVALIDA', 3, 200.00, 0.05, 'Tienda', 'Sur'])
    writer.writerow([20503, 5002, 3002, '2024-02-30', 2, 150.00, 0.0, 'Online', 'Centro'])
    writer.writerow([20504, 5003, 3003, '2024-03-10', -5, 'NO_PRECIO', 0.15, 'Marketplace', 'Este'])
    writer.writerow([20505, 5004, 3004, '2024-04-01', 1, 50.00, 1.5, 'Distribuidor'])  # Falta 1 columna

print("✓ ventas_con_errores.csv generado (505 registros, 5 con errores)")

# ============================================================
# 5. eventos_web.json - 200 registros
# ============================================================
tipos_evento = ['page_view', 'add_to_cart', 'purchase', 'search', 'click', 'logout']
dispositivos = ['desktop', 'mobile', 'tablet']
navegadores = ['Chrome', 'Firefox', 'Safari', 'Edge']

eventos = []
for i in range(1, 201):
    evento = {
        "event_id": 80000 + i,
        "user_id": random.randint(5001, 5100),
        "event_type": random.choice(tipos_evento),
        "timestamp": (datetime.now() - timedelta(
            hours=random.randint(1, 720))).isoformat(),
        "page_url": f"https://tienda.ejemplo.com/{fake.uri_path()}",
        "device": random.choice(dispositivos),
        "browser": random.choice(navegadores),
        "session_id": fake.uuid4(),
        "duration_seconds": random.randint(1, 300),
        "metadata": {
            "ip_address": fake.ipv4(),
            "country": random.choice(paises),
            "referrer": random.choice(['google.com', 'direct', 'facebook.com', 'email'])
        }
    }
    eventos.append(evento)

with open('data/eventos_web.json', 'w', encoding='utf-8') as f:
    json.dump(eventos, f, ensure_ascii=False, indent=2)

print("✓ eventos_web.json generado (200 registros)")
print("\n✅ Todos los archivos generados exitosamente en ./data/")
```

4. Ejecuta el script:

```bash
python genera_datos.py
```

### Resultado Esperado

```
✓ clientes_nuevos.csv generado (100 registros)
✓ productos_nuevos.csv generado (50 registros)
✓ ventas_nuevas.csv generado (500 registros)
✓ ventas_con_errores.csv generado (505 registros, 5 con errores)
✓ eventos_web.json generado (200 registros)

✅ Todos los archivos generados exitosamente en ./data/
```

### Verificación

```bash
# Verificar que los archivos existen y tienen contenido
# Windows
dir data\

# macOS/Linux
ls -la data/
```

Deberás ver 5 archivos con tamaños mayores a 0 bytes.

---

## Paso 2: Crear File Formats en Snowflake

### Objetivo
Definir los formatos de archivo `FF_CSV_STANDARD` y `FF_JSON_STANDARD` en el esquema `SCH_STAGING` para estandarizar la interpretación de archivos CSV y JSON durante la carga.

### Instrucciones

1. Abre Snowsight y crea una nueva worksheet llamada `LAB05_Carga_Batch`.

2. Configura el contexto de sesión:

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE WH_CURSO_LOAD_XS;
USE DATABASE DB_CURSO_SNOWFLAKE;
USE SCHEMA SCH_STAGING;
```

3. Crea el file format para archivos CSV:

```sql
CREATE OR REPLACE FILE FORMAT FF_CSV_STANDARD
    TYPE = 'CSV'
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1
    NULL_IF = ('NULL', 'null', '')
    EMPTY_FIELD_AS_NULL = TRUE
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    COMPRESSION = AUTO
    ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE
    COMMENT = 'Formato CSV estándar para carga batch - Lab 05';
```

4. Crea el file format para archivos JSON:

```sql
CREATE OR REPLACE FILE FORMAT FF_JSON_STANDARD
    TYPE = 'JSON'
    STRIP_OUTER_ARRAY = TRUE
    COMPRESSION = AUTO
    COMMENT = 'Formato JSON estándar para carga batch - Lab 05';
```

5. Verifica la creación de los file formats:

```sql
SHOW FILE FORMATS IN SCHEMA SCH_STAGING;
```

### Resultado Esperado

La consulta `SHOW FILE FORMATS` debe mostrar dos registros:

| name | type |
|------|------|
| FF_CSV_STANDARD | CSV |
| FF_JSON_STANDARD | JSON |

### Verificación

```sql
-- Verificar propiedades específicas del formato CSV
DESCRIBE FILE FORMAT FF_CSV_STANDARD;
```

Confirma que `SKIP_HEADER = 1`, `FIELD_DELIMITER = ','` y `ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE`.

---

## Paso 3: Crear Stages Internos

### Objetivo
Configurar stages internos `STG_INT_CLIENTES_PROD` y `STG_INT_VENTAS_PROD` asociados a sus respectivos file formats para organizar los archivos por dominio de datos.

### Instrucciones

1. Crea el stage interno para clientes y productos:

```sql
CREATE OR REPLACE STAGE STG_INT_CLIENTES_PROD
    FILE_FORMAT = FF_CSV_STANDARD
    COMMENT = 'Stage interno para archivos de clientes y productos';
```

2. Crea el stage interno para ventas y eventos:

```sql
CREATE OR REPLACE STAGE STG_INT_VENTAS_PROD
    FILE_FORMAT = FF_CSV_STANDARD
    COMMENT = 'Stage interno para archivos de ventas y eventos web';
```

3. Verifica los stages creados:

```sql
SHOW STAGES IN SCHEMA SCH_STAGING;
```

### Resultado Esperado

| name | type | comment |
|------|------|---------|
| STG_INT_CLIENTES_PROD | INTERNAL | Stage interno para archivos de clientes y productos |
| STG_INT_VENTAS_PROD | INTERNAL | Stage interno para archivos de ventas y eventos web |

### Verificación

```sql
-- Confirmar que los stages están vacíos inicialmente
LIST @STG_INT_CLIENTES_PROD;
LIST @STG_INT_VENTAS_PROD;
```

Ambos deben retornar 0 filas (stages vacíos).

---

## Paso 4: Cargar Archivos al Stage con PUT (SnowSQL)

### Objetivo
Subir los archivos generados a los stages internos utilizando el comando `PUT` desde SnowSQL CLI.

### Instrucciones

1. Abre una terminal y conéctate a Snowflake con SnowSQL:

```bash
snowsql -a <account_identifier> -u ADMIN
```

> **Nota:** Reemplaza `<account_identifier>` con tu identificador de cuenta (formato: `xxxxxxx-xxxxxxx`).

2. Configura el contexto:

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE WH_CURSO_LOAD_XS;
USE DATABASE DB_CURSO_SNOWFLAKE;
USE SCHEMA SCH_STAGING;
```

3. Sube los archivos de clientes y productos al stage `STG_INT_CLIENTES_PROD`:

```sql
-- Windows
PUT file://C:\curso_snowflake\lab05\data\clientes_nuevos.csv @STG_INT_CLIENTES_PROD AUTO_COMPRESS=TRUE;
PUT file://C:\curso_snowflake\lab05\data\productos_nuevos.csv @STG_INT_CLIENTES_PROD AUTO_COMPRESS=TRUE;

-- macOS/Linux
PUT file://~/curso_snowflake/lab05/data/clientes_nuevos.csv @STG_INT_CLIENTES_PROD AUTO_COMPRESS=TRUE;
PUT file://~/curso_snowflake/lab05/data/productos_nuevos.csv @STG_INT_CLIENTES_PROD AUTO_COMPRESS=TRUE;
```

4. Sube los archivos de ventas y eventos al stage `STG_INT_VENTAS_PROD`:

```sql
-- Windows
PUT file://C:\curso_snowflake\lab05\data\ventas_nuevas.csv @STG_INT_VENTAS_PROD AUTO_COMPRESS=TRUE;
PUT file://C:\curso_snowflake\lab05\data\ventas_con_errores.csv @STG_INT_VENTAS_PROD AUTO_COMPRESS=TRUE;
PUT file://C:\curso_snowflake\lab05\data\eventos_web.json @STG_INT_VENTAS_PROD AUTO_COMPRESS=TRUE;

-- macOS/Linux
PUT file://~/curso_snowflake/lab05/data/ventas_nuevas.csv @STG_INT_VENTAS_PROD AUTO_COMPRESS=TRUE;
PUT file://~/curso_snowflake/lab05/data/ventas_con_errores.csv @STG_INT_VENTAS_PROD AUTO_COMPRESS=TRUE;
PUT file://~/curso_snowflake/lab05/data/eventos_web.json @STG_INT_VENTAS_PROD AUTO_COMPRESS=TRUE;
```

### Resultado Esperado

Cada comando `PUT` debe mostrar un resultado similar a:

```
+-----------------------+------------------------+-------------+-------------+--------------------+--------------------+----------+---------+
| source                | target                 | source_size | target_size | source_compression | target_compression | status   | message |
|-----------------------+------------------------+-------------+-------------+--------------------+--------------------+----------+---------|
| clientes_nuevos.csv   | clientes_nuevos.csv.gz |        8432 |        2891 | NONE               | GZIP               | UPLOADED |         |
+-----------------------+------------------------+-------------+-------------+--------------------+--------------------+----------+---------+
```

El `status` debe ser `UPLOADED` para cada archivo.

### Verificación

Ejecuta desde SnowSQL o Snowsight:

```sql
-- Verificar archivos en el stage de clientes/productos
LIST @STG_INT_CLIENTES_PROD;

-- Verificar archivos en el stage de ventas/eventos
LIST @STG_INT_VENTAS_PROD;
```

Debes ver:
- `STG_INT_CLIENTES_PROD`: 2 archivos (`.csv.gz`)
- `STG_INT_VENTAS_PROD`: 3 archivos (2 `.csv.gz` + 1 `.json.gz`)

---

## Paso 5: Validar Archivos con VALIDATION_MODE

### Objetivo
Usar `VALIDATION_MODE` para inspeccionar los archivos antes de ejecutar la carga real, detectando posibles errores sin modificar las tablas destino.

### Instrucciones

1. Regresa a Snowsight (worksheet `LAB05_Carga_Batch`). Valida las primeras 10 filas del archivo de clientes:

```sql
COPY INTO STG_CLIENTES
FROM @STG_INT_CLIENTES_PROD/clientes_nuevos.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
VALIDATION_MODE = RETURN_10_ROWS;
```

2. Valida el archivo de ventas sin errores:

```sql
COPY INTO STG_VENTAS
FROM @STG_INT_VENTAS_PROD/ventas_nuevas.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
VALIDATION_MODE = RETURN_ERRORS;
```

3. Valida el archivo de ventas CON errores para detectar problemas:

```sql
COPY INTO STG_VENTAS
FROM @STG_INT_VENTAS_PROD/ventas_con_errores.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
VALIDATION_MODE = RETURN_ERRORS;
```

4. Para obtener TODOS los errores del archivo problemático:

```sql
COPY INTO STG_VENTAS
FROM @STG_INT_VENTAS_PROD/ventas_con_errores.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
VALIDATION_MODE = RETURN_ALL_ERRORS;
```

### Resultado Esperado

- **Paso 1:** Retorna 10 filas con los datos de clientes correctamente parseados (columnas $1 a $10).
- **Paso 2:** Retorna 0 filas (sin errores en `ventas_nuevas.csv`).
- **Paso 3:** Retorna filas con información de errores detectados (columnas `ERROR`, `FILE`, `LINE`, `CHARACTER`, `REJECTED_RECORD`).
- **Paso 4:** Retorna los 5 errores intencionales con detalles de cada fila rechazada.

### Verificación

Confirma que el paso 3 muestra errores relacionados con:
- Conversión de tipo (texto donde se espera número)
- Fecha inválida (`2024-02-30`)
- Número de columnas incorrecto (fila con campo faltante)

> **Importante:** Ninguna de estas operaciones modifica datos en las tablas destino.

---

## Paso 6: Ejecutar COPY INTO para Carga de Datos Válidos

### Objetivo
Cargar los archivos CSV válidos (clientes, productos, ventas) en las tablas de staging correspondientes.

### Instrucciones

1. Carga los datos de clientes:

```sql
COPY INTO STG_CLIENTES
FROM @STG_INT_CLIENTES_PROD/clientes_nuevos.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
ON_ERROR = ABORT_STATEMENT;
```

2. Carga los datos de productos:

```sql
COPY INTO STG_PRODUCTOS
FROM @STG_INT_CLIENTES_PROD/productos_nuevos.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
ON_ERROR = ABORT_STATEMENT;
```

3. Carga los datos de ventas (archivo sin errores):

```sql
COPY INTO STG_VENTAS
FROM @STG_INT_VENTAS_PROD/ventas_nuevas.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
ON_ERROR = ABORT_STATEMENT;
```

### Resultado Esperado

Cada comando debe retornar un resultado similar a:

| file | status | rows_parsed | rows_loaded | error_limit | errors_seen |
|------|--------|-------------|-------------|-------------|-------------|
| clientes_nuevos.csv.gz | LOADED | 100 | 100 | 1 | 0 |

```
-- Clientes: 100 rows loaded
-- Productos: 50 rows loaded
-- Ventas: 500 rows loaded
```

### Verificación

```sql
-- Confirmar conteos en cada tabla
SELECT 'STG_CLIENTES' AS tabla, COUNT(*) AS registros FROM STG_CLIENTES
UNION ALL
SELECT 'STG_PRODUCTOS', COUNT(*) FROM STG_PRODUCTOS
UNION ALL
SELECT 'STG_VENTAS', COUNT(*) FROM STG_VENTAS;
```

> **Nota:** Los conteos incluyen datos previos del lab 02 más los nuevos registros cargados.

---

## Paso 7: Manejo de Errores con ON_ERROR

### Objetivo
Comparar el comportamiento de `ON_ERROR` con sus distintos valores (`ABORT_STATEMENT`, `CONTINUE`, `SKIP_FILE`) usando el archivo `ventas_con_errores.csv`.

### Instrucciones

1. Crea una tabla temporal para pruebas de errores (evitar contaminar `STG_VENTAS`):

```sql
CREATE OR REPLACE TEMPORARY TABLE STG_VENTAS_TEST LIKE STG_VENTAS;
```

2. **Prueba con ABORT_STATEMENT** (comportamiento por defecto):

```sql
COPY INTO STG_VENTAS_TEST
FROM @STG_INT_VENTAS_PROD/ventas_con_errores.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
ON_ERROR = ABORT_STATEMENT;
```

> **Resultado esperado:** La carga falla completamente. Se muestra un mensaje de error y 0 filas cargadas.

3. Verifica que no se cargó nada:

```sql
SELECT COUNT(*) AS filas_cargadas FROM STG_VENTAS_TEST;
-- Resultado: 0
```

4. **Prueba con CONTINUE** (carga filas válidas, ignora errores):

```sql
TRUNCATE TABLE STG_VENTAS_TEST;

COPY INTO STG_VENTAS_TEST
FROM @STG_INT_VENTAS_PROD/ventas_con_errores.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
ON_ERROR = CONTINUE
FORCE = TRUE;
```

> **Resultado esperado:** Se cargan ~500 filas válidas y se reportan 5 errores.

5. Verifica las filas cargadas:

```sql
SELECT COUNT(*) AS filas_cargadas FROM STG_VENTAS_TEST;
-- Resultado esperado: 500 (las filas válidas)
```

6. **Prueba con SKIP_FILE**:

```sql
TRUNCATE TABLE STG_VENTAS_TEST;

COPY INTO STG_VENTAS_TEST
FROM @STG_INT_VENTAS_PROD/ventas_con_errores.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
ON_ERROR = SKIP_FILE
FORCE = TRUE;
```

> **Resultado esperado:** El archivo completo se omite porque contiene errores. 0 filas cargadas.

7. Verifica:

```sql
SELECT COUNT(*) AS filas_cargadas FROM STG_VENTAS_TEST;
-- Resultado: 0
```

### Resultado Esperado — Resumen Comparativo

| ON_ERROR | Filas Cargadas | Comportamiento |
|----------|---------------|----------------|
| ABORT_STATEMENT | 0 | Aborta al primer error, rollback completo |
| CONTINUE | ~500 | Carga válidas, ignora erróneas |
| SKIP_FILE | 0 | Omite el archivo entero |

### Verificación

Ejecuta la inspección de errores con `VALIDATE()`:

```sql
-- Guardar el query_id de la última carga con CONTINUE
-- (vuelve a ejecutar con CONTINUE para obtener el query_id)
TRUNCATE TABLE STG_VENTAS_TEST;

COPY INTO STG_VENTAS_TEST
FROM @STG_INT_VENTAS_PROD/ventas_con_errores.csv.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_STANDARD')
ON_ERROR = CONTINUE
FORCE = TRUE;

-- Inspeccionar errores detallados
SELECT
    rejected_record,
    error,
    file,
    line
FROM TABLE(VALIDATE(STG_VENTAS_TEST, JOB_ID => LAST_QUERY_ID()));
```

Debes ver 5 registros rechazados con mensajes de error descriptivos.

---

## Paso 8: Cargar Archivo JSON (Eventos Web)

### Objetivo
Crear una tabla para eventos web y cargar el archivo JSON usando el file format `FF_JSON_STANDARD`.

### Instrucciones

1. Crea la tabla para eventos web con columna VARIANT:

```sql
CREATE OR REPLACE TABLE STG_EVENTOS_WEB (
    evento_raw VARIANT,
    fecha_carga TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

2. Carga el archivo JSON:

```sql
COPY INTO STG_EVENTOS_WEB (evento_raw)
FROM @STG_INT_VENTAS_PROD/eventos_web.json.gz
FILE_FORMAT = (FORMAT_NAME = 'FF_JSON_STANDARD')
ON_ERROR = ABORT_STATEMENT;
```

3. Verifica la carga y consulta datos JSON:

```sql
-- Conteo total
SELECT COUNT(*) AS total_eventos FROM STG_EVENTOS_WEB;

-- Consulta de muestra con acceso a campos JSON
SELECT
    evento_raw:event_id::INTEGER AS event_id,
    evento_raw:user_id::INTEGER AS user_id,
    evento_raw:event_type::STRING AS event_type,
    evento_raw:timestamp::TIMESTAMP AS event_timestamp,
    evento_raw:device::STRING AS device,
    evento_raw:metadata:country::STRING AS country
FROM STG_EVENTOS_WEB
LIMIT 10;
```

### Resultado Esperado

- `COUNT(*)` = 200 registros
- La consulta de muestra muestra datos correctamente extraídos del JSON con tipos apropiados

### Verificación

```sql
-- Verificar distribución por tipo de evento
SELECT
    evento_raw:event_type::STRING AS tipo_evento,
    COUNT(*) AS cantidad
FROM STG_EVENTOS_WEB
GROUP BY tipo_evento
ORDER BY cantidad DESC;
```

---

## Paso 9: Revisar COPY_HISTORY para Auditoría

### Objetivo
Consultar el historial de cargas para auditar todas las operaciones `COPY INTO` ejecutadas durante el laboratorio.

### Instrucciones

1. Consulta el historial de cargas de la tabla `STG_CLIENTES`:

```sql
SELECT
    file_name,
    stage_location,
    last_load_time,
    status,
    row_count,
    row_parsed,
    error_count,
    first_error_message
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'STG_CLIENTES',
    START_TIME => DATEADD('hours', -2, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```

2. Consulta el historial de cargas de `STG_VENTAS`:

```sql
SELECT
    file_name,
    last_load_time,
    status,
    row_count,
    error_count,
    first_error_message
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'STG_VENTAS',
    START_TIME => DATEADD('hours', -2, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```

3. Consulta el historial de la tabla temporal con errores:

```sql
SELECT
    file_name,
    last_load_time,
    status,
    row_count,
    row_parsed,
    error_count,
    first_error_message
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'STG_VENTAS_TEST',
    START_TIME => DATEADD('hours', -2, CURRENT_TIMESTAMP())
))
ORDER BY last_load_time DESC;
```

### Resultado Esperado

Para `STG_CLIENTES`:

| file_name | status | row_count | error_count |
|-----------|--------|-----------|-------------|
| clientes_nuevos.csv.gz | Loaded | 100 | 0 |

Para `STG_VENTAS_TEST` deberás ver múltiples entradas con diferentes estados (`Loaded`, `Load failed`, `Partially loaded`) correspondientes a las pruebas con distintos valores de `ON_ERROR`.

### Verificación

Confirma que puedes identificar en el historial:
- Cargas exitosas (status = `Loaded`)
- Cargas fallidas (status = `Load failed`)
- Cargas parciales con `ON_ERROR = CONTINUE` (status = `Partially loaded`)

---

## Paso 10: Consulta de Auditoría Final

### Objetivo
Ejecutar una consulta integral que compare los registros esperados (archivos fuente) contra los registros efectivamente cargados en las tablas, validando la integridad del proceso.

### Instrucciones

1. Ejecuta la consulta de auditoría consolidada:

```sql
-- Auditoría de integridad: archivos fuente vs tablas cargadas
WITH archivos_fuente AS (
    SELECT 'STG_CLIENTES' AS tabla, 100 AS registros_esperados
    UNION ALL
    SELECT 'STG_PRODUCTOS', 50
    UNION ALL
    SELECT 'STG_VENTAS', 500
    UNION ALL
    SELECT 'STG_EVENTOS_WEB', 200
),
tablas_cargadas AS (
    SELECT 'STG_CLIENTES' AS tabla, COUNT(*) AS registros_cargados FROM STG_CLIENTES
    UNION ALL
    SELECT 'STG_PRODUCTOS', COUNT(*) FROM STG_PRODUCTOS
    UNION ALL
    SELECT 'STG_VENTAS', COUNT(*) FROM STG_VENTAS
    UNION ALL
    SELECT 'STG_EVENTOS_WEB', COUNT(*) FROM STG_EVENTOS_WEB
)
SELECT
    f.tabla,
    f.registros_esperados,
    t.registros_cargados,
    CASE
        WHEN t.registros_cargados >= f.registros_esperados THEN '✓ OK'
        ELSE '✗ DISCREPANCIA'
    END AS estado_validacion
FROM archivos_fuente f
JOIN tablas_cargadas t ON f.tabla = t.tabla
ORDER BY f.tabla;
```

2. Revisa el resumen del historial completo de cargas del laboratorio:

```sql
-- Resumen de todas las cargas realizadas en las últimas 2 horas
SELECT
    table_name,
    COUNT(*) AS total_operaciones,
    SUM(CASE WHEN status = 'Loaded' THEN 1 ELSE 0 END) AS cargas_exitosas,
    SUM(CASE WHEN status = 'Load failed' THEN 1 ELSE 0 END) AS cargas_fallidas,
    SUM(CASE WHEN status = 'Partially loaded' THEN 1 ELSE 0 END) AS cargas_parciales,
    SUM(row_count) AS total_filas_cargadas
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'STG_VENTAS_TEST',
    START_TIME => DATEADD('hours', -2, CURRENT_TIMESTAMP())
))
GROUP BY table_name;
```

### Resultado Esperado

| tabla | registros_esperados | registros_cargados | estado_validacion |
|-------|--------------------|--------------------|-------------------|
| STG_CLIENTES | 100 | ≥100 | ✓ OK |
| STG_EVENTOS_WEB | 200 | 200 | ✓ OK |
| STG_PRODUCTOS | 50 | ≥50 | ✓ OK |
| STG_VENTAS | 500 | ≥500 | ✓ OK |

> **Nota:** Los valores de `registros_cargados` pueden ser mayores que los esperados si las tablas tenían datos previos del lab 02.

### Verificación

Todos los registros deben mostrar `✓ OK` en la columna `estado_validacion`.

---

## Validación y Testing

Ejecuta el siguiente bloque completo para confirmar que todos los objetivos del laboratorio se cumplieron:

```sql
-- ============================================================
-- VALIDACIÓN FINAL DEL LABORATORIO 05-02-01
-- ============================================================

-- 1. Verificar file formats
SELECT 'File Formats' AS check_item,
       COUNT(*) AS resultado,
       CASE WHEN COUNT(*) = 2 THEN 'PASS' ELSE 'FAIL' END AS estado
FROM INFORMATION_SCHEMA.FILE_FORMATS
WHERE FILE_FORMAT_NAME IN ('FF_CSV_STANDARD', 'FF_JSON_STANDARD');

-- 2. Verificar stages
SELECT 'Stages Internos' AS check_item,
       COUNT(*) AS resultado,
       CASE WHEN COUNT(*) >= 2 THEN 'PASS' ELSE 'FAIL' END AS estado
FROM INFORMATION_SCHEMA.STAGES
WHERE STAGE_NAME IN ('STG_INT_CLIENTES_PROD', 'STG_INT_VENTAS_PROD')
  AND STAGE_TYPE = 'INTERNAL';

-- 3. Verificar datos cargados
SELECT 'Clientes Cargados' AS check_item,
       COUNT(*) AS resultado,
       CASE WHEN COUNT(*) >= 100 THEN 'PASS' ELSE 'FAIL' END AS estado
FROM STG_CLIENTES
UNION ALL
SELECT 'Productos Cargados',
       COUNT(*),
       CASE WHEN COUNT(*) >= 50 THEN 'PASS' ELSE 'FAIL' END
FROM STG_PRODUCTOS
UNION ALL
SELECT 'Ventas Cargadas',
       COUNT(*),
       CASE WHEN COUNT(*) >= 500 THEN 'PASS' ELSE 'FAIL' END
FROM STG_VENTAS
UNION ALL
SELECT 'Eventos Web Cargados',
       COUNT(*),
       CASE WHEN COUNT(*) >= 200 THEN 'PASS' ELSE 'FAIL' END
FROM STG_EVENTOS_WEB;
```

Todos los ítems deben mostrar `PASS`.

---

## Solución de Problemas

### Problema 1: PUT falla con "File not found" en SnowSQL

**Síntomas:**
```
PUT file://C:\curso_snowflake\lab05\data\clientes_nuevos.csv @STG_INT_CLIENTES_PROD;
-- Error: File not found: C:\curso_snowflake\lab05\data\clientes_nuevos.csv
```

**Causa:** La ruta del archivo es incorrecta o contiene caracteres especiales. En Windows, SnowSQL puede requerir barras diagonales (`/`) en lugar de barras invertidas (`\`), o la ruta puede tener espacios sin escapar.

**Solución:**
```sql
-- Opción 1: Usar barras diagonales en Windows
PUT file://C:/curso_snowflake/lab05/data/clientes_nuevos.csv @STG_INT_CLIENTES_PROD;

-- Opción 2: Verificar que el archivo existe desde la terminal antes de ejecutar PUT
-- Windows: dir C:\curso_snowflake\lab05\data\clientes_nuevos.csv
-- Linux/macOS: ls ~/curso_snowflake/lab05/data/clientes_nuevos.csv

-- Opción 3: Usar wildcard para subir todos los CSV de una vez
PUT file://C:/curso_snowflake/lab05/data/*.csv @STG_INT_CLIENTES_PROD;
```

---

### Problema 2: COPY INTO falla con "Number of columns in file does not match"

**Síntomas:**
```
Error: Number of columns in file (9) does not match that of the corresponding table (10)
```

**Causa:** El archivo tiene un número diferente de columnas al esperado por la tabla destino. Esto ocurre frecuentemente cuando `ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE` en el file format y el archivo tiene columnas extras o faltantes.

**Solución:**
```sql
-- Opción 1: Verificar la estructura del archivo consultando el stage directamente
SELECT $1, $2, $3, $4, $5, $6, $7, $8, $9, $10
FROM @STG_INT_CLIENTES_PROD/clientes_nuevos.csv.gz
(FILE_FORMAT => 'FF_CSV_STANDARD')
LIMIT 5;

-- Opción 2: Crear un file format temporal más permisivo para diagnóstico
CREATE OR REPLACE TEMPORARY FILE FORMAT FF_CSV_DIAGNOSTICO
    TYPE = 'CSV'
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1
    ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;

-- Opción 3: Si el archivo tiene menos columnas, desactivar la validación
-- y mapear explícitamente las columnas disponibles
COPY INTO STG_VENTAS (id_venta, id_cliente, id_producto, fecha_venta,
                      cantidad, precio_venta, descuento, canal_venta, region)
FROM (
    SELECT $1, $2, $3, $4, $5, $6, $7, $8, $9
    FROM @STG_INT_VENTAS_PROD/ventas_con_errores.csv.gz
)
FILE_FORMAT = (FORMAT_NAME = 'FF_CSV_DIAGNOSTICO')
ON_ERROR = CONTINUE;
```

---

## Limpieza

Ejecuta los siguientes comandos para limpiar los recursos temporales creados durante el laboratorio. **No elimines** los file formats, stages ni datos permanentes, ya que podrían ser utilizados en laboratorios posteriores.

```sql
-- Eliminar tabla temporal de pruebas de errores
DROP TABLE IF EXISTS STG_VENTAS_TEST;

-- Opcional: Limpiar archivos del stage si ya no se necesitan
-- REMOVE @STG_INT_CLIENTES_PROD;
-- REMOVE @STG_INT_VENTAS_PROD;

-- Suspender el warehouse para evitar consumo innecesario
ALTER WAREHOUSE WH_CURSO_LOAD_XS SUSPEND;
```

> **Nota:** Los archivos en los stages, los file formats y la tabla `STG_EVENTOS_WEB` se conservan como parte del entorno persistente del curso.

---

## Resumen

En este laboratorio completaste el flujo end-to-end de carga batch en Snowflake:

| Paso | Acción | Resultado |
|------|--------|-----------|
| 1 | Generación de datos con Python/Faker | 5 archivos (3 CSV + 1 CSV con errores + 1 JSON) |
| 2 | Creación de file formats | `FF_CSV_STANDARD` y `FF_JSON_STANDARD` |
| 3 | Configuración de stages internos | `STG_INT_CLIENTES_PROD` y `STG_INT_VENTAS_PROD` |
| 4 | Carga de archivos con PUT | 5 archivos subidos y comprimidos automáticamente |
| 5 | Validación con VALIDATION_MODE | Errores detectados sin modificar datos |
| 6 | Carga exitosa con COPY INTO | 850 registros CSV cargados correctamente |
| 7 | Comparación de ON_ERROR | ABORT vs CONTINUE vs SKIP_FILE documentados |
| 8 | Carga de JSON | 200 eventos web en formato VARIANT |
| 9 | Auditoría con COPY_HISTORY | Historial completo de operaciones revisado |
| 10 | Validación de integridad | Archivos fuente vs tablas verificados |

### Conceptos Clave Aplicados

- **VALIDATION_MODE** permite inspeccionar archivos antes de comprometer datos
- **ON_ERROR** controla la estrategia ante filas erróneas (abortar, continuar o saltar)
- **COPY_HISTORY** proporciona trazabilidad completa de todas las operaciones de carga
- **VALIDATE()** permite inspección post-carga de filas rechazadas
- **FORCE = TRUE** permite recargar archivos previamente procesados

### Recursos Adicionales

- [Documentación oficial: COPY INTO](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table)
- [Documentación oficial: PUT](https://docs.snowflake.com/en/sql-reference/sql/put)
- [Documentación oficial: VALIDATE function](https://docs.snowflake.com/en/sql-reference/functions/validate)
- [Documentación oficial: COPY_HISTORY](https://docs.snowflake.com/en/sql-reference/functions/copy_history)
