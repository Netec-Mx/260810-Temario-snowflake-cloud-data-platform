# Ingesta Continua con Snowpipe — Pipeline Completo con Monitoreo y Validación

## Metadatos del Laboratorio

| Campo | Valor |
|-------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Tecnologías** | Snowpipe (AUTO_INGEST=TRUE), Amazon S3, AWS SQS, IAM Storage Integration, SYSTEM$PIPE_STATUS, COPY_HISTORY, Python 3.11+ / boto3, SnowSQL CLI |

---

## Descripción General

En este laboratorio construirás un pipeline de ingesta continua completo utilizando Snowpipe con AUTO_INGEST habilitado. Configurarás la integración de almacenamiento con AWS IAM, crearás un external stage apuntando a un bucket S3, definirás una tabla destino para datos de ventas incrementales, y activarás un pipe que procesa archivos automáticamente al detectar notificaciones SQS. Simularás carga continua con un script Python que genera y sube archivos CSV cada 30 segundos, monitorearás el estado del pipe en tiempo real y analizarás el comportamiento ante archivos duplicados.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Crear un storage integration IAM y un external stage S3 desacoplando credenciales del código SQL
- [ ] Definir una tabla destino con esquema apropiado para recibir datos de ventas incrementales en CSV
- [ ] Crear y activar un Snowpipe con AUTO_INGEST=TRUE asociado a notificaciones SQS
- [ ] Simular carga continua subiendo múltiples archivos CSV al bucket S3 en intervalos controlados
- [ ] Validar archivos procesados usando SYSTEM$PIPE_STATUS y consultar el historial con COPY_HISTORY
- [ ] Identificar y documentar el comportamiento de Snowpipe ante archivos duplicados

---

## Prerrequisitos

### Conocimiento Previo

- Comprensión de conceptos de staging y el comando `COPY INTO` en Snowflake
- Familiaridad básica con AWS S3 (buckets, objetos, permisos)
- Experiencia con Python y ejecución de scripts desde terminal
- Conocimiento de IAM roles y políticas en AWS

### Acceso Requerido

| Recurso | Detalle |
|---------|---------|
| Cuenta Snowflake | Enterprise Edition Trial, rol ACCOUNTADMIN disponible |
| AWS CLI | v2.15.40 configurado con credenciales válidas (permisos S3 + IAM) |
| Python | 3.11.9+ con `boto3>=1.34.84` y `snowflake-connector-python>=3.10.0` |
| Bucket S3 | `snowflake-lab-ingest-<tu_aws_account_id>` en región `us-east-1` |
| Base de datos | `LAB_DB` creada en Snowflake |

---

## Entorno del Laboratorio

### Software Necesario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Snowflake (Snowsight) | Release 8.x Enterprise Trial | Ejecución de DDL y monitoreo |
| SnowSQL CLI | 1.2.32 | Ejecución de comandos SQL desde terminal |
| AWS CLI | 2.15.40 | Gestión de bucket S3 y configuración IAM |
| Python | 3.11.9 | Script de generación y carga de datos |
| boto3 | 1.34.84 | Interacción con S3 desde Python |
| Visual Studio Code | 1.89.1 | Edición de scripts |

### Estructura de Directorios

```
~/curso_snowflake/lab06/
├── data/                    # Archivos CSV generados
├── scripts/
│   └── genera_y_sube.py    # Script de generación y carga
└── sql/
    └── setup.sql            # DDL de objetos Snowflake
```

### Configuración Inicial del Directorio

```bash
# macOS/Linux
mkdir -p ~/curso_snowflake/lab06/{data,scripts,sql}
cd ~/curso_snowflake/lab06

# Windows (PowerShell)
New-Item -ItemType Directory -Force -Path C:\curso_snowflake\lab06\data
New-Item -ItemType Directory -Force -Path C:\curso_snowflake\lab06\scripts
New-Item -ItemType Directory -Force -Path C:\curso_snowflake\lab06\sql
Set-Location C:\curso_snowflake\lab06
```

---

## Paso a Paso

### Paso 1: Crear el Bucket S3 y Configurar Notificaciones de Eventos

**Objetivo:** Preparar la infraestructura AWS necesaria para que Snowpipe reciba notificaciones automáticas cuando se depositen archivos nuevos.

**Instrucciones:**

1. Obtén tu AWS Account ID y expórtalo como variable de entorno:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
echo "Tu AWS Account ID es: ${AWS_ACCOUNT_ID}"
```

2. Crea el bucket S3:

```bash
export BUCKET_NAME="snowflake-lab-ingest-${AWS_ACCOUNT_ID}"

aws s3 mb s3://${BUCKET_NAME} --region us-east-1
```

3. Crea un subdirectorio (prefijo) para organizar los archivos de ventas:

```bash
aws s3api put-object --bucket ${BUCKET_NAME} --key ventas/
```

4. Verifica la creación del bucket:

```bash
aws s3 ls s3://${BUCKET_NAME}/
```

**Resultado Esperado:**

```
PRE ventas/
```

**Verificación:** El bucket existe y contiene el prefijo `ventas/`.

> **Nota:** La configuración de notificaciones SQS se completará en el Paso 4 cuando Snowflake genere el ARN del canal de notificación del pipe.

---

### Paso 2: Crear la Storage Integration en Snowflake

**Objetivo:** Configurar una integración de almacenamiento IAM que permita a Snowflake acceder al bucket S3 sin credenciales embebidas en el stage.

**Instrucciones:**

1. Conéctate a Snowflake con rol ACCOUNTADMIN en Snowsight o SnowSQL:

```sql
USE ROLE ACCOUNTADMIN;
```

2. Crea el schema de trabajo:

```sql
CREATE DATABASE IF NOT EXISTS LAB_DB;
CREATE SCHEMA IF NOT EXISTS LAB_DB.INGEST_SCHEMA;
USE SCHEMA LAB_DB.INGEST_SCHEMA;
```

3. Crea la storage integration. Primero verifica tu AWS Account ID (obtenido en el Paso 1) y reemplázalo en la sentencia:

```sql
-- Reemplaza 123456789012 con tu AWS Account ID real obtenido en el Paso 1
CREATE OR REPLACE STORAGE INTEGRATION S3_INGEST_INTEGRATION
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = 'S3'
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-ingest-role'
    STORAGE_ALLOWED_LOCATIONS = ('s3://snowflake-lab-ingest-123456789012/ventas/');
```

> **Importante:** Sustituye `123456789012` en ambas ubicaciones por el valor real de `${AWS_ACCOUNT_ID}` que obtuviste en el Paso 1.

4. Obtén los valores de confianza para configurar el IAM role en AWS:

```sql
DESC INTEGRATION S3_INGEST_INTEGRATION;
```

5. De la salida, localiza y copia estos dos valores (los necesitarás en el Paso 3):
   - `STORAGE_AWS_IAM_USER_ARN` → ejemplo: `arn:aws:iam::987654321098:user/vflo-b-self1234`
   - `STORAGE_AWS_EXTERNAL_ID` → ejemplo: `AB12345_SFCRole=2_f4gh5ij6klmn7op=`

6. Guarda estos valores en variables de entorno en tu terminal para usarlos en el siguiente paso:

```bash
# Copia los valores exactos que obtuviste de DESC INTEGRATION
export SF_IAM_USER_ARN="arn:aws:iam::987654321098:user/vflo-b-self1234"
export SF_EXTERNAL_ID="AB12345_SFCRole=2_f4gh5ij6klmn7op="
```

> **Importante:** Los valores anteriores son ejemplos. Debes reemplazarlos con los valores reales que devolvió `DESC INTEGRATION` en tu cuenta.

**Resultado Esperado:**

| property | property_value |
|----------|---------------|
| STORAGE_AWS_IAM_USER_ARN | arn:aws:iam::987654321098:user/vflo-b-self1234 |
| STORAGE_AWS_EXTERNAL_ID | AB12345_SFCRole=2_f4gh5ij6klmn7op= |

**Verificación:** Ambos valores se muestran correctamente en la salida de `DESC INTEGRATION`.

---

### Paso 3: Configurar el IAM Role en AWS

**Objetivo:** Crear el rol IAM con la política de confianza que permite a Snowflake asumir el rol y acceder al bucket.

**Instrucciones:**

1. Crea el archivo de política de confianza (`trust-policy.json`) usando las variables de entorno que exportaste al final del Paso 2:

```bash
cat > ~/curso_snowflake/lab06/trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "${SF_IAM_USER_ARN}"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "${SF_EXTERNAL_ID}"
        }
      }
    }
  ]
}
EOF
```

2. Verifica que el archivo se generó correctamente con tus valores reales:

```bash
cat ~/curso_snowflake/lab06/trust-policy.json
```

3. Crea el IAM role:

```bash
aws iam create-role \
    --role-name snowflake-ingest-role \
    --assume-role-policy-document file://~/curso_snowflake/lab06/trust-policy.json
```

4. Crea la política de acceso al bucket (`s3-access-policy.json`):

```bash
cat > ~/curso_snowflake/lab06/s3-access-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}",
        "arn:aws:s3:::${BUCKET_NAME}/ventas/*"
      ]
    }
  ]
}
EOF
```

5. Adjunta la política al rol:

```bash
aws iam put-role-policy \
    --role-name snowflake-ingest-role \
    --policy-name snowflake-s3-read \
    --policy-document file://~/curso_snowflake/lab06/s3-access-policy.json
```

6. Verifica que el rol existe:

```bash
aws iam get-role --role-name snowflake-ingest-role --query "Role.Arn" --output text
```

**Resultado Esperado:**

```
arn:aws:iam::123456789012:role/snowflake-ingest-role
```

(Donde `123456789012` será tu AWS Account ID real)

**Verificación:** El ARN del rol se muestra correctamente.

---

### Paso 4: Crear el External Stage, la Tabla Destino y el Pipe

**Objetivo:** Definir los objetos Snowflake necesarios para el pipeline: file format, external stage, tabla SALES_RAW y pipe SALES_PIPE con AUTO_INGEST=TRUE.

**Instrucciones:**

1. En Snowsight, abre un nuevo worksheet nombrado `LAB06_SnowpipeSetup` y ejecuta:

```sql
USE ROLE ACCOUNTADMIN;
USE SCHEMA LAB_DB.INGEST_SCHEMA;
USE WAREHOUSE WH_CURSO_LOAD_XS;
```

2. Crea el file format para CSV:

```sql
CREATE OR REPLACE FILE FORMAT FF_CSV_VENTAS
    TYPE = 'CSV'
    FIELD_DELIMITER = ','
    RECORD_DELIMITER = '\n'
    SKIP_HEADER = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    NULL_IF = ('NULL', 'null', '')
    EMPTY_FIELD_AS_NULL = TRUE
    DATE_FORMAT = 'YYYY-MM-DD'
    TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS'
    ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```

3. Crea el external stage usando la storage integration (reemplaza `123456789012` con tu AWS Account ID):

```sql
-- Reemplaza 123456789012 con tu AWS Account ID real
CREATE OR REPLACE STAGE SALES_STAGE
    STORAGE_INTEGRATION = S3_INGEST_INTEGRATION
    URL = 's3://snowflake-lab-ingest-123456789012/ventas/'
    FILE_FORMAT = FF_CSV_VENTAS;
```

4. Verifica la conectividad del stage:

```sql
LIST @SALES_STAGE;
```

5. Crea la tabla destino para datos de ventas incrementales:

```sql
CREATE OR REPLACE TABLE SALES_RAW (
    id_venta        VARCHAR(50)     NOT NULL,
    id_cliente      VARCHAR(50)     NOT NULL,
    id_producto     VARCHAR(50)     NOT NULL,
    fecha_venta     DATE            NOT NULL,
    cantidad        INTEGER         NOT NULL,
    precio_venta    DECIMAL(12,2)   NOT NULL,
    descuento       DECIMAL(5,2)    DEFAULT 0.00,
    canal_venta     VARCHAR(30),
    region          VARCHAR(50),
    archivo_origen  VARCHAR(200)    DEFAULT METADATA$FILENAME,
    fecha_ingesta   TIMESTAMP_LTZ   DEFAULT CURRENT_TIMESTAMP()
);
```

6. Crea el pipe con AUTO_INGEST habilitado:

```sql
CREATE OR REPLACE PIPE SALES_PIPE
    AUTO_INGEST = TRUE
    AS
    COPY INTO SALES_RAW (
        id_venta, id_cliente, id_producto, fecha_venta,
        cantidad, precio_venta, descuento, canal_venta,
        region, archivo_origen, fecha_ingesta
    )
    FROM (
        SELECT
            $1,                          -- id_venta
            $2,                          -- id_cliente
            $3,                          -- id_producto
            $4,                          -- fecha_venta
            $5,                          -- cantidad
            $6,                          -- precio_venta
            $7,                          -- descuento
            $8,                          -- canal_venta
            $9,                          -- region
            METADATA$FILENAME,           -- archivo_origen
            CURRENT_TIMESTAMP()          -- fecha_ingesta
        FROM @SALES_STAGE
    );
```

7. Obtén el ARN de la cola SQS del pipe para configurar las notificaciones del bucket:

```sql
SHOW PIPES LIKE 'SALES_PIPE';
```

Busca la columna `notification_channel` y copia el ARN SQS completo (formato: `arn:aws:sqs:us-east-1:123456789012:sf-snowpipe-AIDAXXXXXXXXXXXXXXX-AbCdEfGhIjKlMnOpQr`).

8. Exporta el ARN del notification channel como variable de entorno en tu terminal:

```bash
# Reemplaza el valor siguiente con el ARN real de la columna notification_channel
export SQS_ARN="arn:aws:sqs:us-east-1:123456789012:sf-snowpipe-AIDAXXXXXXXXXXXXXXX-AbCdEfGhIjKlMnOpQr"
echo "SQS ARN configurado: ${SQS_ARN}"
```

> **Importante:** El valor de `SQS_ARN` anterior es un ejemplo. Copia el valor exacto que aparece en la columna `notification_channel` del resultado de `SHOW PIPES`.

9. Configura la notificación de eventos en el bucket S3 usando el ARN obtenido:

```bash
aws s3api put-bucket-notification-configuration \
    --bucket ${BUCKET_NAME} \
    --notification-configuration '{
        "QueueConfigurations": [
            {
                "QueueArn": "'"${SQS_ARN}"'",
                "Events": ["s3:ObjectCreated:*"],
                "Filter": {
                    "Key": {
                        "FilterRules": [
                            {"Name": "prefix", "Value": "ventas/"},
                            {"Name": "suffix", "Value": ".csv"}
                        ]
                    }
                }
            }
        ]
    }'
```

10. Verifica que la notificación se configuró correctamente:

```bash
aws s3api get-bucket-notification-configuration --bucket ${BUCKET_NAME}
```

**Resultado Esperado:**

```json
{
    "QueueConfigurations": [
        {
            "QueueArn": "arn:aws:sqs:us-east-1:123456789012:sf-snowpipe-...",
            "Events": ["s3:ObjectCreated:*"],
            "Filter": {
                "Key": {
                    "FilterRules": [
                        {"Name": "prefix", "Value": "ventas/"},
                        {"Name": "suffix", "Value": ".csv"}
                    ]
                }
            }
        }
    ]
}
```

11. Verifica el estado inicial del pipe:

```sql
SELECT SYSTEM$PIPE_STATUS('LAB_DB.INGEST_SCHEMA.SALES_PIPE');
```

**Resultado Esperado:**

```json
{"executionState":"RUNNING","pendingFileCount":0}
```

**Verificación:** El pipe está en estado `RUNNING` y no tiene archivos pendientes.

---

### Paso 5: Crear el Script de Generación y Carga Continua

**Objetivo:** Desarrollar un script Python que genere archivos CSV con datos de ventas simulados y los suba al bucket S3 cada 30 segundos.

**Instrucciones:**

1. Crea el script `genera_y_sube.py`:

```bash
cat > ~/curso_snowflake/lab06/scripts/genera_y_sube.py << 'PYTHON_EOF'
#!/usr/bin/env python3
"""
Script de generación y carga continua de datos de ventas a S3.
Genera archivos CSV con datos simulados y los sube al bucket S3
para ser procesados por Snowpipe.
"""

import boto3
import csv
import os
import random
import time
import uuid
from datetime import datetime, timedelta

# Configuración
BUCKET_NAME = os.environ.get("BUCKET_NAME", "snowflake-lab-ingest-123456789012")
S3_PREFIX = "ventas/"
LOCAL_DATA_DIR = os.path.expanduser("~/curso_snowflake/lab06/data/")
INTERVALO_SEGUNDOS = 30
NUM_REGISTROS_POR_ARCHIVO = 50
TOTAL_ARCHIVOS = 10

# Datos de referencia para generación
PRODUCTOS = [f"PROD-{str(i).zfill(4)}" for i in range(1, 51)]
CLIENTES = [f"CLI-{str(i).zfill(5)}" for i in range(1, 201)]
CANALES = ["online", "tienda", "mayorista", "marketplace", "telefono"]
REGIONES = ["Norte", "Sur", "Centro", "Este", "Oeste",
            "Noreste", "Noroeste", "Sureste", "Suroeste"]

def generar_registro():
    """Genera un registro de venta aleatorio."""
    fecha_base = datetime.now() - timedelta(days=random.randint(0, 30))
    return {
        "id_venta": str(uuid.uuid4())[:12].upper(),
        "id_cliente": random.choice(CLIENTES),
        "id_producto": random.choice(PRODUCTOS),
        "fecha_venta": fecha_base.strftime("%Y-%m-%d"),
        "cantidad": random.randint(1, 20),
        "precio_venta": round(random.uniform(10.00, 999.99), 2),
        "descuento": round(random.uniform(0.00, 25.00), 2),
        "canal_venta": random.choice(CANALES),
        "region": random.choice(REGIONES),
    }

def generar_archivo_csv(numero_archivo):
    """Genera un archivo CSV con registros de ventas."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    nombre_archivo = f"ventas_{timestamp}_batch{str(numero_archivo).zfill(3)}.csv"
    ruta_completa = os.path.join(LOCAL_DATA_DIR, nombre_archivo)

    os.makedirs(LOCAL_DATA_DIR, exist_ok=True)

    campos = ["id_venta", "id_cliente", "id_producto", "fecha_venta",
              "cantidad", "precio_venta", "descuento", "canal_venta", "region"]

    with open(ruta_completa, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=campos)
        writer.writeheader()
        for _ in range(NUM_REGISTROS_POR_ARCHIVO):
            writer.writerow(generar_registro())

    print(f"  [LOCAL] Archivo generado: {nombre_archivo} ({NUM_REGISTROS_POR_ARCHIVO} registros)")
    return ruta_completa, nombre_archivo

def subir_a_s3(ruta_local, nombre_archivo):
    """Sube un archivo al bucket S3."""
    s3_client = boto3.client("s3", region_name="us-east-1")
    s3_key = f"{S3_PREFIX}{nombre_archivo}"

    s3_client.upload_file(ruta_local, BUCKET_NAME, s3_key)
    print(f"  [S3] Subido: s3://{BUCKET_NAME}/{s3_key}")
    return s3_key

def main():
    """Función principal: genera y sube archivos en intervalos."""
    print("=" * 70)
    print("SIMULADOR DE CARGA CONTINUA - Snowpipe Lab 06")
    print("=" * 70)
    print(f"Bucket: {BUCKET_NAME}")
    print(f"Prefijo: {S3_PREFIX}")
    print(f"Intervalo: {INTERVALO_SEGUNDOS}s")
    print(f"Registros por archivo: {NUM_REGISTROS_POR_ARCHIVO}")
    print(f"Total archivos a generar: {TOTAL_ARCHIVOS}")
    print("=" * 70)

    for i in range(1, TOTAL_ARCHIVOS + 1):
        print(f"\n--- Batch {i}/{TOTAL_ARCHIVOS} [{datetime.now().strftime('%H:%M:%S')}] ---")

        ruta_local, nombre = generar_archivo_csv(i)
        subir_a_s3(ruta_local, nombre)

        if i < TOTAL_ARCHIVOS:
            print(f"  Esperando {INTERVALO_SEGUNDOS} segundos...")
            time.sleep(INTERVALO_SEGUNDOS)

    print("\n" + "=" * 70)
    print(f"COMPLETADO: {TOTAL_ARCHIVOS} archivos subidos")
    print(f"Total registros generados: {TOTAL_ARCHIVOS * NUM_REGISTROS_POR_ARCHIVO}")
    print("=" * 70)

if __name__ == "__main__":
    main()
PYTHON_EOF
```

2. Asegúrate de que la variable `BUCKET_NAME` esté exportada en tu terminal:

```bash
echo "Bucket configurado: ${BUCKET_NAME}"
```

3. Instala las dependencias necesarias:

```bash
pip install boto3>=1.34.84
```

4. Ejecuta el script:

```bash
cd ~/curso_snowflake/lab06
python scripts/genera_y_sube.py
```

**Resultado Esperado:**

```
======================================================================
SIMULADOR DE CARGA CONTINUA - Snowpipe Lab 06
======================================================================
Bucket: snowflake-lab-ingest-123456789012
Prefijo: ventas/
Intervalo: 30s
Registros por archivo: 50
Total archivos a generar: 10
======================================================================

--- Batch 1/10 [14:30:00] ---
  [LOCAL] Archivo generado: ventas_20240515_143000_batch001.csv (50 registros)
  [S3] Subido: s3://snowflake-lab-ingest-123456789012/ventas/ventas_20240515_143000_batch001.csv
  Esperando 30 segundos...
...
```

**Verificación:** El script genera y sube archivos correctamente sin errores de permisos.

---

### Paso 6: Monitorear el Pipeline en Tiempo Real

**Objetivo:** Verificar que Snowpipe procesa los archivos automáticamente y consultar el estado del pipeline mientras el script sigue ejecutándose.

**Instrucciones:**

1. Mientras el script del Paso 5 sigue ejecutándose, abre una nueva sesión de Snowsight y ejecuta:

```sql
USE ROLE ACCOUNTADMIN;
USE SCHEMA LAB_DB.INGEST_SCHEMA;
```

2. Consulta el estado actual del pipe:

```sql
SELECT SYSTEM$PIPE_STATUS('LAB_DB.INGEST_SCHEMA.SALES_PIPE');
```

3. Verifica los archivos listados en el stage:

```sql
LIST @SALES_STAGE;
```

4. Consulta el conteo de registros en la tabla destino (ejecutar periódicamente):

```sql
SELECT COUNT(*) AS total_registros FROM SALES_RAW;
```

5. Consulta el historial de carga del pipe:

```sql
SELECT
    FILE_NAME,
    STATUS,
    ROW_COUNT,
    ROW_PARSED,
    FIRST_ERROR_MESSAGE,
    PIPE_RECEIVED_TIME,
    LAST_LOAD_TIME
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'LAB_DB.INGEST_SCHEMA.SALES_RAW',
    START_TIME => DATEADD(HOUR, -1, CURRENT_TIMESTAMP())
))
ORDER BY PIPE_RECEIVED_TIME DESC;
```

6. Monitorea el progreso con una consulta resumen:

```sql
SELECT
    COUNT(DISTINCT FILE_NAME) AS archivos_procesados,
    SUM(ROW_COUNT) AS total_filas_cargadas,
    MIN(PIPE_RECEIVED_TIME) AS primera_recepcion,
    MAX(LAST_LOAD_TIME) AS ultima_carga,
    DATEDIFF(SECOND, MIN(PIPE_RECEIVED_TIME), MAX(LAST_LOAD_TIME)) AS duracion_total_seg
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'LAB_DB.INGEST_SCHEMA.SALES_RAW',
    START_TIME => DATEADD(HOUR, -1, CURRENT_TIMESTAMP())
))
WHERE STATUS = 'Loaded';
```

**Resultado Esperado (después de que todos los archivos se procesen):**

| archivos_procesados | total_filas_cargadas | primera_recepcion | ultima_carga | duracion_total_seg |
|---------------------|---------------------|-------------------|--------------|-------------------|
| 10 | 500 | 2024-05-15 14:30:05 | 2024-05-15 14:35:12 | 307 |

7. Verifica una muestra de datos cargados:

```sql
SELECT * FROM SALES_RAW
ORDER BY fecha_ingesta DESC
LIMIT 10;
```

**Verificación:** Los 10 archivos se procesaron exitosamente con 50 registros cada uno (500 total), sin errores.

---

### Paso 7: Probar el Comportamiento ante Archivos Duplicados

**Objetivo:** Documentar cómo Snowpipe maneja archivos que ya fueron procesados previamente.

**Instrucciones:**

1. Primero, registra el conteo actual de registros:

```sql
SELECT COUNT(*) AS registros_antes FROM SALES_RAW;
```

2. En tu terminal, re-sube uno de los archivos ya procesados:

```bash
# Busca el primer archivo generado
PRIMER_ARCHIVO=$(ls ~/curso_snowflake/lab06/data/ | head -1)
echo "Re-subiendo archivo: ${PRIMER_ARCHIVO}"

aws s3 cp ~/curso_snowflake/lab06/data/${PRIMER_ARCHIVO} \
    s3://${BUCKET_NAME}/ventas/${PRIMER_ARCHIVO}
```

3. Espera 60 segundos para dar tiempo a Snowpipe de procesar la notificación:

```bash
echo "Esperando 60 segundos para que Snowpipe procese..."
sleep 60
```

4. Verifica que el conteo NO cambió (Snowpipe ignora archivos duplicados):

```sql
SELECT COUNT(*) AS registros_despues FROM SALES_RAW;
```

5. Confirma en el historial que el archivo duplicado fue ignorado:

```sql
SELECT
    FILE_NAME,
    STATUS,
    ROW_COUNT,
    FIRST_ERROR_MESSAGE
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'LAB_DB.INGEST_SCHEMA.SALES_RAW',
    START_TIME => DATEADD(HOUR, -1, CURRENT_TIMESTAMP())
))
ORDER BY PIPE_RECEIVED_TIME DESC
LIMIT 5;
```

6. Ahora prueba subir el mismo contenido con un nombre diferente:

```bash
PRIMER_ARCHIVO=$(ls ~/curso_snowflake/lab06/data/ | head -1)
aws s3 cp ~/curso_snowflake/lab06/data/${PRIMER_ARCHIVO} \
    s3://${BUCKET_NAME}/ventas/ventas_duplicado_nombre_diferente.csv
```

7. Espera 60 segundos y verifica:

```bash
sleep 60
```

```sql
SELECT COUNT(*) AS registros_con_nombre_diferente FROM SALES_RAW;
```

**Resultado Esperado:**

- El re-upload del mismo archivo (mismo nombre y contenido): **NO genera filas adicionales** — Snowpipe detecta que ya fue procesado usando metadatos internos (file path + size + last modified).
- El upload con nombre diferente pero mismo contenido: **SÍ genera 50 filas adicionales** — Snowpipe trata cada nombre de archivo como único.

8. Documenta tus hallazgos:

```sql
-- Resumen final de archivos procesados
SELECT
    FILE_NAME,
    STATUS,
    ROW_COUNT,
    PIPE_RECEIVED_TIME
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'LAB_DB.INGEST_SCHEMA.SALES_RAW',
    START_TIME => DATEADD(HOUR, -1, CURRENT_TIMESTAMP())
))
ORDER BY PIPE_RECEIVED_TIME ASC;
```

**Verificación:** Se confirma que Snowpipe usa el nombre del archivo como identificador de deduplicación, no el contenido.

---

### Paso 8: Consultas de Validación Final

**Objetivo:** Ejecutar consultas analíticas sobre los datos ingestados para confirmar la integridad del pipeline.

**Instrucciones:**

1. Resumen de carga por archivo origen:

```sql
SELECT
    archivo_origen,
    COUNT(*) AS registros,
    MIN(fecha_venta) AS fecha_min,
    MAX(fecha_venta) AS fecha_max,
    SUM(cantidad * precio_venta) AS venta_bruta_total
FROM SALES_RAW
GROUP BY archivo_origen
ORDER BY archivo_origen;
```

2. Distribución por canal de venta:

```sql
SELECT
    canal_venta,
    COUNT(*) AS num_ventas,
    ROUND(AVG(precio_venta), 2) AS precio_promedio,
    SUM(cantidad) AS unidades_totales
FROM SALES_RAW
GROUP BY canal_venta
ORDER BY num_ventas DESC;
```

3. Latencia de ingesta (diferencia entre carga y timestamp de ingesta):

```sql
SELECT
    archivo_origen,
    MIN(fecha_ingesta) AS primera_ingesta,
    MAX(fecha_ingesta) AS ultima_ingesta
FROM SALES_RAW
GROUP BY archivo_origen
ORDER BY primera_ingesta;
```

4. Estado final del pipe:

```sql
SELECT SYSTEM$PIPE_STATUS('LAB_DB.INGEST_SCHEMA.SALES_PIPE');
```

**Resultado Esperado:**

```json
{"executionState":"RUNNING","pendingFileCount":0}
```

**Verificación:** Todos los archivos fueron procesados, el pipe sigue activo y sin archivos pendientes.

---

### Paso 9: Limpieza de Recursos

**Objetivo:** Eliminar los recursos creados para evitar costos innecesarios.

**Instrucciones:**

1. Pausa el pipe (detiene el consumo de créditos de Snowpipe):

```sql
ALTER PIPE LAB_DB.INGEST_SCHEMA.SALES_PIPE SET PIPE_EXECUTION_PAUSED = TRUE;
```

2. Verifica que el pipe está pausado:

```sql
SELECT SYSTEM$PIPE_STATUS('LAB_DB.INGEST_SCHEMA.SALES_PIPE');
```

**Resultado Esperado:**

```json
{"executionState":"PAUSED","pendingFileCount":0}
```

3. Elimina los objetos Snowflake:

```sql
USE ROLE ACCOUNTADMIN;

DROP PIPE IF EXISTS LAB_DB.INGEST_SCHEMA.SALES_PIPE;
DROP TABLE IF EXISTS LAB_DB.INGEST_SCHEMA.SALES_RAW;
DROP STAGE IF EXISTS LAB_DB.INGEST_SCHEMA.SALES_STAGE;
DROP FILE FORMAT IF EXISTS LAB_DB.INGEST_SCHEMA.FF_CSV_VENTAS;
DROP SCHEMA IF EXISTS LAB_DB.INGEST_SCHEMA;
DROP STORAGE INTEGRATION IF EXISTS S3_INGEST_INTEGRATION;
```

4. Elimina los recursos AWS:

```bash
# Vaciar y eliminar el bucket S3
aws s3 rm s3://${BUCKET_NAME} --recursive
aws s3 rb s3://${BUCKET_NAME}

# Eliminar la política y el rol IAM
aws iam delete-role-policy \
    --role-name snowflake-ingest-role \
    --policy-name snowflake-s3-read

aws iam delete-role --role-name snowflake-ingest-role
```

5. Limpia archivos locales:

```bash
rm -rf ~/curso_snowflake/lab06/data/*
rm -f ~/curso_snowflake/lab06/trust-policy.json
rm -f ~/curso_snowflake/lab06/s3-access-policy.json
```

**Verificación:** Todos los recursos han sido eliminados exitosamente.

---

## Resumen del Laboratorio

| Componente | Estado Esperado |
|------------|----------------|
| Storage Integration | Creada y funcional con IAM role |
| External Stage | Conectado al bucket S3 |
| Tabla SALES_RAW | 550 registros (500 originales + 50 del archivo con nombre diferente) |
| Pipe SALES_PIPE | AUTO_INGEST=TRUE, procesó 11 archivos |
| Deduplicación | Confirmada: mismo archivo ignorado, nombre diferente procesado |
| Latencia típica | 30-90 segundos desde upload hasta disponibilidad en tabla |

---

## Troubleshooting Común

| Problema | Causa Probable | Solución |
|----------|---------------|----------|
| `SYSTEM$PIPE_STATUS` muestra `STOPPED_FEATURE_DISABLED` | Snowpipe no está habilitado en la edición de cuenta | Verificar que la cuenta es Enterprise Edition |
| Archivos en stage pero no en tabla | Notificación SQS no configurada correctamente | Verificar `notification_channel` y configuración del bucket |
| Error `Access Denied` en LIST @STAGE | IAM role no tiene los permisos correctos o trust policy incorrecta | Revisar `STORAGE_AWS_IAM_USER_ARN` y `STORAGE_AWS_EXTERNAL_ID` en trust policy |
| `pendingFileCount` no disminuye | El warehouse asociado puede estar suspendido | Snowpipe usa compute serverless; verificar que el pipe esté en estado RUNNING |
| Error `Insufficient privileges` | El rol no tiene permisos sobre la integration | Ejecutar con `ACCOUNTADMIN` o conceder `USAGE` en la integration |
| Script Python falla con `NoCredentialsError` | AWS CLI no configurado | Ejecutar `aws configure` y verificar credenciales |

---

## Conceptos Clave Reforzados

1. **Storage Integration**: Desacopla credenciales del código SQL, delegando acceso a un IAM role con trust policy.
2. **AUTO_INGEST=TRUE**: Snowpipe escucha eventos SQS del bucket S3 y procesa archivos automáticamente.
3. **Deduplicación nativa**: Snowpipe rastrea archivos procesados por path+nombre; el mismo archivo no se recarga.
4. **METADATA$FILENAME**: Función especial que captura el nombre del archivo origen durante la carga.
5. **SYSTEM$PIPE_STATUS**: Función del sistema para monitoreo en tiempo real del estado del pipe.
6. **COPY_HISTORY**: Vista de Information Schema que proporciona historial detallado de operaciones de carga.
