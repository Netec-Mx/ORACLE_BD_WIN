# Lab 01-00-01: Exploración de la Arquitectura Fundamental de Oracle Database 19c

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 45 minutos |
| **Complejidad** | Principiante |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 1.1 – Conceptos Fundamentales |

---

## Descripción General

En esta práctica explorarás de forma activa los componentes arquitectónicos fundamentales de una instancia Oracle Database 19c en ejecución. Utilizando vistas dinámicas de rendimiento (vistas `V$`) y comandos del sistema operativo Linux, verificarás la existencia y el estado real de cada capa de la arquitectura Oracle: memoria (SGA y PGA), procesos de fondo y estructuras de almacenamiento físico.

Esta exploración te permitirá conectar los conceptos teóricos de la Lección 1.1 con la realidad operativa de una base de datos activa, desarrollando la habilidad de "leer" el estado de una instancia Oracle a través de sus vistas dinámicas, una competencia esencial para cualquier DBA.

---

## Objetivos de Aprendizaje

Al completar esta práctica, serás capaz de:

- [ ] Consultar e interpretar las vistas `V$INSTANCE` y `V$DATABASE` para identificar el estado y características de la instancia activa
- [ ] Explorar la configuración de memoria SGA y PGA mediante las vistas `V$SGA`, `V$SGAINFO` y parámetros de inicialización
- [ ] Identificar y verificar los procesos de fondo activos de Oracle correlacionando `V$BGPROCESS` con los procesos del sistema operativo Linux
- [ ] Examinar las estructuras de almacenamiento físico (datafiles, control files, redo log files) mediante vistas `V$DATAFILE`, `V$CONTROLFILE` y `V$LOGFILE`
- [ ] Utilizar el comando `SHOW PARAMETER` y la vista `V$PARAMETER` para consultar parámetros de configuración de la instancia

---

## Prerrequisitos

### Conocimientos Requeridos

- Comprensión de los conceptos de instancia Oracle vs. base de datos (Lección 1.1)
- Conocimientos básicos de SQL: sentencia `SELECT`, cláusulas `WHERE`, `ORDER BY` y funciones de fila
- Familiaridad con comandos básicos de Linux: `ps`, `ls`, `df`, `grep`
- Noción básica de navegación en terminal Linux

### Acceso Requerido

- Acceso SSH a la máquina virtual Oracle Linux 8.x donde está instalado Oracle Database 19c
- Credenciales del usuario `oracle` del sistema operativo (o equivalente con permisos DBA)
- Acceso a SQL*Plus como usuario `SYS` con rol `SYSDBA` o como usuario `SYSTEM`
- Instancia Oracle `ORCL` en estado `OPEN` (verificar antes de comenzar)

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | Intel Core i5 8va gen. o AMD Ryzen 5 (mínimo) |
| Memoria RAM | 16 GB mínimo (8 GB asignados a la VM) |
| Almacenamiento | 100 GB libres en disco (SSD recomendado) |
| Red | Adaptador de red funcional (loopback habilitado) |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) | Motor de base de datos principal |
| Oracle Linux | 8.x (8.7 u 8.8) | Sistema operativo del servidor |
| SQL*Plus | Incluido con Oracle 19c | Interfaz de línea de comandos para consultas |
| Oracle SQL Developer | 23.1 o superior | IDE gráfico (opcional, complementario) |
| PuTTY / Terminal SSH | 0.79+ / Nativo | Acceso remoto a la VM |

### Configuración Inicial

Antes de comenzar, verifica que el entorno esté listo ejecutando los siguientes comandos en la terminal Linux como usuario `oracle`:

```bash
# Verificar que el usuario oracle tiene las variables de entorno correctas
echo $ORACLE_HOME
echo $ORACLE_SID
echo $PATH

# Si las variables no están definidas, cargarlas manualmente
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH

# Verificar que el listener está activo
lsnrctl status

# Verificar que la instancia está en ejecución
ps -ef | grep pmon | grep -v grep
```

**Resultado esperado de la verificación inicial:**

```
ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1
ORACLE_SID  = ORCL
...
oracle    1234     1  0 10:00 ?  00:00:00 ora_pmon_ORCL
```

> ⚠️ **Importante:** Si el proceso `ora_pmon_ORCL` no aparece, la instancia no está activa. Notifica al instructor antes de continuar.

---

## Instrucciones Paso a Paso

### Paso 1: Conectarse a la Instancia Oracle con SQL*Plus

**Objetivo:** Establecer una conexión autenticada a la instancia Oracle como `SYSDBA` y verificar el estado general de la base de datos.

**Instrucciones:**

1. Abre una terminal SSH en tu máquina virtual Oracle Linux y cambia al usuario `oracle`:

   ```bash
   su - oracle
   ```

2. Inicia SQL*Plus conectándote como `SYSDBA` (autenticación del sistema operativo, sin contraseña en red):

   ```bash
   sqlplus / as sysdba
   ```

3. Una vez dentro de SQL*Plus, configura el formato de salida para mayor legibilidad:

   ```sql
   SET LINESIZE 120
   SET PAGESIZE 50
   SET COLSEP ' | '
   ```

4. Consulta la vista `V$INSTANCE` para obtener información básica de la instancia activa:

   ```sql
   SELECT INSTANCE_NUMBER,
          INSTANCE_NAME,
          HOST_NAME,
          VERSION,
          STATUS,
          DATABASE_STATUS,
          ARCHIVER
   FROM   V$INSTANCE;
   ```

5. Consulta la vista `V$DATABASE` para obtener información de la base de datos montada:

   ```sql
   SELECT DBID,
          NAME,
          DB_UNIQUE_NAME,
          CREATED,
          LOG_MODE,
          OPEN_MODE,
          CDB
   FROM   V$DATABASE;
   ```

**Resultado Esperado:**

```
INSTANCE_NUMBER | INSTANCE_NAME | HOST_NAME       | VERSION    | STATUS  | DATABASE_STATUS | ARCHIVER
--------------- | ------------- | --------------- | ---------- | ------- | --------------- | --------
              1 | ORCL          | oraclelinux8.lab| 19.0.0.0.0 | OPEN    | ACTIVE          | STOPPED

DBID       | NAME | DB_UNIQUE_NAME | CREATED   | LOG_MODE    | OPEN_MODE  | CDB
---------- | ---- | -------------- | --------- | ----------- | ---------- | ---
1234567890 | ORCL | orcl           | 01-JAN-23 | NOARCHIVELOG| READ WRITE | NO
```

**Verificación:**

- El campo `STATUS` debe mostrar `OPEN`
- El campo `DATABASE_STATUS` debe mostrar `ACTIVE`
- El campo `OPEN_MODE` debe mostrar `READ WRITE`
- Anota el valor del campo `LOG_MODE` (ARCHIVELOG o NOARCHIVELOG) — lo usarás más adelante

---

### Paso 2: Explorar las Estructuras de Memoria SGA

**Objetivo:** Consultar e interpretar la configuración y distribución de la SGA (System Global Area) de la instancia Oracle activa.

**Instrucciones:**

1. Obtén un resumen de alto nivel de la SGA con la vista `V$SGA`:

   ```sql
   SELECT NAME,
          VALUE,
          ROUND(VALUE/1024/1024, 2) AS VALOR_MB
   FROM   V$SGA
   ORDER BY VALUE DESC;
   ```

2. Obtén el detalle granular de todos los componentes de la SGA con `V$SGAINFO`:

   ```sql
   SELECT NAME,
          BYTES,
          ROUND(BYTES/1024/1024, 2) AS MEGAS,
          RESIZEABLE
   FROM   V$SGAINFO
   ORDER BY BYTES DESC;
   ```

3. Consulta los parámetros de memoria más importantes usando `SHOW PARAMETER`:

   ```sql
   SHOW PARAMETER sga_target
   SHOW PARAMETER pga_aggregate_target
   SHOW PARAMETER memory_target
   SHOW PARAMETER db_cache_size
   SHOW PARAMETER shared_pool_size
   ```

4. Obtén una vista consolidada de los parámetros de memoria desde `V$PARAMETER`:

   ```sql
   SELECT NAME,
          VALUE,
          DESCRIPTION
   FROM   V$PARAMETER
   WHERE  NAME IN ('sga_target',
                   'pga_aggregate_target',
                   'memory_target',
                   'db_cache_size',
                   'shared_pool_size',
                   'large_pool_size',
                   'java_pool_size')
   ORDER BY NAME;
   ```

**Resultado Esperado:**

```
-- V$SGA (valores aproximados, dependen de la configuración)
NAME                      VALUE       VALOR_MB
------------------------- ----------- --------
Fixed Size                  9137152       8.71
Redo Buffers                5369856       5.12
Database Buffers         1275068416    1216.00
Variable Size             956301312     912.00

-- V$SGAINFO (fragmento)
NAME                            BYTES       MEGAS  RES
------------------------------- ----------- ------ ---
Buffer Cache Size          1275068416    1216.00 Yes
Shared Pool Size            956301312     912.00 Yes
Large Pool Size              16777216      16.00 Yes
Java Pool Size               16777216      16.00 Yes
...
```

**Verificación:**

- Confirma que el campo `RESIZEABLE` aparece como `Yes` para los componentes principales (Buffer Cache, Shared Pool) — esto indica que Oracle puede ajustar automáticamente estos componentes
- Anota el tamaño del **Buffer Cache** en MB — es el componente más grande de la SGA y el que más impacta el rendimiento de consultas
- Si `sga_target` tiene un valor mayor a 0, significa que la gestión automática de memoria (AMM o ASMM) está habilitada

---

### Paso 3: Verificar los Procesos de Fondo de Oracle

**Objetivo:** Identificar los procesos de fondo activos de Oracle tanto desde las vistas dinámicas como desde el sistema operativo Linux, correlacionando ambas fuentes de información.

**Instrucciones:**

1. Consulta la vista `V$BGPROCESS` para ver todos los procesos de fondo conocidos por Oracle y su estado:

   ```sql
   SELECT PADDR,
          NAME,
          DESCRIPTION,
          ERROR
   FROM   V$BGPROCESS
   WHERE  PADDR <> '00'
   ORDER BY NAME;
   ```

2. Para ver solo los procesos de fondo más importantes con mayor detalle, usa esta consulta combinada con `V$PROCESS`:

   ```sql
   SELECT p.SPID        AS PID_SO,
          b.NAME        AS PROCESO_ORACLE,
          b.DESCRIPTION AS DESCRIPCION
   FROM   V$BGPROCESS b
   JOIN   V$PROCESS   p ON b.PADDR = p.ADDR
   WHERE  b.NAME IN ('PMON','SMON','DBW0','LGWR','CKPT','MMON','MMNL','ARC0')
   ORDER BY b.NAME;
   ```

3. **Sin cerrar SQL*Plus**, abre una segunda terminal SSH y verifica los mismos procesos desde el sistema operativo Linux:

   ```bash
   # Ver todos los procesos de Oracle para la instancia ORCL
   ps -ef | grep ora_ | grep ORCL | grep -v grep | sort
   ```

4. Compara el PID del sistema operativo (`SPID` de la consulta anterior) con los PIDs que aparecen en la salida del comando `ps`. Busca un proceso específico, por ejemplo PMON:

   ```bash
   # Buscar específicamente el proceso PMON
   ps -ef | grep ora_pmon_ORCL | grep -v grep
   ```

5. Regresa a SQL*Plus y verifica la memoria utilizada por los procesos Oracle con `V$PROCESS`:

   ```sql
   SELECT SPID,
          PROGRAM,
          BACKGROUND,
          ROUND(PGA_USED_MEM/1024/1024, 2)   AS PGA_USADA_MB,
          ROUND(PGA_ALLOC_MEM/1024/1024, 2)  AS PGA_ASIGNADA_MB
   FROM   V$PROCESS
   WHERE  BACKGROUND = 1
   ORDER BY PGA_ALLOC_MEM DESC
   FETCH FIRST 10 ROWS ONLY;
   ```

**Resultado Esperado:**

```
-- V$BGPROCESS (fragmento de procesos activos)
PADDR    | NAME | DESCRIPTION                                        | ERROR
-------- | ---- | -------------------------------------------------- | -----
60A12340 | CKPT | checkpoint                                         |     0
60A14560 | DBW0 | db writer process 0                               |     0
60A16780 | LGWR | Redo etc.                                          |     0
60A18900 | PMON | process cleanup                                    |     0
60A1AB20 | SMON | System Monitor Process                             |     0

-- ps -ef (fragmento)
oracle  1234  1  0 10:00 ?  00:00:00 ora_pmon_ORCL
oracle  1236  1  0 10:00 ?  00:00:02 ora_smon_ORCL
oracle  1238  1  0 10:00 ?  00:00:00 ora_dbw0_ORCL
oracle  1240  1  0 10:00 ?  00:00:01 ora_lgwr_ORCL
oracle  1242  1  0 10:00 ?  00:00:00 ora_ckpt_ORCL
```

**Verificación:**

- Confirma que los cinco procesos esenciales están activos: `PMON`, `SMON`, `DBW0` (o `DBW`), `LGWR`, `CKPT`
- Verifica que el `SPID` devuelto por `V$PROCESS` coincide con el PID que muestra el comando `ps` en el sistema operativo — esta correlación confirma que ambas vistas representan el mismo proceso físico
- Si `ARC0` aparece activo, significa que la base de datos está en modo `ARCHIVELOG`

---

### Paso 4: Examinar las Estructuras de Almacenamiento Físico

**Objetivo:** Localizar y describir los archivos físicos que conforman la base de datos Oracle: datafiles, control files y redo log files, utilizando las vistas dinámicas correspondientes.

**Instrucciones:**

1. Consulta los **datafiles** (archivos de datos) con la vista `V$DATAFILE`:

   ```sql
   SELECT FILE#,
          NAME,
          STATUS,
          ENABLED,
          ROUND(BYTES/1024/1024, 2)   AS TAMANIO_MB,
          ROUND(MAXBYTES/1024/1024, 2) AS MAX_MB
   FROM   V$DATAFILE
   ORDER BY FILE#;
   ```

2. Consulta los **control files** (archivos de control) con `V$CONTROLFILE`:

   ```sql
   SELECT STATUS,
          NAME,
          IS_RECOVERY_DEST_FILE,
          BLOCK_SIZE,
          FILE_SIZE_BLKS
   FROM   V$CONTROLFILE;
   ```

3. Consulta los **redo log files** (archivos de redo log) con `V$LOGFILE` y `V$LOG`:

   ```sql
   -- Información de los grupos de redo log
   SELECT l.GROUP#,
          l.MEMBERS,
          l.BYTES/1024/1024 AS TAMANIO_MB,
          l.STATUS,
          l.ARCHIVED
   FROM   V$LOG l
   ORDER BY l.GROUP#;
   ```

   ```sql
   -- Información de los miembros individuales de redo log
   SELECT lf.GROUP#,
          lf.MEMBER,
          lf.STATUS,
          lf.TYPE
   FROM   V$LOGFILE lf
   ORDER BY lf.GROUP#, lf.MEMBER;
   ```

4. Verifica la existencia física de estos archivos en el sistema operativo. Abre la segunda terminal y ejecuta:

   ```bash
   # Listar los datafiles (ajusta la ruta según tu instalación)
   ls -lh /u01/app/oracle/oradata/ORCL/*.dbf

   # Listar los control files
   ls -lh /u01/app/oracle/oradata/ORCL/*.ctl

   # Listar los redo log files
   ls -lh /u01/app/oracle/oradata/ORCL/*.log
   ```

5. Regresa a SQL*Plus y consulta el archivo de parámetros (SPFILE/PFILE):

   ```sql
   -- Verificar si se usa SPFILE o PFILE
   SHOW PARAMETER spfile

   -- Ver la ruta del SPFILE si está en uso
   SELECT VALUE FROM V$PARAMETER WHERE NAME = 'spfile';
   ```

**Resultado Esperado:**

```
-- V$DATAFILE (fragmento)
FILE# | NAME                                          | STATUS  | ENABLED    | TAMANIO_MB | MAX_MB
----- | --------------------------------------------- | ------- | ---------- | ---------- | ------
    1 | /u01/app/oracle/oradata/ORCL/system01.dbf     | SYSTEM  | READ WRITE |     880.00 | 32767.98
    2 | /u01/app/oracle/oradata/ORCL/sysaux01.dbf     | ONLINE  | READ WRITE |     620.00 | 32767.98
    3 | /u01/app/oracle/oradata/ORCL/undotbs01.dbf    | ONLINE  | READ WRITE |     200.00 | 32767.98
    4 | /u01/app/oracle/oradata/ORCL/users01.dbf      | ONLINE  | READ WRITE |       5.00 | 32767.98

-- V$CONTROLFILE
STATUS | NAME                                          | IS_REC | BLOCK_SIZE | FILE_SIZE_BLKS
------ | --------------------------------------------- | ------ | ---------- | --------------
       | /u01/app/oracle/oradata/ORCL/control01.ctl   | NO     |      16384 |            594
       | /u01/app/oracle/fast_recovery_area/ORCL/...  | NO     |      16384 |            594

-- V$LOG
GROUP# | MEMBERS | TAMANIO_MB | STATUS   | ARC
------ | ------- | ---------- | -------- | ---
     1 |       1 |     200.00 | INACTIVE | NO
     2 |       1 |     200.00 | CURRENT  | NO
     3 |       1 |     200.00 | INACTIVE | NO
```

**Verificación:**

- Confirma que existe al menos un datafile para el tablespace `SYSTEM` (siempre presente)
- Verifica que hay al menos 2 control files (Oracle recomienda multiplexarlos para redundancia)
- Confirma que hay al menos 2 grupos de redo log y que uno de ellos tiene estado `CURRENT` (el que está siendo escrito actualmente)
- Verifica que los archivos físicos listados por el comando `ls` coinciden con las rutas mostradas por las vistas `V$`

---

### Paso 5: Consultar Parámetros de Inicialización Clave

**Objetivo:** Utilizar `SHOW PARAMETER` y la vista `V$PARAMETER` para identificar y comprender los parámetros de configuración más relevantes de la instancia Oracle.

**Instrucciones:**

1. Muestra los parámetros relacionados con la identificación de la base de datos:

   ```sql
   SHOW PARAMETER db_name
   SHOW PARAMETER db_domain
   SHOW PARAMETER instance_name
   SHOW PARAMETER service_names
   ```

2. Consulta los parámetros relacionados con el almacenamiento de diagnóstico y logs:

   ```sql
   SHOW PARAMETER diagnostic_dest
   SHOW PARAMETER db_recovery_file_dest
   SHOW PARAMETER log_archive_dest_1
   ```

3. Realiza una consulta más amplia sobre parámetros relevantes para la arquitectura:

   ```sql
   SELECT NAME,
          VALUE,
          ISDEFAULT,
          ISMODIFIED,
          DESCRIPTION
   FROM   V$PARAMETER
   WHERE  NAME IN (
              'db_name',
              'instance_name',
              'db_block_size',
              'processes',
              'sessions',
              'undo_tablespace',
              'undo_management',
              'db_recovery_file_dest',
              'diagnostic_dest'
          )
   ORDER BY NAME;
   ```

4. Verifica el directorio de diagnóstico (ADR - Automatic Diagnostic Repository) en el sistema operativo:

   ```bash
   # Verificar el directorio de diagnóstico de Oracle
   ls -la /u01/app/oracle/diag/rdbms/orcl/ORCL/

   # Ver los logs de alerta recientes (últimas 20 líneas)
   tail -20 /u01/app/oracle/diag/rdbms/orcl/ORCL/trace/alert_ORCL.log
   ```

5. Vuelve a SQL*Plus y genera un reporte consolidado de la instancia para el reporte de laboratorio:

   ```sql
   -- Reporte consolidado de la instancia
   SELECT 'INSTANCIA'       AS COMPONENTE,
          INSTANCE_NAME     AS NOMBRE,
          STATUS            AS ESTADO,
          TO_CHAR(STARTUP_TIME, 'DD-MON-YYYY HH24:MI') AS INICIO
   FROM   V$INSTANCE
   UNION ALL
   SELECT 'BASE DE DATOS',
          NAME,
          OPEN_MODE,
          TO_CHAR(CREATED, 'DD-MON-YYYY HH24:MI')
   FROM   V$DATABASE;
   ```

**Resultado Esperado:**

```
-- SHOW PARAMETER db_name
NAME     TYPE   VALUE
-------- ------ -----
db_name  string ORCL

-- V$PARAMETER (fragmento)
NAME                   VALUE          ISDEFAULT ISMODIFIED
---------------------- -------------- --------- ----------
db_block_size          8192           TRUE      FALSE
db_name                ORCL           FALSE     FALSE
db_recovery_file_dest  /u01/app/...   FALSE     FALSE
diagnostic_dest        /u01/app/...   FALSE     FALSE
processes              300            FALSE     FALSE
sessions               472            FALSE     FALSE
undo_management        AUTO           TRUE      FALSE
undo_tablespace        UNDOTBS1       FALSE     FALSE

-- Reporte consolidado
COMPONENTE     NOMBRE  ESTADO      INICIO
-------------- ------- ----------- ----------------
INSTANCIA      ORCL    OPEN        15-JAN-2024 10:00
BASE DE DATOS  ORCL    READ WRITE  01-JAN-2023 08:30
```

**Verificación:**

- Confirma que `db_block_size` tiene un valor (típicamente 8192 bytes = 8 KB) — este es el tamaño del bloque Oracle básico
- Verifica que `undo_management` es `AUTO` — indica gestión automática de undo
- Confirma que `diagnostic_dest` apunta a un directorio existente en el sistema de archivos
- El archivo `alert_ORCL.log` debe mostrar mensajes de inicio de la instancia sin errores críticos (mensajes `ORA-` graves)

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] Se conectó exitosamente a SQL*Plus como `SYSDBA` y verificó que la instancia está en estado `OPEN`
- [ ] Consultó `V$INSTANCE` y `V$DATABASE` e identificó correctamente el nombre, versión y modo de apertura de la base de datos
- [ ] Ejecutó consultas sobre `V$SGA` y `V$SGAINFO` e identificó el tamaño del Buffer Cache y del Shared Pool
- [ ] Identificó al menos 5 procesos de fondo activos usando `V$BGPROCESS` y los correlacionó con los procesos del sistema operativo mediante `ps`
- [ ] Localizó y listó los datafiles, control files y redo log files usando las vistas `V$DATAFILE`, `V$CONTROLFILE` y `V$LOGFILE`
- [ ] Verificó la existencia física de los archivos de la base de datos en el sistema de archivos Linux
- [ ] Consultó parámetros de inicialización clave usando `SHOW PARAMETER` y `V$PARAMETER`

### Procedimiento de Prueba

1. Verifica el estado completo de la instancia ejecutando esta consulta de validación final:

   ```sql
   SELECT 'Instancia: ' || INSTANCE_NAME ||
          ' | Estado: ' || STATUS ||
          ' | Version: ' || VERSION AS RESUMEN_INSTANCIA
   FROM   V$INSTANCE;
   ```

   **Resultado Esperado:** `Instancia: ORCL | Estado: OPEN | Version: 19.0.0.0.0`

2. Verifica que el Buffer Cache está activo y tiene datos:

   ```sql
   SELECT COUNT(*) AS BLOQUES_EN_CACHE
   FROM   V$BH
   WHERE  STATUS != 'free';
   ```

   **Resultado Esperado:** Un número mayor a 0 (indica que hay bloques de datos cargados en memoria)

3. Verifica que todos los datafiles están en línea:

   ```sql
   SELECT COUNT(*) AS DATAFILES_OFFLINE
   FROM   V$DATAFILE
   WHERE  STATUS NOT IN ('SYSTEM', 'ONLINE');
   ```

   **Resultado Esperado:** `0` (ningún datafile debe estar offline en una instancia sana)

4. Confirma el conteo de procesos de fondo activos:

   ```sql
   SELECT COUNT(*) AS PROCESOS_BG_ACTIVOS
   FROM   V$BGPROCESS
   WHERE  PADDR <> '00';
   ```

   **Resultado Esperado:** Un número entre 20 y 50 procesos (varía según la configuración)

---

## Solución de Problemas

### Problema 1: Error ORA-01034 al Conectarse con SQL*Plus

**Síntomas:**
- Al ejecutar `sqlplus / as sysdba`, aparece el mensaje: `ORA-01034: ORACLE not available`
- El comando `ps -ef | grep pmon` no muestra ningún proceso `ora_pmon_ORCL`

**Causa:**
La instancia Oracle no está iniciada. El proceso PMON no existe, lo que indica que la instancia está completamente detenida o nunca fue iniciada.

**Solución:**

```bash
# Como usuario oracle, inicia la instancia
export ORACLE_SID=ORCL
sqlplus / as sysdba

# Dentro de SQL*Plus, inicia la instancia
STARTUP;

# Verifica el estado después del inicio
SELECT STATUS FROM V$INSTANCE;
```

---

### Problema 2: Las Variables de Entorno ORACLE_HOME u ORACLE_SID No Están Definidas

**Síntomas:**
- Al ejecutar `sqlplus`, aparece el error: `bash: sqlplus: command not found`
- El comando `echo $ORACLE_SID` devuelve una línea vacía

**Causa:**
Las variables de entorno del usuario `oracle` no están configuradas en la sesión actual. Esto ocurre cuando el archivo `.bash_profile` del usuario oracle no se ejecutó correctamente o la sesión se abrió sin cargar el perfil.

**Solución:**

```bash
# Cargar el perfil del usuario oracle manualmente
source ~/.bash_profile

# O definir las variables manualmente
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH

# Verificar que sqlplus ahora es accesible
which sqlplus
sqlplus -version
```

---

### Problema 3: La Vista V$BGPROCESS No Muestra Procesos Activos

**Síntomas:**
- La consulta a `V$BGPROCESS WHERE PADDR <> '00'` devuelve 0 filas
- La instancia aparece como `OPEN` en `V$INSTANCE`

**Causa:**
Es poco probable que ocurra si la instancia está abierta, pero puede deberse a un problema de permisos o a que se está consultando la vista sin los privilegios `SYSDBA` necesarios.

**Solución:**

```sql
-- Verificar el usuario actual de la sesión
SHOW USER;

-- Si no eres SYS o no tienes SYSDBA, reconectar correctamente
CONNECT / AS SYSDBA

-- Verificar nuevamente
SELECT COUNT(*) FROM V$BGPROCESS WHERE PADDR <> '00';

-- Si el problema persiste, verificar los procesos directamente en el SO
-- (en una segunda terminal)
-- ps -ef | grep ora_ | grep ORCL | wc -l
```

---

### Problema 4: Los Archivos Físicos No Se Encuentran en la Ruta Mostrada por V$DATAFILE

**Síntomas:**
- Las rutas mostradas en `V$DATAFILE` apuntan a `/u01/app/oracle/oradata/ORCL/`
- El comando `ls` en esa ruta devuelve "No such file or directory"

**Causa:**
La instalación de Oracle puede haberse realizado en un directorio diferente al estándar, o los datafiles pueden estar en una ruta personalizada definida durante la creación de la base de datos (DBCA).

**Solución:**

```bash
# Verificar la ruta real de los datafiles consultando la vista
# (dentro de SQL*Plus)
# SELECT NAME FROM V$DATAFILE;
# Toma nota de la ruta exacta mostrada

# Luego en el SO, navega a esa ruta
# Por ejemplo, si la ruta es /opt/oracle/oradata/ORCL:
ls -lh /opt/oracle/oradata/ORCL/

# También puedes buscar los datafiles en todo el sistema
find / -name "system01.dbf" -type f 2>/dev/null
```

---

## Limpieza

Esta práctica es de solo lectura (únicamente consultas `SELECT` y comandos de visualización), por lo que no se crearon objetos que deban eliminarse. Sin embargo, ejecuta los siguientes pasos de cierre ordenado:

```sql
-- Dentro de SQL*Plus, cerrar la sesión correctamente
EXIT;
```

```bash
# En la terminal Linux, verificar que no quedaron sesiones SQL*Plus activas
ps -ef | grep sqlplus | grep -v grep

# Si quedaron sesiones colgadas, terminarlas (reemplaza XXXX con el PID)
# kill -9 XXXX

# Opcional: limpiar el historial de comandos de SQL*Plus si contiene información sensible
rm -f ~/.sqlplus_history 2>/dev/null || true
```

> ⚠️ **Advertencia:** No ejecutes comandos `SHUTDOWN` o `STARTUP` a menos que el instructor lo indique explícitamente. La instancia debe permanecer activa para las prácticas siguientes del curso. Recuerda que las prácticas tienen dependencias progresivas.

---

## Resumen

### Lo Que Lograste

- Estableciste una conexión exitosa a Oracle Database 19c como `SYSDBA` usando SQL*Plus con autenticación del sistema operativo
- Consultaste e interpretaste las vistas `V$INSTANCE` y `V$DATABASE` para obtener información del estado y características de la instancia activa
- Exploraste la distribución de memoria de la SGA mediante `V$SGA` y `V$SGAINFO`, identificando el Buffer Cache, Shared Pool y otros componentes
- Identificaste los procesos de fondo activos de Oracle (PMON, SMON, DBWn, LGWR, CKPT) y los correlacionaste con los procesos del sistema operativo Linux
- Localizaste y describiste los archivos físicos de la base de datos: datafiles, control files y redo log files, verificando su existencia en el sistema de archivos
- Consultaste parámetros de inicialización clave usando `SHOW PARAMETER` y la vista `V$PARAMETER`

### Conceptos Clave Aprendidos

- **Instancia vs. Base de Datos:** La instancia (memoria + procesos en RAM) y la base de datos (archivos en disco) son entidades distintas pero interdependientes. Las vistas `V$INSTANCE` y `V$DATABASE` reflejan esta distinción.
- **Vistas V$ como ventana a la instancia:** Las vistas dinámicas de rendimiento (vistas `V$`) son la principal herramienta de un DBA para inspeccionar el estado interno de Oracle en tiempo real. Son "ventanas" a la memoria de la instancia.
- **Las tres capas de la arquitectura Oracle** son visibles y consultables: la memoria (V$SGA), los procesos (V$BGPROCESS) y el almacenamiento (V$DATAFILE, V$CONTROLFILE, V$LOGFILE).
- **Correlación SO-Oracle:** Los procesos de fondo de Oracle son procesos reales del sistema operativo. El campo `SPID` en `V$PROCESS` conecta el mundo Oracle con el mundo Linux.
- **Buffer Cache y rendimiento:** El Buffer Cache es el componente más grande de la SGA porque es el principal mecanismo de Oracle para evitar lecturas de disco, mejorando dramáticamente el rendimiento.

### Próximos Pasos

- Continúa con la **Lección 1.2** para profundizar en las estructuras internas de la SGA: Buffer Cache, Shared Pool, Large Pool, Java Pool y Redo Log Buffer
- En la **Lección 1.3** estudiarás en detalle las responsabilidades de cada proceso de fondo (DBWn, LGWR, CKPT, SMON, PMON, ARCn) y cómo interactúan entre sí
- En la **Lección 1.4** explorarás la jerarquía de almacenamiento lógico: tablespaces → segmentos → extents → bloques
- Considera habilitar el modo `ARCHIVELOG` en tu instancia (si aún no lo está) para prepararte para las prácticas de RMAN del Módulo 8

---

## Recursos Adicionales

- **Oracle Database Concepts 19c – Chapter 1 (Introduction to Oracle Database):** Documentación oficial que describe la arquitectura fundamental. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/)
- **Oracle Database Reference 19c – V$ Views:** Referencia completa de todas las vistas dinámicas `V$` con descripción de cada columna. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)
- **Oracle Database 2 Day DBA 19c – Getting Started:** Guía práctica para administradores con ejemplos de consultas a vistas `V$`. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admqs/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admqs/)
- **Oracle Learning Explorer – Oracle Database Foundations:** Curso introductorio gratuito sobre arquitectura Oracle. Disponible en [education.oracle.com](https://education.oracle.com)
- **Oracle Database Administrator's Guide 19c – Managing Processes:** Documentación detallada sobre procesos de fondo y su gestión. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admin/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/)
