# Lab 07-00-01: Tipos de Datos LOB — Creación, Almacenamiento y Manipulación con DBMS_LOB

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 90 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 7 — Large Objects (LOBs) |
| **Versión Oracle** | 19c (19.3+) |

---

## Descripción General

En este laboratorio explorarás de forma práctica los cuatro tipos de datos LOB disponibles en Oracle Database: `BLOB`, `CLOB`, `NCLOB` y `BFILE`. Crearás tablas con columnas LOB configurando parámetros de almacenamiento avanzados, insertarás y recuperarás datos utilizando el paquete `DBMS_LOB`, y evaluarás las diferencias entre el almacenamiento BasicFiles y SecureFiles. Al finalizar, habrás construido un sistema de gestión documental simplificado que demuestra el uso real de LOBs en aplicaciones empresariales, comprendiendo cómo Oracle organiza internamente estos objetos de gran tamaño dentro de sus tablespaces.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Crear tablas Oracle con columnas de tipo `BLOB`, `CLOB` y `NCLOB` especificando cláusulas `LOB STORE AS` con parámetros de almacenamiento personalizados (`CHUNK`, `CACHE`, `STORAGE IN ROW`).
- [ ] Insertar, actualizar y recuperar datos LOB utilizando operaciones SQL estándar y el paquete `DBMS_LOB` (`WRITE`, `READ`, `APPEND`, `SUBSTR`, `GETLENGTH`).
- [ ] Implementar operaciones avanzadas sobre CLOBs incluyendo búsqueda de contenido con `DBMS_LOB.INSTR` y lectura parcial con `DBMS_LOB.SUBSTR`.
- [ ] Configurar objetos `DIRECTORY` de Oracle y utilizar `BFILENAME` con `DBMS_LOB.LOADFROMFILE` para cargar contenido desde el sistema de archivos.
- [ ] Comparar las características de almacenamiento BasicFiles vs SecureFiles y crear LOBs con deduplicación y compresión habilitadas.
- [ ] Consultar vistas del diccionario de datos para monitorear segmentos LOB y su consumo de espacio en tablespaces.

---

## Prerrequisitos

### Conocimiento Requerido

- Conocimiento de PL/SQL básico: bloques anónimos, variables, cursores y manejo de excepciones con `EXCEPTION WHEN`.
- Comprensión de tipos de datos Oracle estándar (`VARCHAR2`, `NUMBER`, `DATE`) y DDL para creación y modificación de tablas.
- Familiaridad con el concepto de tablespaces, segmentos y extents en Oracle Database.
- Conocimiento básico de cómo invocar procedimientos y funciones de un paquete PL/SQL (notación `PAQUETE.PROCEDIMIENTO`).
- Haber completado los laboratorios de los módulos anteriores (01 al 06) con la instancia Oracle operativa.

### Acceso Requerido

- Conexión a Oracle Database 19c como usuario `PRACTICA_USER` con privilegios `CREATE TABLE`, `CREATE DIRECTORY` (o que el DBA cree el directorio), `UNLIMITED TABLESPACE` o cuota en el tablespace de práctica.
- Acceso como `SYSDBA` (usuario `SYS`) para crear tablespaces dedicados y objetos `DIRECTORY`.
- Acceso SSH a la máquina virtual Oracle Linux donde reside la base de datos.
- SQL*Plus o SQL Developer disponible para ejecutar los scripts del laboratorio.

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | Intel Core i5 8va gen o AMD Ryzen 5 (mínimo) |
| RAM | 16 GB mínimo (recomendado 32 GB) |
| Almacenamiento | 100 GB libres en disco (SSD recomendado) |
| Red | Adaptador de red funcional con loopback |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) | Motor de base de datos principal |
| SQL*Plus | Incluido con 19c | Ejecución de scripts SQL y PL/SQL |
| SQL Developer | 23.1+ | Visualización de contenido LOB y resultados |
| Oracle Linux | 8.x | Sistema operativo huésped |
| PuTTY / Terminal SSH | 0.79+ / nativo | Acceso remoto a la VM |

### Configuración Inicial

Antes de comenzar el laboratorio, verifica que la instancia Oracle esté activa y establece las variables de entorno necesarias:

```bash
# Conectarse a la VM Oracle Linux via SSH
ssh oracle@192.168.56.101

# Verificar que la instancia esté activa
ps -ef | grep pmon | grep -v grep

# Establecer variables de entorno Oracle
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH

# Verificar conectividad a la base de datos
sqlplus -S / as sysdba <<EOF
SELECT instance_name, status FROM v\$instance;
EXIT;
EOF
```

```bash
# Crear el directorio del sistema de archivos para archivos LOB externos
mkdir -p /u01/app/oracle/lob_files
chmod 755 /u01/app/oracle/lob_files

# Crear archivos de prueba para el laboratorio
cat > /u01/app/oracle/lob_files/contrato_001.txt << 'ENDOFFILE'
CONTRATO DE SERVICIOS PROFESIONALES
=====================================
Entre las partes: Empresa ABC S.A. de C.V. (en adelante "El Cliente")
y Consultora XYZ (en adelante "El Proveedor").

PRIMERA CLAUSULA - OBJETO DEL CONTRATO:
El Proveedor se compromete a prestar servicios de consultoria tecnologica
incluyendo analisis, diseno e implementacion de soluciones de base de datos
Oracle para los sistemas internos del Cliente.

SEGUNDA CLAUSULA - DURACION:
El presente contrato tendra una vigencia de doce (12) meses contados
a partir de la fecha de firma del mismo.

TERCERA CLAUSULA - HONORARIOS:
El Cliente pagara al Proveedor la cantidad de $50,000 MXN mensuales
por los servicios descritos en la primera clausula.

CUARTA CLAUSULA - CONFIDENCIALIDAD:
Ambas partes se comprometen a mantener confidencialidad absoluta
sobre toda informacion tecnica y comercial intercambiada durante
la vigencia de este contrato y por un periodo de dos anios posterior.

Firmado en la Ciudad de Mexico a los 15 dias del mes de enero de 2024.
ENDOFFILE

cat > /u01/app/oracle/lob_files/manual_tecnico.txt << 'ENDOFFILE'
MANUAL TECNICO - SISTEMA DE GESTION DOCUMENTAL v2.0
=====================================================
Capitulo 1: Introduccion al Sistema
Este manual describe los procedimientos de instalacion, configuracion
y uso del Sistema de Gestion Documental basado en Oracle Database 19c.

Capitulo 2: Arquitectura del Sistema
El sistema utiliza una arquitectura de tres capas:
- Capa de Presentacion: Interfaz web desarrollada en Java EE
- Capa de Negocio: Servicios REST con Spring Boot
- Capa de Datos: Oracle Database 19c con almacenamiento LOB

Capitulo 3: Tipos de Documentos Soportados
- Contratos legales (CLOB - texto plano)
- Imagenes de productos (BLOB - binario)
- Manuales tecnicos (CLOB - texto extenso)
- Documentos PDF (BLOB - binario)
- Archivos de configuracion XML (CLOB - texto estructurado)

Capitulo 4: Procedimientos de Respaldo
Los LOBs internos se respaldan automaticamente con RMAN.
Los archivos BFILE requieren respaldo independiente del SO.
ENDOFFILE

echo "Archivos de prueba creados exitosamente."
ls -la /u01/app/oracle/lob_files/
```

---

## Instrucciones Paso a Paso

### Paso 1: Crear el Tablespace Dedicado para LOBs

**Objetivo:** Crear un tablespace separado específicamente para almacenar los segmentos LOB del laboratorio, siguiendo la mejor práctica de aislar los LOBs en su propio espacio de almacenamiento para facilitar la administración y el monitoreo.

**Instrucciones:**

1. Conéctate a SQL*Plus como `SYSDBA` para crear los objetos de infraestructura necesarios:

   ```bash
   sqlplus / as sysdba
   ```

2. Crea el tablespace dedicado para LOBs con las especificaciones apropiadas:

   ```sql
   -- Verificar espacio disponible antes de crear el tablespace
   SELECT
       name,
       ROUND(total_mb, 2) AS total_mb,
       ROUND(free_mb, 2)  AS free_mb
   FROM (
       SELECT
           tf.name,
           tf.bytes / 1024 / 1024 AS total_mb,
           (SELECT SUM(bytes) / 1024 / 1024
            FROM v$datafile
            WHERE file# = tf.file#) AS used_mb,
           0 AS free_mb
       FROM v$tempfile tf
   )
   UNION ALL
   SELECT
       tablespace_name,
       ROUND(SUM(bytes) / 1024 / 1024, 2),
       ROUND(SUM(DECODE(autoextensible, 'YES', maxbytes, bytes)) / 1024 / 1024, 2)
   FROM dba_data_files
   GROUP BY tablespace_name
   ORDER BY 1;
   ```

   ```sql
   -- Crear tablespace dedicado para segmentos LOB
   CREATE TABLESPACE lob_data_ts
       DATAFILE '/u01/app/oracle/oradata/ORCL/lob_data01.dbf'
       SIZE 200M
       AUTOEXTEND ON NEXT 50M MAXSIZE 2G
       EXTENT MANAGEMENT LOCAL
       SEGMENT SPACE MANAGEMENT AUTO;

   -- Verificar creación del tablespace
   SELECT
       tablespace_name,
       status,
       extent_management,
       segment_space_management,
       ROUND(bytes / 1024 / 1024, 2) AS size_mb
   FROM dba_tablespaces t
   JOIN dba_data_files d USING (tablespace_name)
   WHERE tablespace_name = 'LOB_DATA_TS';
   ```

3. Crea el objeto `DIRECTORY` de Oracle que apunta al directorio del sistema de archivos donde residen los archivos para `BFILE`:

   ```sql
   -- Crear el objeto DIRECTORY para acceso a archivos externos
   CREATE OR REPLACE DIRECTORY dir_lob_files AS '/u01/app/oracle/lob_files';

   -- Otorgar privilegios de lectura y escritura al usuario de práctica
   GRANT READ, WRITE ON DIRECTORY dir_lob_files TO practica_user;

   -- Verificar que el directorio fue creado
   SELECT
       directory_name,
       directory_path
   FROM dba_directories
   WHERE directory_name = 'DIR_LOB_FILES';
   ```

4. Otorga los privilegios necesarios al usuario de práctica:

   ```sql
   -- Otorgar cuota en el tablespace LOB
   ALTER USER practica_user QUOTA UNLIMITED ON lob_data_ts;

   -- Otorgar privilegio para crear directorios (opcional si el DBA los crea)
   GRANT CREATE ANY DIRECTORY TO practica_user;

   -- Confirmar privilegios
   SELECT
       privilege,
       admin_option
   FROM dba_sys_privs
   WHERE grantee = 'PRACTICA_USER'
   ORDER BY privilege;
   ```

**Resultado Esperado:**

```
Tablespace LOB_DATA_TS creado.

TABLESPACE_NAME    STATUS  EXTENT_MANAGEMENT  SEGMENT_SPACE_MANAGEMENT  SIZE_MB
-----------------  ------  -----------------  ------------------------  -------
LOB_DATA_TS        ONLINE  LOCAL              AUTO                       200

DIRECTORY_NAME     DIRECTORY_PATH
-----------------  ------------------------------------
DIR_LOB_FILES      /u01/app/oracle/lob_files
```

**Verificación:**

- Confirma que el tablespace `LOB_DATA_TS` aparece con estado `ONLINE`.
- Confirma que el directorio `DIR_LOB_FILES` existe en `DBA_DIRECTORIES`.
- Verifica que `PRACTICA_USER` tiene cuota ilimitada en `LOB_DATA_TS`.

---

### Paso 2: Crear Tablas con Columnas LOB y Parámetros de Almacenamiento

**Objetivo:** Crear las tablas del sistema de gestión documental con columnas `BLOB`, `CLOB`, `NCLOB` y `BFILE`, especificando cláusulas `LOB STORE AS` con diferentes parámetros de almacenamiento para comprender su impacto.

**Instrucciones:**

1. Conéctate como `PRACTICA_USER`:

   ```bash
   sqlplus practica_user/Oracle123@ORCL
   ```

2. Crea la tabla principal de documentos con columna `CLOB` usando almacenamiento BasicFiles con parámetros explícitos:

   ```sql
   -- Tabla de contratos con CLOB - BasicFiles con parámetros detallados
   CREATE TABLE lab07_contratos (
       id_contrato      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       numero_contrato  VARCHAR2(20)   NOT NULL,
       cliente          VARCHAR2(200)  NOT NULL,
       tipo_contrato    VARCHAR2(50),
       fecha_firma      DATE           DEFAULT SYSDATE,
       monto            NUMBER(15,2),
       texto_contrato   CLOB,
       notas_internas   NCLOB,
       estado           VARCHAR2(20)   DEFAULT 'ACTIVO'
   )
   LOB (texto_contrato) STORE AS BASICFILE lob_contratos_texto (
       TABLESPACE  lob_data_ts
       CHUNK       8192
       NOCACHE
       NOLOGGING
       ENABLE STORAGE IN ROW
   )
   LOB (notas_internas) STORE AS BASICFILE lob_contratos_notas (
       TABLESPACE  lob_data_ts
       CHUNK       4096
       CACHE
       LOGGING
       DISABLE STORAGE IN ROW
   );

   -- Verificar estructura de la tabla
   DESC lab07_contratos;
   ```

3. Crea la tabla de imágenes de productos con columna `BLOB` usando almacenamiento SecureFiles:

   ```sql
   -- Tabla de multimedia con BLOB - SecureFiles con compresión y deduplicación
   CREATE TABLE lab07_multimedia (
       id_archivo       NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       nombre_archivo   VARCHAR2(255)  NOT NULL,
       tipo_contenido   VARCHAR2(100),
       descripcion      VARCHAR2(500),
       fecha_carga      TIMESTAMP      DEFAULT SYSTIMESTAMP,
       tamano_bytes     NUMBER,
       contenido_blob   BLOB,
       miniatura_blob   BLOB
   )
   LOB (contenido_blob) STORE AS SECUREFILE sf_multimedia_contenido (
       TABLESPACE    lob_data_ts
       COMPRESS      MEDIUM
       DEDUPLICATE   LOB
       CACHE
       NOLOGGING
   )
   LOB (miniatura_blob) STORE AS SECUREFILE sf_multimedia_miniatura (
       TABLESPACE    lob_data_ts
       NOCOMPRESS
       NODEDUPLICATE
       NOCACHE
       LOGGING
   );

   -- Verificar estructura
   DESC lab07_multimedia;
   ```

4. Crea la tabla de manuales técnicos que utiliza `BFILE` para referenciar archivos externos:

   ```sql
   -- Tabla de manuales con BFILE (referencia externa)
   CREATE TABLE lab07_manuales_ext (
       id_manual        NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       codigo_manual    VARCHAR2(20)   NOT NULL,
       titulo           VARCHAR2(300)  NOT NULL,
       version          VARCHAR2(10),
       idioma           VARCHAR2(20)   DEFAULT 'ES',
       fecha_publicacion DATE,
       nombre_directorio VARCHAR2(128),
       nombre_archivo   VARCHAR2(255),
       archivo_externo  BFILE
   );

   -- Verificar las tres tablas creadas
   SELECT
       table_name,
       num_rows,
       last_analyzed
   FROM user_tables
   WHERE table_name LIKE 'LAB07%'
   ORDER BY table_name;
   ```

5. Consulta las vistas del diccionario para ver los segmentos LOB creados:

   ```sql
   -- Ver los segmentos LOB asociados a las tablas del laboratorio
   SELECT
       l.table_name,
       l.column_name,
       l.segment_name,
       l.tablespace_name,
       l.index_name,
       l.in_row,
       l.chunk,
       l.cache,
       l.logging,
       l.securefile
   FROM user_lobs l
   WHERE l.table_name LIKE 'LAB07%'
   ORDER BY l.table_name, l.column_name;
   ```

**Resultado Esperado:**

```
TABLE_NAME          COLUMN_NAME       SEGMENT_NAME              TABLESPACE   IN_ROW CHUNK  CACHE LOGGING SECUREFILE
------------------  ----------------  ------------------------  -----------  ------ -----  ----- ------- ----------
LAB07_CONTRATOS     NOTAS_INTERNAS    LOB_CONTRATOS_NOTAS       LOB_DATA_TS  NO     4096   YES   YES     NO
LAB07_CONTRATOS     TEXTO_CONTRATO    LOB_CONTRATOS_TEXTO       LOB_DATA_TS  YES    8192   NO    NO      NO
LAB07_MULTIMEDIA    CONTENIDO_BLOB    SF_MULTIMEDIA_CONTENIDO   LOB_DATA_TS  YES    8192   YES   NO      YES
LAB07_MULTIMEDIA    MINIATURA_BLOB    SF_MULTIMEDIA_MINIATURA   LOB_DATA_TS  YES    8192   NO    YES     YES
```

**Verificación:**

- Confirma que los segmentos LOB de BasicFiles muestran `SECUREFILE = NO`.
- Confirma que los segmentos LOB de SecureFiles muestran `SECUREFILE = YES`.
- Verifica que `LOB_CONTRATOS_NOTAS` tiene `IN_ROW = NO` (por `DISABLE STORAGE IN ROW`).
- Verifica que `LOB_CONTRATOS_TEXTO` tiene `IN_ROW = YES` (por `ENABLE STORAGE IN ROW`).

---

### Paso 3: Insertar Datos CLOB con SQL Estándar y DBMS_LOB

**Objetivo:** Practicar la inserción de datos de tipo `CLOB` utilizando tanto operaciones SQL directas como el paquete `DBMS_LOB`, comprendiendo cuándo usar cada enfoque según el tamaño del contenido.

**Instrucciones:**

1. Inserta un contrato con contenido CLOB directamente mediante SQL (adecuado para contenido menor a 32,767 bytes):

   ```sql
   -- Inserción directa con TO_CLOB para contenido moderado
   INSERT INTO lab07_contratos (
       numero_contrato,
       cliente,
       tipo_contrato,
       fecha_firma,
       monto,
       texto_contrato,
       notas_internas
   ) VALUES (
       'CONT-2024-001',
       'Empresa Tecnologica Alpha S.A.',
       'SERVICIOS_TI',
       TO_DATE('15/01/2024', 'DD/MM/YYYY'),
       75000.00,
       TO_CLOB(
           'CONTRATO DE SERVICIOS DE TECNOLOGIA DE INFORMACION' || CHR(10) ||
           '=============================================' || CHR(10) ||
           'PARTES CONTRATANTES:' || CHR(10) ||
           'Proveedor: Consultora Oracle Experts S.C.' || CHR(10) ||
           'Cliente: Empresa Tecnologica Alpha S.A.' || CHR(10) || CHR(10) ||
           'OBJETO: Implementacion y administracion de base de datos Oracle 19c' || CHR(10) ||
           'DURACION: 12 meses a partir del 15 de enero de 2024' || CHR(10) ||
           'MONTO MENSUAL: $75,000 MXN mas IVA' || CHR(10) || CHR(10) ||
           'CLAUSULAS ESPECIALES:' || CHR(10) ||
           '1. El proveedor garantiza disponibilidad 24x7 para incidentes criticos.' || CHR(10) ||
           '2. Se incluyen hasta 40 horas mensuales de consultoria presencial.' || CHR(10) ||
           '3. Todos los entregables seran documentados en Oracle Documentation.' || CHR(10) ||
           '4. El cliente proporcionara acceso VPN a sus instalaciones.' || CHR(10) || CHR(10) ||
           'Firmado digitalmente el 15 de enero de 2024.'
       ),
       TO_NCLOB(
           N'NOTAS CONFIDENCIALES - SOLO USO INTERNO' || CHR(10) ||
           N'Cliente con historial de pagos excelente.' || CHR(10) ||
           N'Contacto principal: Ing. María González - ext. 2341' || CHR(10) ||
           N'Renovacion probable al vencimiento: SI'
       )
   );

   COMMIT;

   -- Verificar la inserción
   SELECT
       id_contrato,
       numero_contrato,
       cliente,
       DBMS_LOB.GETLENGTH(texto_contrato)  AS longitud_contrato,
       DBMS_LOB.GETLENGTH(notas_internas)  AS longitud_notas
   FROM lab07_contratos
   WHERE numero_contrato = 'CONT-2024-001';
   ```

2. Inserta un segundo contrato usando un bloque PL/SQL con `DBMS_LOB` para construir el contenido de forma incremental:

   ```sql
   -- Inserción con PL/SQL y DBMS_LOB.WRITE para contenido construido dinámicamente
   DECLARE
       v_clob      CLOB;
       v_nclob     NCLOB;
       v_buffer    VARCHAR2(32767);
       v_offset    INTEGER := 1;
       i           INTEGER;
   BEGIN
       -- Inicializar el LOB temporal
       DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);
       DBMS_LOB.CREATETEMPORARY(v_nclob, TRUE);

       -- Escribir el encabezado del contrato
       v_buffer := 'CONTRATO MARCO DE COLABORACION EMPRESARIAL' || CHR(10);
       v_buffer := v_buffer || RPAD('=', 50, '=') || CHR(10);
       v_buffer := v_buffer || 'Numero: CONT-2024-002' || CHR(10);
       v_buffer := v_buffer || 'Fecha: ' || TO_CHAR(SYSDATE, 'DD/MM/YYYY') || CHR(10) || CHR(10);
       DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_buffer), v_buffer);

       -- Agregar cláusulas del contrato en un bucle
       FOR i IN 1..10 LOOP
           v_buffer := 'CLAUSULA ' || TO_CHAR(i) || ': ';
           CASE i
               WHEN 1  THEN v_buffer := v_buffer || 'OBJETO - Desarrollo de software a medida para gestion empresarial.';
               WHEN 2  THEN v_buffer := v_buffer || 'VIGENCIA - El contrato inicia el 01/02/2024 y vence el 31/01/2025.';
               WHEN 3  THEN v_buffer := v_buffer || 'PRECIO - $120,000 MXN mensuales pagaderos los primeros 5 dias.';
               WHEN 4  THEN v_buffer := v_buffer || 'ENTREGABLES - Reportes mensuales de avance y demos quincenales.';
               WHEN 5  THEN v_buffer := v_buffer || 'SOPORTE - Atencion de incidentes en maximo 4 horas habiles.';
               WHEN 6  THEN v_buffer := v_buffer || 'PROPIEDAD INTELECTUAL - El codigo desarrollado es propiedad del cliente.';
               WHEN 7  THEN v_buffer := v_buffer || 'CONFIDENCIALIDAD - Vigente durante el contrato y 3 anios posteriores.';
               WHEN 8  THEN v_buffer := v_buffer || 'RESCISION - Cualquier parte puede rescindir con 30 dias de aviso.';
               WHEN 9  THEN v_buffer := v_buffer || 'JURISDICCION - Tribunales de la Ciudad de Mexico, Mexico.';
               WHEN 10 THEN v_buffer := v_buffer || 'FIRMAS - Representantes legales de ambas partes con poder notarial.';
           END CASE;
           v_buffer := v_buffer || CHR(10);
           DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_buffer), v_buffer);
       END LOOP;

       -- Agregar pie de contrato
       v_buffer := CHR(10) || 'Firmado en Ciudad de Mexico a 01 de febrero de 2024.' || CHR(10);
       DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_buffer), v_buffer);

       -- Construir las notas internas en NCLOB
       v_buffer := 'Cliente nuevo - Referido por Empresa Alpha.' || CHR(10);
       v_buffer := v_buffer || 'Potencial de expansion a 3 proyectos adicionales.' || CHR(10);
       v_buffer := v_buffer || 'Presupuesto anual estimado: $2,000,000 MXN.';
       DBMS_LOB.WRITEAPPEND(v_nclob, LENGTH(v_buffer), v_buffer);

       -- Insertar el registro con los LOBs construidos
       INSERT INTO lab07_contratos (
           numero_contrato, cliente, tipo_contrato,
           fecha_firma, monto, texto_contrato, notas_internas
       ) VALUES (
           'CONT-2024-002',
           'Corporativo Beta Internacional',
           'DESARROLLO_SW',
           TO_DATE('01/02/2024', 'DD/MM/YYYY'),
           120000.00,
           v_clob,
           v_nclob
       );

       COMMIT;

       -- Liberar LOBs temporales
       DBMS_LOB.FREETEMPORARY(v_clob);
       DBMS_LOB.FREETEMPORARY(v_nclob);

       DBMS_OUTPUT.PUT_LINE('Contrato CONT-2024-002 insertado exitosamente.');
       DBMS_OUTPUT.PUT_LINE('Longitud del contrato: ' ||
           (SELECT DBMS_LOB.GETLENGTH(texto_contrato)
            FROM lab07_contratos WHERE numero_contrato = 'CONT-2024-002') || ' caracteres.');
   END;
   /
   ```

3. Verifica los datos insertados consultando la tabla:

   ```sql
   -- Resumen de contratos insertados
   SELECT
       id_contrato,
       numero_contrato,
       cliente,
       tipo_contrato,
       TO_CHAR(fecha_firma, 'DD/MM/YYYY')          AS fecha_firma,
       TO_CHAR(monto, '$999,999,990.00')            AS monto,
       DBMS_LOB.GETLENGTH(texto_contrato)           AS chars_contrato,
       DBMS_LOB.GETLENGTH(notas_internas)           AS chars_notas,
       estado
   FROM lab07_contratos
   ORDER BY id_contrato;
   ```

**Resultado Esperado:**

```
ID_CONTRATO  NUMERO_CONTRATO  CLIENTE                          TIPO_CONTRATO  FECHA_FIRMA  MONTO              CHARS_CONTRATO  CHARS_NOTAS  ESTADO
-----------  ---------------  -------------------------------  -------------  -----------  -----------------  --------------  -----------  ------
1            CONT-2024-001    Empresa Tecnologica Alpha S.A.   SERVICIOS_TI   15/01/2024   $75,000.00         (aprox 650)     (aprox 150)  ACTIVO
2            CONT-2024-002    Corporativo Beta Internacional   DESARROLLO_SW  01/02/2024   $120,000.00        (aprox 1,400)   (aprox 170)  ACTIVO

Contrato CONT-2024-002 insertado exitosamente.
Longitud del contrato: 1412 caracteres.
```

**Verificación:**

- Confirma que ambos contratos tienen valores no nulos en `CHARS_CONTRATO` y `CHARS_NOTAS`.
- Verifica que `CONT-2024-002` tiene mayor longitud de contrato que `CONT-2024-001` (más cláusulas).
- Comprueba que el bloque PL/SQL finalizó sin errores.

---

### Paso 4: Operaciones Avanzadas de Lectura y Búsqueda sobre CLOBs

**Objetivo:** Practicar las operaciones de lectura parcial, extracción de fragmentos y búsqueda de patrones dentro de columnas CLOB utilizando las funciones del paquete `DBMS_LOB`.

**Instrucciones:**

1. Lee los primeros 200 caracteres de un contrato usando `DBMS_LOB.SUBSTR`:

   ```sql
   -- Leer los primeros 200 caracteres del contrato 1
   SELECT
       numero_contrato,
       DBMS_LOB.SUBSTR(
           texto_contrato,   -- LOB a leer
           200,              -- cantidad de caracteres a extraer
           1                 -- posición de inicio (1 = inicio del LOB)
       ) AS primeros_200_chars
   FROM lab07_contratos
   WHERE numero_contrato = 'CONT-2024-001';
   ```

2. Extrae un fragmento del medio del contrato para simular una lectura parcial:

   ```sql
   -- Leer caracteres 100 al 300 del contrato 2 (lectura de fragmento intermedio)
   DECLARE
       v_clob      CLOB;
       v_buffer    VARCHAR2(32767);
       v_offset    INTEGER := 100;
       v_amount    INTEGER := 200;
       v_longitud  INTEGER;
   BEGIN
       -- Obtener el LOB de la base de datos
       SELECT texto_contrato
       INTO v_clob
       FROM lab07_contratos
       WHERE numero_contrato = 'CONT-2024-002';

       -- Obtener la longitud total
       v_longitud := DBMS_LOB.GETLENGTH(v_clob);
       DBMS_OUTPUT.PUT_LINE('Longitud total del contrato: ' || v_longitud || ' caracteres');

       -- Leer fragmento con DBMS_LOB.READ
       DBMS_LOB.READ(
           v_clob,     -- LOB fuente
           v_amount,   -- cantidad a leer (se actualiza con bytes realmente leídos)
           v_offset,   -- posición de inicio
           v_buffer    -- buffer donde se almacena el resultado
       );

       DBMS_OUTPUT.PUT_LINE('Caracteres leídos: ' || v_amount);
       DBMS_OUTPUT.PUT_LINE('Fragmento (pos 100-300):');
       DBMS_OUTPUT.PUT_LINE(v_buffer);
   END;
   /
   ```

3. Busca la posición de una palabra clave dentro del contrato usando `DBMS_LOB.INSTR`:

   ```sql
   -- Buscar la posición de palabras clave en los contratos
   SELECT
       numero_contrato,
       cliente,
       -- Buscar la primera ocurrencia de 'CLAUSULA'
       DBMS_LOB.INSTR(
           texto_contrato,
           'CLAUSULA',   -- patrón a buscar
           1,            -- posición de inicio de la búsqueda
           1             -- número de ocurrencia (1 = primera)
       ) AS pos_primera_clausula,
       -- Buscar la segunda ocurrencia de 'CLAUSULA'
       DBMS_LOB.INSTR(
           texto_contrato,
           'CLAUSULA',
           1,
           2             -- número de ocurrencia (2 = segunda)
       ) AS pos_segunda_clausula,
       -- Verificar si contiene la palabra 'CONFIDENCIALIDAD'
       CASE
           WHEN DBMS_LOB.INSTR(texto_contrato, 'CONFIDENCIALIDAD', 1, 1) > 0
           THEN 'SI'
           ELSE 'NO'
       END AS tiene_clausula_confidencialidad
   FROM lab07_contratos
   ORDER BY id_contrato;
   ```

4. Extrae el texto de una cláusula específica usando la combinación de `DBMS_LOB.INSTR` y `DBMS_LOB.SUBSTR`:

   ```sql
   -- Extraer el texto de la primera cláusula del contrato 2
   DECLARE
       v_clob          CLOB;
       v_pos_inicio    INTEGER;
       v_pos_fin       INTEGER;
       v_longitud      INTEGER;
       v_clausula      VARCHAR2(500);
   BEGIN
       SELECT texto_contrato
       INTO v_clob
       FROM lab07_contratos
       WHERE numero_contrato = 'CONT-2024-002';

       -- Encontrar inicio de CLAUSULA 1
       v_pos_inicio := DBMS_LOB.INSTR(v_clob, 'CLAUSULA 1', 1, 1);

       -- Encontrar inicio de CLAUSULA 2 (para delimitar el final de CLAUSULA 1)
       v_pos_fin := DBMS_LOB.INSTR(v_clob, 'CLAUSULA 2', 1, 1);

       IF v_pos_inicio > 0 THEN
           -- Calcular longitud del fragmento
           v_longitud := v_pos_fin - v_pos_inicio - 1;

           -- Extraer el texto de la cláusula
           v_clausula := DBMS_LOB.SUBSTR(v_clob, v_longitud, v_pos_inicio);

           DBMS_OUTPUT.PUT_LINE('Posicion inicio CLAUSULA 1: ' || v_pos_inicio);
           DBMS_OUTPUT.PUT_LINE('Posicion inicio CLAUSULA 2: ' || v_pos_fin);
           DBMS_OUTPUT.PUT_LINE('Longitud de CLAUSULA 1: ' || v_longitud || ' chars');
           DBMS_OUTPUT.PUT_LINE('Contenido: ' || v_clausula);
       ELSE
           DBMS_OUTPUT.PUT_LINE('No se encontro la clausula 1 en el contrato.');
       END IF;
   END;
   /
   ```

5. Actualiza el contenido de un CLOB existente agregando texto al final:

   ```sql
   -- Agregar una adenda al contrato 1 usando DBMS_LOB.APPEND
   DECLARE
       v_clob_original  CLOB;
       v_clob_adenda    CLOB;
       v_adenda_texto   VARCHAR2(500);
       v_longitud_antes INTEGER;
       v_longitud_despues INTEGER;
   BEGIN
       -- Seleccionar el LOB para actualización (FOR UPDATE bloquea la fila)
       SELECT texto_contrato
       INTO v_clob_original
       FROM lab07_contratos
       WHERE numero_contrato = 'CONT-2024-001'
       FOR UPDATE;

       v_longitud_antes := DBMS_LOB.GETLENGTH(v_clob_original);

       -- Crear el LOB temporal con el texto de la adenda
       DBMS_LOB.CREATETEMPORARY(v_clob_adenda, TRUE);

       v_adenda_texto :=
           CHR(10) || CHR(10) ||
           'ADENDA NO. 1 - MODIFICACION AL CONTRATO CONT-2024-001' || CHR(10) ||
           RPAD('-', 55, '-') || CHR(10) ||
           'Fecha de la adenda: ' || TO_CHAR(SYSDATE, 'DD/MM/YYYY') || CHR(10) ||
           'Se modifica la CLAUSULA 2 para extender la duracion del' || CHR(10) ||
           'contrato por 6 meses adicionales, hasta el 15/01/2026.' || CHR(10) ||
           'El monto mensual se incrementa a $80,000 MXN mas IVA.' || CHR(10) ||
           'Firmado por ambas partes de conformidad.';

       DBMS_LOB.WRITEAPPEND(v_clob_adenda, LENGTH(v_adenda_texto), v_adenda_texto);

       -- Agregar la adenda al contrato original
       DBMS_LOB.APPEND(v_clob_original, v_clob_adenda);

       v_longitud_despues := DBMS_LOB.GETLENGTH(v_clob_original);

       COMMIT;

       DBMS_LOB.FREETEMPORARY(v_clob_adenda);

       DBMS_OUTPUT.PUT_LINE('Adenda agregada exitosamente.');
       DBMS_OUTPUT.PUT_LINE('Longitud antes: ' || v_longitud_antes || ' chars');
       DBMS_OUTPUT.PUT_LINE('Longitud despues: ' || v_longitud_despues || ' chars');
       DBMS_OUTPUT.PUT_LINE('Caracteres agregados: ' || (v_longitud_despues - v_longitud_antes));
   END;
   /
   ```

**Resultado Esperado:**

```
-- Para la búsqueda de palabras clave:
NUMERO_CONTRATO  CLIENTE                          POS_PRIMERA_CLAUSULA  POS_SEGUNDA_CLAUSULA  TIENE_CLAUSULA_CONFIDENCIALIDAD
---------------  -------------------------------  --------------------  --------------------  --------------------------------
CONT-2024-001    Empresa Tecnologica Alpha S.A.   (posición > 0)        0                     SI
CONT-2024-002    Corporativo Beta Internacional   (posición > 0)        (posición > 0)        SI

-- Para la adenda:
Adenda agregada exitosamente.
Longitud antes: 650 chars
Longitud despues: 1050 chars
Caracteres agregados: 400
```

**Verificación:**

- Confirma que `DBMS_LOB.INSTR` retorna posiciones mayores a 0 cuando encuentra el patrón.
- Verifica que después de `DBMS_LOB.APPEND`, la longitud del contrato 1 aumentó.
- Comprueba que el bloque PL/SQL de extracción de cláusula imprime el contenido correcto.

---

### Paso 5: Trabajar con BLOBs — Carga desde Sistema de Archivos

**Objetivo:** Demostrar cómo cargar contenido binario desde archivos del sistema de archivos hacia columnas `BLOB` en la base de datos, utilizando `BFILENAME` y `DBMS_LOB.LOADFROMFILE`, y cómo trabajar con `BFILE` para referenciar archivos externos.

**Instrucciones:**

1. Primero, crea algunos archivos binarios de prueba en el servidor (simulando imágenes pequeñas):

   ```bash
   # Ejecutar en el shell de Oracle Linux como usuario oracle
   # Crear archivos binarios de prueba (simulando imágenes PNG con cabecera válida)

   # Crear un archivo que simula una imagen PNG (cabecera PNG + datos de relleno)
   python3 -c "
   import struct
   import zlib

   # Cabecera PNG válida (8 bytes)
   png_header = b'\x89PNG\r\n\x1a\n'

   # Chunk IHDR (imagen 100x100 RGB)
   ihdr_data = struct.pack('>IIBBBBB', 100, 100, 8, 2, 0, 0, 0)
   ihdr_crc = zlib.crc32(b'IHDR' + ihdr_data) & 0xffffffff
   ihdr_chunk = struct.pack('>I', 13) + b'IHDR' + ihdr_data + struct.pack('>I', ihdr_crc)

   # Chunk IDAT (datos de imagen - relleno simple)
   raw_data = b'\x00' + b'\xff\x00\x00' * 100  # línea roja
   raw_data = raw_data * 100
   compressed = zlib.compress(raw_data)
   idat_crc = zlib.crc32(b'IDAT' + compressed) & 0xffffffff
   idat_chunk = struct.pack('>I', len(compressed)) + b'IDAT' + compressed + struct.pack('>I', idat_crc)

   # Chunk IEND
   iend_crc = zlib.crc32(b'IEND') & 0xffffffff
   iend_chunk = struct.pack('>I', 0) + b'IEND' + struct.pack('>I', iend_crc)

   # Escribir archivo PNG
   with open('/u01/app/oracle/lob_files/imagen_producto_001.png', 'wb') as f:
       f.write(png_header + ihdr_chunk + idat_chunk + iend_chunk)

   print('Imagen PNG creada exitosamente')
   print('Tamanio:', len(png_header + ihdr_chunk + idat_chunk + iend_chunk), 'bytes')
   "

   # Crear un segundo archivo binario (simulando un PDF pequeño)
   python3 -c "
   pdf_content = b'%PDF-1.4\n'
   pdf_content += b'1 0 obj\n<< /Type /Catalog /Pages 2 0 R >>\nendobj\n'
   pdf_content += b'2 0 obj\n<< /Type /Pages /Kids [3 0 R] /Count 1 >>\nendobj\n'
   pdf_content += b'3 0 obj\n<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] >>\nendobj\n'
   pdf_content += b'xref\n0 4\n0000000000 65535 f\n'
   pdf_content += b'trailer\n<< /Size 4 /Root 1 0 R >>\nstartxref\n9\n%%EOF\n'

   with open('/u01/app/oracle/lob_files/manual_tecnico_v2.pdf', 'wb') as f:
       f.write(pdf_content)
   print('PDF creado exitosamente')
   print('Tamanio:', len(pdf_content), 'bytes')
   "

   # Verificar archivos creados
   ls -lh /u01/app/oracle/lob_files/
   ```

2. Carga el contenido de la imagen PNG al `BLOB` en la base de datos:

   ```sql
   -- Conectado como practica_user en SQL*Plus
   -- Cargar imagen PNG desde sistema de archivos a columna BLOB

   DECLARE
       v_blob          BLOB;
       v_bfile         BFILE;
       v_dest_offset   INTEGER := 1;
       v_src_offset    INTEGER := 1;
       v_tamano        INTEGER;
   BEGIN
       -- Inicializar el BLOB vacío en la tabla
       INSERT INTO lab07_multimedia (
           nombre_archivo,
           tipo_contenido,
           descripcion,
           tamano_bytes,
           contenido_blob
       ) VALUES (
           'imagen_producto_001.png',
           'image/png',
           'Imagen de producto - Servidor Oracle Database 19c',
           0,
           EMPTY_BLOB()  -- Inicializar con BLOB vacío
       )
       RETURNING contenido_blob INTO v_blob;

       -- Crear el localizador BFILE apuntando al archivo en disco
       v_bfile := BFILENAME('DIR_LOB_FILES', 'imagen_producto_001.png');

       -- Abrir el BFILE para lectura
       DBMS_LOB.OPEN(v_bfile, DBMS_LOB.LOB_READONLY);

       -- Obtener el tamaño del archivo
       v_tamano := DBMS_LOB.GETLENGTH(v_bfile);
       DBMS_OUTPUT.PUT_LINE('Tamano del archivo fuente: ' || v_tamano || ' bytes');

       -- Cargar el contenido del BFILE al BLOB
       DBMS_LOB.LOADBLOBFROMFILE(
           v_blob,         -- LOB destino (BLOB interno)
           v_bfile,        -- LOB fuente (BFILE externo)
           v_tamano,       -- cantidad de bytes a copiar
           v_dest_offset,  -- offset de destino
           v_src_offset    -- offset de fuente
       );

       -- Cerrar el BFILE
       DBMS_LOB.CLOSE(v_bfile);

       -- Actualizar el tamaño en la tabla
       UPDATE lab07_multimedia
       SET tamano_bytes = DBMS_LOB.GETLENGTH(contenido_blob)
       WHERE nombre_archivo = 'imagen_producto_001.png';

       COMMIT;

       DBMS_OUTPUT.PUT_LINE('Imagen cargada exitosamente al BLOB.');
       DBMS_OUTPUT.PUT_LINE('Tamano en BLOB: ' ||
           (SELECT DBMS_LOB.GETLENGTH(contenido_blob)
            FROM lab07_multimedia
            WHERE nombre_archivo = 'imagen_producto_001.png') || ' bytes');
   EXCEPTION
       WHEN OTHERS THEN
           IF DBMS_LOB.ISOPEN(v_bfile) = 1 THEN
               DBMS_LOB.CLOSE(v_bfile);
           END IF;
           ROLLBACK;
           DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
           RAISE;
   END;
   /
   ```

3. Inserta registros en la tabla de manuales externos usando `BFILE`:

   ```sql
   -- Insertar referencias BFILE a archivos externos (sin copiar el contenido)
   INSERT INTO lab07_manuales_ext (
       codigo_manual,
       titulo,
       version,
       idioma,
       fecha_publicacion,
       nombre_directorio,
       nombre_archivo,
       archivo_externo
   ) VALUES (
       'MAN-ORA-001',
       'Manual de Instalacion Oracle Database 19c',
       '2.0',
       'ES',
       TO_DATE('01/01/2024', 'DD/MM/YYYY'),
       'DIR_LOB_FILES',
       'manual_tecnico.txt',
       BFILENAME('DIR_LOB_FILES', 'manual_tecnico.txt')
   );

   INSERT INTO lab07_manuales_ext (
       codigo_manual,
       titulo,
       version,
       idioma,
       fecha_publicacion,
       nombre_directorio,
       nombre_archivo,
       archivo_externo
   ) VALUES (
       'MAN-ORA-002',
       'Manual Tecnico del Sistema de Gestion Documental v2.0',
       '2.0',
       'ES',
       TO_DATE('15/01/2024', 'DD/MM/YYYY'),
       'DIR_LOB_FILES',
       'manual_tecnico_v2.pdf',
       BFILENAME('DIR_LOB_FILES', 'manual_tecnico_v2.pdf')
   );

   COMMIT;

   -- Verificar los registros BFILE insertados
   SELECT
       id_manual,
       codigo_manual,
       titulo,
       version,
       nombre_archivo,
       -- Verificar si el archivo existe físicamente
       CASE
           WHEN DBMS_LOB.FILEEXISTS(archivo_externo) = 1 THEN 'EXISTE'
           ELSE 'NO EXISTE'
       END AS archivo_fisico,
       -- Obtener el tamaño del archivo externo
       CASE
           WHEN DBMS_LOB.FILEEXISTS(archivo_externo) = 1
           THEN DBMS_LOB.GETLENGTH(archivo_externo)
           ELSE NULL
       END AS tamano_bytes
   FROM lab07_manuales_ext
   ORDER BY id_manual;
   ```

4. Lee el contenido de un archivo `BFILE` directamente desde Oracle:

   ```sql
   -- Leer los primeros 500 bytes del manual técnico vía BFILE
   DECLARE
       v_bfile     BFILE;
       v_buffer    RAW(500);
       v_amount    INTEGER := 500;
       v_offset    INTEGER := 1;
   BEGIN
       SELECT archivo_externo
       INTO v_bfile
       FROM lab07_manuales_ext
       WHERE codigo_manual = 'MAN-ORA-001';

       -- Verificar existencia del archivo
       IF DBMS_LOB.FILEEXISTS(v_bfile) = 0 THEN
           DBMS_OUTPUT.PUT_LINE('ERROR: El archivo fisico no existe.');
           RETURN;
       END IF;

       -- Abrir el BFILE
       DBMS_LOB.OPEN(v_bfile, DBMS_LOB.LOB_READONLY);

       -- Leer los primeros 500 bytes como RAW
       DBMS_LOB.READ(v_bfile, v_amount, v_offset, v_buffer);

       -- Cerrar el BFILE
       DBMS_LOB.CLOSE(v_bfile);

       DBMS_OUTPUT.PUT_LINE('Bytes leidos del BFILE: ' || v_amount);
       DBMS_OUTPUT.PUT_LINE('Contenido (como texto):');
       -- Convertir RAW a VARCHAR2 para visualizar texto
       DBMS_OUTPUT.PUT_LINE(UTL_RAW.CAST_TO_VARCHAR2(v_buffer));
   EXCEPTION
       WHEN OTHERS THEN
           IF DBMS_LOB.ISOPEN(v_bfile) = 1 THEN
               DBMS_LOB.CLOSE(v_bfile);
           END IF;
           DBMS_OUTPUT.PUT_LINE('Error al leer BFILE: ' || SQLERRM);
   END;
   /
   ```

**Resultado Esperado:**

```
Tamano del archivo fuente: (bytes del PNG)
Imagen cargada exitosamente al BLOB.
Tamano en BLOB: (mismo tamaño) bytes

ID_MANUAL  CODIGO_MANUAL  TITULO                                         VERSION  NOMBRE_ARCHIVO          ARCHIVO_FISICO  TAMANO_BYTES
---------  -------------  ---------------------------------------------  -------  ----------------------  --------------  ------------
1          MAN-ORA-001    Manual de Instalacion Oracle Database 19c      2.0      manual_tecnico.txt      EXISTE          (bytes)
2          MAN-ORA-002    Manual Tecnico del Sistema de Gestion...       2.0      manual_tecnico_v2.pdf   EXISTE          (bytes)

Bytes leidos del BFILE: 500
Contenido (como texto):
MANUAL TECNICO - SISTEMA DE GESTION DOCUMENTAL v2.0
...
```

**Verificación:**

- Confirma que `DBMS_LOB.FILEEXISTS` retorna `1` (EXISTE) para ambos archivos.
- Verifica que el tamaño del BLOB cargado coincide con el tamaño del archivo fuente.
- Comprueba que la lectura del BFILE imprime el texto del archivo correctamente.

---

### Paso 6: Comparar BasicFiles vs SecureFiles y Monitorear Almacenamiento LOB

**Objetivo:** Analizar las diferencias entre los tipos de almacenamiento LOB BasicFiles y SecureFiles consultando las vistas del diccionario de datos, y monitorear el espacio consumido por los segmentos LOB en los tablespaces.

**Instrucciones:**

1. Crea una tabla adicional con SecureFiles para comparar directamente:

   ```sql
   -- Tabla de prueba con SecureFiles habilitando todas las características
   CREATE TABLE lab07_sf_demo (
       id          NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       descripcion VARCHAR2(100),
       contenido   CLOB
   )
   LOB (contenido) STORE AS SECUREFILE sf_demo_contenido (
       TABLESPACE  lob_data_ts
       COMPRESS    HIGH
       DEDUPLICATE LOB
       CACHE
       LOGGING
   );

   -- Insertar datos de prueba con contenido duplicado (para ver efecto de deduplicación)
   DECLARE
       v_texto_repetido CLOB;
       v_texto          VARCHAR2(4000);
       i                INTEGER;
   BEGIN
       -- Crear texto de 4000 caracteres
       v_texto := RPAD('Este es el contenido de prueba para demostrar deduplicacion en SecureFiles. ', 4000, 'X');

       FOR i IN 1..5 LOOP
           DBMS_LOB.CREATETEMPORARY(v_texto_repetido, TRUE);
           DBMS_LOB.WRITEAPPEND(v_texto_repetido, LENGTH(v_texto), v_texto);

           INSERT INTO lab07_sf_demo (descripcion, contenido)
           VALUES ('Registro duplicado #' || i, v_texto_repetido);

           DBMS_LOB.FREETEMPORARY(v_texto_repetido);
       END LOOP;

       COMMIT;
       DBMS_OUTPUT.PUT_LINE('5 registros con contenido identico insertados en SecureFiles.');
   END;
   /
   ```

2. Consulta las características de almacenamiento de todos los LOBs del laboratorio:

   ```sql
   -- Comparativa completa de LOBs BasicFiles vs SecureFiles
   SELECT
       l.table_name                    AS tabla,
       l.column_name                   AS columna,
       l.segment_name                  AS segmento_lob,
       l.securefile                    AS es_securefile,
       l.in_row                        AS almacen_en_fila,
       l.chunk                         AS chunk_bytes,
       l.cache                         AS cache,
       l.logging                       AS logging,
       l.encrypt                       AS encriptado,
       l.compress                      AS compresion,
       l.deduplication                 AS deduplicacion
   FROM user_lobs l
   WHERE l.table_name LIKE 'LAB07%'
   ORDER BY l.securefile, l.table_name, l.column_name;
   ```

3. Monitorea el espacio consumido por los segmentos LOB:

   ```sql
   -- Espacio consumido por segmentos LOB del laboratorio
   SELECT
       s.segment_name,
       s.segment_type,
       s.tablespace_name,
       s.extents,
       ROUND(s.bytes / 1024, 2)           AS size_kb,
       ROUND(s.bytes / 1024 / 1024, 2)    AS size_mb
   FROM user_segments s
   WHERE s.segment_name IN (
       SELECT segment_name FROM user_lobs
       WHERE table_name LIKE 'LAB07%'
   )
   OR s.segment_name IN (
       SELECT index_name FROM user_lobs
       WHERE table_name LIKE 'LAB07%'
   )
   ORDER BY s.bytes DESC;
   ```

4. Verifica el espacio total usado en el tablespace LOB:

   ```sql
   -- Espacio total del tablespace LOB_DATA_TS
   -- (Ejecutar como SYSDBA o con privilegios DBA)
   -- Si eres practica_user, usa la vista USER_SEGMENTS

   SELECT
       tablespace_name,
       ROUND(SUM(bytes) / 1024 / 1024, 2)  AS total_mb_asignado
   FROM user_segments
   WHERE tablespace_name = 'LOB_DATA_TS'
   GROUP BY tablespace_name;
   ```

5. Analiza las propiedades de los LOBs en las tablas del diccionario:

   ```sql
   -- Resumen ejecutivo de LOBs del laboratorio
   SELECT
       l.table_name,
       COUNT(*) AS num_columnas_lob,
       SUM(CASE WHEN l.securefile = 'YES' THEN 1 ELSE 0 END) AS securefile_count,
       SUM(CASE WHEN l.securefile = 'NO'  THEN 1 ELSE 0 END) AS basicfile_count,
       SUM(CASE WHEN l.cache = 'YES'      THEN 1 ELSE 0 END) AS con_cache,
       SUM(CASE WHEN l.in_row = 'YES'     THEN 1 ELSE 0 END) AS inline_storage
   FROM user_lobs l
   WHERE l.table_name LIKE 'LAB07%'
   GROUP BY l.table_name
   ORDER BY l.table_name;
   ```

**Resultado Esperado:**

```
-- Comparativa BasicFiles vs SecureFiles:
TABLA               COLUMNA           SEGMENTO                    ES_SECUREFILE  ALMACEN_EN_FILA  CHUNK  CACHE  LOGGING  ENCRIPTADO  COMPRESION  DEDUPLICACION
------------------  ----------------  --------------------------  -------------  ---------------  -----  -----  -------  ----------  ----------  -------------
LAB07_CONTRATOS     NOTAS_INTERNAS    LOB_CONTRATOS_NOTAS         NO             NO               4096   YES    YES      NONE        DISABLED    NO
LAB07_CONTRATOS     TEXTO_CONTRATO    LOB_CONTRATOS_TEXTO         NO             YES              8192   NO     NO       NONE        DISABLED    NO
LAB07_MULTIMEDIA    CONTENIDO_BLOB    SF_MULTIMEDIA_CONTENIDO     YES            YES              8192   YES    NO       NONE        MEDIUM      LOB
LAB07_MULTIMEDIA    MINIATURA_BLOB    SF_MULTIMEDIA_MINIATURA     YES            YES              8192   NO     YES      NONE        DISABLED    NONE
LAB07_SF_DEMO       CONTENIDO         SF_DEMO_CONTENIDO           YES            YES              8192   YES    YES      NONE        HIGH        LOB

-- Resumen ejecutivo:
TABLE_NAME          NUM_COLUMNAS_LOB  SECUREFILE_COUNT  BASICFILE_COUNT  CON_CACHE  INLINE_STORAGE
------------------  ----------------  ----------------  ---------------  ---------  --------------
LAB07_CONTRATOS     2                 0                 2                1          1
LAB07_MULTIMEDIA    2                 2                 0                1          2
LAB07_SF_DEMO       1                 1                 0                1          1
```

**Verificación:**

- Confirma que las tablas creadas con `BASICFILE` muestran `ES_SECUREFILE = NO`.
- Confirma que las tablas creadas con `SECUREFILE` muestran `ES_SECUREFILE = YES`.
- Verifica que `SF_DEMO_CONTENIDO` tiene `COMPRESION = HIGH` y `DEDUPLICACION = LOB`.

---

### Paso 7: Caso Práctico — Sistema de Gestión Documental

**Objetivo:** Integrar todos los conceptos aprendidos en este laboratorio construyendo un caso práctico completo de un sistema de gestión documental que utiliza múltiples tipos LOB, incluyendo procedimientos almacenados para la gestión de documentos.

**Instrucciones:**

1. Crea la tabla central del sistema de gestión documental:

   ```sql
   -- Tabla principal del sistema de gestión documental
   CREATE TABLE lab07_gestion_docs (
       id_documento     NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       codigo_doc       VARCHAR2(30)   NOT NULL UNIQUE,
       titulo           VARCHAR2(300)  NOT NULL,
       categoria        VARCHAR2(50),
       autor            VARCHAR2(100),
       fecha_creacion   TIMESTAMP      DEFAULT SYSTIMESTAMP,
       fecha_modificacion TIMESTAMP,
       version          NUMBER(5,2)    DEFAULT 1.0,
       estado           VARCHAR2(20)   DEFAULT 'BORRADOR',
       -- LOBs para diferentes tipos de contenido
       contenido_texto  CLOB,          -- Contenido textual del documento
       contenido_bin    BLOB,          -- Versión binaria (PDF, DOCX, etc.)
       metadatos_xml    CLOB,          -- Metadatos en formato XML
       archivo_original BFILE,         -- Referencia al archivo fuente original
       -- Metadata adicional
       tamano_texto     NUMBER,
       tamano_binario   NUMBER,
       palabras_clave   VARCHAR2(500)
   )
   LOB (contenido_texto) STORE AS SECUREFILE sf_gd_texto (
       TABLESPACE lob_data_ts COMPRESS MEDIUM CACHE LOGGING
   )
   LOB (contenido_bin) STORE AS SECUREFILE sf_gd_binario (
       TABLESPACE lob_data_ts COMPRESS HIGH DEDUPLICATE LOB NOCACHE NOLOGGING
   )
   LOB (metadatos_xml) STORE AS BASICFILE bf_gd_xml (
       TABLESPACE lob_data_ts CHUNK 4096 CACHE LOGGING ENABLE STORAGE IN ROW
   );
   ```

2. Crea un procedimiento almacenado para registrar documentos en el sistema:

   ```sql
   -- Procedimiento para registrar un nuevo documento
   CREATE OR REPLACE PROCEDURE lab07_registrar_documento (
       p_codigo        IN  VARCHAR2,
       p_titulo        IN  VARCHAR2,
       p_categoria     IN  VARCHAR2,
       p_autor         IN  VARCHAR2,
       p_contenido     IN  VARCHAR2,   -- Contenido como VARCHAR2 (se convierte a CLOB)
       p_palabras_clave IN VARCHAR2,
       p_id_resultado  OUT NUMBER
   )
   AS
       v_clob_contenido  CLOB;
       v_clob_xml        CLOB;
       v_xml_metadata    VARCHAR2(2000);
   BEGIN
       -- Construir metadatos XML
       v_xml_metadata :=
           '<?xml version="1.0" encoding="UTF-8"?>' || CHR(10) ||
           '<documento>' || CHR(10) ||
           '  <codigo>' || p_codigo || '</codigo>' || CHR(10) ||
           '  <titulo>' || p_titulo || '</titulo>' || CHR(10) ||
           '  <categoria>' || p_categoria || '</categoria>' || CHR(10) ||
           '  <autor>' || p_autor || '</autor>' || CHR(10) ||
           '  <fecha_creacion>' || TO_CHAR(SYSTIMESTAMP, 'YYYY-MM-DD"T"HH24:MI:SS') || '</fecha_creacion>' || CHR(10) ||
           '  <palabras_clave>' || p_palabras_clave || '</palabras_clave>' || CHR(10) ||
           '  <longitud_contenido>' || LENGTH(p_contenido) || '</longitud_contenido>' || CHR(10) ||
           '</documento>';

       -- Crear LOBs temporales
       DBMS_LOB.CREATETEMPORARY(v_clob_contenido, TRUE);
       DBMS_LOB.CREATETEMPORARY(v_clob_xml, TRUE);

       -- Escribir contenido
       DBMS_LOB.WRITEAPPEND(v_clob_contenido, LENGTH(p_contenido), p_contenido);
       DBMS_LOB.WRITEAPPEND(v_clob_xml, LENGTH(v_xml_metadata), v_xml_metadata);

       -- Insertar el documento
       INSERT INTO lab07_gestion_docs (
           codigo_doc, titulo, categoria, autor,
           contenido_texto, metadatos_xml, palabras_clave,
           tamano_texto, estado
       ) VALUES (
           p_codigo, p_titulo, p_categoria, p_autor,
           v_clob_contenido, v_clob_xml, p_palabras_clave,
           LENGTH(p_contenido), 'PUBLICADO'
       )
       RETURNING id_documento INTO p_id_resultado;

       COMMIT;

       -- Liberar LOBs temporales
       DBMS_LOB.FREETEMPORARY(v_clob_contenido);
       DBMS_LOB.FREETEMPORARY(v_clob_xml);

       DBMS_OUTPUT.PUT_LINE('Documento registrado con ID: ' || p_id_resultado);
   EXCEPTION
       WHEN DUP_VAL_ON_INDEX THEN
           ROLLBACK;
           RAISE_APPLICATION_ERROR(-20001, 'Ya existe un documento con el codigo: ' || p_codigo);
       WHEN OTHERS THEN
           ROLLBACK;
           IF DBMS_LOB.ISTEMPORARY(v_clob_contenido) = 1 THEN
               DBMS_LOB.FREETEMPORARY(v_clob_contenido);
           END IF;
           RAISE;
   END lab07_registrar_documento;
   /
   ```

3. Crea una función para buscar documentos por contenido:

   ```sql
   -- Función para buscar documentos que contengan una palabra clave
   CREATE OR REPLACE FUNCTION lab07_buscar_en_docs (
       p_termino_busqueda IN VARCHAR2
   )
   RETURN SYS_REFCURSOR
   AS
       v_cursor SYS_REFCURSOR;
   BEGIN
       OPEN v_cursor FOR
           SELECT
               id_documento,
               codigo_doc,
               titulo,
               categoria,
               autor,
               TO_CHAR(fecha_creacion, 'DD/MM/YYYY HH24:MI') AS fecha,
               DBMS_LOB.GETLENGTH(contenido_texto)  AS longitud_chars,
               -- Posición donde aparece el término buscado
               DBMS_LOB.INSTR(contenido_texto, p_termino_busqueda, 1, 1) AS posicion_termino,
               -- Extracto de 100 chars alrededor del término encontrado
               CASE
                   WHEN DBMS_LOB.INSTR(contenido_texto, p_termino_busqueda, 1, 1) > 0
                   THEN DBMS_LOB.SUBSTR(
                           contenido_texto,
                           100,
                           GREATEST(1, DBMS_LOB.INSTR(contenido_texto, p_termino_busqueda, 1, 1) - 20)
                        )
                   ELSE NULL
               END AS extracto_contexto,
               palabras_clave
           FROM lab07_gestion_docs
           WHERE DBMS_LOB.INSTR(contenido_texto, p_termino_busqueda, 1, 1) > 0
              OR INSTR(UPPER(palabras_clave), UPPER(p_termino_busqueda)) > 0
           ORDER BY id_documento;

       RETURN v_cursor;
   END lab07_buscar_en_docs;
   /
   ```

4. Prueba el sistema registrando varios documentos y realizando búsquedas:

   ```sql
   -- Registrar documentos de prueba usando el procedimiento
   DECLARE
       v_id NUMBER;
   BEGIN
       -- Documento 1: Política de seguridad
       lab07_registrar_documento(
           p_codigo         => 'POL-SEG-001',
           p_titulo         => 'Politica de Seguridad de la Informacion v3.0',
           p_categoria      => 'POLITICAS',
           p_autor          => 'Departamento de TI',
           p_contenido      =>
               'POLITICA DE SEGURIDAD DE LA INFORMACION' || CHR(10) ||
               '======================================' || CHR(10) ||
               'Esta politica establece los lineamientos para la proteccion de datos.' || CHR(10) ||
               'Todo empleado debe cumplir con las medidas de seguridad establecidas.' || CHR(10) ||
               'El acceso a sistemas criticos requiere autenticacion de dos factores.' || CHR(10) ||
               'Las contrasenas deben tener minimo 12 caracteres y renovarse cada 90 dias.' || CHR(10) ||
               'Queda prohibido el uso de dispositivos personales en redes corporativas.' || CHR(10) ||
               'Los incidentes de seguridad deben reportarse dentro de las 2 horas.' || CHR(10) ||
               'Esta politica es de cumplimiento obligatorio para todo el personal.',
           p_palabras_clave => 'seguridad, contrasenas, acceso, autenticacion, politica',
           p_id_resultado   => v_id
       );

       -- Documento 2: Procedimiento de respaldo
       lab07_registrar_documento(
           p_codigo         => 'PROC-DB-001',
           p_titulo         => 'Procedimiento de Respaldo de Base de Datos Oracle',
           p_categoria      => 'PROCEDIMIENTOS',
           p_autor          => 'DBA Team',
           p_contenido      =>
               'PROCEDIMIENTO DE RESPALDO CON ORACLE RMAN' || CHR(10) ||
               '==========================================' || CHR(10) ||
               '1. OBJETIVO: Garantizar la disponibilidad de los datos mediante respaldos.' || CHR(10) ||
               '2. ALCANCE: Todas las bases de datos Oracle en produccion.' || CHR(10) ||
               '3. FRECUENCIA: Respaldo completo semanal, incremental diario.' || CHR(10) ||
               '4. HERRAMIENTA: Oracle Recovery Manager (RMAN).' || CHR(10) ||
               '5. RETENCION: Los respaldos se conservan por 30 dias.' || CHR(10) ||
               '6. VERIFICACION: Se valida la integridad de cada respaldo con RMAN VALIDATE.' || CHR(10) ||
               '7. RESTAURACION: El RTO objetivo es de 4 horas para bases de datos criticas.' || CHR(10) ||
               '8. RESPONSABLE: El equipo DBA es responsable de ejecutar y monitorear.',
           p_palabras_clave => 'rman, respaldo, backup, oracle, recuperacion, dba',
           p_id_resultado   => v_id
       );

       -- Documento 3: Manual de usuario
       lab07_registrar_documento(
           p_codigo         => 'MAN-USR-001',
           p_titulo         => 'Manual de Usuario - Sistema de Gestion Documental',
           p_categoria      => 'MANUALES',
           p_autor          => 'Equipo de Desarrollo',
           p_contenido      =>
               'MANUAL DE USUARIO - SGD v1.0' || CHR(10) ||
               '=============================' || CHR(10) ||
               'Bienvenido al Sistema de Gestion Documental.' || CHR(10) ||
               'Este manual describe como utilizar todas las funcionalidades del sistema.' || CHR(10) ||
               'INICIO DE SESION: Use sus credenciales corporativas para acceder.' || CHR(10) ||
               'CARGA DE DOCUMENTOS: Seleccione Nuevo > Documento y complete el formulario.' || CHR(10) ||
               'BUSQUEDA: Use el campo de busqueda para encontrar documentos por titulo o contenido.' || CHR(10) ||
               'CATEGORIAS: Los documentos se organizan en Politicas, Procedimientos y Manuales.' || CHR(10) ||
               'VERSIONAMIENTO: Cada modificacion genera una nueva version del documento.' || CHR(10) ||
               'SOPORTE: Contacte a helpdesk@empresa.com para reportar problemas.',
           p_palabras_clave => 'manual, usuario, sistema, documentos, gestion, busqueda',
           p_id_resultado   => v_id
       );

       DBMS_OUTPUT.PUT_LINE('Todos los documentos registrados exitosamente.');
   END;
   /
   ```

5. Ejecuta búsquedas en el sistema documental:

   ```sql
   -- Buscar documentos que contengan 'seguridad'
   DECLARE
       v_cursor    SYS_REFCURSOR;
       v_id        NUMBER;
       v_codigo    VARCHAR2(30);
       v_titulo    VARCHAR2(300);
       v_categoria VARCHAR2(50);
       v_fecha     VARCHAR2(20);
       v_longitud  NUMBER;
       v_posicion  NUMBER;
       v_extracto  VARCHAR2(200);
       v_palabras  VARCHAR2(500);
   BEGIN
       DBMS_OUTPUT.PUT_LINE('=== RESULTADOS DE BUSQUEDA: "seguridad" ===');
       DBMS_OUTPUT.PUT_LINE(RPAD('-', 60, '-'));

       v_cursor := lab07_buscar_en_docs('seguridad');

       LOOP
           FETCH v_cursor INTO v_id, v_codigo, v_titulo, v_categoria,
                               v_fecha, v_longitud, v_posicion, v_extracto, v_palabras;
           EXIT WHEN v_cursor%NOTFOUND;

           DBMS_OUTPUT.PUT_LINE('ID: ' || v_id || ' | Codigo: ' || v_codigo);
           DBMS_OUTPUT.PUT_LINE('Titulo: ' || v_titulo);
           DBMS_OUTPUT.PUT_LINE('Categoria: ' || v_categoria || ' | Fecha: ' || v_fecha);
           DBMS_OUTPUT.PUT_LINE('Longitud: ' || v_longitud || ' chars | Posicion termino: ' || v_posicion);
           IF v_extracto IS NOT NULL THEN
               DBMS_OUTPUT.PUT_LINE('Extracto: ...' || v_extracto || '...');
           END IF;
           DBMS_OUTPUT.PUT_LINE(RPAD('-', 60, '-'));
       END LOOP;

       CLOSE v_cursor;

       -- Segunda búsqueda
       DBMS_OUTPUT.PUT_LINE(CHR(10) || '=== RESULTADOS DE BUSQUEDA: "RMAN" ===');
       DBMS_OUTPUT.PUT_LINE(RPAD('-', 60, '-'));

       v_cursor := lab07_buscar_en_docs('RMAN');

       LOOP
           FETCH v_cursor INTO v_id, v_codigo, v_titulo, v_categoria,
                               v_fecha, v_longitud, v_posicion, v_extracto, v_palabras;
           EXIT WHEN v_cursor%NOTFOUND;

           DBMS_OUTPUT.PUT_LINE('ID: ' || v_id || ' | Codigo: ' || v_codigo);
           DBMS_OUTPUT.PUT_LINE('Titulo: ' || v_titulo);
           DBMS_OUTPUT.PUT_LINE('Extracto: ...' || NVL(v_extracto, '(en palabras clave)') || '...');
           DBMS_OUTPUT.PUT_LINE(RPAD('-', 60, '-'));
       END LOOP;

       CLOSE v_cursor;
   END;
   /
   ```

**Resultado Esperado:**

```
Documento registrado con ID: 1
Documento registrado con ID: 2
Documento registrado con ID: 3
Todos los documentos registrados exitosamente.

=== RESULTADOS DE BUSQUEDA: "seguridad" ===
------------------------------------------------------------
ID: 1 | Codigo: POL-SEG-001
Titulo: Politica de Seguridad de la Informacion v3.0
Categoria: POLITICAS | Fecha: (fecha actual)
Longitud: (chars) | Posicion termino: (posición)
Extracto: ...Esta politica establece los lineamientos para la proteccion...
------------------------------------------------------------

=== RESULTADOS DE BUSQUEDA: "RMAN" ===
------------------------------------------------------------
ID: 2 | Codigo: PROC-DB-001
Titulo: Procedimiento de Respaldo de Base de Datos Oracle
Extracto: ...4. HERRAMIENTA: Oracle Recovery Manager (RMAN)...
------------------------------------------------------------
```

**Verificación:**

- Confirma que el procedimiento `lab07_registrar_documento` se ejecuta sin errores.
- Verifica que la función `lab07_buscar_en_docs` retorna los documentos correctos para cada término.
- Comprueba que el extracto de contexto muestra texto alrededor del término buscado.

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El tablespace `LOB_DATA_TS` fue creado exitosamente y está en estado `ONLINE`.
- [ ] El objeto `DIRECTORY` `DIR_LOB_FILES` existe y apunta al directorio correcto del sistema de archivos.
- [ ] Las tablas `LAB07_CONTRATOS`, `LAB07_MULTIMEDIA`, `LAB07_MANUALES_EXT`, `LAB07_SF_DEMO` y `LAB07_GESTION_DOCS` fueron creadas con los segmentos LOB en el tablespace `LOB_DATA_TS`.
- [ ] Los contratos `CONT-2024-001` y `CONT-2024-002` tienen contenido CLOB con longitudes mayores a 500 caracteres.
- [ ] La adenda fue agregada al contrato `CONT-2024-001` incrementando su longitud.
- [ ] El archivo PNG fue cargado exitosamente al BLOB con `DBMS_LOB.LOADBLOBFROMFILE`.
- [ ] Los archivos BFILE existen físicamente y `DBMS_LOB.FILEEXISTS` retorna `1`.
- [ ] Los segmentos SecureFiles muestran `SECUREFILE = YES` en `USER_LOBS`.
- [ ] El sistema de gestión documental registró 3 documentos y las búsquedas retornan resultados correctos.

### Procedimiento de Prueba

1. Verifica el estado completo de todas las tablas del laboratorio:

   ```sql
   -- Prueba integral: resumen del laboratorio
   SELECT
       'LAB07_CONTRATOS'   AS tabla,
       COUNT(*)            AS registros,
       SUM(DBMS_LOB.GETLENGTH(texto_contrato)) AS total_chars_clob
   FROM lab07_contratos
   UNION ALL
   SELECT
       'LAB07_MULTIMEDIA',
       COUNT(*),
       SUM(DBMS_LOB.GETLENGTH(contenido_blob))
   FROM lab07_multimedia
   UNION ALL
   SELECT
       'LAB07_MANUALES_EXT',
       COUNT(*),
       NULL
   FROM lab07_manuales_ext
   UNION ALL
   SELECT
       'LAB07_GESTION_DOCS',
       COUNT(*),
       SUM(DBMS_LOB.GETLENGTH(contenido_texto))
   FROM lab07_gestion_docs;
   ```

   **Resultado Esperado:**
   ```
   TABLA                REGISTROS  TOTAL_CHARS_CLOB
   -------------------  ---------  ----------------
   LAB07_CONTRATOS      2          (> 2000 chars)
   LAB07_MULTIMEDIA     1          (bytes del PNG)
   LAB07_MANUALES_EXT   2          (null - BFILE)
   LAB07_GESTION_DOCS   3          (> 1500 chars)
   ```

2. Valida las propiedades de los segmentos LOB:

   ```sql
   -- Validación: contar LOBs por tipo de almacenamiento
   SELECT
       securefile                      AS tipo_almacenamiento,
       COUNT(*)                        AS cantidad_lobs,
       COUNT(DISTINCT table_name)      AS tablas_afectadas
   FROM user_lobs
   WHERE table_name LIKE 'LAB07%'
   GROUP BY securefile
   ORDER BY securefile;
   ```

   **Resultado Esperado:**
   ```
   TIPO_ALMACENAMIENTO  CANTIDAD_LOBS  TABLAS_AFECTADAS
   -------------------  -------------  ----------------
   NO                   3              2
   YES                  5              3
   ```

3. Verifica la funcionalidad de búsqueda en el sistema documental:

   ```sql
   -- Prueba de búsqueda: verificar que DBMS_LOB.INSTR funciona correctamente
   SELECT
       codigo_doc,
       titulo,
       CASE
           WHEN DBMS_LOB.INSTR(contenido_texto, 'RMAN', 1, 1) > 0
           THEN 'ENCONTRADO en posicion ' ||
                DBMS_LOB.INSTR(contenido_texto, 'RMAN', 1, 1)
           ELSE 'NO ENCONTRADO'
       END AS busqueda_rman,
       CASE
           WHEN DBMS_LOB.INSTR(contenido_texto, 'seguridad', 1, 1) > 0
           THEN 'ENCONTRADO en posicion ' ||
                DBMS_LOB.INSTR(contenido_texto, 'seguridad', 1, 1)
           ELSE 'NO ENCONTRADO'
       END AS busqueda_seguridad
   FROM lab07_gestion_docs
   ORDER BY id_documento;
   ```

   **Resultado Esperado:**
   ```
   CODIGO_DOC   TITULO                                          BUSQUEDA_RMAN                BUSQUEDA_SEGURIDAD
   -----------  ----------------------------------------------  ---------------------------  ---------------------
   POL-SEG-001  Politica de Seguridad de la Informacion v3.0   NO ENCONTRADO                ENCONTRADO en posicion X
   PROC-DB-001  Procedimiento de Respaldo de Base de Datos...  ENCONTRADO en posicion X     NO ENCONTRADO
   MAN-USR-001  Manual de Usuario - Sistema de Gestion...      NO ENCONTRADO                NO ENCONTRADO
   ```

---

## Solución de Problemas

### Problema 1: ORA-22285 — No se puede abrir el BFILE porque el archivo no existe

**Síntomas:**
- Al ejecutar `DBMS_LOB.OPEN(v_bfile, ...)` se recibe el error `ORA-22285: non-existent directory or file for FILEOPEN operation`.
- `DBMS_LOB.FILEEXISTS(archivo_externo)` retorna `0`.

**Causa:**
El archivo físico no existe en la ruta especificada en el objeto `DIRECTORY`, o el objeto `DIRECTORY` apunta a una ruta incorrecta, o el usuario del sistema operativo `oracle` no tiene permisos de lectura sobre el directorio.

**Solución:**
```bash
# 1. Verificar que el archivo existe en el sistema de archivos
ls -la /u01/app/oracle/lob_files/

# 2. Verificar permisos del directorio y archivos
ls -ld /u01/app/oracle/lob_files/
stat /u01/app/oracle/lob_files/manual_tecnico.txt

# 3. Corregir permisos si es necesario
chmod 755 /u01/app/oracle/lob_files/
chmod 644 /u01/app/oracle/lob_files/*.txt
chmod 644 /u01/app/oracle/lob_files/*.pdf

# 4. Verificar que el proceso Oracle puede leer el archivo
sudo -u oracle ls -la /u01/app/oracle/lob_files/
```

```sql
-- 5. Verificar la ruta del objeto DIRECTORY en la base de datos
SELECT directory_name, directory_path
FROM dba_directories
WHERE directory_name = 'DIR_LOB_FILES';

-- 6. Si la ruta es incorrecta, recrear el directorio
CREATE OR REPLACE DIRECTORY dir_lob_files AS '/u01/app/oracle/lob_files';

-- 7. Verificar existencia del archivo desde SQL
DECLARE
    v_bfile BFILE := BFILENAME('DIR_LOB_FILES', 'manual_tecnico.txt');
BEGIN
    DBMS_OUTPUT.PUT_LINE('Archivo existe: ' || DBMS_LOB.FILEEXISTS(v_bfile));
END;
/
```

---

### Problema 2: ORA-01536 — Se excedió la cuota de espacio en el tablespace

**Síntomas:**
- Al insertar datos LOB se recibe `ORA-01536: space quota exceeded for tablespace 'LOB_DATA_TS'`.
- Las inserciones fallan después de cargar varios archivos grandes.

**Causa:**
El usuario `PRACTICA_USER` no tiene cuota suficiente asignada en el tablespace `LOB_DATA_TS`, o el tablespace se llenó y no puede extenderse automáticamente.

**Solución:**
```sql
-- Conectarse como SYSDBA para verificar y corregir

-- 1. Verificar cuota actual del usuario
SELECT
    username,
    tablespace_name,
    bytes_used,
    bytes,
    max_bytes
FROM dba_ts_quotas
WHERE username = 'PRACTICA_USER';

-- 2. Ampliar la cuota del usuario
ALTER USER practica_user QUOTA UNLIMITED ON lob_data_ts;

-- 3. Verificar espacio libre en el tablespace
SELECT
    tablespace_name,
    ROUND(SUM(bytes) / 1024 / 1024, 2) AS free_mb
FROM dba_free_space
WHERE tablespace_name = 'LOB_DATA_TS'
GROUP BY tablespace_name;

-- 4. Si el tablespace está lleno, agregar un datafile
ALTER TABLESPACE lob_data_ts
ADD DATAFILE '/u01/app/oracle/oradata/ORCL/lob_data02.dbf'
SIZE 200M AUTOEXTEND ON NEXT 50M MAXSIZE 2G;

-- 5. Verificar el nuevo datafile
SELECT file_name, bytes/1024/1024 AS size_mb, autoextensible
FROM dba_data_files
WHERE tablespace_name = 'LOB_DATA_TS';
```

---

### Problema 3: ORA-22275 — Localizador LOB inválido o nulo al usar DBMS_LOB

**Síntomas:**
- Al ejecutar operaciones `DBMS_LOB.READ` o `DBMS_LOB.WRITE` se recibe `ORA-22275: invalid LOB locator specified` o `ORA-06502: PL/SQL: numeric or value error`.
- El LOB aparece como `NULL` en la consulta aunque se insertó un valor.

**Causa:**
El LOB no fue inicializado correctamente antes de intentar escribir en él. Para escribir en un LOB interno, la columna debe contener `EMPTY_BLOB()` o `EMPTY_CLOB()` (no `NULL`), y el registro debe haber sido insertado y la fila bloqueada con `SELECT ... FOR UPDATE ... RETURNING ... INTO`.

**Solución:**
```sql
-- Patrón CORRECTO para insertar y escribir en un LOB interno:
DECLARE
    v_clob  CLOB;
    v_texto VARCHAR2(500) := 'Contenido del documento de prueba.';
BEGIN
    -- PASO 1: Insertar fila con LOB vacío (NO NULL)
    INSERT INTO lab07_contratos (
        numero_contrato, cliente, texto_contrato
    ) VALUES (
        'CONT-TEST-001', 'Cliente Prueba', EMPTY_CLOB()
    )
    RETURNING texto_contrato INTO v_clob;  -- Obtener el localizador

    -- PASO 2: Ahora el localizador v_clob es válido, escribir contenido
    DBMS_LOB.WRITEAPPEND(v_clob, LENGTH(v_texto), v_texto);

    COMMIT;
    DBMS_OUTPUT.PUT_LINE('LOB escrito correctamente. Longitud: ' ||
        DBMS_LOB.GETLENGTH(v_clob));
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/

-- INCORRECTO: No hacer esto (el localizador no es válido fuera de la transacción)
-- SELECT texto_contrato INTO v_clob FROM tabla WHERE id = 1;
-- DBMS_LOB.WRITE(v_clob, ...);  -- ERROR: necesita FOR UPDATE
```

---

### Problema 4: ORA-22288 — Error al ejecutar DBMS_LOB.LOADBLOBFROMFILE

**Síntomas:**
- `DBMS_LOB.LOADBLOBFROMFILE` falla con `ORA-22288: file or LOB operation FILEOPEN failed` o `ORA-22289: cannot perform operation on an unopened file or LOB`.
- El procedimiento de carga de archivos al BLOB no completa.

**Causa:**
El `BFILE` no fue abierto antes de intentar leer de él, o se está usando `DBMS_LOB.LOADFROMFILE` en lugar de `DBMS_LOB.LOADBLOBFROMFILE` (la versión correcta para Oracle 10g+), o el BLOB destino es `NULL` en lugar de `EMPTY_BLOB()`.

**Solución:**
```sql
-- Verificar y corregir el procedimiento de carga BLOB
DECLARE
    v_blob          BLOB;
    v_bfile         BFILE;
    v_dest_offset   INTEGER := 1;
    v_src_offset    INTEGER := 1;
    v_tamano        INTEGER;
BEGIN
    -- Asegurar que el BLOB destino está inicializado con EMPTY_BLOB()
    INSERT INTO lab07_multimedia (nombre_archivo, tipo_contenido, contenido_blob)
    VALUES ('test.png', 'image/png', EMPTY_BLOB())
    RETURNING contenido_blob INTO v_blob;

    -- Crear el localizador BFILE
    v_bfile := BFILENAME('DIR_LOB_FILES', 'imagen_producto_001.png');

    -- VERIFICAR existencia antes de abrir
    IF DBMS_LOB.FILEEXISTS(v_bfile) = 0 THEN
        RAISE_APPLICATION_ERROR(-20002, 'Archivo no encontrado en el directorio.');
    END IF;

    -- ABRIR el BFILE ANTES de llamar a LOADBLOBFROMFILE
    DBMS_LOB.OPEN(v_bfile, DBMS_LOB.LOB_READONLY);

    v_tamano := DBMS_LOB.GETLENGTH(v_bfile);

    -- Usar LOADBLOBFROMFILE (no LOADFROMFILE que es obsoleto)
    DBMS_LOB.LOADBLOBFROMFILE(v_blob, v_bfile, v_tamano, v_dest_offset, v_src_offset);

    -- CERRAR el BFILE después de usarlo
    DBMS_LOB.CLOSE(v_bfile);

    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Carga exitosa. Bytes: ' || DBMS_LOB.GETLENGTH(v_blob));
EXCEPTION
    WHEN OTHERS THEN
        IF DBMS_LOB.ISOPEN(v_bfile) = 1 THEN
            DBMS_LOB.CLOSE(v_bfile);
        END IF;
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

---

## Limpieza

```sql
-- ============================================================
-- SCRIPT DE LIMPIEZA DEL LABORATORIO 07-00-01
-- Ejecutar como PRACTICA_USER primero, luego como SYSDBA
-- ============================================================

-- PASO 1: Conectado como PRACTICA_USER - Eliminar objetos del esquema
-- Eliminar procedimientos y funciones
DROP PROCEDURE lab07_registrar_documento;
DROP FUNCTION  lab07_buscar_en_docs;

-- Eliminar tablas (los segmentos LOB se eliminan automáticamente)
DROP TABLE lab07_gestion_docs   PURGE;
DROP TABLE lab07_sf_demo        PURGE;
DROP TABLE lab07_manuales_ext   PURGE;
DROP TABLE lab07_multimedia     PURGE;
DROP TABLE lab07_contratos      PURGE;

-- Verificar que no quedan objetos del laboratorio
SELECT object_name, object_type
FROM user_objects
WHERE object_name LIKE 'LAB07%'
ORDER BY object_type, object_name;
```

```sql
-- PASO 2: Conectado como SYSDBA - Eliminar objetos de infraestructura
-- Revocar privilegios del usuario de práctica
REVOKE READ, WRITE ON DIRECTORY dir_lob_files FROM practica_user;

-- Eliminar el objeto DIRECTORY
DROP DIRECTORY dir_lob_files;

-- Eliminar el tablespace y sus datafiles
DROP TABLESPACE lob_data_ts INCLUDING CONTENTS AND DATAFILES;

-- Verificar que el tablespace fue eliminado
SELECT tablespace_name FROM dba_tablespaces
WHERE tablespace_name = 'LOB_DATA_TS';
```

```bash
# PASO 3: Limpieza del sistema de archivos (como usuario oracle en Linux)
# Eliminar los archivos de prueba creados durante el laboratorio
rm -f /u01/app/oracle/lob_files/contrato_001.txt
rm -f /u01/app/oracle/lob_files/manual_tecnico.txt
rm -f /u01/app/oracle/lob_files/manual_tecnico_v2.pdf
rm -f /u01/app/oracle/lob_files/imagen_producto_001.png

# Verificar que el directorio está vacío
ls -la /u01/app/oracle/lob_files/

# Opcionalmente eliminar el directorio
rmdir /u01/app/oracle/lob_files/

echo "Limpieza del laboratorio completada."
```

> ⚠️ **Advertencia:** La instrucción `DROP TABLESPACE ... INCLUDING CONTENTS AND DATAFILES` elimina permanentemente el datafile del disco. Asegúrate de haber completado todas las prácticas y exportado cualquier dato que necesites conservar antes de ejecutar este comando. Si el tablespace contiene datos de otros laboratorios, omite este paso y solo elimina las tablas del laboratorio.

> ⚠️ **Nota:** Si planeas continuar con el Laboratorio 07-00-02 (Almacenamiento y Gestión de BLOBs), considera mantener el tablespace `LOB_DATA_TS` y el directorio `DIR_LOB_FILES` ya que serán reutilizados en la siguiente práctica.

---

## Resumen

### Lo que Lograste

- Creaste un tablespace dedicado `LOB_DATA_TS` para aislar los segmentos LOB y configuraste un objeto `DIRECTORY` de Oracle para acceso controlado a archivos del sistema de archivos.
- Diseñaste y creaste cinco tablas con columnas de tipo `BLOB`, `CLOB`, `NCLOB` y `BFILE`, especificando cláusulas `LOB STORE AS` con parámetros explícitos de almacenamiento (`CHUNK`, `CACHE`, `LOGGING`, `ENABLE/DISABLE STORAGE IN ROW`).
- Insertaste datos CLOB utilizando tanto `TO_CLOB()` en SQL directo como el paquete `DBMS_LOB` con `CREATETEMPORARY`, `WRITEAPPEND` y `APPEND` para construcción incremental de contenido.
- Ejecutaste operaciones avanzadas de lectura con `DBMS_LOB.READ`, `DBMS_LOB.SUBSTR` y búsqueda de patrones con `DBMS_LOB.INSTR` para localizar y extraer fragmentos de texto dentro de CLOBs.
- Cargaste contenido binario desde archivos del sistema de archivos a columnas `BLOB` usando `DBMS_LOB.LOADBLOBFROMFILE`, y referenciaste archivos externos con `BFILE` y `BFILENAME`.
- Comparaste las características de BasicFiles y SecureFiles consultando la vista `USER_LOBS` y verificando propiedades como compresión y deduplicación.
- Construiste un sistema de gestión documental funcional con procedimientos almacenados que encapsulan la lógica de inserción y búsqueda sobre LOBs.

### Conceptos Clave Aprendidos

- **Localizador LOB:** Oracle almacena un puntero de 20 bytes en la fila de la tabla; el contenido real reside en un segmento LOB separado en el tablespace. Para LOBs pequeños (< ~4,000 bytes), Oracle puede almacenarlos directamente en línea (`ENABLE STORAGE IN ROW`).
- **EMPTY_BLOB() / EMPTY_CLOB():** Para escribir en un LOB interno mediante `DBMS_LOB`, la columna debe ser inicializada con estos constructores (no con `NULL`), y el localizador debe obtenerse en la misma transacción con `RETURNING ... INTO`.
- **BasicFiles vs SecureFiles:** SecureFiles es el formato moderno (Oracle 11g+) que soporta compresión, deduplicación y encriptación transparente. BasicFiles es el formato legacy sin estas características pero con mayor compatibilidad.
- **BFILE — Solo lectura:** Oracle no puede modificar archivos externos; `BFILE` es estrictamente de lectura. El archivo debe existir físicamente y ser accesible cuando se intenta leer. No participa en transacciones ACID ni en respaldos RMAN automáticos.
- **DBMS_LOB es el paquete central:** Todas las operaciones programáticas sobre LOBs (leer, escribir, buscar, comparar, cargar) se realizan a través de `DBMS_LOB`. Familiarizarse con sus procedimientos y funciones es fundamental para el desarrollo con LOBs.

### Próximos Pasos

- Continúa con el **Laboratorio 07-00-02: Almacenamiento y Gestión Avanzada de BLOBs**, donde profundizarás en la configuración de parámetros de almacenamiento LOB, la migración entre BasicFiles y SecureFiles, y el monitoreo detallado del espacio consumido por segmentos LOB.
- Explora el uso de **Oracle Text** (`DBMS_TEXT`) para realizar búsquedas de texto completo sobre columnas CLOB con índices de texto, superando las limitaciones de `DBMS_LOB.INSTR` en documentos de gran tamaño.
- Investiga las opciones de **encriptación transparente de LOBs** con `ENCRYPT USING AES256` en SecureFiles para proteger datos sensibles almacenados como LOBs.

---

## Recursos Adicionales

- **Oracle Database SecureFiles and Large Objects Developer's Guide 19c** — Documentación oficial completa sobre todos los tipos LOB, arquitectura de almacenamiento, API DBMS_LOB y mejores prácticas. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/adlob/
- **Oracle Database PL/SQL Packages and Types Reference 19c — DBMS_LOB** — Referencia completa de todos los procedimientos y funciones del paquete `DBMS_LOB` con ejemplos detallados. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_LOB.html
- **Oracle Database SQL Language Reference 19c — LOB Data Types** — Sección dedicada a los tipos de datos LOB en SQL, incluyendo DDL para `LOB STORE AS`, restricciones y comportamiento en DML. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/
- **Oracle Database Administrator's Guide 19c — Managing LOBs** — Guía para administradores sobre gestión de segmentos LOB, monitoreo de espacio y estrategias de mantenimiento. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/
- **Vista USER_LOBS** — Consulta `SELECT * FROM user_lobs` o `SELECT * FROM dba_lobs` para inspeccionar todas las propiedades de los segmentos LOB en tu esquema o en toda la base de datos, incluyendo `SECUREFILE`, `COMPRESS`, `DEDUPLICATE`, `ENCRYPT`, `CACHE`, `LOGGING` y `IN_ROW`.
