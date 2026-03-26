# Lab 04-00-01: Gestión de Tablespaces, Datafiles, Undo, Temp y ASM en Oracle Database 19c

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 90 minutos |
| **Complejidad** | Avanzado |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 4 — Gestión de Almacenamiento Lógico y Físico |
| **Práctica** | 04-00-01 |

---

## Descripción General

En esta práctica el estudiante creará, modificará y administrará tablespaces permanentes, temporales y de undo en Oracle Database 19c, explorando la jerarquía completa del almacenamiento lógico (tablespace → segmento → extent → bloque). Se trabajará con comandos `CREATE TABLESPACE`, `ALTER TABLESPACE`, `ALTER DATABASE DATAFILE` y se consultarán las vistas del diccionario de datos más relevantes (`DBA_TABLESPACES`, `DBA_DATA_FILES`, `DBA_SEGMENTS`, `DBA_EXTENTS`, `DBA_FREE_SPACE`) para monitorear el estado del almacenamiento en tiempo real.

La práctica tiene relevancia directa en escenarios de producción: un DBA debe ser capaz de aprovisionar espacio antes de que las aplicaciones lo agoten, reorganizar tablespaces para optimizar el rendimiento y gestionar el espacio de undo y temporal para soportar cargas transaccionales intensas. Al finalizar, el estudiante habrá completado el ciclo completo de administración de almacenamiento que incluye creación, expansión, monitoreo, cambio de estado y eliminación de tablespaces y datafiles.

---

## Objetivos de Aprendizaje

Al completar esta práctica, serás capaz de:

- [ ] Crear tablespaces permanentes con gestión local de extensiones usando `AUTOALLOCATE` y `UNIFORM SIZE`, configurando `AUTOEXTEND` con límites de crecimiento
- [ ] Agregar y redimensionar datafiles asociados a tablespaces existentes usando `ALTER TABLESPACE ADD DATAFILE` y `ALTER DATABASE DATAFILE RESIZE`
- [ ] Examinar la jerarquía de almacenamiento consultando `DBA_SEGMENTS`, `DBA_EXTENTS` y `DBA_BLOCKS` para entender cómo Oracle organiza los datos internamente
- [ ] Crear y activar un tablespace de undo adicional, configurar el parámetro `UNDO_RETENTION` y monitorear el estado con `V$UNDOSTAT`
- [ ] Gestionar el tablespace temporal agregando tempfiles y monitoreando el uso con `V$TEMP_SPACE_HEADER` y `V$TEMPSEG_USAGE`
- [ ] Explorar los conceptos de Automatic Storage Management (ASM) consultando las vistas `V$ASM_DISKGROUP`, `V$ASM_DISK` y ejecutando comandos básicos de `asmcmd`

---

## Prerrequisitos

### Conocimiento Requerido

- Comprensión de la arquitectura Oracle: instancia, base de datos, SGA, procesos de fondo (especialmente DBWn)
- Familiaridad con SQL*Plus y conexión como usuario con privilegios DBA
- Conceptos básicos de almacenamiento: bloques Oracle, extensiones (extents) y segmentos
- Prácticas 01-00-01, 02-00-01 y 03-00-01 completadas exitosamente
- Conocimiento del sistema de archivos Linux: rutas, permisos y comandos básicos (`ls`, `df`, `du`)

### Acceso Requerido

- Conexión SSH a la máquina virtual Oracle Linux 8.x
- Acceso a SQL*Plus como `SYS` con rol `SYSDBA` o como `SYSTEM`
- Usuario de práctica `PRACTICA_USER` creado en prácticas anteriores con privilegios `DBA` o equivalentes
- Al menos **5 GB de espacio libre en disco** en la partición donde residen los datafiles (generalmente `/u01`)
- Acceso al sistema operativo como usuario `oracle` para verificar archivos físicos

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | Intel Core i5 8va gen o superior (virtualización habilitada) |
| RAM | Mínimo 8 GB asignados a la VM (16 GB recomendado) |
| Almacenamiento | Mínimo 5 GB libres en `/u01` para datafiles de práctica |
| Red | Adaptador de red funcional (loopback para conexión local) |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) | Motor de base de datos principal |
| Oracle Linux | 8.x (8.7 o 8.8) | Sistema operativo huésped |
| SQL*Plus | Incluido con Oracle 19c | Ejecución de comandos SQL y DBA |
| PuTTY / Terminal SSH | 0.79+ / nativo | Acceso remoto a la VM |
| Oracle Grid Infrastructure | 19c (opcional para ASM) | Instancia ASM para sección final |

### Configuración Inicial

Antes de comenzar, verificar que la instancia Oracle esté activa y que exista espacio suficiente en disco:

```bash
# Conectarse a la VM Oracle Linux via SSH
ssh oracle@192.168.56.101

# Verificar que la instancia Oracle esté activa
sqlplus / as sysdba <<EOF
SELECT instance_name, status, version FROM v\$instance;
EXIT;
EOF

# Verificar espacio disponible en el sistema de archivos
df -h /u01

# Verificar que el directorio de datafiles existe y tiene permisos correctos
ls -la /u01/oradata/

# Identificar el nombre de la base de datos (SID)
echo "ORACLE_SID: $ORACLE_SID"
echo "ORACLE_BASE: $ORACLE_BASE"
echo "ORACLE_HOME: $ORACLE_HOME"
```

**Salida esperada de verificación:**

```
INSTANCE_NAME    STATUS       VERSION
---------------- ------------ -----------------
ORCL             OPEN         19.0.0.0.0

Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       200G   85G  115G  43% /u01

ORACLE_SID: ORCL
ORACLE_BASE: /u01/app/oracle
ORACLE_HOME: /u01/app/oracle/product/19.3.0/dbhome_1
```

---

## Instrucciones Paso a Paso

### Paso 1: Exploración del Estado Inicial del Almacenamiento

**Objetivo:** Comprender el estado actual de los tablespaces y datafiles antes de realizar modificaciones, estableciendo una línea base de referencia.

**Instrucciones:**

1. Conectarse a SQL*Plus como `SYSDBA` y ejecutar una consulta de diagnóstico inicial del almacenamiento:

   ```sql
   sqlplus / as sysdba
   ```

2. Consultar todos los tablespaces existentes con su estado y tipo de gestión:

   ```sql
   -- Estado completo de todos los tablespaces
   SELECT
       tablespace_name,
       status,
       contents,
       extent_management,
       segment_space_management,
       bigfile
   FROM dba_tablespaces
   ORDER BY tablespace_name;
   ```

3. Consultar los datafiles asociados a cada tablespace con información de tamaño:

   ```sql
   -- Datafiles con tamaño actual y máximo
   SELECT
       d.tablespace_name,
       d.file_name,
       ROUND(d.bytes / 1024 / 1024, 2)     AS size_mb,
       d.autoextensible,
       ROUND(d.maxbytes / 1024 / 1024, 2)  AS maxsize_mb,
       ROUND(d.increment_by * 8192 / 1024 / 1024, 2) AS next_mb
   FROM dba_data_files d
   ORDER BY d.tablespace_name, d.file_name;
   ```

4. Verificar el espacio libre disponible en cada tablespace:

   ```sql
   -- Espacio libre por tablespace
   SELECT
       f.tablespace_name,
       ROUND(SUM(f.bytes) / 1024 / 1024, 2) AS free_mb,
       COUNT(f.file_id)                       AS free_chunks
   FROM dba_free_space f
   GROUP BY f.tablespace_name
   ORDER BY f.tablespace_name;
   ```

5. Crear un reporte consolidado de uso de almacenamiento:

   ```sql
   -- Reporte consolidado: total, usado y libre por tablespace
   SELECT
       t.tablespace_name,
       t.status,
       ROUND(NVL(d.total_mb, 0), 2)                          AS total_mb,
       ROUND(NVL(d.total_mb, 0) - NVL(f.free_mb, 0), 2)     AS used_mb,
       ROUND(NVL(f.free_mb, 0), 2)                           AS free_mb,
       ROUND((NVL(f.free_mb, 0) / NULLIF(d.total_mb, 0)) * 100, 1) AS pct_free
   FROM dba_tablespaces t
   LEFT JOIN (
       SELECT tablespace_name, SUM(bytes) / 1024 / 1024 AS total_mb
       FROM dba_data_files
       GROUP BY tablespace_name
   ) d ON t.tablespace_name = d.tablespace_name
   LEFT JOIN (
       SELECT tablespace_name, SUM(bytes) / 1024 / 1024 AS free_mb
       FROM dba_free_space
       GROUP BY tablespace_name
   ) f ON t.tablespace_name = f.tablespace_name
   WHERE t.contents != 'TEMPORARY'
   ORDER BY t.tablespace_name;
   ```

**Salida Esperada:**

```
TABLESPACE_NAME    STATUS  TOTAL_MB  USED_MB  FREE_MB  PCT_FREE
------------------ ------- --------- -------- -------- --------
SYSAUX             ONLINE   860.00   755.25   104.75     12.2
SYSTEM             ONLINE   860.00   812.50    47.50      5.5
UNDOTBS1           ONLINE   235.00    18.75   216.25     92.0
USERS              ONLINE     5.00     1.25     3.75     75.0
```

**Verificación:**

- Confirmar que los tablespaces `SYSTEM`, `SYSAUX`, `UNDOTBS1`, `TEMP` y `USERS` están presentes y en estado `ONLINE`
- Anotar el espacio libre disponible para comparar al final de la práctica
- Verificar que `EXTENT_MANAGEMENT` sea `LOCAL` para todos los tablespaces modernos

---

### Paso 2: Creación de Tablespaces Permanentes con Diferentes Configuraciones

**Objetivo:** Crear tablespaces permanentes usando `AUTOALLOCATE` y `UNIFORM SIZE` para comprender las diferencias en la gestión de extensiones y cómo afectan el rendimiento y la fragmentación.

**Instrucciones:**

1. Crear el primer tablespace con gestión `AUTOALLOCATE` (Oracle decide el tamaño de las extensiones automáticamente):

   ```sql
   -- Tablespace con AUTOALLOCATE: Oracle gestiona el tamaño de extensiones
   -- Adecuado para objetos de tamaño variable y mixto
   CREATE TABLESPACE lab_autoalloc_ts
     DATAFILE '/u01/oradata/ORCL/lab_autoalloc_ts01.dbf'
     SIZE 100M
     AUTOEXTEND ON NEXT 50M MAXSIZE 500M
     EXTENT MANAGEMENT LOCAL AUTOALLOCATE
     SEGMENT SPACE MANAGEMENT AUTO;
   ```

2. Crear el segundo tablespace con `UNIFORM SIZE` (todas las extensiones tienen el mismo tamaño):

   ```sql
   -- Tablespace con UNIFORM SIZE 1M: todas las extensiones son de 1 MB
   -- Adecuado para objetos de tamaño similar, reduce fragmentación
   CREATE TABLESPACE lab_uniform_ts
     DATAFILE '/u01/oradata/ORCL/lab_uniform_ts01.dbf'
     SIZE 100M
     AUTOEXTEND ON NEXT 50M MAXSIZE 500M
     EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
     SEGMENT SPACE MANAGEMENT AUTO;
   ```

3. Crear un tablespace para datos de la aplicación de práctica (simula un entorno real de aplicación):

   ```sql
   -- Tablespace de aplicación: datos transaccionales
   CREATE TABLESPACE lab_app_data_ts
     DATAFILE '/u01/oradata/ORCL/lab_app_data01.dbf'
     SIZE 200M
     AUTOEXTEND ON NEXT 100M MAXSIZE 2G
     EXTENT MANAGEMENT LOCAL AUTOALLOCATE
     SEGMENT SPACE MANAGEMENT AUTO;
   ```

4. Crear un tablespace para índices de la aplicación (buena práctica: separar datos de índices):

   ```sql
   -- Tablespace de índices: separado para optimización de I/O
   CREATE TABLESPACE lab_app_idx_ts
     DATAFILE '/u01/oradata/ORCL/lab_app_idx01.dbf'
     SIZE 100M
     AUTOEXTEND ON NEXT 50M MAXSIZE 1G
     EXTENT MANAGEMENT LOCAL UNIFORM SIZE 2M
     SEGMENT SPACE MANAGEMENT AUTO;
   ```

5. Verificar que los tablespaces fueron creados correctamente:

   ```sql
   -- Verificar los nuevos tablespaces
   SELECT
       tablespace_name,
       status,
       extent_management,
       allocation_type,
       segment_space_management,
       ROUND(initial_extent / 1024, 2) AS initial_extent_kb,
       ROUND(next_extent / 1024, 2)    AS next_extent_kb
   FROM dba_tablespaces
   WHERE tablespace_name IN (
       'LAB_AUTOALLOC_TS',
       'LAB_UNIFORM_TS',
       'LAB_APP_DATA_TS',
       'LAB_APP_IDX_TS'
   )
   ORDER BY tablespace_name;
   ```

6. Verificar los datafiles creados en el sistema de archivos:

   ```bash
   -- Desde el terminal Linux (abrir nueva sesión SSH)
   ls -lh /u01/oradata/ORCL/lab_*.dbf
   ```

**Salida Esperada:**

```sql
TABLESPACE_NAME    STATUS  EXTENT_MGMT  ALLOCATION_TYPE  SEG_SPACE_MGMT
------------------ ------- ------------ ---------------- ---------------
LAB_APP_DATA_TS    ONLINE  LOCAL        SYSTEM           AUTO
LAB_APP_IDX_TS     ONLINE  LOCAL        UNIFORM          AUTO
LAB_AUTOALLOC_TS   ONLINE  LOCAL        SYSTEM           AUTO
LAB_UNIFORM_TS     ONLINE  LOCAL        UNIFORM          AUTO
```

```bash
-rw-r----- 1 oracle oinstall 104857600 Jan 15 10:23 lab_autoalloc_ts01.dbf
-rw-r----- 1 oracle oinstall 104857600 Jan 15 10:23 lab_uniform_ts01.dbf
-rw-r----- 1 oracle oinstall 209715200 Jan 15 10:24 lab_app_data01.dbf
-rw-r----- 1 oracle oinstall 104857600 Jan 15 10:24 lab_app_idx01.dbf
```

**Verificación:**

- Los 4 tablespaces deben aparecer en `DBA_TABLESPACES` con estado `ONLINE`
- `LAB_UNIFORM_TS` y `LAB_APP_IDX_TS` deben mostrar `ALLOCATION_TYPE = UNIFORM`
- Los archivos físicos deben existir en `/u01/oradata/ORCL/`
- Confirmar que `EXTENT_MANAGEMENT = LOCAL` para todos

---

### Paso 3: Gestión de Datafiles — Agregar, Redimensionar y Configurar AUTOEXTEND

**Objetivo:** Practicar las operaciones más comunes de administración de datafiles: agregar nuevos datafiles a tablespaces existentes, redimensionar datafiles y modificar la configuración de `AUTOEXTEND`.

**Instrucciones:**

1. Agregar un segundo datafile al tablespace `LAB_APP_DATA_TS` para aumentar su capacidad:

   ```sql
   -- Agregar segundo datafile: simula expansión de capacidad cuando el primero se llena
   ALTER TABLESPACE lab_app_data_ts
     ADD DATAFILE '/u01/oradata/ORCL/lab_app_data02.dbf'
     SIZE 150M
     AUTOEXTEND ON NEXT 100M MAXSIZE 2G;
   ```

2. Agregar un segundo datafile al tablespace de índices:

   ```sql
   -- Agregar segundo datafile a tablespace de índices
   ALTER TABLESPACE lab_app_idx_ts
     ADD DATAFILE '/u01/oradata/ORCL/lab_app_idx02.dbf'
     SIZE 100M
     AUTOEXTEND OFF;
   ```

3. Redimensionar el primer datafile de `LAB_APP_DATA_TS` (aumentar su tamaño manualmente):

   ```sql
   -- Aumentar el tamaño del primer datafile a 300 MB
   ALTER DATABASE DATAFILE '/u01/oradata/ORCL/lab_app_data01.dbf'
     RESIZE 300M;
   ```

4. Modificar la configuración de `AUTOEXTEND` en un datafile existente:

   ```sql
   -- Habilitar AUTOEXTEND en el datafile que se creó sin esta opción
   ALTER DATABASE DATAFILE '/u01/oradata/ORCL/lab_app_idx02.dbf'
     AUTOEXTEND ON NEXT 50M MAXSIZE 500M;
   ```

5. Verificar el estado actualizado de todos los datafiles de los tablespaces de práctica:

   ```sql
   -- Estado completo de datafiles de tablespaces de práctica
   SELECT
       d.tablespace_name,
       d.file_id,
       SUBSTR(d.file_name, INSTR(d.file_name, '/', -1) + 1) AS filename,
       ROUND(d.bytes / 1024 / 1024, 2)                      AS current_mb,
       d.autoextensible                                       AS autoext,
       ROUND(d.increment_by * 8192 / 1024 / 1024, 2)        AS next_mb,
       ROUND(d.maxbytes / 1024 / 1024, 2)                   AS max_mb,
       d.status
   FROM dba_data_files d
   WHERE d.tablespace_name IN (
       'LAB_APP_DATA_TS',
       'LAB_APP_IDX_TS',
       'LAB_AUTOALLOC_TS',
       'LAB_UNIFORM_TS'
   )
   ORDER BY d.tablespace_name, d.file_id;
   ```

6. Verificar el espacio total disponible en los tablespaces de práctica:

   ```sql
   -- Capacidad total por tablespace (suma de todos sus datafiles)
   SELECT
       tablespace_name,
       COUNT(file_id)                              AS num_datafiles,
       ROUND(SUM(bytes) / 1024 / 1024, 2)         AS total_mb,
       ROUND(SUM(maxbytes) / 1024 / 1024, 2)      AS max_possible_mb
   FROM dba_data_files
   WHERE tablespace_name IN (
       'LAB_APP_DATA_TS',
       'LAB_APP_IDX_TS'
   )
   GROUP BY tablespace_name
   ORDER BY tablespace_name;
   ```

**Salida Esperada:**

```
TABLESPACE_NAME  FILE_ID  FILENAME              CURRENT_MB  AUTOEXT  NEXT_MB  MAX_MB
---------------- -------- --------------------- ----------- -------- -------- -------
LAB_APP_DATA_TS       5   lab_app_data01.dbf      300.00    YES      100.00   2048.00
LAB_APP_DATA_TS       6   lab_app_data02.dbf      150.00    YES      100.00   2048.00
LAB_APP_IDX_TS        7   lab_app_idx01.dbf       100.00    YES       50.00   1024.00
LAB_APP_IDX_TS        8   lab_app_idx02.dbf       100.00    YES       50.00    500.00

TABLESPACE_NAME  NUM_DATAFILES  TOTAL_MB  MAX_POSSIBLE_MB
---------------- ------------- --------- ---------------
LAB_APP_DATA_TS             2    450.00         4096.00
LAB_APP_IDX_TS              2    200.00         1524.00
```

**Verificación:**

- `LAB_APP_DATA_TS` debe tener 2 datafiles con un total de 450 MB
- `LAB_APP_IDX_TS` debe tener 2 datafiles, ambos con `AUTOEXT = YES`
- El datafile `lab_app_data01.dbf` debe mostrar 300 MB (después del RESIZE)
- Confirmar los tamaños en el sistema operativo: `ls -lh /u01/oradata/ORCL/lab_app*.dbf`

---

### Paso 4: Exploración de la Jerarquía de Almacenamiento (Segmentos, Extents y Bloques)

**Objetivo:** Crear objetos en los tablespaces de práctica y examinar cómo Oracle organiza internamente los datos a través de la jerarquía tablespace → segmento → extent → bloque, usando las vistas del diccionario de datos.

**Instrucciones:**

1. Crear una tabla de prueba en el tablespace `LAB_APP_DATA_TS`:

   ```sql
   -- Conectarse como PRACTICA_USER o crear la tabla como SYSTEM con tablespace especificado
   -- Si se usa SYSTEM, especificar el tablespace explícitamente
   CREATE TABLE system.lab_empleados (
       emp_id      NUMBER(10)    NOT NULL,
       nombre      VARCHAR2(100) NOT NULL,
       salario     NUMBER(12, 2),
       depto_id    NUMBER(5),
       fecha_alta  DATE DEFAULT SYSDATE,
       descripcion VARCHAR2(4000),
       CONSTRAINT pk_lab_empleados PRIMARY KEY (emp_id)
   )
   TABLESPACE lab_app_data_ts
   STORAGE (INITIAL 1M NEXT 1M PCTINCREASE 0);
   ```

2. Crear un índice en el tablespace de índices:

   ```sql
   -- Índice en tablespace separado: buena práctica de administración
   CREATE INDEX system.idx_lab_emp_depto
     ON system.lab_empleados (depto_id)
     TABLESPACE lab_app_idx_ts;
   ```

3. Insertar datos de prueba para generar extensiones:

   ```sql
   -- Insertar 10,000 filas para generar múltiples extensiones
   BEGIN
     FOR i IN 1..10000 LOOP
       INSERT INTO system.lab_empleados (
           emp_id, nombre, salario, depto_id, descripcion
       ) VALUES (
           i,
           'Empleado_' || LPAD(i, 5, '0'),
           ROUND(DBMS_RANDOM.VALUE(30000, 150000), 2),
           MOD(i, 20) + 1,
           RPAD('Descripcion del empleado ' || i || ' - datos de prueba para laboratorio Oracle ', 200, 'X')
       );
       -- Commit cada 1000 filas para no saturar el undo
       IF MOD(i, 1000) = 0 THEN
         COMMIT;
       END IF;
     END LOOP;
     COMMIT;
   END;
   /
   ```

4. Examinar los segmentos creados en los tablespaces de práctica:

   ```sql
   -- Ver segmentos en tablespaces de práctica
   SELECT
       owner,
       segment_name,
       segment_type,
       tablespace_name,
       ROUND(bytes / 1024 / 1024, 3)   AS size_mb,
       extents,
       blocks,
       initial_extent / 1024           AS initial_kb,
       next_extent / 1024              AS next_kb
   FROM dba_segments
   WHERE tablespace_name IN ('LAB_APP_DATA_TS', 'LAB_APP_IDX_TS')
   ORDER BY tablespace_name, segment_type, segment_name;
   ```

5. Examinar las extensiones (extents) del segmento de la tabla:

   ```sql
   -- Ver extensiones del segmento de la tabla lab_empleados
   SELECT
       segment_name,
       extent_id,
       file_id,
       block_id,
       ROUND(bytes / 1024, 2) AS size_kb,
       blocks
   FROM dba_extents
   WHERE segment_name = 'LAB_EMPLEADOS'
     AND owner = 'SYSTEM'
   ORDER BY extent_id;
   ```

6. Analizar la distribución de bloques Oracle en el segmento:

   ```sql
   -- Estadísticas del segmento: bloques usados vs. bloques libres
   -- Primero recolectar estadísticas
   EXEC DBMS_STATS.GATHER_TABLE_STATS('SYSTEM', 'LAB_EMPLEADOS');

   -- Consultar estadísticas de la tabla
   SELECT
       num_rows,
       blocks,
       empty_blocks,
       avg_space,
       chain_cnt,
       avg_row_len,
       last_analyzed
   FROM dba_tables
   WHERE table_name = 'LAB_EMPLEADOS'
     AND owner = 'SYSTEM';
   ```

7. Visualizar la jerarquía completa en una sola consulta:

   ```sql
   -- Jerarquía completa: tablespace -> datafile -> segmento -> extents
   SELECT
       e.tablespace_name,
       d.file_name                                          AS datafile,
       e.segment_name,
       e.segment_type,
       COUNT(e.extent_id)                                   AS num_extents,
       ROUND(SUM(e.bytes) / 1024 / 1024, 3)               AS total_mb,
       SUM(e.blocks)                                        AS total_blocks
   FROM dba_extents e
   JOIN dba_data_files d
     ON e.tablespace_name = d.tablespace_name
    AND e.file_id = d.file_id
   WHERE e.tablespace_name IN ('LAB_APP_DATA_TS', 'LAB_APP_IDX_TS')
   GROUP BY e.tablespace_name, d.file_name, e.segment_name, e.segment_type
   ORDER BY e.tablespace_name, e.segment_name;
   ```

**Salida Esperada:**

```sql
-- Segmentos (resultado aproximado):
OWNER   SEGMENT_NAME          SEGMENT_TYPE  TABLESPACE_NAME   SIZE_MB  EXTENTS  BLOCKS
------- --------------------- ------------- ----------------- -------- -------- ------
SYSTEM  LAB_EMPLEADOS         TABLE         LAB_APP_DATA_TS      8.000       8    1024
SYSTEM  IDX_LAB_EMP_DEPTO     INDEX         LAB_APP_IDX_TS       2.000       2     256
SYSTEM  PK_LAB_EMPLEADOS      INDEX         LAB_APP_DATA_TS      1.000       1     128

-- Extents de LAB_EMPLEADOS (resultado aproximado):
SEGMENT_NAME    EXTENT_ID  FILE_ID  BLOCK_ID  SIZE_KB  BLOCKS
--------------- ---------- -------- --------- -------- ------
LAB_EMPLEADOS           0        5      128    128.00      16
LAB_EMPLEADOS           1        5      144   1024.00     128
LAB_EMPLEADOS           2        5      272   1024.00     128
...
```

**Verificación:**

- La tabla `LAB_EMPLEADOS` debe tener exactamente 10,000 filas: `SELECT COUNT(*) FROM system.lab_empleados;`
- El segmento debe aparecer en `DBA_SEGMENTS` con `TABLESPACE_NAME = LAB_APP_DATA_TS`
- El índice debe aparecer en `DBA_SEGMENTS` con `TABLESPACE_NAME = LAB_APP_IDX_TS`
- Deben existir múltiples extensiones en `DBA_EXTENTS` para el segmento `LAB_EMPLEADOS`

---

### Paso 5: Gestión del Tablespace de Undo

**Objetivo:** Crear un tablespace de undo adicional, cambiar el tablespace de undo activo, configurar el parámetro `UNDO_RETENTION` y monitorear el estado del undo con las vistas dinámicas de rendimiento.

**Instrucciones:**

1. Verificar la configuración actual del undo:

   ```sql
   -- Parámetros actuales de undo
   SHOW PARAMETER undo;
   ```

   ```sql
   -- Tablespace de undo activo y su estado
   SELECT
       tablespace_name,
       status,
       contents,
       ROUND(SUM(d.bytes) / 1024 / 1024, 2) AS size_mb
   FROM dba_tablespaces t
   JOIN dba_data_files d USING (tablespace_name)
   WHERE t.contents = 'UNDO'
   GROUP BY t.tablespace_name, t.status, t.contents;
   ```

2. Consultar el estado actual del undo con `V$UNDOSTAT`:

   ```sql
   -- Estadísticas de undo de las últimas horas
   SELECT
       TO_CHAR(begin_time, 'DD-MON HH24:MI') AS period_start,
       undoblks,
       txncount,
       maxconcurrency,
       maxquerylen,
       ROUND(undoblks * 8192 / 1024 / 1024, 2) AS undo_mb_used,
       ssolderrcnt,
       nospaceerrcnt
   FROM v$undostat
   WHERE rownum <= 10
   ORDER BY begin_time DESC;
   ```

3. Crear un segundo tablespace de undo (útil para mantenimiento o cuando el primero se llena):

   ```sql
   -- Crear un tablespace de undo alternativo
   CREATE UNDO TABLESPACE undotbs2
     DATAFILE '/u01/oradata/ORCL/undotbs2_01.dbf'
     SIZE 200M
     AUTOEXTEND ON NEXT 50M MAXSIZE 1G
     RETENTION GUARANTEE;
   ```

4. Cambiar el tablespace de undo activo al nuevo (operación en caliente, sin reinicio):

   ```sql
   -- Cambiar el tablespace de undo activo
   ALTER SYSTEM SET UNDO_TABLESPACE = UNDOTBS2 SCOPE = BOTH;
   ```

5. Verificar el cambio:

   ```sql
   -- Confirmar que el nuevo tablespace de undo está activo
   SHOW PARAMETER undo_tablespace;
   ```

   ```sql
   -- Verificar que UNDOTBS2 está siendo usado
   SELECT
       t.tablespace_name,
       t.status,
       t.contents,
       ROUND(SUM(d.bytes) / 1024 / 1024, 2) AS size_mb
   FROM dba_tablespaces t
   JOIN dba_data_files d USING (tablespace_name)
   WHERE t.contents = 'UNDO'
   GROUP BY t.tablespace_name, t.status, t.contents
   ORDER BY t.tablespace_name;
   ```

6. Configurar el parámetro `UNDO_RETENTION` para garantizar consistencia en consultas largas:

   ```sql
   -- Aumentar UNDO_RETENTION a 1800 segundos (30 minutos)
   -- Importante para reportes y consultas analíticas largas
   ALTER SYSTEM SET UNDO_RETENTION = 1800 SCOPE = BOTH;
   ```

7. Monitorear el uso del undo por transacciones activas:

   ```sql
   -- Transacciones activas y su uso de undo
   SELECT
       t.xidusn,
       t.xidslot,
       t.xidsqn,
       ROUND(t.used_ublk * 8192 / 1024, 2) AS used_kb,
       t.start_time,
       s.username,
       s.sid,
       s.serial#,
       s.status
   FROM v$transaction t
   JOIN v$session s ON t.ses_addr = s.saddr
   ORDER BY t.used_ublk DESC;
   ```

8. Consultar el estado de los segmentos de undo:

   ```sql
   -- Estado de los segmentos de rollback/undo
   SELECT
       usn,
       name,
       status,
       ROUND(rssize / 1024 / 1024, 2) AS size_mb,
       writes,
       xacts,
       shrinks,
       extends
   FROM v$rollstat r
   JOIN v$rollname n ON r.usn = n.usn
   ORDER BY usn;
   ```

**Salida Esperada:**

```sql
-- SHOW PARAMETER undo_tablespace:
NAME                     TYPE        VALUE
------------------------ ----------- --------
undo_tablespace          string      UNDOTBS2

-- SHOW PARAMETER undo_retention:
NAME                     TYPE        VALUE
------------------------ ----------- --------
undo_retention           integer     1800
```

**Verificación:**

- `SHOW PARAMETER undo_tablespace` debe mostrar `UNDOTBS2`
- `SHOW PARAMETER undo_retention` debe mostrar `1800`
- El tablespace `UNDOTBS2` debe aparecer en `DBA_TABLESPACES` con `CONTENTS = UNDO`
- El archivo `/u01/oradata/ORCL/undotbs2_01.dbf` debe existir en disco

---

### Paso 6: Gestión del Tablespace Temporal

**Objetivo:** Examinar el tablespace temporal actual, agregar un tempfile para aumentar la capacidad y monitorear el uso del espacio temporal durante operaciones que requieren ordenamiento o joins de gran volumen.

**Instrucciones:**

1. Verificar el tablespace temporal actual y sus tempfiles:

   ```sql
   -- Información del tablespace temporal
   SELECT
       tablespace_name,
       status,
       contents,
       extent_management,
       allocation_type
   FROM dba_tablespaces
   WHERE contents = 'TEMPORARY';
   ```

   ```sql
   -- Tempfiles del tablespace temporal
   SELECT
       tablespace_name,
       file_name,
       ROUND(bytes / 1024 / 1024, 2)    AS size_mb,
       autoextensible,
       ROUND(maxbytes / 1024 / 1024, 2) AS max_mb,
       status
   FROM dba_temp_files
   ORDER BY tablespace_name, file_name;
   ```

2. Verificar el uso actual del tablespace temporal:

   ```sql
   -- Estado del espacio temporal por tempfile
   SELECT
       file#,
       status,
       ROUND(bytes / 1024 / 1024, 2)   AS total_mb,
       ROUND(free_space / 1024 / 1024, 2) AS free_mb
   FROM v$temp_space_header;
   ```

3. Agregar un segundo tempfile al tablespace `TEMP`:

   ```sql
   -- Agregar tempfile adicional para mayor capacidad de operaciones temporales
   ALTER TABLESPACE temp
     ADD TEMPFILE '/u01/oradata/ORCL/temp02.dbf'
     SIZE 100M
     AUTOEXTEND ON NEXT 50M MAXSIZE 500M;
   ```

4. Generar una operación que use el espacio temporal (ordenamiento de gran volumen):

   ```sql
   -- Esta consulta generará uso del tablespace temporal
   -- (ordenamiento de datos con GROUP BY y ORDER BY sobre tabla grande)
   SELECT
       depto_id,
       COUNT(*)                      AS num_empleados,
       ROUND(AVG(salario), 2)        AS salario_promedio,
       MIN(salario)                  AS salario_min,
       MAX(salario)                  AS salario_max,
       ROUND(SUM(salario), 2)        AS masa_salarial
   FROM system.lab_empleados
   GROUP BY depto_id
   ORDER BY masa_salarial DESC;
   ```

5. Monitorear el uso del espacio temporal en tiempo real (ejecutar en otra sesión SQL*Plus):

   ```sql
   -- Monitorear uso del tablespace temporal (ejecutar en sesión separada)
   SELECT
       s.username,
       s.sid,
       s.serial#,
       s.sql_id,
       ROUND(su.blocks * 8192 / 1024 / 1024, 2) AS temp_used_mb,
       su.segtype
   FROM v$tempseg_usage su
   JOIN v$session s ON su.session_addr = s.saddr
   ORDER BY temp_used_mb DESC;
   ```

6. Verificar el estado del tablespace temporal después de agregar el tempfile:

   ```sql
   -- Verificar ambos tempfiles
   SELECT
       tablespace_name,
       file_name,
       ROUND(bytes / 1024 / 1024, 2)    AS size_mb,
       autoextensible                    AS autoext,
       ROUND(maxbytes / 1024 / 1024, 2) AS max_mb
   FROM dba_temp_files
   ORDER BY tablespace_name, file_id;
   ```

7. Verificar el grupo de tablespaces temporales por defecto para usuarios:

   ```sql
   -- Tablespace temporal asignado a cada usuario
   SELECT
       username,
       default_tablespace,
       temporary_tablespace,
       account_status
   FROM dba_users
   WHERE username IN ('SYSTEM', 'SYS', 'PRACTICA_USER', 'HR')
   ORDER BY username;
   ```

**Salida Esperada:**

```sql
-- Tempfiles después de agregar el segundo:
TABLESPACE_NAME  FILE_NAME               SIZE_MB  AUTOEXT  MAX_MB
---------------- ----------------------- -------- -------- -------
TEMP             /u01/oradata/ORCL/temp01.dbf  100.00  YES    32767.98
TEMP             /u01/oradata/ORCL/temp02.dbf  100.00  YES      500.00
```

**Verificación:**

- `DBA_TEMP_FILES` debe mostrar 2 tempfiles para el tablespace `TEMP`
- El archivo `/u01/oradata/ORCL/temp02.dbf` debe existir en disco
- `V$TEMP_SPACE_HEADER` debe mostrar ambos tempfiles con su estado

---

### Paso 7: Cambio de Estado de Tablespaces y Operaciones de Mantenimiento

**Objetivo:** Practicar las operaciones de cambio de estado de tablespaces (`READ ONLY`, `OFFLINE`, `ONLINE`) y entender cuándo y cómo se usan en escenarios reales de mantenimiento.

**Instrucciones:**

1. Poner el tablespace `LAB_AUTOALLOC_TS` en modo `READ ONLY` (simula datos históricos de solo lectura):

   ```sql
   -- Cambiar a modo READ ONLY: útil para datos históricos o de referencia
   ALTER TABLESPACE lab_autoalloc_ts READ ONLY;
   ```

2. Verificar el cambio de estado:

   ```sql
   -- Confirmar el estado READ ONLY
   SELECT tablespace_name, status FROM dba_tablespaces
   WHERE tablespace_name = 'LAB_AUTOALLOC_TS';
   ```

3. Intentar insertar datos en el tablespace `READ ONLY` (debe fallar):

   ```sql
   -- Crear tabla en tablespace READ ONLY (debe generar error ORA-01647)
   CREATE TABLE system.test_readonly (id NUMBER)
   TABLESPACE lab_autoalloc_ts;
   ```

   > **Nota:** Se espera el error `ORA-01647: tablespace 'LAB_AUTOALLOC_TS' is read-only, cannot allocate space in it`. Esto confirma que el modo funciona correctamente.

4. Restaurar el tablespace a modo `READ WRITE`:

   ```sql
   -- Restaurar a modo lectura/escritura
   ALTER TABLESPACE lab_autoalloc_ts READ WRITE;
   ```

5. Poner el tablespace `LAB_UNIFORM_TS` en modo `OFFLINE` (simula mantenimiento):

   ```sql
   -- Poner OFFLINE para mantenimiento (NORMAL = espera a que las transacciones terminen)
   ALTER TABLESPACE lab_uniform_ts OFFLINE NORMAL;
   ```

6. Verificar el estado `OFFLINE`:

   ```sql
   -- Confirmar OFFLINE
   SELECT tablespace_name, status FROM dba_tablespaces
   WHERE tablespace_name IN ('LAB_AUTOALLOC_TS', 'LAB_UNIFORM_TS');
   ```

7. Restaurar el tablespace `LAB_UNIFORM_TS` a `ONLINE`:

   ```sql
   -- Restaurar a ONLINE después del mantenimiento
   ALTER TABLESPACE lab_uniform_ts ONLINE;
   ```

8. Verificar el estado final de todos los tablespaces de práctica:

   ```sql
   -- Estado final de todos los tablespaces de práctica
   SELECT
       tablespace_name,
       status,
       contents,
       extent_management,
       allocation_type
   FROM dba_tablespaces
   WHERE tablespace_name LIKE 'LAB_%'
      OR tablespace_name IN ('UNDOTBS2')
   ORDER BY tablespace_name;
   ```

**Salida Esperada:**

```sql
-- Error esperado al intentar crear tabla en READ ONLY:
ORA-01647: tablespace 'LAB_AUTOALLOC_TS' is read-only, cannot allocate space in it

-- Estado final de tablespaces de práctica:
TABLESPACE_NAME    STATUS  CONTENTS   EXTENT_MGMT  ALLOCATION_TYPE
------------------ ------- ---------- ------------ ---------------
LAB_APP_DATA_TS    ONLINE  PERMANENT  LOCAL        SYSTEM
LAB_APP_IDX_TS     ONLINE  PERMANENT  LOCAL        UNIFORM
LAB_AUTOALLOC_TS   ONLINE  PERMANENT  LOCAL        SYSTEM
LAB_UNIFORM_TS     ONLINE  PERMANENT  LOCAL        UNIFORM
UNDOTBS2           ONLINE  UNDO       LOCAL        SYSTEM
```

**Verificación:**

- Todos los tablespaces `LAB_*` deben estar en estado `ONLINE` al finalizar
- El error `ORA-01647` confirma que el modo `READ ONLY` funciona correctamente
- Registrar el comportamiento observado para el reporte de laboratorio

---

### Paso 8: Exploración de ASM (Automatic Storage Management)

**Objetivo:** Explorar los conceptos de ASM consultando las vistas dinámicas de rendimiento si hay una instancia ASM disponible, o simulando la exploración mediante la documentación de las vistas. Esta sección es parcialmente conceptual si no hay instancia ASM configurada.

**Instrucciones:**

1. Verificar si hay una instancia ASM disponible en el entorno:

   ```bash
   # Verificar si el proceso ASM está corriendo
   ps -ef | grep asm_pmon | grep -v grep
   ```

   ```bash
   # Verificar si hay instancias de Grid Infrastructure
   /u01/app/grid/product/19.3.0/grid/bin/crsctl status resource -t 2>/dev/null || echo "Grid Infrastructure no instalado"
   ```

2. **Si hay instancia ASM disponible:** Conectarse a la instancia ASM y consultar disk groups:

   ```bash
   # Conectarse a la instancia ASM (requiere Grid Infrastructure instalado)
   export ORACLE_SID=+ASM
   sqlplus / as sysasm
   ```

   ```sql
   -- Consultar disk groups de ASM
   SELECT
       group_number,
       name,
       state,
       type,
       ROUND(total_mb / 1024, 2)       AS total_gb,
       ROUND(free_mb / 1024, 2)        AS free_gb,
       ROUND((total_mb - free_mb) / total_mb * 100, 1) AS pct_used
   FROM v$asm_diskgroup
   ORDER BY name;
   ```

   ```sql
   -- Consultar discos en cada disk group
   SELECT
       group_number,
       disk_number,
       name,
       path,
       state,
       mode_status,
       ROUND(total_mb / 1024, 2) AS total_gb,
       ROUND(free_mb / 1024, 2)  AS free_gb,
       header_status
   FROM v$asm_disk
   ORDER BY group_number, disk_number;
   ```

   ```sql
   -- Consultar archivos de base de datos en ASM
   SELECT
       group_number,
       file_number,
       compound_index,
       incarnation,
       ROUND(bytes / 1024 / 1024, 2)     AS size_mb,
       ROUND(space / 1024 / 1024, 2)     AS space_mb,
       type,
       redundancy
   FROM v$asm_file
   WHERE rownum <= 20
   ORDER BY group_number, file_number;
   ```

3. **Si hay instancia ASM disponible:** Usar `asmcmd` para navegar la estructura ASM:

   ```bash
   # Conectarse a asmcmd
   asmcmd -p
   ```

   ```bash
   # Dentro de asmcmd, ejecutar estos comandos:
   # Listar disk groups
   lsdg

   # Listar contenido del disk group DATA (ajustar nombre según entorno)
   ls +DATA/

   # Listar archivos de la base de datos
   ls +DATA/ORCL/

   # Ver espacio disponible
   du +DATA/ORCL/DATAFILE/

   # Salir de asmcmd
   exit
   ```

4. **Si NO hay instancia ASM disponible:** Explorar las vistas ASM conectado a la base de datos regular (mostrarán 0 filas pero la estructura es visible):

   ```bash
   # Regresar a la instancia de base de datos regular
   export ORACLE_SID=ORCL
   sqlplus / as sysdba
   ```

   ```sql
   -- Explorar estructura de vistas ASM (puede estar vacía sin instancia ASM)
   -- Esto muestra la estructura de las vistas aunque no haya datos
   DESCRIBE v$asm_diskgroup;
   ```

   ```sql
   DESCRIBE v$asm_disk;
   ```

   ```sql
   DESCRIBE v$asm_file;
   ```

   ```sql
   -- Consultar la vista (puede retornar 0 filas sin ASM)
   SELECT * FROM v$asm_diskgroup;
   ```

5. Documentar los conceptos clave de ASM identificados durante la exploración:

   ```sql
   -- Consulta conceptual: comparar gestión de archivos tradicional vs. ASM
   -- Ver datafiles actuales con rutas del sistema de archivos tradicional
   SELECT
       tablespace_name,
       file_name,
       ROUND(bytes / 1024 / 1024, 2) AS size_mb,
       autoextensible
   FROM dba_data_files
   ORDER BY tablespace_name, file_id;
   ```

   > **Concepto ASM:** En un entorno ASM, las rutas de `FILE_NAME` tendrían el formato `+DISKGROUP/DBNAME/DATAFILE/tablespace.nnn.nnnnnnnnn` en lugar de rutas del sistema de archivos como `/u01/oradata/ORCL/...`. ASM abstrae la ubicación física de los archivos igual que los tablespaces abstraen los datafiles para las aplicaciones.

**Salida Esperada (con ASM disponible):**

```sql
-- V$ASM_DISKGROUP (ejemplo):
GROUP_NUMBER  NAME    STATE    TYPE    TOTAL_GB  FREE_GB  PCT_USED
------------ ------- -------- ------- --------- -------- --------
           1  DATA    MOUNTED  EXTERN     40.00    25.00     37.5
           2  FRA     MOUNTED  EXTERN     20.00    18.00     10.0

-- ASMCMD lsdg:
State    Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  ...
MOUNTED  EXTERN  N         512   4096  1048576     40960    25600  DATA/
MOUNTED  EXTERN  N         512   4096  1048576     20480    18432  FRA/
```

**Salida Esperada (sin ASM disponible):**

```sql
-- DESCRIBE v$asm_diskgroup mostrará la estructura de columnas
-- SELECT * FROM v$asm_diskgroup retornará 0 filas
-- Esto es normal y esperado en entornos sin Grid Infrastructure

no rows selected
```

**Verificación:**

- Si hay ASM: confirmar que `V$ASM_DISKGROUP` muestra al menos un disk group en estado `MOUNTED`
- Si no hay ASM: confirmar que se puede hacer `DESCRIBE` de las vistas ASM sin errores
- Documentar en el reporte si el entorno tiene ASM o no, y por qué

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] Los 4 tablespaces permanentes de práctica (`LAB_AUTOALLOC_TS`, `LAB_UNIFORM_TS`, `LAB_APP_DATA_TS`, `LAB_APP_IDX_TS`) existen y están en estado `ONLINE`
- [ ] `LAB_APP_DATA_TS` tiene 2 datafiles con un total de al menos 450 MB
- [ ] `LAB_APP_IDX_TS` tiene 2 datafiles, ambos con `AUTOEXTEND = YES`
- [ ] La tabla `SYSTEM.LAB_EMPLEADOS` existe con exactamente 10,000 filas en `LAB_APP_DATA_TS`
- [ ] El índice `IDX_LAB_EMP_DEPTO` existe en `LAB_APP_IDX_TS`
- [ ] El tablespace `UNDOTBS2` existe y está configurado como tablespace de undo activo
- [ ] El parámetro `UNDO_RETENTION` está configurado en 1800 segundos
- [ ] El tablespace `TEMP` tiene 2 tempfiles
- [ ] Se documentó el comportamiento del error `ORA-01647` al intentar escribir en tablespace `READ ONLY`
- [ ] Se exploraron las vistas ASM (`V$ASM_DISKGROUP`, `V$ASM_DISK`, `V$ASM_FILE`)

### Procedimiento de Pruebas

1. Verificar todos los tablespaces de práctica:

   ```sql
   -- Prueba 1: Verificar existencia y estado de tablespaces
   SELECT tablespace_name, status, contents, extent_management, allocation_type
   FROM dba_tablespaces
   WHERE tablespace_name IN (
       'LAB_AUTOALLOC_TS', 'LAB_UNIFORM_TS',
       'LAB_APP_DATA_TS', 'LAB_APP_IDX_TS', 'UNDOTBS2'
   )
   ORDER BY tablespace_name;
   ```
   **Resultado Esperado:** 5 filas, todos con `STATUS = ONLINE`

2. Verificar datafiles de los tablespaces de práctica:

   ```sql
   -- Prueba 2: Contar datafiles por tablespace
   SELECT tablespace_name, COUNT(*) AS num_files,
          ROUND(SUM(bytes)/1024/1024, 2) AS total_mb
   FROM dba_data_files
   WHERE tablespace_name IN ('LAB_APP_DATA_TS', 'LAB_APP_IDX_TS')
   GROUP BY tablespace_name;
   ```
   **Resultado Esperado:** `LAB_APP_DATA_TS` con 2 archivos y ≥450 MB; `LAB_APP_IDX_TS` con 2 archivos y ≥200 MB

3. Verificar los datos cargados:

   ```sql
   -- Prueba 3: Contar filas en la tabla de prueba
   SELECT COUNT(*) AS total_filas FROM system.lab_empleados;
   ```
   **Resultado Esperado:** `TOTAL_FILAS = 10000`

4. Verificar la configuración de undo:

   ```sql
   -- Prueba 4: Verificar parámetros de undo
   SELECT name, value
   FROM v$parameter
   WHERE name IN ('undo_tablespace', 'undo_retention', 'undo_management');
   ```
   **Resultado Esperado:** `undo_tablespace = UNDOTBS2`, `undo_retention = 1800`, `undo_management = AUTO`

5. Verificar tempfiles:

   ```sql
   -- Prueba 5: Verificar tempfiles
   SELECT tablespace_name, COUNT(*) AS num_tempfiles,
          ROUND(SUM(bytes)/1024/1024, 2) AS total_mb
   FROM dba_temp_files
   GROUP BY tablespace_name;
   ```
   **Resultado Esperado:** `TEMP` con 2 tempfiles

6. Verificar la jerarquía de almacenamiento completa:

   ```sql
   -- Prueba 6: Confirmar segmentos y extents
   SELECT segment_name, segment_type, tablespace_name,
          ROUND(bytes/1024/1024, 3) AS mb, extents
   FROM dba_segments
   WHERE owner = 'SYSTEM'
     AND segment_name IN ('LAB_EMPLEADOS', 'IDX_LAB_EMP_DEPTO', 'PK_LAB_EMPLEADOS')
   ORDER BY segment_name;
   ```
   **Resultado Esperado:** 3 segmentos con extents > 0

---

## Solución de Problemas

### Problema 1: Error ORA-01119 al Crear Datafile — Archivo Ya Existe

**Síntomas:**
- Al ejecutar `CREATE TABLESPACE` o `ALTER TABLESPACE ADD DATAFILE` aparece:
  `ORA-01119: error creating database file '/u01/oradata/ORCL/lab_app_data01.dbf'`
  `ORA-27038: created file already exists`

**Causa:**
El archivo físico ya existe en el sistema de archivos (posiblemente de una ejecución anterior del script o de una práctica previa no limpiada correctamente).

**Solución:**

```bash
# Desde el terminal Linux, verificar si el archivo existe
ls -lh /u01/oradata/ORCL/lab_*.dbf

# Si el archivo existe pero el tablespace NO existe en Oracle, eliminar el archivo
rm /u01/oradata/ORCL/lab_app_data01.dbf

# Si tanto el archivo como el tablespace existen, primero eliminar el tablespace
# y luego el archivo (o usar INCLUDING CONTENTS AND DATAFILES)
```

```sql
-- Si el tablespace existe pero está en estado incompleto, eliminarlo primero
DROP TABLESPACE lab_app_data_ts INCLUDING CONTENTS AND DATAFILES;

-- Luego recrearlo
CREATE TABLESPACE lab_app_data_ts
  DATAFILE '/u01/oradata/ORCL/lab_app_data01.dbf'
  SIZE 200M
  AUTOEXTEND ON NEXT 100M MAXSIZE 2G
  EXTENT MANAGEMENT LOCAL AUTOALLOCATE
  SEGMENT SPACE MANAGEMENT AUTO;
```

---

### Problema 2: Error ORA-30012 al Crear UNDO TABLESPACE — Nombre Duplicado

**Síntomas:**
- Al ejecutar `CREATE UNDO TABLESPACE undotbs2` aparece:
  `ORA-30012: undo tablespace 'UNDOTBS2' cannot be created`
  o `ORA-01543: tablespace 'UNDOTBS2' already exists`

**Causa:**
El tablespace de undo `UNDOTBS2` ya fue creado en una ejecución anterior de la práctica.

**Solución:**

```sql
-- Verificar si UNDOTBS2 ya existe
SELECT tablespace_name, status, contents FROM dba_tablespaces
WHERE tablespace_name = 'UNDOTBS2';

-- Si existe y está activo como tablespace de undo actual, primero cambiar a UNDOTBS1
ALTER SYSTEM SET UNDO_TABLESPACE = UNDOTBS1 SCOPE = BOTH;

-- Luego eliminar UNDOTBS2
DROP TABLESPACE undotbs2 INCLUDING CONTENTS AND DATAFILES;

-- Y recrearlo
CREATE UNDO TABLESPACE undotbs2
  DATAFILE '/u01/oradata/ORCL/undotbs2_01.dbf'
  SIZE 200M
  AUTOEXTEND ON NEXT 50M MAXSIZE 1G
  RETENTION GUARANTEE;
```

---

### Problema 3: Error ORA-01552 al Insertar Datos — Tablespace de Undo Sin Espacio

**Síntomas:**
- Durante la inserción masiva de 10,000 filas aparece:
  `ORA-01552: cannot use system rollback segment for non-system tablespace 'LAB_APP_DATA_TS'`
  o `ORA-30036: unable to extend segment by 8 in undo tablespace 'UNDOTBS1'`

**Causa:**
El tablespace de undo no tiene suficiente espacio para la transacción grande, o `AUTOEXTEND` está deshabilitado y el tablespace se llenó.

**Solución:**

```sql
-- Verificar espacio libre en el tablespace de undo activo
SELECT
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024, 2) AS free_mb
FROM dba_free_space
WHERE tablespace_name = (SELECT value FROM v$parameter WHERE name = 'undo_tablespace')
GROUP BY tablespace_name;

-- Si no hay espacio, agregar un datafile al tablespace de undo activo
ALTER TABLESPACE undotbs1
  ADD DATAFILE '/u01/oradata/ORCL/undotbs1_02.dbf'
  SIZE 200M AUTOEXTEND ON NEXT 50M MAXSIZE 1G;
```

```sql
-- Si el error persiste, dividir la inserción en lotes más pequeños
-- (ya está implementado en el script con COMMIT cada 1000 filas)
-- Verificar que el bloque PL/SQL incluye el COMMIT intermedio
BEGIN
  FOR i IN 1..10000 LOOP
    INSERT INTO system.lab_empleados (emp_id, nombre, salario, depto_id, descripcion)
    VALUES (i, 'Empleado_' || LPAD(i,5,'0'),
            ROUND(DBMS_RANDOM.VALUE(30000,150000),2),
            MOD(i,20)+1,
            RPAD('Desc ' || i, 200, 'X'));
    IF MOD(i, 500) = 0 THEN COMMIT; END IF;  -- Commit más frecuente
  END LOOP;
  COMMIT;
END;
/
```

---

### Problema 4: El Tablespace OFFLINE No Puede Volver a ONLINE

**Síntomas:**
- Al ejecutar `ALTER TABLESPACE lab_uniform_ts ONLINE` aparece:
  `ORA-01113: file N needs media recovery`
  `ORA-01110: data file N: '/u01/oradata/ORCL/lab_uniform_ts01.dbf'`

**Causa:**
El tablespace fue puesto `OFFLINE IMMEDIATE` (o hubo un fallo durante el modo offline) y los datafiles necesitan recuperación de medios antes de poder volver a `ONLINE`.

**Solución:**

```sql
-- Si la base de datos está en modo ARCHIVELOG, ejecutar recuperación
RECOVER TABLESPACE lab_uniform_ts;

-- Después de la recuperación exitosa, poner ONLINE
ALTER TABLESPACE lab_uniform_ts ONLINE;
```

```sql
-- Si no hay archivos de archive log disponibles y los datos no son críticos:
-- (SOLO para tablespaces de práctica, NUNCA en producción)
-- Opción 1: Eliminar y recrear el tablespace
DROP TABLESPACE lab_uniform_ts INCLUDING CONTENTS AND DATAFILES;

CREATE TABLESPACE lab_uniform_ts
  DATAFILE '/u01/oradata/ORCL/lab_uniform_ts01.dbf'
  SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE 500M
  EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M
  SEGMENT SPACE MANAGEMENT AUTO;
```

---

### Problema 5: Error al Acceder a V$ASM_DISKGROUP — Instancia ASM No Disponible

**Síntomas:**
- Al consultar `V$ASM_DISKGROUP` aparece:
  `ORA-04045: errors during recompilation/revalidation of SYS.V_$ASM_DISKGROUP`
  o la consulta retorna 0 filas sin error

**Causa:**
No hay una instancia de Oracle Grid Infrastructure / ASM instalada en el entorno de práctica. Las vistas `V$ASM_*` existen en la base de datos pero no tienen datos porque no hay instancia ASM conectada.

**Solución:**

```sql
-- Verificar si hay instancia ASM registrada
SELECT inst_id, instance_name, status
FROM gv$instance
WHERE instance_name LIKE '+ASM%';
-- Si retorna 0 filas, no hay ASM instalado

-- Alternativa: Explorar la estructura de las vistas ASM sin datos
-- Esto es válido para el aprendizaje conceptual
DESCRIBE v$asm_diskgroup;
DESCRIBE v$asm_disk;
DESCRIBE v$asm_file;
DESCRIBE v$asm_operation;

-- Consultar la documentación de referencia de las vistas
SELECT column_name, data_type, comments
FROM dict_columns
WHERE table_name = 'V$ASM_DISKGROUP'
ORDER BY column_id;
```

```bash
# Para verificar definitivamente si Grid Infrastructure está instalado:
ls /u01/app/grid/ 2>/dev/null || echo "Grid Infrastructure no instalado en /u01/app/grid"
ls /u01/app/19.3.0/grid/ 2>/dev/null || echo "Grid Infrastructure no instalado en /u01/app/19.3.0/grid"
```

> **Nota para el instructor:** Si el entorno no tiene ASM, esta sección puede completarse de forma conceptual documentando la estructura de las vistas y los comandos `asmcmd` que se usarían. El objetivo de aprendizaje se cumple al comprender la arquitectura ASM aunque no se ejecuten los comandos en vivo.

---

## Limpieza

Al finalizar la práctica, ejecutar el siguiente script de limpieza para liberar espacio en disco y dejar el entorno en estado limpio para prácticas posteriores:

```sql
-- ============================================================
-- SCRIPT DE LIMPIEZA - LAB 04-00-01
-- ADVERTENCIA: Ejecutar SOLO al finalizar la práctica
-- ============================================================

-- Conectarse como SYSDBA
-- sqlplus / as sysdba

-- Paso 1: Restaurar el tablespace de undo original antes de eliminar UNDOTBS2
ALTER SYSTEM SET UNDO_RETENTION = 900 SCOPE = BOTH;
ALTER SYSTEM SET UNDO_TABLESPACE = UNDOTBS1 SCOPE = BOTH;

-- Esperar 30 segundos para que las transacciones activas en UNDOTBS2 terminen
-- (En entorno de práctica sin transacciones activas, es inmediato)

-- Paso 2: Eliminar tabla y objetos de práctica
DROP TABLE system.lab_empleados CASCADE CONSTRAINTS PURGE;

-- Paso 3: Eliminar tablespace de undo adicional
DROP TABLESPACE undotbs2 INCLUDING CONTENTS AND DATAFILES;

-- Paso 4: Eliminar tablespaces de práctica (en orden para evitar dependencias)
DROP TABLESPACE lab_app_idx_ts INCLUDING CONTENTS AND DATAFILES;
DROP TABLESPACE lab_app_data_ts INCLUDING CONTENTS AND DATAFILES;
DROP TABLESPACE lab_autoalloc_ts INCLUDING CONTENTS AND DATAFILES;
DROP TABLESPACE lab_uniform_ts INCLUDING CONTENTS AND DATAFILES;

-- Paso 5: Eliminar el tempfile adicional del tablespace TEMP
-- (Los tempfiles se eliminan con el comando ALTER TABLESPACE)
ALTER TABLESPACE temp DROP TEMPFILE '/u01/oradata/ORCL/temp02.dbf';

-- Paso 6: Verificar que la limpieza fue exitosa
SELECT tablespace_name, status FROM dba_tablespaces
WHERE tablespace_name LIKE 'LAB_%' OR tablespace_name = 'UNDOTBS2';
-- Debe retornar 0 filas

SELECT tablespace_name, file_name FROM dba_temp_files;
-- Debe retornar solo temp01.dbf

SHOW PARAMETER undo_tablespace;
-- Debe mostrar UNDOTBS1
```

```bash
# Verificar que los archivos físicos fueron eliminados del sistema operativo
ls -lh /u01/oradata/ORCL/lab_*.dbf 2>/dev/null || echo "Archivos de práctica eliminados correctamente"
ls -lh /u01/oradata/ORCL/undotbs2*.dbf 2>/dev/null || echo "Archivo UNDOTBS2 eliminado correctamente"
ls -lh /u01/oradata/ORCL/temp02.dbf 2>/dev/null || echo "Tempfile adicional eliminado correctamente"

# Verificar espacio liberado
df -h /u01
```

> ⚠️ **Advertencia:** El comando `DROP TABLESPACE ... INCLUDING CONTENTS AND DATAFILES` elimina permanentemente tanto los objetos de Oracle como los archivos físicos del disco. Esta operación es **irreversible**. Asegurarse de que no hay datos importantes en los tablespaces de práctica antes de ejecutar la limpieza.

> ⚠️ **Advertencia:** No eliminar los tablespaces del sistema (`SYSTEM`, `SYSAUX`, `UNDOTBS1`, `TEMP`, `USERS`). Solo eliminar los tablespaces creados durante esta práctica (prefijo `LAB_` y `UNDOTBS2`).

---

## Resumen

### Lo que Lograste

- **Creación y configuración de tablespaces:** Creaste 4 tablespaces permanentes con diferentes configuraciones de gestión de extensiones (`AUTOALLOCATE` vs `UNIFORM SIZE`) y configuraste `AUTOEXTEND` con límites de crecimiento máximo, siguiendo las mejores prácticas de separación de datos e índices
- **Gestión de datafiles:** Practicaste las operaciones fundamentales de administración de datafiles: agregar nuevos datafiles a tablespaces existentes, redimensionar datafiles con `ALTER DATABASE DATAFILE RESIZE` y modificar la configuración de `AUTOEXTEND` en datafiles existentes
- **Exploración de la jerarquía de almacenamiento:** Creaste objetos reales (tabla e índice) con datos y exploraste la jerarquía completa tablespace → segmento → extent → bloque usando las vistas `DBA_SEGMENTS`, `DBA_EXTENTS` y `DBA_TABLES`
- **Gestión del Undo Tablespace:** Creaste un tablespace de undo adicional (`UNDOTBS2`), cambiaste el tablespace de undo activo en caliente, configuraste `UNDO_RETENTION` y monitoreaste el estado con `V$UNDOSTAT` y `V$ROLLSTAT`
- **Gestión del Tablespace Temporal:** Agregaste un segundo tempfile al tablespace `TEMP` y monitoreaste el uso del espacio temporal con `V$TEMP_SPACE_HEADER` y `V$TEMPSEG_USAGE`
- **Cambio de estados de tablespaces:** Practicaste los modos `READ ONLY`, `OFFLINE` y `ONLINE`, comprendiendo cuándo usar cada uno en escenarios reales de mantenimiento
- **Exploración de ASM:** Exploraste las vistas `V$ASM_DISKGROUP`, `V$ASM_DISK` y `V$ASM_FILE` y/o los comandos `asmcmd` para comprender la arquitectura de Automatic Storage Management

### Conceptos Clave Aprendidos

- **Separación lógica/física:** Los tablespaces son la abstracción lógica que Oracle presenta a las aplicaciones y DBAs; los datafiles son la realidad física en el disco. Esta separación permite administrar el almacenamiento sin afectar las aplicaciones
- **AUTOALLOCATE vs UNIFORM SIZE:** `AUTOALLOCATE` es flexible y eficiente para objetos de tamaño variable; `UNIFORM SIZE` reduce la fragmentación cuando los objetos tienen tamaños similares. Ambos son superiores al obsoleto Dictionary Managed
- **Jerarquía de almacenamiento:** Tablespace → Segmento → Extent → Bloque Oracle. Cada nivel tiene su propósito: el tablespace organiza objetos, el segmento es la unidad de un objeto, el extent es la unidad de asignación de espacio, y el bloque es la unidad mínima de I/O
- **Gestión del Undo:** El tablespace de undo es crítico para la consistencia de lectura y la recuperación de transacciones. `UNDO_RETENTION` debe configurarse según la duración de las consultas más largas del sistema
- **ASM como abstracción adicional:** ASM añade otra capa de abstracción sobre el sistema de archivos del sistema operativo, gestionando automáticamente el striping, mirroring y balanceo de carga entre discos

### Próximos Pasos

- Continuar con la **Práctica 04-00-02: Segmentos, Extents y Bloques Oracle** para profundizar en la estructura interna de los tablespaces y aprender a diagnosticar fragmentación y problemas de espacio a nivel de bloque
- Explorar la **gestión de tablespaces bigfile** (`BIGFILE TABLESPACE`) como alternativa para bases de datos muy grandes donde un único datafile de gran tamaño es preferible a múltiples datafiles pequeños
- Investigar la **compresión de tablespaces** (`COMPRESS FOR OLTP`) como técnica para reducir el espacio en disco en entornos con datos altamente repetitivos
- Revisar la documentación de Oracle sobre **Automatic Segment Space Management (ASSM)** para comprender cómo Oracle gestiona el espacio libre dentro de los segmentos usando bitmaps de bloque

---

## Recursos Adicionales

- **Oracle Database Administrator's Guide 19c — Managing Tablespaces** — Referencia completa para administración de tablespaces, datafiles y operaciones de mantenimiento: [https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tablespaces.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tablespaces.html)

- **Oracle Database Administrator's Guide 19c — Managing Undo** — Guía completa para configuración y gestión del espacio de undo, incluyendo cálculo del tamaño óptimo de undo tablespace: [https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-undo.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-undo.html)

- **Oracle Database Storage Administrator's Guide — ASM** — Documentación completa de Automatic Storage Management incluyendo configuración de disk groups, redundancia y administración: [https://docs.oracle.com/en/database/oracle/oracle-database/19/ostmg/](https://docs.oracle.com/en/database/oracle/oracle-database/19/ostmg/)

- **Oracle Database Reference 19c — V$UNDOSTAT** — Descripción detallada de la vista `V$UNDOSTAT` y cómo interpretar sus columnas para diagnóstico de problemas de undo: [https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-UNDOSTAT.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-UNDOSTAT.html)

- **Oracle Database Reference 19c — DBA_TABLESPACES, DBA_DATA_FILES, DBA_SEGMENTS, DBA_EXTENTS** — Referencia de las vistas del diccionario de datos más importantes para administración de almacenamiento: [https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)

- **Oracle Support Note 1551288.1 — Tablespace Space Management Best Practices** — Disponible en My Oracle Support (MOS) para suscriptores, cubre mejores prácticas para gestión de espacio en tablespaces de producción
