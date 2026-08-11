---LAB_START---
LAB_ID: 03-02-01
---MARKDOWN---
# Consultas SQL Analíticas sobre un Modelo de Ventas

## Metadata

| Campo | Detalle |
|-------|---------|
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Tecnologías clave** | DML INSERT, funciones escalares, agregaciones, window functions, JOINs |

## Descripción General

En este laboratorio construirás el contenido analítico del modelo de ventas insertando datos de ejemplo en las tablas `STG_CLIENTES`, `STG_PRODUCTOS` y `STG_VENTAS`, y ejecutarás una progresión completa de consultas SQL: desde filtros básicos hasta window functions y joins avanzados. Al finalizar, validarás cinco métricas de negocio que confirman la integridad y utilidad de los datos cargados.

## Objetivos de Aprendizaje

- [ ] Insertar datos de ejemplo en las tablas `STG_CLIENTES` (50 registros), `STG_PRODUCTOS` (30 registros) y `STG_VENTAS` (200 registros) usando `INSERT INTO … VALUES` múltiples.
- [ ] Ejecutar consultas SQL con `SELECT`, `WHERE`, `ORDER BY`, `LIMIT` y funciones escalares de Snowflake (`UPPER`, `LOWER`, `ROUND`, `DATEDIFF`, `TO_DATE`, `COALESCE`, `IFF`).
- [ ] Aplicar `GROUP BY`, `HAVING` y funciones de agregación (`SUM`, `COUNT`, `AVG`, `MAX`, `MIN`, `COUNT DISTINCT`) para calcular métricas de negocio.
- [ ] Implementar window functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `SUM OVER PARTITION BY`, `LAG`, `LEAD`) para análisis avanzados.
- [ ] Ejecutar los cinco tipos de JOIN (`INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, `CROSS`) sobre el modelo de ventas y verificar resultados.

## Prerrequisitos

### Conocimientos previos
- SQL básico: `SELECT`, `WHERE`, `JOIN` (nivel introductorio).
- Familiaridad con la interfaz Snowsight (navegación de worksheets, ejecución de sentencias).

### Acceso y configuración previa
- Laboratorio 02-02-01 completado exitosamente:
  - Base de datos `DB_CURSO_SNOWFLAKE` existente.
  - Esquema `SCH_STAGING` con tablas `STG_CLIENTES`, `STG_PRODUCTOS`, `STG_VENTAS` creadas y **vacías**.
  - Virtual Warehouse `WH_CURSO_XS` operativo.
- Sesión activa en Snowsight con rol `SYSADMIN`.

## Entorno del Laboratorio

| Componente | Especificación |
|------------|---------------|
| Cuenta Snowflake | Enterprise Trial, AWS us-east-1 |
| Warehouse | `WH_CURSO_XS` (X-Small, AUTO_SUSPEND=60, AUTO_RESUME=TRUE) |
| Base de datos | `DB_CURSO_SNOWFLAKE` |
| Esquema | `SCH_STAGING` |
| Interfaz | Snowsight (navegador Chrome 124+, Firefox 125+ o Edge 124+) |

### Configuración inicial de sesión

Abre una nueva worksheet en Snowsight y nómbrala **`LAB03_Consultas_Analiticas`**. Ejecuta el siguiente bloque para establecer el contexto:

```sql
USE ROLE SYSADMIN;
USE WAREHOUSE WH_CURSO_XS;
USE DATABASE DB_CURSO_SNOWFLAKE;
USE SCHEMA SCH_STAGING;

-- Verificar que las tablas existen y están vacías
SELECT 'STG_CLIENTES' AS tabla, COUNT(*) AS registros FROM STG_CLIENTES
UNION ALL
SELECT 'STG_PRODUCTOS', COUNT(*) FROM STG_PRODUCTOS
UNION ALL
SELECT 'STG_VENTAS', COUNT(*) FROM STG_VENTAS;
```

**Resultado esperado:** Tres filas, todas con `registros = 0`.

---

## Paso a Paso

### Paso 1: Insertar datos en STG_CLIENTES (50 registros)

**Objetivo:** Poblar la tabla de clientes con 50 registros representativos que incluyan variedad de ciudades, países, segmentos y estados activo/inactivo.

**Instrucciones:**

1. En la worksheet `LAB03_Consultas_Analiticas`, copia y ejecuta el siguiente bloque INSERT:

```sql
INSERT INTO STG_CLIENTES (id_cliente, nombre, apellido, email, telefono, ciudad, pais, segmento, fecha_registro, activo)
VALUES
(1,  'Carlos',    'García',     'carlos.garcia@email.com',    '+5211234001', 'Ciudad de México', 'México',    'Corporativo', '2022-01-15', TRUE),
(2,  'María',     'López',      'maria.lopez@email.com',      '+5211234002', 'Guadalajara',      'México',    'Consumidor',  '2022-02-20', TRUE),
(3,  'Juan',      'Martínez',   'juan.martinez@email.com',    '+5211234003', 'Monterrey',        'México',    'Corporativo', '2022-03-10', TRUE),
(4,  'Ana',       'Rodríguez',  'ana.rodriguez@email.com',    '+5711234004', 'Bogotá',           'Colombia',  'Consumidor',  '2022-03-22', TRUE),
(5,  'Pedro',     'Hernández',  'pedro.hernandez@email.com',  '+5411234005', 'Buenos Aires',     'Argentina', 'PyME',        '2022-04-01', TRUE),
(6,  'Laura',     'González',   'laura.gonzalez@email.com',   '+5611234006', 'São Paulo',        'Brasil',    'Corporativo', '2022-04-15', TRUE),
(7,  'Diego',     'Sánchez',    'diego.sanchez@email.com',    '+5211234007', 'Ciudad de México', 'México',    'PyME',        '2022-05-03', TRUE),
(8,  'Sofía',     'Ramírez',    'sofia.ramirez@email.com',    '+5611234008', 'Río de Janeiro',   'Brasil',    'Consumidor',  '2022-05-18', TRUE),
(9,  'Andrés',    'Torres',     'andres.torres@email.com',    '+5711234009', 'Medellín',         'Colombia',  'PyME',        '2022-06-07', FALSE),
(10, 'Valentina', 'Flores',     'valentina.flores@email.com', '+5611234010', 'São Paulo',        'Brasil',    'Corporativo', '2022-06-20', TRUE),
(11, 'Roberto',   'Díaz',       'roberto.diaz@email.com',     '+5211234011', 'Guadalajara',      'México',    'Consumidor',  '2022-07-01', TRUE),
(12, 'Camila',    'Morales',    'camila.morales@email.com',   '+5411234012', 'Córdoba',          'Argentina', 'PyME',        '2022-07-14', TRUE),
(13, 'Fernando',  'Jiménez',    'fernando.jimenez@email.com', '+5211234013', 'Monterrey',        'México',    'Corporativo', '2022-07-28', TRUE),
(14, 'Isabella',  'Ruiz',       'isabella.ruiz@email.com',    '+5711234014', 'Cali',             'Colombia',  'Consumidor',  '2022-08-10', FALSE),
(15, 'Miguel',    'Vargas',     'miguel.vargas@email.com',    '+5411234015', 'Buenos Aires',     'Argentina', 'Corporativo', '2022-08-25', TRUE),
(16, 'Lucía',     'Castro',     'lucia.castro@email.com',     '+5211234016', 'Ciudad de México', 'México',    'Consumidor',  '2022-09-05', TRUE),
(17, 'Alejandro', 'Mendoza',    'alejandro.mendoza@email.com','+5711234017', 'Bogotá',           'Colombia',  'PyME',        '2022-09-18', TRUE),
(18, 'Gabriela',  'Ortiz',      'gabriela.ortiz@email.com',   '+5611234018', 'São Paulo',        'Brasil',    'Corporativo', '2022-10-02', TRUE),
(19, 'Javier',    'Guerrero',   'javier.guerrero@email.com',  '+5211234019', 'Guadalajara',      'México',    'PyME',        '2022-10-15', TRUE),
(20, 'Paula',     'Reyes',      'paula.reyes@email.com',      '+5411234020', 'Rosario',          'Argentina', 'Consumidor',  '2022-10-30', TRUE),
(21, 'Ricardo',   'Cruz',       'ricardo.cruz@email.com',     '+5211234021', 'Monterrey',        'México',    'Corporativo', '2022-11-08', TRUE),
(22, 'Daniela',   'Herrera',    'daniela.herrera@email.com',  '+5711234022', 'Bogotá',           'Colombia',  'Consumidor',  '2022-11-20', FALSE),
(23, 'Emilio',    'Aguilar',    'emilio.aguilar@email.com',   '+5611234023', 'Río de Janeiro',   'Brasil',    'PyME',        '2022-12-01', TRUE),
(24, 'Mariana',   'Medina',     'mariana.medina@email.com',   '+5211234024', 'Ciudad de México', 'México',    'Corporativo', '2022-12-14', TRUE),
(25, 'Óscar',     'Peña',       'oscar.pena@email.com',       '+5411234025', 'Buenos Aires',     'Argentina', 'Consumidor',  '2023-01-05', TRUE),
(26, 'Natalia',   'Romero',     'natalia.romero@email.com',   '+5711234026', 'Medellín',         'Colombia',  'Corporativo', '2023-01-18', TRUE),
(27, 'Sergio',    'Navarro',    'sergio.navarro@email.com',   '+5211234027', 'Guadalajara',      'México',    'PyME',        '2023-02-02', TRUE),
(28, 'Valeria',   'Domínguez',  'valeria.dominguez@email.com','+5611234028', 'São Paulo',        'Brasil',    'Consumidor',  '2023-02-15', TRUE),
(29, 'Héctor',    'Suárez',     'hector.suarez@email.com',    '+5211234029', 'Monterrey',        'México',    'Corporativo', '2023-03-01', FALSE),
(30, 'Carolina',  'Ramos',      'carolina.ramos@email.com',   '+5411234030', 'Córdoba',          'Argentina', 'PyME',        '2023-03-14', TRUE),
(31, 'Luis',      'Molina',     'luis.molina@email.com',       '+5711234031', 'Cali',             'Colombia',  'Consumidor',  '2023-03-28', TRUE),
(32, 'Elena',     'Contreras',  'elena.contreras@email.com',  '+5211234032', 'Ciudad de México', 'México',    'Corporativo', '2023-04-10', TRUE),
(33, 'Tomás',     'Sandoval',   'tomas.sandoval@email.com',   '+5611234033', 'Río de Janeiro',   'Brasil',    'PyME',        '2023-04-22', TRUE),
(34, 'Andrea',    'Ibarra',     'andrea.ibarra@email.com',    '+5411234034', 'Buenos Aires',     'Argentina', 'Consumidor',  '2023-05-05', TRUE),
(35, 'Raúl',      'Espinoza',   'raul.espinoza@email.com',    '+5211234035', 'Guadalajara',      'México',    'Corporativo', '2023-05-18', TRUE),
(36, 'Mónica',    'Delgado',    'monica.delgado@email.com',   '+5711234036', 'Bogotá',           'Colombia',  'PyME',        '2023-06-01', FALSE),
(37, 'Iván',      'Cabrera',    'ivan.cabrera@email.com',     '+5611234037', 'São Paulo',        'Brasil',    'Consumidor',  '2023-06-14', TRUE),
(38, 'Patricia',  'Vega',       'patricia.vega@email.com',    '+5211234038', 'Monterrey',        'México',    'Corporativo', '2023-06-28', TRUE),
(39, 'Alberto',   'Fuentes',    'alberto.fuentes@email.com',  '+5411234039', 'Rosario',          'Argentina', 'PyME',        '2023-07-10', TRUE),
(40, 'Adriana',   'Campos',     'adriana.campos@email.com',   '+5711234040', 'Medellín',         'Colombia',  'Consumidor',  '2023-07-22', TRUE),
(41, 'Francisco', 'León',       'francisco.leon@email.com',   '+5211234041', 'Ciudad de México', 'México',    'Corporativo', '2023-08-03', TRUE),
(42, 'Teresa',    'Pacheco',    'teresa.pacheco@email.com',   '+5611234042', 'Río de Janeiro',   'Brasil',    'PyME',        '2023-08-16', TRUE),
(43, 'Arturo',    'Ríos',       'arturo.rios@email.com',      '+5411234043', 'Buenos Aires',     'Argentina', 'Consumidor',  '2023-08-29', TRUE),
(44, 'Claudia',   'Salazar',    'claudia.salazar@email.com',  '+5211234044', 'Guadalajara',      'México',    'Corporativo', '2023-09-10', TRUE),
(45, 'Enrique',   'Guzmán',     'enrique.guzman@email.com',   '+5711234045', 'Cali',             'Colombia',  'PyME',        '2023-09-22', TRUE),
(46, 'Rosa',      'Acosta',     'rosa.acosta@email.com',      '+5611234046', 'São Paulo',        'Brasil',    'Consumidor',  '2023-10-05', TRUE),
(47, 'Gustavo',   'Miranda',    'gustavo.miranda@email.com',  '+5211234047', 'Monterrey',        'México',    'Corporativo', '2023-10-18', TRUE),
(48, 'Silvia',    'Rojas',      'silvia.rojas@email.com',     '+5411234048', 'Córdoba',          'Argentina', 'PyME',        '2023-11-01', FALSE),
(49, 'Manuel',    'Pereira',    'manuel.pereira@email.com',   '+5611234049', 'Río de Janeiro',   'Brasil',    'Consumidor',  '2023-11-14', TRUE),
(50, 'Lorena',    'Figueroa',   'lorena.figueroa@email.com',  '+5711234050', 'Bogotá',           'Colombia',  'Corporativo', '2023-11-28', TRUE);
```

2. Verifica la carga:

```sql
SELECT COUNT(*) AS total_clientes FROM STG_CLIENTES;
```

**Resultado esperado:**

| TOTAL_CLIENTES |
|----------------|
| 50 |

**Verificación:** Confirma que existen clientes de los 4 países y 3 segmentos:

```sql
SELECT pais, segmento, COUNT(*) AS cantidad
FROM STG_CLIENTES
GROUP BY pais, segmento
ORDER BY pais, segmento;
```

Debes obtener combinaciones de México, Colombia, Argentina y Brasil con segmentos Corporativo, Consumidor y PyME.

---

### Paso 2: Insertar datos en STG_PRODUCTOS (30 registros)

**Objetivo:** Cargar 30 productos con categorías, precios y costos variados para habilitar análisis de rentabilidad.

**Instrucciones:**

1. Ejecuta el siguiente INSERT:

```sql
INSERT INTO STG_PRODUCTOS (id_producto, nombre_producto, categoria, subcategoria, precio_unitario, costo_unitario, unidad_medida, activo)
VALUES
(1,  'Laptop Pro 15',         'Tecnología',    'Computadoras',     18500.00, 12950.00, 'Pieza', TRUE),
(2,  'Monitor 27 4K',         'Tecnología',    'Monitores',         8900.00,  5340.00, 'Pieza', TRUE),
(3,  'Teclado Mecánico RGB',  'Tecnología',    'Accesorios',        2100.00,  1050.00, 'Pieza', TRUE),
(4,  'Mouse Ergonómico',      'Tecnología',    'Accesorios',         890.00,   445.00, 'Pieza', TRUE),
(5,  'Auriculares BT Pro',    'Tecnología',    'Audio',             3500.00,  1750.00, 'Pieza', TRUE),
(6,  'Webcam HD 1080p',       'Tecnología',    'Accesorios',        1200.00,   600.00, 'Pieza', TRUE),
(7,  'Tablet 10 pulgadas',    'Tecnología',    'Tablets',           7200.00,  4320.00, 'Pieza', TRUE),
(8,  'Disco SSD 1TB',         'Tecnología',    'Almacenamiento',    2800.00,  1680.00, 'Pieza', TRUE),
(9,  'Impresora Láser',       'Tecnología',    'Impresoras',        5600.00,  3360.00, 'Pieza', TRUE),
(10, 'Router WiFi 6',         'Tecnología',    'Redes',             3200.00,  1920.00, 'Pieza', TRUE),
(11, 'Escritorio Ejecutivo',  'Mobiliario',    'Escritorios',       12000.00,  7200.00, 'Pieza', TRUE),
(12, 'Silla Ergonómica',      'Mobiliario',    'Sillas',            8500.00,  5100.00, 'Pieza', TRUE),
(13, 'Archivero 4 Cajones',   'Mobiliario',    'Almacenamiento',    4500.00,  2700.00, 'Pieza', TRUE),
(14, 'Mesa de Juntas',        'Mobiliario',    'Mesas',            15000.00,  9000.00, 'Pieza', TRUE),
(15, 'Librero Modular',       'Mobiliario',    'Almacenamiento',    6800.00,  4080.00, 'Pieza', TRUE),
(16, 'Silla Visitante',       'Mobiliario',    'Sillas',            2200.00,  1320.00, 'Pieza', TRUE),
(17, 'Lámpara de Escritorio', 'Mobiliario',    'Iluminación',       1500.00,   750.00, 'Pieza', TRUE),
(18, 'Pizarrón Blanco 2m',   'Mobiliario',    'Accesorios',        3800.00,  2280.00, 'Pieza', TRUE),
(19, 'Papel Bond A4 (5000)',  'Suministros',   'Papel',              450.00,   270.00, 'Paquete', TRUE),
(20, 'Tóner Negro Universal', 'Suministros',   'Impresión',         1800.00,  1080.00, 'Pieza', TRUE),
(21, 'Bolígrafos (caja 50)',  'Suministros',   'Escritura',          320.00,   160.00, 'Caja',  TRUE),
(22, 'Carpetas Manila (100)', 'Suministros',   'Archivo',            280.00,   140.00, 'Paquete', TRUE),
(23, 'Post-it Colores (12)',  'Suministros',   'Escritura',          180.00,    90.00, 'Paquete', TRUE),
(24, 'Grapadora Industrial',  'Suministros',   'Herramientas',       650.00,   325.00, 'Pieza', TRUE),
(25, 'Cinta Adhesiva (6)',    'Suministros',   'Herramientas',       120.00,    60.00, 'Paquete', TRUE),
(26, 'Marcadores (set 10)',   'Suministros',   'Escritura',          250.00,   125.00, 'Set',   TRUE),
(27, 'Organizador Escritorio','Suministros',   'Accesorios',         480.00,   240.00, 'Pieza', TRUE),
(28, 'Calculadora Científica','Suministros',   'Herramientas',       550.00,   275.00, 'Pieza', TRUE),
(29, 'Hub USB-C 7 puertos',  'Tecnología',    'Accesorios',        1600.00,   800.00, 'Pieza', TRUE),
(30, 'Cable HDMI 3m',        'Tecnología',    'Accesorios',         350.00,   175.00, 'Pieza', FALSE);
```

2. Verifica:

```sql
SELECT categoria, COUNT(*) AS productos, ROUND(AVG(precio_unitario), 2) AS precio_promedio
FROM STG_PRODUCTOS
GROUP BY categoria
ORDER BY categoria;
```

**Resultado esperado:** Tres categorías (Mobiliario, Suministros, Tecnología) con sus respectivos conteos y precios promedio.

---

### Paso 3: Insertar datos en STG_VENTAS (200 registros)

**Objetivo:** Generar 200 transacciones de venta distribuidas entre enero 2023 y diciembre 2023, con variedad de clientes, productos, canales y regiones.

**Instrucciones:**

1. Ejecuta el siguiente INSERT (dividido en bloques de 50 para mejor legibilidad). Ejecuta los cuatro bloques secuencialmente:

```sql
-- Bloque 1: Ventas 1-50
INSERT INTO STG_VENTAS (id_venta, id_cliente, id_producto, fecha_venta, cantidad, precio_venta, descuento, canal_venta, region)
VALUES
(1,   1,  1,  '2023-01-05', 2, 18500.00, 0.05, 'Online',    'Norte'),
(2,   2,  3,  '2023-01-07', 5, 2100.00,  0.00, 'Tienda',    'Occidente'),
(3,   3,  11, '2023-01-10', 1, 12000.00, 0.10, 'Online',    'Norte'),
(4,   4,  19, '2023-01-12', 10,450.00,   0.00, 'Distribuidor','Sur'),
(5,   5,  5,  '2023-01-15', 3, 3500.00,  0.05, 'Online',    'Sur'),
(6,   6,  12, '2023-01-18', 2, 8500.00,  0.08, 'Tienda',    'Este'),
(7,   7,  20, '2023-01-20', 4, 1800.00,  0.00, 'Online',    'Centro'),
(8,   8,  7,  '2023-01-22', 1, 7200.00,  0.12, 'Tienda',    'Este'),
(9,   9,  21, '2023-01-25', 8, 320.00,   0.00, 'Distribuidor','Sur'),
(10, 10,  2,  '2023-01-28', 2, 8900.00,  0.05, 'Online',    'Este'),
(11,  1,  8,  '2023-02-01', 3, 2800.00,  0.00, 'Online',    'Norte'),
(12, 11,  14, '2023-02-03', 1, 15000.00, 0.15, 'Tienda',    'Occidente'),
(13, 12,  22, '2023-02-05', 15,280.00,   0.00, 'Distribuidor','Sur'),
(14, 13,  9,  '2023-02-08', 1, 5600.00,  0.05, 'Online',    'Norte'),
(15, 14,  4,  '2023-02-10', 6, 890.00,   0.00, 'Tienda',    'Sur'),
(16, 15,  1,  '2023-02-12', 1, 18500.00, 0.10, 'Online',    'Sur'),
(17, 16,  23, '2023-02-15', 20,180.00,   0.00, 'Distribuidor','Centro'),
(18, 17,  13, '2023-02-18', 2, 4500.00,  0.05, 'Tienda',    'Sur'),
(19, 18,  6,  '2023-02-20', 4, 1200.00,  0.00, 'Online',    'Este'),
(20, 19,  15, '2023-02-22', 1, 6800.00,  0.08, 'Tienda',    'Occidente'),
(21, 20,  24, '2023-02-25', 3, 650.00,   0.00, 'Online',    'Sur'),
(22, 21,  10, '2023-02-28', 2, 3200.00,  0.05, 'Online',    'Norte'),
(23, 22,  16, '2023-03-02', 4, 2200.00,  0.00, 'Tienda',    'Sur'),
(24, 23,  25, '2023-03-05', 10,120.00,   0.00, 'Distribuidor','Este'),
(25, 24,  2,  '2023-03-07', 3, 8900.00,  0.10, 'Online',    'Centro'),
(26, 25,  11, '2023-03-10', 1, 12000.00, 0.05, 'Tienda',    'Sur'),
(27, 26,  17, '2023-03-12', 5, 1500.00,  0.00, 'Online',    'Sur'),
(28, 27,  3,  '2023-03-15', 3, 2100.00,  0.00, 'Tienda',    'Occidente'),
(29, 28,  26, '2023-03-18', 8, 250.00,   0.00, 'Distribuidor','Este'),
(30, 29,  12, '2023-03-20', 1, 8500.00,  0.10, 'Online',    'Norte'),
(31, 30,  19, '2023-03-22', 12,450.00,   0.00, 'Distribuidor','Sur'),
(32, 31,  5,  '2023-03-25', 2, 3500.00,  0.05, 'Online',    'Sur'),
(33, 32,  14, '2023-03-28', 1, 15000.00, 0.12, 'Tienda',    'Centro'),
(34, 33,  27, '2023-03-30', 6, 480.00,   0.00, 'Distribuidor','Este'),
(35, 34,  7,  '2023-04-02', 1, 7200.00,  0.05, 'Online',    'Sur'),
(36, 35,  18, '2023-04-05', 2, 3800.00,  0.00, 'Tienda',    'Occidente'),
(37, 36,  28, '2023-04-08', 4, 550.00,   0.00, 'Online',    'Sur'),
(38, 37,  1,  '2023-04-10', 1, 18500.00, 0.08, 'Online',    'Este'),
(39, 38,  8,  '2023-04-12', 2, 2800.00,  0.00, 'Tienda',    'Norte'),
(40, 39,  20, '2023-04-15', 3, 1800.00,  0.05, 'Distribuidor','Sur'),
(41, 40,  9,  '2023-04-18', 1, 5600.00,  0.00, 'Online',    'Sur'),
(42, 41,  29, '2023-04-20', 5, 1600.00,  0.00, 'Online',    'Centro'),
(43, 42,  13, '2023-04-22', 1, 4500.00,  0.10, 'Tienda',    'Este'),
(44, 43,  4,  '2023-04-25', 8, 890.00,   0.00, 'Tienda',    'Sur'),
(45, 44,  10, '2023-04-28', 2, 3200.00,  0.05, 'Online',    'Occidente'),
(46, 45,  15, '2023-05-01', 1, 6800.00,  0.00, 'Tienda',    'Sur'),
(47, 46,  30, '2023-05-03', 10,350.00,   0.00, 'Distribuidor','Este'),
(48, 47,  2,  '2023-05-05', 1, 8900.00,  0.05, 'Online',    'Norte'),
(49, 48,  21, '2023-05-08', 12,320.00,   0.00, 'Distribuidor','Sur'),
(50,  1,  11, '2023-05-10', 2, 12000.00, 0.10, 'Online',    'Norte');
```

```sql
-- Bloque 2: Ventas 51-100
INSERT INTO STG_VENTAS (id_venta, id_cliente, id_producto, fecha_venta, cantidad, precio_venta, descuento, canal_venta, region)
VALUES
(51,  2,  6,  '2023-05-12', 3, 1200.00,  0.00, 'Tienda',    'Occidente'),
(52,  3,  16, '2023-05-15', 6, 2200.00,  0.05, 'Online',    'Norte'),
(53,  4,  22, '2023-05-18', 20,280.00,   0.00, 'Distribuidor','Sur'),
(54,  5,  1,  '2023-05-20', 1, 18500.00, 0.12, 'Online',    'Sur'),
(55,  6,  9,  '2023-05-22', 2, 5600.00,  0.05, 'Tienda',    'Este'),
(56,  7,  23, '2023-05-25', 15,180.00,   0.00, 'Distribuidor','Centro'),
(57,  8,  3,  '2023-05-28', 4, 2100.00,  0.00, 'Online',    'Este'),
(58,  9,  14, '2023-05-30', 1, 15000.00, 0.10, 'Tienda',    'Sur'),
(59, 10,  17, '2023-06-02', 3, 1500.00,  0.00, 'Online',    'Este'),
(60, 11,  5,  '2023-06-05', 2, 3500.00,  0.05, 'Tienda',    'Occidente'),
(61, 12,  24, '2023-06-08', 5, 650.00,   0.00, 'Distribuidor','Sur'),
(62, 13,  7,  '2023-06-10', 1, 7200.00,  0.08, 'Online',    'Norte'),
(63, 14,  18, '2023-06-12', 2, 3800.00,  0.00, 'Tienda',    'Sur'),
(64, 15,  25, '2023-06-15', 8, 120.00,   0.00, 'Distribuidor','Sur'),
(65, 16,  12, '2023-06-18', 1, 8500.00,  0.10, 'Online',    'Centro'),
(66, 17,  19, '2023-06-20', 6, 450.00,   0.00, 'Distribuidor','Sur'),
(67, 18,  8,  '2023-06-22', 3, 2800.00,  0.00, 'Online',    'Este'),
(68, 19,  26, '2023-06-25', 10,250.00,   0.00, 'Tienda',    'Occidente'),
(69, 20,  10, '2023-06-28', 1, 3200.00,  0.05, 'Online',    'Sur'),
(70, 21,  13, '2023-06-30', 2, 4500.00,  0.00, 'Tienda',    'Norte'),
(71, 22,  4,  '2023-07-02', 5, 890.00,   0.00, 'Online',    'Sur'),
(72, 23,  27, '2023-07-05', 4, 480.00,   0.00, 'Distribuidor','Este'),
(73, 24,  1,  '2023-07-08', 1, 18500.00, 0.05, 'Online',    'Centro'),
(74, 25,  15, '2023-07-10', 2, 6800.00,  0.00, 'Tienda',    'Sur'),
(75, 26,  20, '2023-07-12', 3, 1800.00,  0.00, 'Online',    'Sur'),
(76, 27,  11, '2023-07-15', 1, 12000.00, 0.08, 'Tienda',    'Occidente'),
(77, 28,  28, '2023-07-18', 6, 550.00,   0.00, 'Online',    'Este'),
(78, 29,  6,  '2023-07-20', 2, 1200.00,  0.00, 'Tienda',    'Norte'),
(79, 30,  2,  '2023-07-22', 1, 8900.00,  0.10, 'Online',    'Sur'),
(80, 31,  16, '2023-07-25', 3, 2200.00,  0.00, 'Tienda',    'Sur'),
(81, 32,  29, '2023-07-28', 4, 1600.00,  0.05, 'Online',    'Centro'),
(82, 33,  9,  '2023-07-30', 1, 5600.00,  0.00, 'Tienda',    'Este'),
(83, 34,  22, '2023-08-02', 10,280.00,   0.00, 'Distribuidor','Sur'),
(84, 35,  3,  '2023-08-05', 6, 2100.00,  0.00, 'Online',    'Occidente'),
(85, 36,  12, '2023-08-08', 1, 8500.00,  0.05, 'Tienda',    'Sur'),
(86, 37,  19, '2023-08-10', 8, 450.00,   0.00, 'Distribuidor','Este'),
(87, 38,  7,  '2023-08-12', 2, 7200.00,  0.10, 'Online',    'Norte'),
(88, 39,  14, '2023-08-15', 1, 15000.00, 0.08, 'Tienda',    'Sur'),
(89, 40,  23, '2023-08-18', 12,180.00,   0.00, 'Distribuidor','Sur'),
(90, 41,  5,  '2023-08-20', 3, 3500.00,  0.00, 'Online',    'Centro'),
(91, 42,  17, '2023-08-22', 2, 1500.00,  0.00, 'Tienda',    'Este'),
(92, 43,  10, '2023-08-25', 1, 3200.00,  0.05, 'Online',    'Sur'),
(93, 44,  24, '2023-08-28', 4, 650.00,   0.00, 'Tienda',    'Occidente'),
(94, 45,  8,  '2023-08-30', 2, 2800.00,  0.00, 'Online',    'Sur'),
(95, 46,  13, '2023-09-02', 1, 4500.00,  0.10, 'Tienda',    'Este'),
(96, 47,  21, '2023-09-05', 6, 320.00,   0.00, 'Distribuidor','Norte'),
(97, 48,  11, '2023-09-08', 1, 12000.00, 0.05, 'Online',    'Sur'),
(98, 49,  4,  '2023-09-10', 10,890.00,   0.00, 'Tienda',    'Este'),
(99, 50,  18, '2023-09-12', 1, 3800.00,  0.00, 'Online',    'Sur'),
(100, 1,  2,  '2023-09-15', 2, 8900.00,  0.05, 'Online',    'Norte');
```

```sql
-- Bloque 3: Ventas 101-150
INSERT INTO STG_VENTAS (id_venta, id_cliente, id_producto, fecha_venta, cantidad, precio_venta, descuento, canal_venta, region)
VALUES
(101, 2,  15, '2023-09-18', 1, 6800.00,  0.00, 'Tienda',    'Occidente'),
(102, 3,  25, '2023-09-20', 12,120.00,   0.00, 'Distribuidor','Norte'),
(103, 4,  9,  '2023-09-22', 1, 5600.00,  0.05, 'Online',    'Sur'),
(104, 5,  26, '2023-09-25', 6, 250.00,   0.00, 'Distribuidor','Sur'),
(105, 6,  1,  '2023-09-28', 1, 18500.00, 0.10, 'Online',    'Este'),
(106, 7,  16, '2023-09-30', 4, 2200.00,  0.00, 'Tienda',    'Centro'),
(107, 8,  27, '2023-10-02', 5, 480.00,   0.00, 'Online',    'Este'),
(108, 9,  5,  '2023-10-05', 2, 3500.00,  0.05, 'Tienda',    'Sur'),
(109,10,  14, '2023-10-08', 1, 15000.00, 0.12, 'Online',    'Este'),
(110,11,  20, '2023-10-10', 3, 1800.00,  0.00, 'Distribuidor','Occidente'),
(111,12,  3,  '2023-10-12', 4, 2100.00,  0.00, 'Tienda',    'Sur'),
(112,13,  12, '2023-10-15', 2, 8500.00,  0.08, 'Online',    'Norte'),
(113,14,  28, '2023-10-18', 3, 550.00,   0.00, 'Tienda',    'Sur'),
(114,15,  7,  '2023-10-20', 1, 7200.00,  0.05, 'Online',    'Sur'),
(115,16,  22, '2023-10-22', 15,280.00,   0.00, 'Distribuidor','Centro'),
(116,17,  6,  '2023-10-25', 3, 1200.00,  0.00, 'Online',    'Sur'),
(117,18,  11, '2023-10-28', 1, 12000.00, 0.10, 'Tienda',    'Este'),
(118,19,  19, '2023-10-30', 8, 450.00,   0.00, 'Distribuidor','Occidente'),
(119,20,  29, '2023-11-02', 3, 1600.00,  0.00, 'Online',    'Sur'),
(120,21,  8,  '2023-11-05', 2, 2800.00,  0.00, 'Online',    'Norte'),
(121,22,  13, '2023-11-08', 1, 4500.00,  0.05, 'Tienda',    'Sur'),
(122,23,  4,  '2023-11-10', 7, 890.00,   0.00, 'Online',    'Este'),
(123,24,  17, '2023-11-12', 4, 1500.00,  0.00, 'Tienda',    'Centro'),
(124,25,  1,  '2023-11-15', 1, 18500.00, 0.05, 'Online',    'Sur'),
(125,26,  10, '2023-11-18', 2, 3200.00,  0.00, 'Online',    'Sur'),
(126,27,  23, '2023-11-20', 10,180.00,   0.00, 'Distribuidor','Occidente'),
(127,28,  15, '2023-11-22', 1, 6800.00,  0.08, 'Tienda',    'Este'),
(128,29,  2,  '2023-11-25', 1, 8900.00,  0.00, 'Online',    'Norte'),
(129,30,  24, '2023-11-28', 5, 650.00,   0.00, 'Distribuidor','Sur'),
(130,31,  9,  '2023-11-30', 1, 5600.00,  0.00, 'Online',    'Sur'),
(131,32,  18, '2023-12-02', 3, 3800.00,  0.05, 'Tienda',    'Centro'),
(132,33,  30, '2023-12-05', 8, 350.00,   0.00, 'Distribuidor','Este'),
(133,34,  12, '2023-12-08', 1, 8500.00,  0.00, 'Online',    'Sur'),
(134,35,  5,  '2023-12-10', 4, 3500.00,  0.05, 'Tienda',    'Occidente'),
(135,36,  21, '2023-12-12', 6, 320.00,   0.00, 'Distribuidor','Sur'),
(136,37,  7,  '2023-12-15', 2, 7200.00,  0.10, 'Online',    'Este'),
(137,38,  14, '2023-12-18', 1, 15000.00, 0.05, 'Tienda',    'Norte'),
(138,39,  26, '2023-12-20', 5, 250.00,   0.00, 'Distribuidor','Sur'),
(139,40,  3,  '2023-12-22', 3, 2100.00,  0.00, 'Online',    'Sur'),
(140,41,  16, '2023-12-25', 2, 2200.00,  0.00, 'Tienda',    'Centro'),
(141,42,  8,  '2023-12-28', 1, 2800.00,  0.00, 'Online',    'Este'),
(142,43,  11, '2023-12-30', 1, 12000.00, 0.08, 'Tienda',    'Sur'),
(143,44,  20, '2023-01-08', 2, 1800.00,  0.00, 'Online',    'Occidente'),
(144,45,  27, '2023-01-14', 3, 480.00,   0.00, 'Tienda',    'Sur'),
(145,46,  1,  '2023-02-06', 1, 18500.00, 0.05, 'Online',    'Este'),
(146,47,  13, '2023-02-14', 2, 4500.00,  0.00, 'Tienda',    'Norte'),
(147,48,  6,  '2023-03-08', 5, 1200.00,  0.00, 'Online',    'Sur'),
(148,49,  10, '2023-03-16', 1, 3200.00,  0.00, 'Tienda',    'Este'),
(149,50,  22, '2023-04-04', 8, 280.00,   0.00, 'Distribuidor','Sur'),
(150, 1,  5,  '2023-04-14', 2, 3500.00,  0.05, 'Online',    'Norte');
```

```sql
-- Bloque 4: Ventas 151-200
INSERT INTO STG_VENTAS (id_venta, id_cliente, id_producto, fecha_venta, cantidad, precio_venta, descuento, canal_venta, region)
VALUES
(151, 2,  9,  '2023-04-20', 1, 5600.00,  0.00, 'Tienda',    'Occidente'),
(152, 3,  17, '2023-04-26', 3, 1500.00,  0.00, 'Online',    'Norte'),
(153, 4,  28, '2023-05-04', 2, 550.00,   0.00, 'Tienda',    'Sur'),
(154, 5,  12, '2023-05-14', 1, 8500.00,  0.10, 'Online',    'Sur'),
(155, 6,  25, '2023-05-22', 6, 120.00,   0.00, 'Distribuidor','Este'),
(156, 7,  2,  '2023-06-04', 1, 8900.00,  0.05, 'Online',    'Centro'),
(157, 8,  14, '2023-06-14', 1, 15000.00, 0.10, 'Tienda',    'Este'),
(158, 9,  4,  '2023-06-22', 4, 890.00,   0.00, 'Online',    'Sur'),
(159,10,  18, '2023-07-04', 2, 3800.00,  0.00, 'Tienda',    'Este'),
(160,11,  29, '2023-07-14', 3, 1600.00,  0.00, 'Online',    'Occidente'),
(161,12,  7,  '2023-07-22', 1, 7200.00,  0.05, 'Tienda',    'Sur'),
(162,13,  23, '2023-08-04', 8, 180.00,   0.00, 'Distribuidor','Norte'),
(163,14,  11, '2023-08-14', 1, 12000.00, 0.10, 'Online',    'Sur'),
(164,15,  19, '2023-08-22', 5, 450.00,   0.00, 'Distribuidor','Sur'),
(165,16,  8,  '2023-09-04', 2, 2800.00,  0.00, 'Online',    'Centro'),
(166,17,  15, '2023-09-14', 1, 6800.00,  0.05, 'Tienda',    'Sur'),
(167,18,  30, '2023-09-22', 6, 350.00,   0.00, 'Distribuidor','Este'),
(168,19,  1,  '2023-10-04', 1, 18500.00, 0.08, 'Online',    'Occidente'),
(169,20,  13, '2023-10-14', 1, 4500.00,  0.00, 'Tienda',    'Sur'),
(170,21,  6,  '2023-10-22', 4, 1200.00,  0.00, 'Online',    'Norte'),
(171,22,  20, '2023-11-04', 2, 1800.00,  0.00, 'Tienda',    'Sur'),
(172,23,  3,  '2023-11-14', 5, 2100.00,  0.00, 'Online',    'Este'),
(173,24,  16, '2023-11-22', 3, 2200.00,  0.00, 'Tienda',    'Centro'),
(174,25,  9,  '2023-12-04', 1, 5600.00,  0.00, 'Online',    'Sur'),
(175,26,  24, '2023-12-14', 4, 650.00,   0.00, 'Distribuidor','Sur'),
(176,27,  7,  '2023-12-22', 1, 7200.00,  0.05, 'Tienda',    'Occidente'),
(177,28,  11, '2023-01-20', 1, 12000.00, 0.00, 'Online',    'Este'),
(178,29,  21, '2023-02-18', 5, 320.00,   0.00, 'Distribuidor','Norte'),
(179,30,  5,  '2023-03-14', 2, 3500.00,  0.00, 'Online',    'Sur'),
(180,31,  14, '2023-04-12', 1, 15000.00, 0.10, 'Tienda',    'Sur'),
(181,32,  26, '2023-05-08', 7, 250.00,   0.00, 'Distribuidor','Centro'),
(182,33,  10, '2023-06-06', 2, 3200.00,  0.05, 'Online',    'Este'),
(183,34,  17, '2023-07-04', 3, 1500.00,  0.00, 'Tienda',    'Sur'),
(184,35,  22, '2023-08-02', 10,280.00,   0.00, 'Distribuidor','Occidente'),
(185,36,  8,  '2023-09-06', 1, 2800.00,  0.00, 'Online',    'Sur'),
(186,37,  13, '2023-10-08', 2, 4500.00,  0.00, 'Tienda',    'Este'),
(187,38,  25, '2023-11-06', 8, 120.00,   0.00, 'Distribuidor','Norte'),
(188,39,  2,  '2023-12-08', 1, 8900.00,  0.00, 'Online',    'Sur'),
(189,40,  19, '2023-01-28', 6, 450.00,   0.00, 'Distribuidor','Sur'),
(190,41,  12, '2023-02-26', 1, 8500.00,  0.05, 'Tienda',    'Centro'),
(191,42,  4,  '2023-03-24', 5, 890.00,   0.00, 'Online',    'Este'),
(192,43,  15, '2023-04-22', 1, 6800.00,  0.08, 'Tienda',    'Sur'),
(193,44,  28, '2023-05-18', 3, 550.00,   0.00, 'Online',    'Occidente'),
(194,45,  6,  '2023-06-16', 2, 1200.00,  0.00, 'Tienda',    'Sur'),
(195,46,  11, '2023-07-14', 1, 12000.00, 0.10, 'Online',    'Este'),
(196,47,  23, '2023-08-12', 10,180.00,   0.00, 'Distribuidor','Norte'),
(197,48,  1,  '2023-09-14', 1, 18500.00, 0.05, 'Online',    'Sur'),
(198,49,  16, '2023-10-12', 4, 2200.00,  0.00, 'Tienda',    'Este'),
(199,50,  9,  '2023-11-10', 1, 5600.00,  0.00, 'Online',    'Sur'),
(200, 1,  3,  '2023-12-15', 3, 2100.00,  0.00, 'Online',    'Norte');
```

2. Verifica el total de registros:

```sql
SELECT COUNT(*) AS total_ventas FROM STG_VENTAS;
```

**Resultado esperado:**

| TOTAL_VENTAS |
|--------------|
| 200 |

**Verificación adicional:**

```sql
SELECT
    MIN(fecha_venta) AS primera_venta,
    MAX(fecha_venta) AS ultima_venta,
    COUNT(DISTINCT id_cliente) AS clientes_unicos,
    COUNT(DISTINCT id_producto) AS productos_unicos
FROM STG_VENTAS;
```

Debes obtener: rango de fechas en 2023, al menos 40 clientes únicos y al menos 25 productos únicos.

---

### Paso 4: Bloque 1 — Consultas básicas con SELECT, aliases y funciones escalares

**Objetivo:** Practicar SELECT con expresiones calculadas, aliases y funciones escalares de Snowflake.

**Instrucciones:**

1. Ejecuta las siguientes consultas una por una, observando los resultados:

```sql
-- 4.1 Listado de clientes con nombre completo en mayúsculas y antigüedad en días
SELECT
    id_cliente,
    UPPER(nombre || ' ' || apellido)           AS nombre_completo,
    email,
    ciudad,
    segmento,
    DATEDIFF('day', fecha_registro, CURRENT_DATE()) AS dias_como_cliente
FROM STG_CLIENTES
ORDER BY dias_como_cliente DESC
LIMIT 10;
```

```sql
-- 4.2 Productos con margen de ganancia calculado
SELECT
    id_producto,
    nombre_producto,
    categoria,
    precio_unitario,
    costo_unitario,
    ROUND(precio_unitario - costo_unitario, 2)                    AS margen_bruto,
    ROUND((precio_unitario - costo_unitario) / precio_unitario * 100, 1) AS pct_margen
FROM STG_PRODUCTOS
WHERE activo = TRUE
ORDER BY pct_margen DESC;
```

```sql
-- 4.3 Ventas con monto neto (aplicando descuento) y clasificación por tamaño
SELECT
    id_venta,
    fecha_venta,
    cantidad,
    precio_venta,
    descuento,
    ROUND(cantidad * precio_venta * (1 - descuento), 2) AS monto_neto,
    IFF(cantidad * precio_venta * (1 - descuento) >= 10000, 'Grande',
        IFF(cantidad * precio_venta * (1 - descuento) >= 3000, 'Mediana', 'Pequeña')
    ) AS clasificacion
FROM STG_VENTAS
ORDER BY monto_neto DESC
LIMIT 15;
```

**Resultado esperado:** Cada consulta devuelve filas formateadas con los aliases definidos. La consulta 4.1 muestra los 10 clientes más antiguos; la 4.2 muestra todos los productos activos con su margen; la 4.3 muestra las 15 ventas de mayor monto neto.

---

### Paso 5: Bloque 2 — Filtros avanzados y ordenamiento

**Objetivo:** Aplicar `WHERE` con múltiples condiciones, `BETWEEN`, `IN`, `LIKE`, `IS NULL`, `ORDER BY` múltiple y `LIMIT` con `OFFSET`.

**Instrucciones:**

```sql
-- 5.1 Clientes corporativos de México o Colombia registrados en 2022
SELECT id_cliente, nombre, apellido, ciudad, pais, fecha_registro
FROM STG_CLIENTES
WHERE segmento = 'Corporativo'
  AND pais IN ('México', 'Colombia')
  AND fecha_registro BETWEEN '2022-01-01' AND '2022-12-31'
ORDER BY pais, fecha_registro;
```

```sql
-- 5.2 Ventas del canal Online en el segundo semestre 2023 con descuento aplicado
SELECT id_venta, fecha_venta, canal_venta, region,
       ROUND(cantidad * precio_venta * (1 - descuento), 2) AS monto_neto
FROM STG_VENTAS
WHERE canal_venta = 'Online'
  AND fecha_venta >= '2023-07-01'
  AND descuento > 0
ORDER BY monto_neto DESC;
```

```sql
-- 5.3 Productos cuyo nombre contiene 'USB' o 'HD' (LIKE)
SELECT id_producto, nombre_producto, categoria, precio_unitario
FROM STG_PRODUCTOS
WHERE nombre_producto LIKE '%USB%'
   OR nombre_producto LIKE '%HD%';
```

```sql
-- 5.4 Paginación: ventas 11 a 20 ordenadas por fecha (LIMIT + OFFSET)
SELECT id_venta, fecha_venta, id_cliente, id_producto,
       ROUND(cantidad * precio_venta * (1 - descuento), 2) AS monto_neto
FROM STG_VENTAS
ORDER BY fecha_venta, id_venta
LIMIT 10 OFFSET 10;
```

**Resultado esperado:** La consulta 5.1 devuelve solo clientes corporativos de MX/CO en 2022. La 5.2 filtra ventas Online con descuento del H2-2023. La 5.3 encuentra productos con esos patrones de texto. La 5.4 retorna exactamente 10 filas (posiciones 11-20).

---

### Paso 6: Bloque 3 — Agregaciones con GROUP BY y HAVING

**Objetivo:** Calcular métricas de negocio usando funciones de agregación, agrupamiento simple y múltiple, y filtrado de grupos.

**Instrucciones:**

```sql
-- 6.1 Ventas totales por región
SELECT
    region,
    COUNT(*)                                              AS num_transacciones,
    SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) AS ingresos_totales,
    ROUND(AVG(cantidad * precio_venta * (1 - descuento)), 2) AS ticket_promedio,
    MAX(cantidad * precio_venta * (1 - descuento))           AS venta_maxima
FROM STG_VENTAS
GROUP BY region
ORDER BY ingresos_totales DESC;
```

```sql
-- 6.2 Ventas mensuales (GROUP BY con DATE_TRUNC)
SELECT
    DATE_TRUNC('month', fecha_venta)::DATE AS mes,
    COUNT(*)                               AS num_ventas,
    SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) AS ingresos_mes
FROM STG_VENTAS
GROUP BY DATE_TRUNC('month', fecha_venta)
ORDER BY mes;
```

```sql
-- 6.3 Top clientes con más de 3 compras (HAVING)
SELECT
    id_cliente,
    COUNT(*)                                              AS num_compras,
    SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) AS total_gastado,
    MIN(fecha_venta)                                     AS primera_compra,
    MAX(fecha_venta)                                     AS ultima_compra
FROM STG_VENTAS
GROUP BY id_cliente
HAVING COUNT(*) > 3
ORDER BY total_gastado DESC;
```

```sql
-- 6.4 Categoría y canal: agrupamiento múltiple
SELECT
    p.categoria,
    v.canal_venta,
    COUNT(*)                                              AS transacciones,
    SUM(ROUND(v.cantidad * v.precio_venta * (1 - v.descuento), 2)) AS ingresos
FROM STG_VENTAS v
INNER JOIN STG_PRODUCTOS p ON v.id_producto = p.id_producto
GROUP BY p.categoria, v.canal_venta
ORDER BY p.categoria, ingresos DESC;
```

```sql
-- 6.5 Conteo de valores distintos
SELECT
    COUNT(DISTINCT canal_venta) AS canales_distintos,
    COUNT(DISTINCT region)      AS regiones_distintas,
    COUNT(DISTINCT id_cliente)  AS clientes_activos_ventas
FROM STG_VENTAS;
```

**Resultado esperado:** La consulta 6.1 muestra 5 regiones con sus métricas. La 6.2 muestra 12 meses. La 6.3 muestra solo clientes con más de 3 compras. La 6.4 combina categoría × canal. La 6.5 devuelve una sola fila con los conteos distintos.

---

### Paso 7: Bloque 4 — Window Functions

**Objetivo:** Implementar funciones de ventana para rankings, acumulados y comparaciones con periodos anteriores.

**Instrucciones:**

```sql
-- 7.1 Ranking de clientes por ingresos totales (ROW_NUMBER, RANK, DENSE_RANK)
SELECT
    id_cliente,
    SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) AS ingresos_cliente,
    ROW_NUMBER() OVER (ORDER BY SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) DESC) AS row_num,
    RANK()       OVER (ORDER BY SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) DESC) AS ranking,
    DENSE_RANK() OVER (ORDER BY SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) DESC) AS dense_ranking
FROM STG_VENTAS
GROUP BY id_cliente
ORDER BY row_num
LIMIT 10;
```

```sql
-- 7.2 Ingresos acumulados por mes (SUM OVER con ORDER BY)
SELECT
    DATE_TRUNC('month', fecha_venta)::DATE AS mes,
    SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) AS ingresos_mes,
    SUM(SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2))) OVER (
        ORDER BY DATE_TRUNC('month', fecha_venta)
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ingresos_acumulados
FROM STG_VENTAS
GROUP BY DATE_TRUNC('month', fecha_venta)
ORDER BY mes;
```

```sql
-- 7.3 Ranking de productos por ventas dentro de cada categoría (PARTITION BY)
WITH ventas_producto AS (
    SELECT
        p.categoria,
        p.nombre_producto,
        SUM(ROUND(v.cantidad * v.precio_venta * (1 - v.descuento), 2)) AS ingresos_producto
    FROM STG_VENTAS v
    INNER JOIN STG_PRODUCTOS p ON v.id_producto = p.id_producto
    GROUP BY p.categoria, p.nombre_producto
)
SELECT
    categoria,
    nombre_producto,
    ingresos_producto,
    ROW_NUMBER() OVER (PARTITION BY categoria ORDER BY ingresos_producto DESC) AS rank_en_categoria
FROM ventas_producto
ORDER BY categoria, rank_en_categoria;
```

```sql
-- 7.4 Comparación con mes anterior usando LAG y cálculo de variación
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)::DATE AS mes,
        SUM(ROUND(cantidad * precio_venta * (1 - descuento), 2)) AS ingresos_mes
    FROM STG_VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    mes,
    ingresos_mes,
    LAG(ingresos_mes, 1) OVER (ORDER BY mes)  AS ingresos_mes_anterior,
    LEAD(ingresos_mes, 1) OVER (ORDER BY mes) AS ingresos_mes_siguiente,
    ROUND(
        (ingresos_mes - LAG(ingresos_mes, 1) OVER (ORDER BY mes))
        / NULLIF(LAG(ingresos_mes, 1) OVER (ORDER BY mes), 0) * 100
    , 1) AS pct_variacion_vs_anterior
FROM ventas_mensuales
ORDER BY mes;
```

**Resultado esperado:** La consulta 7.1 muestra el top 10 de clientes con tres columnas de ranking. La 7.2 muestra 12 meses con columna de acumulado creciente. La 7.3 muestra productos rankeados dentro de su categoría. La 7.4 muestra 12 meses con variación porcentual (el primer mes tendrá NULL en `ingresos_mes_anterior`).

---

### Paso 8: Bloque 5 — Joins completos

**Objetivo:** Ejecutar los cinco tipos de JOIN sobre el modelo de ventas y comprender cuándo usar cada uno.

**Instrucciones:**

```sql
-- 8.1 INNER JOIN: Ventas enriquecidas con datos de cliente y producto
SELECT
    v.id_venta,
    v.fecha_venta,
    c.nombre || ' ' || c.apellido AS cliente,
    c.segmento,
    p.nombre_producto,
    p.categoria,
    v.cantidad,
    ROUND(v.cantidad * v.precio_venta * (1 - v.descuento), 2) AS monto_neto,
    v.canal_venta,
    v.region
FROM STG_VENTAS v
INNER JOIN STG_CLIENTES c ON v.id_cliente = c.id_cliente
INNER JOIN STG_PRODUCTOS p ON v.id_producto = p.id_producto
ORDER BY v.fecha_venta DESC
LIMIT 20;
```

```sql
-- 8.2 LEFT JOIN: Identificar clientes que NO tienen ventas
SELECT
    c.id_cliente,
    c.nombre || ' ' || c.apellido AS cliente,
    c.segmento,
    c.pais,
    COUNT(v.id_venta) AS num_ventas
FROM STG_CLIENTES c
LEFT JOIN STG_VENTAS v ON c.id_cliente = v.id_cliente
GROUP BY c.id_cliente, c.nombre, c.apellido, c.segmento, c.pais
HAVING COUNT(v.id_venta) = 0
ORDER BY c.id_cliente;
```

```sql
-- 8.3 RIGHT JOIN: Todos los productos, incluyendo los que no se vendieron
SELECT
    p.id_producto,
    p.nombre_producto,
    p.categoria,
    COALESCE(SUM(v.cantidad), 0)                              AS unidades_vendidas,
    COALESCE(SUM(ROUND(v.cantidad * v.precio_venta * (1 - v.descuento), 2)), 0) AS ingresos_producto
FROM STG_VENTAS v
RIGHT JOIN STG_PRODUCTOS p ON v.id_producto = p.id_producto
GROUP BY p.id_producto, p.nombre_producto, p.categoria
ORDER BY ingresos_producto DESC;
```

```sql
-- 8.4 FULL OUTER JOIN: Análisis de completitud (todos los clientes y todas las ventas)
SELECT
    COALESCE(c.id_cliente, v.id_cliente)    AS id_cliente,
    c.nombre,
    c.apellido,
    COUNT(v.id_venta)                       AS ventas_encontradas,
    IFF(c.id_cliente IS NULL, 'CLIENTE_FALTANTE',
        IFF(COUNT(v.id_venta) = 0, 'SIN_VENTAS', 'OK')
    ) AS estado
FROM STG_CLIENTES c
FULL OUTER JOIN STG_VENTAS v ON c.id_cliente = v.id_cliente
GROUP BY COALESCE(c.id_cliente, v.id_cliente), c.id_cliente, c.nombre, c.apellido
ORDER BY id_cliente;
```

```sql
-- 8.5 CROSS JOIN controlado: Combinaciones de segmento × canal (para análisis de cobertura)
SELECT
    seg.segmento,
    can.canal_venta,
    COALESCE(datos.num_ventas, 0)      AS num_ventas,
    COALESCE(datos.ingresos, 0)        AS ingresos
FROM (SELECT DISTINCT segmento FROM STG_CLIENTES) seg
CROSS JOIN (SELECT DISTINCT canal_venta FROM STG_VENTAS) can
LEFT JOIN (
    SELECT
        c.segmento,
        v.canal_venta,
        COUNT(*)                                              AS num_ventas,
        SUM(ROUND(v.cantidad * v.precio_venta * (1 - v.descuento), 2)) AS ingresos
    FROM STG_VENTAS v
    INNER JOIN STG_CLIENTES c ON v.id_cliente = c.id_cliente
    GROUP BY c.segmento, v.canal_venta
) datos ON seg.segmento = datos.segmento AND can.canal_venta = datos.canal_venta
ORDER BY seg.segmento, can.canal_venta;
```

**Resultado esperado:**
- 8.1: 20 filas con datos enriquecidos de las tres tablas.
- 8.2: Lista de clientes sin ventas (puede ser vacía si todos los 50 clientes tienen al menos una venta en los datos insertados; si es vacía, eso confirma cobertura completa).
- 8.3: 30 filas (una por producto), algunos con 0 unidades vendidas.
- 8.4: Todos los clientes con su estado.
- 8.5: Una matriz de 3 segmentos × 3 canales = 9 filas (o 12 si hay 4 canales).

---

### Paso 9: Validación de métricas de negocio

**Objetivo:** Calcular y documentar las 5 métricas clave que validan la integridad del modelo de datos cargado.

**Instrucciones:**

```sql
-- MÉTRICA 1: Total de ingresos del período 2023
SELECT
    ROUND(SUM(cantidad * precio_venta * (1 - descuento)), 2) AS total_ingresos_2023
FROM STG_VENTAS;
```

```sql
-- MÉTRICA 2: Top 5 clientes por monto total gastado
SELECT
    c.id_cliente,
    c.nombre || ' ' || c.apellido AS cliente,
    c.segmento,
    ROUND(SUM(v.cantidad * v.precio_venta * (1 - v.descuento)), 2) AS total_gastado
FROM STG_VENTAS v
INNER JOIN STG_CLIENTES c ON v.id_cliente = c.id_cliente
GROUP BY c.id_cliente, c.nombre, c.apellido, c.segmento
ORDER BY total_gastado DESC
LIMIT 5;
```

```sql
-- MÉTRICA 3: Producto más vendido (por ingresos)
SELECT
    p.id_producto,
    p.nombre_producto,
    p.categoria,
    ROUND(SUM(v.cantidad * v.precio_venta * (1 - v.descuento)), 2) AS ingresos_totales,
    SUM(v.cantidad) AS unidades_totales
FROM STG_VENTAS v
INNER JOIN STG_PRODUCTOS p ON v.id_producto = p.id_producto
GROUP BY p.id_producto, p.nombre_producto, p.categoria
ORDER BY ingresos_totales DESC
LIMIT 1;
```

```sql
-- MÉTRICA 4: Mes con mayor facturación
SELECT
    DATE_TRUNC('month', fecha_venta)::DATE AS mes,
    ROUND(SUM(cantidad * precio_venta * (1 - descuento)), 2) AS ingresos_mes
FROM STG_VENTAS
GROUP BY DATE_TRUNC('month', fecha_venta)
ORDER BY ingresos_mes DESC
LIMIT 1;
```

```sql
-- MÉTRICA 5: Tasa de clientes activos (activo = TRUE)
SELECT
    COUNT(*)                                     AS total_clientes,
    SUM(IFF(activo = TRUE, 1, 0))                AS clientes_activos,
    ROUND(SUM(IFF(activo = TRUE, 1, 0)) / COUNT(*) * 100, 1) AS pct_activos
FROM STG_CLIENTES;
```

**Resultado esperado:**

| Métrica | Valor esperado (aproximado) |
|---------|---------------------------|
| Total ingresos 2023 | ~$1,200,000 - $1,500,000 (depende del cálculo exacto de descuentos) |
| Top 1 cliente | Cliente id=1 (Carlos
