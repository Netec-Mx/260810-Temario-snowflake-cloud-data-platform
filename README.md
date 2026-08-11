# Snowflake Cloud Data Platform

Este curso capacita a los participantes en el uso práctico y seguro de Snowflake como plataforma moderna de datos en la nube. Inicia con fundamentos de arquitectura, operación de la interfaz, objetos principales y SQL; posteriormente avanza hacia carga de datos, ingesta continua, administración de warehouses, optimización, seguridad, recuperación, clonación y automatización de procesos. Durante el curso se trabajan capacidades clave como virtual warehouses, separación de almacenamiento y cómputo, bases de datos, esquemas, tablas, vistas, stages, file formats, COPY INTO, Snowpipe, microparticiones, caching, Time Travel, Zero-Copy Cloning, RBAC, UDFs, stored procedures, Streams y Tasks.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Exploración inicial de Snowflake y ejecución de consultas. El participante accede a Snowflake, explora Snowsight, identifica objetos principales, ejecuta sus primeras consultas SQL y revisa el historial básico de ejecución.](Capitulo01/README.md#exploración-inicial-de-snowflake-y-ejecución-de-consultas-el-participante-accede-a-snowflake-explora-snowsight-identifica-objetos-principales-ejecuta-sus-primeras-consultas-sql-y-revisa-el-historial-básico-de-ejecución)
  - Duración estimada: 60 min

### Capítulo 2

- [Creación de warehouses, bases de datos, esquemas y tablas. El participante crea un warehouse de trabajo, una base de datos, un esquema y tablas iniciales. También observa el comportamiento de auto-resume y auto-suspend.](Capitulo02/README.md#creación-de-warehouses-bases-de-datos-esquemas-y-tablas-el-participante-crea-un-warehouse-de-trabajo-una-base-de-datos-un-esquema-y-tablas-iniciales-también-observa-el-comportamiento-de-auto-resume-y-auto-suspend)
  - Descripción: DESCRIPCION VARCHAR(100),
  - Duración estimada: 60 min

### Capítulo 3

- [Consultas SQL analíticas sobre un modelo de ventas. El participante crea tablas de clientes, productos y ventas; inserta datos de ejemplo; ejecuta consultas con filtros, agregaciones y joins; y valida métricas básicas de negocio.](Capitulo03/README.md#consultas-sql-analíticas-sobre-un-modelo-de-ventas-el-participante-crea-tablas-de-clientes-productos-y-ventas-inserta-datos-de-ejemplo-ejecuta-consultas-con-filtros-agregaciones-y-joins-y-valida-métricas-básicas-de-negocio)
  - Duración estimada: 60 min

### Capítulo 4

- [Buenas prácticas de nomenclatura y ambientes. El participante organiza datos en capas, crea una tabla de hechos, dimensiones y vistas analíticas para responder preguntas de negocio sobre ventas, clientes y productos.](Capitulo04/README.md#buenas-prácticas-de-nomenclatura-y-ambientes-el-participante-organiza-datos-en-capas-crea-una-tabla-de-hechos-dimensiones-y-vistas-analíticas-para-responder-preguntas-de-negocio-sobre-ventas-clientes-y-productos)
  - Duración estimada: 60 min

### Capítulo 5

- [Carga batch con stages, file formats y COPY INTO. El participante crea formatos de archivo, configura stages, carga archivos CSV y JSON, ejecuta COPY INTO, valida datos cargados y revisa errores de carga.](Capitulo05/README.md#carga-batch-con-stages-file-formats-y-copy-into-el-participante-crea-formatos-de-archivo-configura-stages-carga-archivos-csv-y-json-ejecuta-copy-into-valida-datos-cargados-y-revisa-errores-de-carga)
  - Duración estimada: 60 min

### Capítulo 6

- [Ingesta continua con Snowpipe. El participante crea un stage, una tabla destino y un pipe; simula o configura una carga continua; valida archivos procesados y consulta el historial de ingesta.](Capitulo06/README.md#ingesta-continua-con-snowpipe-el-participante-crea-un-stage-una-tabla-destino-y-un-pipe-simula-o-configura-una-carga-continua-valida-archivos-procesados-y-consulta-el-historial-de-ingesta)
  - Duración estimada: 60 min

### Capítulo 7

- [Administración de warehouses y control de consumo. El participante crea diferentes warehouses, configura auto-suspend y auto-resume, ejecuta consultas con distintos tamaños y analiza el comportamiento de consumo y rendimiento.](Capitulo07/README.md#administración-de-warehouses-y-control-de-consumo-el-participante-crea-diferentes-warehouses-configura-auto-suspend-y-auto-resume-ejecuta-consultas-con-distintos-tamaños-y-analiza-el-comportamiento-de-consumo-y-rendimiento)
  - Duración estimada: 60 min

### Capítulo 8

- [Optimización de consultas con Query Profile y caching. El participante ejecuta consultas sobre datos de prueba, compara tiempos, revisa Query Profile, analiza bytes escaneados, observa el efecto de caching y propone mejoras.](Capitulo08/README.md#optimización-de-consultas-con-query-profile-y-caching-el-participante-ejecuta-consultas-sobre-datos-de-prueba-compara-tiempos-revisa-query-profile-analiza-bytes-escaneados-observa-el-efecto-de-caching-y-propone-mejoras)
  - Duración estimada: 60 min

### Capítulo 9

- [Seguridad con roles, privilegios y RBAC. El participante crea roles de prueba, asigna privilegios, valida accesos permitidos y denegados, consulta objetos con diferentes roles y documenta una matriz básica de permisos.](Capitulo09/README.md#seguridad-con-roles-privilegios-y-rbac-el-participante-crea-roles-de-prueba-asigna-privilegios-valida-accesos-permitidos-y-denegados-consulta-objetos-con-diferentes-roles-y-documenta-una-matriz-básica-de-permisos)
  - Duración estimada: 50 min

### Capítulo 10

- [Pipeline incremental con Time Travel, Cloning, Streams, Tasks y lógica procedural. El participante recupera datos con Time Travel, crea un clon de ambiente, define una UDF o stored procedure, configura un Stream y programa una Task para procesar datos incrementales.](Capitulo10/README.md#pipeline-incremental-con-time-travel-cloning-streams-tasks-y-lógica-procedural-el-participante-recupera-datos-con-time-travel-crea-un-clon-de-ambiente-define-una-udf-o-stored-procedure-configura-un-stream-y-programa-una-task-para-procesar-datos-incrementales)
  - Duración estimada: 55 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
