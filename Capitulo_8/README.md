# Lab 08-00-01: Introducción al Cloud, Arquitectura Multitenant y Operaciones RMAN

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 8.1 – Conceptos de Cloud y Modelos de Implementación DBCS |

---

## Descripción General

Este laboratorio integra tres áreas fundamentales del administrador de bases de datos Oracle moderno: los conceptos de servicios cloud aplicados a Oracle Cloud Infrastructure (OCI), la gestión de la arquitectura Multitenant mediante Container Databases (CDB) y Pluggable Databases (PDB), y la ejecución de operaciones de respaldo y recuperación con Oracle Recovery Manager (RMAN). A través de ejercicios progresivos, el estudiante pasará de la comprensión conceptual del cloud al trabajo práctico con instancias Oracle locales que simulan los mismos principios arquitectónicos que se encontrarían en un entorno DBCS en la nube.

La relevancia práctica de este laboratorio radica en que las habilidades de gestión Multitenant y RMAN son exactamente las mismas que se utilizan en Oracle Database Cloud Service: comprender cómo funcionan localmente es el paso previo indispensable para operar con confianza en la nube.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Identificar y clasificar los modelos de servicio cloud (IaaS, PaaS, SaaS) aplicando cada uno a escenarios reales con Oracle Database Cloud Service (DBCS)
- [ ] Conectarse al CDB$ROOT como SYSDBA y ejecutar consultas sobre las vistas V$PDBS y CDB_PDBS para inspeccionar el estado de las Pluggable Databases
- [ ] Abrir, cerrar y crear una PDB nueva a partir de PDB$SEED, y gestionar usuarios locales versus usuarios comunes en la arquitectura Multitenant
- [ ] Configurar parámetros RMAN (canal de respaldo, política de retención) y ejecutar respaldos completos e incrementales de la base de datos
- [ ] Simular la pérdida de un datafile y realizar una recuperación completa utilizando los comandos RESTORE DATABASE y RECOVER DATABASE de RMAN

---

## Prerrequisitos

### Conocimiento Requerido

- Arquitectura física de Oracle Database: datafiles, redo logs, control files y archive logs
- Conceptos básicos de backup y recovery: full backup, incremental backup, point-in-time recovery
- Familiaridad con SQL*Plus: conexión como SYSDBA, comandos STARTUP y SHUTDOWN
- Comprensión básica de conceptos cloud (IaaS, PaaS, SaaS) a nivel conceptual
- Conocimiento de comandos básicos de Linux (navegación de directorios, permisos, variables de entorno)

### Acceso Requerido

- Acceso a Oracle Database 19c o 21c instalada en Oracle Linux 8.x (máquina virtual o servidor dedicado)
- Usuario del sistema operativo `oracle` con variables de entorno configuradas (`ORACLE_SID`, `ORACLE_HOME`, `PATH`)
- La base de datos **debe estar en modo ARCHIVELOG** antes de iniciar la Parte 3 del laboratorio
- Acceso SSH a la máquina virtual (PuTTY o terminal nativo)
- Espacio en disco mínimo de 10 GB disponible para respaldos RMAN en el directorio `/home/oracle/backup_rman`

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | Intel Core i5 8va gen o AMD Ryzen 5 (mínimo) |
| RAM | 16 GB mínimo (8 GB asignados a la VM Oracle Linux) |
| Almacenamiento | 20 GB libres para datafiles + 10 GB para respaldos RMAN |
| Red | Adaptador loopback activo para conexiones locales Oracle Net |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) o 21c | Motor principal de base de datos |
| Oracle Linux | 8.x (8.7 o 8.8) | Sistema operativo huésped |
| SQL*Plus | Incluido con Oracle DB | Administración y consultas |
| Oracle RMAN | Incluido con Oracle DB | Respaldo y recuperación |
| PuTTY / Terminal SSH | PuTTY 0.79+ o nativo | Acceso remoto a la VM |

### Configuración Inicial

Antes de iniciar el laboratorio, conectarse a la máquina virtual Oracle Linux y verificar el entorno:

```bash
# Conectarse a la VM por SSH
ssh oracle@192.168.56.101

# Verificar variables de entorno Oracle
echo "ORACLE_SID: $ORACLE_SID"
echo "ORACLE_HOME: $ORACLE_HOME"
echo "PATH: $PATH"

# Si las variables no están configuradas, cargarlas manualmente
export ORACLE_SID=ORCL
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export PATH=$ORACLE_HOME/bin:$PATH

# Verificar que la instancia Oracle está activa
ps -ef | grep pmon | grep -v grep

# Crear directorio para respaldos RMAN
mkdir -p /home/oracle/backup_rman
chmod 755 /home/oracle/backup_rman

# Verificar espacio disponible en disco
df -h /home/oracle/backup_rman
df -h /u01
```

**Salida esperada de verificación:**

```
ORACLE_SID: ORCL
ORACLE_HOME: /u01/app/oracle/product/19.3.0/dbhome_1
oracle    1234     1  0 08:00 ?  00:00:00 ora_pmon_ORCL
```

---

## Instrucciones Paso a Paso

### Paso 1: Análisis Conceptual de Modelos Cloud y Oracle DBCS

**Objetivo:** Identificar y clasificar los modelos de servicio cloud (IaaS, PaaS, SaaS) en el contexto de Oracle Database Cloud Service, relacionando cada modelo con su nivel de responsabilidad administrativa.

**Instrucciones:**

1. Leer detenidamente la siguiente tabla de referencia de modelos de servicio Oracle Cloud y completar el análisis solicitado:

   | Modelo | Servicio Oracle Ejemplo | Gestiona Oracle | Gestiona el DBA |
   |--------|------------------------|-----------------|-----------------|
   | IaaS | Oracle Cloud VM (Compute) | Hardware, red, virtualización | SO, Oracle DB, parches, datos |
   | PaaS | Oracle DBCS / Autonomous DB | Hardware, SO, parches de SO | Oracle DB config, esquemas, datos |
   | SaaS | Oracle ERP Cloud | Todo el stack | Solo configuración funcional |

2. Conectarse a SQL*Plus como SYSDBA y verificar la versión de Oracle instalada, lo que representa la misma versión disponible en Oracle DBCS:

   ```bash
   sqlplus / as sysdba
   ```

   ```sql
   -- Verificar versión Oracle (equivalente a lo que se aprovisiona en DBCS)
   SELECT BANNER_FULL FROM V$VERSION;

   -- Verificar modo de operación de la base de datos
   SELECT NAME, DB_UNIQUE_NAME, CDB, CON_ID FROM V$DATABASE;

   -- Verificar si la base de datos es Multitenant (CDB)
   -- CDB = YES indica arquitectura Container Database, usada en Oracle DBCS
   SELECT CDB, CON_ID, CURRENT_SCN FROM V$DATABASE;
   ```

3. Registrar los resultados en el reporte de laboratorio y responder las siguientes preguntas de análisis:

   ```sql
   -- Consulta adicional: identificar características que coinciden con Oracle DBCS
   SELECT NAME, VALUE 
   FROM V$PARAMETER 
   WHERE NAME IN ('db_name', 'enable_pluggable_database', 
                  'log_archive_dest_1', 'db_recovery_file_dest');
   ```

**Salida Esperada:**

```
BANNER_FULL
-----------------------------------------------------------------------
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production

NAME        DB_UNIQUE_NAME  CDB  CON_ID
----------- --------------- ---- ------
ORCL        ORCL            YES       0

NAME                          VALUE
----------------------------- ------------------------------------
db_name                       ORCL
enable_pluggable_database     TRUE
log_archive_dest_1            USE_DB_RECOVERY_FILE_DEST
db_recovery_file_dest         /u01/app/oracle/fast_recovery_area
```

**Verificación:**

- El campo `CDB = YES` confirma que la base de datos es una Container Database, equivalente a la arquitectura usada en Oracle DBCS
- `enable_pluggable_database = TRUE` confirma soporte Multitenant activo

---

### Paso 2: Exploración de la Arquitectura Multitenant – Consulta de CDB y PDBs

**Objetivo:** Conectarse al CDB$ROOT como SYSDBA y utilizar las vistas V$PDBS y CDB_PDBS para obtener información completa sobre las Pluggable Databases existentes en el contenedor.

**Instrucciones:**

1. Desde SQL*Plus (ya conectado como SYSDBA), verificar el contenedor actual y listar todas las PDBs disponibles:

   ```sql
   -- Verificar en qué contenedor estamos actualmente
   SHOW CON_NAME;
   SHOW CON_ID;

   -- Listar todas las PDBs usando V$PDBS
   SELECT CON_ID, NAME, OPEN_MODE, RESTRICTED, CREATION_SCN
   FROM V$PDBS
   ORDER BY CON_ID;
   ```

2. Consultar la vista CDB_PDBS para obtener información adicional de administración:

   ```sql
   -- Consulta detallada con CDB_PDBS
   SELECT CON_ID, PDB_NAME, STATUS, LOGGING, FORCE_LOGGING,
          CON_UID, GUID
   FROM CDB_PDBS
   ORDER BY CON_ID;

   -- Ver el estado actual de todas las PDBs con sus modos de apertura
   SELECT NAME, 
          CON_ID,
          OPEN_MODE,
          RESTRICTED,
          TO_CHAR(OPEN_TIME, 'DD-MON-YYYY HH24:MI:SS') AS OPEN_TIME
   FROM V$PDBS
   ORDER BY CON_ID;
   ```

3. Verificar los servicios de red asociados a cada PDB (relevante para conectividad DBCS):

   ```sql
   -- Servicios registrados por cada PDB
   SELECT NAME, PDB, NETWORK_NAME 
   FROM V$SERVICES
   ORDER BY PDB, NAME;
   ```

**Salida Esperada:**

```
CON_NAME
-----------------------------
CDB$ROOT

CON_ID  NAME        OPEN_MODE   RESTRICTED  CREATION_SCN
------  ----------- ----------- ----------  ------------
     2  PDB$SEED    READ ONLY   NO          1
     3  ORCLPDB     READ WRITE  NO          1234567

CON_ID  PDB_NAME    STATUS   LOGGING
------  ----------- -------- -------
     2  PDB$SEED    NORMAL   LOGGING
     3  ORCLPDB     NORMAL   LOGGING
```

**Verificación:**

- `PDB$SEED` debe aparecer en estado `READ ONLY` — es la plantilla de Oracle para crear nuevas PDBs
- Las PDBs de usuario deben aparecer en estado `READ WRITE` para estar operativas
- Si alguna PDB aparece en `MOUNTED`, deberá abrirse en el Paso 3

---

### Paso 3: Gestión de PDBs – Apertura, Cierre y Creación

**Objetivo:** Ejecutar operaciones administrativas sobre PDBs: abrir y cerrar PDBs existentes, y crear una nueva PDB a partir de PDB$SEED simulando el aprovisionamiento de una base de datos en Oracle DBCS.

**Instrucciones:**

1. Abrir todas las PDBs que estén en estado MOUNTED y verificar el resultado:

   ```sql
   -- Abrir todas las PDBs que estén cerradas
   ALTER PLUGGABLE DATABASE ALL OPEN;

   -- Verificar estado después de la apertura
   SELECT CON_ID, NAME, OPEN_MODE FROM V$PDBS ORDER BY CON_ID;
   ```

2. Practicar el cierre y apertura de una PDB específica:

   ```sql
   -- Cerrar la PDB ORCLPDB (simula mantenimiento programado en DBCS)
   ALTER PLUGGABLE DATABASE ORCLPDB CLOSE IMMEDIATE;

   -- Verificar que está cerrada
   SELECT NAME, OPEN_MODE FROM V$PDBS WHERE NAME = 'ORCLPDB';

   -- Volver a abrir la PDB
   ALTER PLUGGABLE DATABASE ORCLPDB OPEN;

   -- Confirmar apertura exitosa
   SELECT NAME, OPEN_MODE FROM V$PDBS WHERE NAME = 'ORCLPDB';
   ```

3. Crear una nueva PDB a partir de PDB$SEED (equivalente a aprovisionar una nueva base de datos en Oracle DBCS):

   ```sql
   -- Crear nueva PDB para el laboratorio
   -- Esta operación simula el aprovisionamiento de un DB System en OCI
   CREATE PLUGGABLE DATABASE LAB08_PDB
     ADMIN USER pdb_admin IDENTIFIED BY Oracle123
     ROLES = (DBA)
     FILE_NAME_CONVERT = (
       '/u01/app/oracle/oradata/ORCL/pdbseed/',
       '/u01/app/oracle/oradata/ORCL/LAB08_PDB/'
     );

   -- Abrir la nueva PDB recién creada
   ALTER PLUGGABLE DATABASE LAB08_PDB OPEN;

   -- Verificar que la nueva PDB está disponible
   SELECT CON_ID, NAME, OPEN_MODE FROM V$PDBS ORDER BY CON_ID;
   ```

4. Configurar apertura automática de la PDB al reiniciar la instancia:

   ```sql
   -- Conectarse a la nueva PDB para configurar apertura automática
   ALTER SESSION SET CONTAINER = LAB08_PDB;
   SHOW CON_NAME;

   -- Guardar el estado de apertura para que persista tras reinicios
   ALTER PLUGGABLE DATABASE LAB08_PDB SAVE STATE;

   -- Regresar al CDB$ROOT
   ALTER SESSION SET CONTAINER = CDB$ROOT;
   SHOW CON_NAME;
   ```

**Salida Esperada:**

```
Pluggable database altered.

CON_ID  NAME        OPEN_MODE
------  ----------- -----------
     2  PDB$SEED    READ ONLY
     3  ORCLPDB     READ WRITE
     4  LAB08_PDB   READ WRITE

Pluggable database created.

CON_NAME
-----------------------------
LAB08_PDB

CON_NAME
-----------------------------
CDB$ROOT
```

**Verificación:**

- La nueva PDB `LAB08_PDB` debe aparecer con `OPEN_MODE = READ WRITE`
- El `CON_ID` asignado debe ser mayor que los existentes (generalmente 4 o superior)
- El comando `SAVE STATE` garantiza que la PDB se abra automáticamente tras reinicios

---

### Paso 4: Gestión de Usuarios en Arquitectura Multitenant

**Objetivo:** Comprender y demostrar la diferencia entre usuarios comunes (Common Users) y usuarios locales (Local Users) en la arquitectura CDB/PDB, una característica crítica en entornos Oracle DBCS multitenancy.

**Instrucciones:**

1. Desde CDB$ROOT, crear un usuario común (Common User) que existe en todos los contenedores:

   ```sql
   -- Asegurarse de estar en CDB$ROOT
   SHOW CON_NAME;

   -- Crear un usuario común (prefijo C## obligatorio en Oracle 19c)
   CREATE USER C##DBA_CLOUD IDENTIFIED BY Oracle123
     CONTAINER = ALL;

   -- Otorgar privilegios al usuario común en todos los contenedores
   GRANT CREATE SESSION TO C##DBA_CLOUD CONTAINER = ALL;
   GRANT DBA TO C##DBA_CLOUD CONTAINER = ALL;

   -- Verificar que el usuario común existe en el CDB
   SELECT USERNAME, COMMON, CON_ID 
   FROM CDB_USERS 
   WHERE USERNAME = 'C##DBA_CLOUD'
   ORDER BY CON_ID;
   ```

2. Conectarse a LAB08_PDB y crear un usuario local (Local User) que solo existe en esa PDB:

   ```sql
   -- Cambiar al contenedor LAB08_PDB
   ALTER SESSION SET CONTAINER = LAB08_PDB;
   SHOW CON_NAME;

   -- Crear usuario local (sin prefijo C##, solo existe en esta PDB)
   CREATE USER LAB_USER IDENTIFIED BY Oracle123;

   -- Otorgar privilegios básicos al usuario local
   GRANT CREATE SESSION TO LAB_USER;
   GRANT CREATE TABLE TO LAB_USER;
   GRANT UNLIMITED TABLESPACE TO LAB_USER;

   -- Verificar la diferencia: usuario local vs usuario común en esta PDB
   SELECT USERNAME, COMMON, CON_ID
   FROM CDB_USERS
   WHERE USERNAME IN ('LAB_USER', 'C##DBA_CLOUD')
   ORDER BY USERNAME;
   ```

3. Crear una tabla de prueba con el usuario local para verificar operatividad:

   ```sql
   -- Aún en LAB08_PDB, crear objeto de prueba como usuario local
   -- (conectarse como LAB_USER usando el servicio de la PDB)
   -- Por simplicidad, crear la tabla como PDB_ADMIN desde el CDB$ROOT
   
   CREATE TABLE LAB_USER.CLOUD_CONCEPTS (
     ID           NUMBER GENERATED ALWAYS AS IDENTITY,
     MODELO       VARCHAR2(10),
     DESCRIPCION  VARCHAR2(200),
     PROVEEDOR    VARCHAR2(50),
     FECHA_REG    DATE DEFAULT SYSDATE,
     CONSTRAINT PK_CLOUD_CONCEPTS PRIMARY KEY (ID)
   );

   -- Insertar datos de referencia sobre modelos cloud
   INSERT INTO LAB_USER.CLOUD_CONCEPTS (MODELO, DESCRIPCION, PROVEEDOR)
   VALUES ('IaaS', 'El DBA gestiona SO, Oracle DB y parches', 'Oracle OCI Compute');

   INSERT INTO LAB_USER.CLOUD_CONCEPTS (MODELO, DESCRIPCION, PROVEEDOR)
   VALUES ('PaaS', 'Oracle gestiona SO; DBA gestiona esquemas y datos', 'Oracle DBCS');

   INSERT INTO LAB_USER.CLOUD_CONCEPTS (MODELO, DESCRIPCION, PROVEEDOR)
   VALUES ('SaaS', 'Oracle gestiona todo el stack tecnologico', 'Oracle ERP Cloud');

   INSERT INTO LAB_USER.CLOUD_CONCEPTS (MODELO, DESCRIPCION, PROVEEDOR)
   VALUES ('PaaS', 'Afinacion automatica de SQL e indices', 'Oracle Autonomous DB');

   COMMIT;

   -- Verificar los datos insertados
   SELECT ID, MODELO, DESCRIPCION, PROVEEDOR FROM LAB_USER.CLOUD_CONCEPTS;

   -- Regresar al CDB$ROOT
   ALTER SESSION SET CONTAINER = CDB$ROOT;
   ```

**Salida Esperada:**

```
CON_NAME
-----------------------------
CDB$ROOT

User created.
Grant succeeded.

USERNAME       COMMON  CON_ID
-------------- ------- ------
C##DBA_CLOUD   YES          0
C##DBA_CLOUD   YES          3
C##DBA_CLOUD   YES          4

CON_NAME
-----------------------------
LAB08_PDB

User created.

USERNAME       COMMON  CON_ID
-------------- ------- ------
C##DBA_CLOUD   YES          4
LAB_USER       NO           4

4 rows inserted.

ID  MODELO  DESCRIPCION                                    PROVEEDOR
--  ------  ---------------------------------------------  ------------------
 1  IaaS    El DBA gestiona SO, Oracle DB y parches        Oracle OCI Compute
 2  PaaS    Oracle gestiona SO; DBA gestiona esquemas...   Oracle DBCS
 3  SaaS    Oracle gestiona todo el stack tecnologico      Oracle ERP Cloud
 4  PaaS    Afinacion automatica de SQL e indices          Oracle Autonomous DB
```

**Verificación:**

- `C##DBA_CLOUD` debe aparecer con `COMMON = YES` y tener entradas en múltiples `CON_ID`
- `LAB_USER` debe aparecer con `COMMON = NO` y solo en el `CON_ID` de `LAB08_PDB`
- La tabla `CLOUD_CONCEPTS` debe contener exactamente 4 filas

---

### Paso 5: Verificación del Modo ARCHIVELOG y Configuración de RMAN

**Objetivo:** Confirmar que la base de datos está en modo ARCHIVELOG (requisito indispensable para RMAN) y configurar los parámetros fundamentales de RMAN: canal de respaldo, política de retención y ubicación de los archivos de respaldo.

**Instrucciones:**

1. Desde SQL*Plus como SYSDBA, verificar el modo ARCHIVELOG de la base de datos:

   ```sql
   -- Verificar modo ARCHIVELOG
   SELECT NAME, LOG_MODE, OPEN_MODE FROM V$DATABASE;

   -- Ver destino de archive logs
   ARCHIVE LOG LIST;
   ```

2. Si la base de datos **NO está en modo ARCHIVELOG**, ejecutar los siguientes comandos para habilitarlo. Si ya está en ARCHIVELOG, saltar al paso 3:

   ```sql
   -- SOLO si LOG_MODE = NOARCHIVELOG
   -- Apagar la base de datos limpiamente
   SHUTDOWN IMMEDIATE;

   -- Montar la base de datos (sin abrirla)
   STARTUP MOUNT;

   -- Habilitar modo ARCHIVELOG
   ALTER DATABASE ARCHIVELOG;

   -- Abrir la base de datos
   ALTER DATABASE OPEN;

   -- Abrir todas las PDBs
   ALTER PLUGGABLE DATABASE ALL OPEN;

   -- Verificar el cambio
   SELECT NAME, LOG_MODE FROM V$DATABASE;
   ARCHIVE LOG LIST;
   ```

3. Salir de SQL*Plus y conectarse a RMAN para configurar los parámetros de respaldo:

   ```bash
   # Salir de SQL*Plus
   EXIT

   # Conectarse a RMAN como SYSDBA
   rman target /
   ```

4. Dentro de RMAN, configurar los parámetros fundamentales:

   ```bash
   # Verificar la configuración actual de RMAN
   SHOW ALL;

   # Configurar la ubicación de los respaldos (Fast Recovery Area)
   CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/home/oracle/backup_rman/%U';

   # Configurar política de retención: mantener respaldos por 7 días
   CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;

   # Configurar compresión de respaldos para ahorrar espacio
   CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';

   # Configurar respaldo automático del control file
   CONFIGURE CONTROLFILE AUTOBACKUP ON;
   CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK 
     TO '/home/oracle/backup_rman/cf_%F';

   # Configurar paralelismo (1 canal para entorno de laboratorio)
   CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO BACKUPSET;

   # Verificar la configuración aplicada
   SHOW ALL;
   ```

**Salida Esperada:**

```sql
NAME   LOG_MODE    OPEN_MODE
------ ----------- ----------
ORCL   ARCHIVELOG  READ WRITE

Archive Mode      : Enabled
Automatic Archival: Enabled
Archive Destination: USE_DB_RECOVERY_FILE_DEST
```

```
Recovery Manager: Release 19.0.0.0.0

RMAN configuration parameters for database with db_unique_name ORCL are:
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE BACKUP OPTIMIZATION OFF; # default
CONFIGURE DEFAULT DEVICE TYPE TO DISK; # default
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/home/oracle/backup_rman/cf_%F';
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT   '/home/oracle/backup_rman/%U';
CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';
```

**Verificación:**

- `LOG_MODE = ARCHIVELOG` es obligatorio para continuar con el laboratorio
- La política de retención debe mostrar `RECOVERY WINDOW OF 7 DAYS`
- El autobackup del control file debe estar `ON`

---

### Paso 6: Ejecución de Respaldo Completo (Full Backup) con RMAN

**Objetivo:** Ejecutar un respaldo completo de la base de datos Oracle incluyendo archive logs, utilizando RMAN con los parámetros configurados en el paso anterior. Este tipo de respaldo es el equivalente al "Full Backup" que Oracle DBCS ejecuta automáticamente en la nube.

**Instrucciones:**

1. Desde la sesión RMAN (si salió, reconectarse con `rman target /`), ejecutar el respaldo completo:

   ```bash
   # Respaldo completo de la base de datos + archive logs
   # Equivale al respaldo automático que Oracle DBCS ejecuta diariamente
   BACKUP AS COMPRESSED BACKUPSET 
     DATABASE PLUS ARCHIVELOG 
     TAG 'FULL_BACKUP_LAB08'
     DELETE INPUT;
   ```

2. Verificar el respaldo ejecutado con LIST BACKUP:

   ```bash
   # Listar todos los respaldos disponibles
   LIST BACKUP SUMMARY;

   # Ver detalles del respaldo más reciente
   LIST BACKUP TAG 'FULL_BACKUP_LAB08';

   # Ver el contenido del directorio de respaldos
   HOST 'ls -lh /home/oracle/backup_rman/';
   ```

3. Validar la integridad del respaldo (simula la verificación automática de DBCS):

   ```bash
   # Verificar que los respaldos son válidos y no están corruptos
   VALIDATE BACKUPSET ALL;

   # Crosscheck: verificar que los archivos físicos de respaldo existen
   CROSSCHECK BACKUP;

   # Ver el estado de los respaldos después del crosscheck
   LIST BACKUP SUMMARY;
   ```

**Salida Esperada:**

```
Starting backup at 15-JUN-2024 10:30:00
using channel ORA_DISK_1
channel ORA_DISK_1: starting compressed full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00001 name=/u01/app/oracle/oradata/ORCL/system01.dbf
input datafile file number=00003 name=/u01/app/oracle/oradata/ORCL/sysaux01.dbf
...
channel ORA_DISK_1: backup set complete, elapsed time: 00:02:15
Finished backup at 15-JUN-2024 10:32:15

List of Backups
===============
Key     TY LV S Device Type Completion Time    #Pieces #Copies Compressed Tag
------- -- -- - ----------- ------------------ ------- ------- ---------- ----
1       B  F  A DISK        15-JUN-24          1       1       YES        FULL_BACKUP_LAB08
2       B  A  A DISK        15-JUN-24          1       1       YES        FULL_BACKUP_LAB08

crosscheck backup completed
validated 2 objects
```

**Verificación:**

- El respaldo debe completarse sin errores (`Finished backup at...`)
- `LIST BACKUP SUMMARY` debe mostrar al menos 2 piezas: una para la base de datos (`F` = Full) y otra para archive logs (`A` = Archivelog)
- `CROSSCHECK BACKUP` debe mostrar todos los backupsets como `AVAILABLE`

---

### Paso 7: Ejecución de Respaldo Incremental (Nivel 0 y Nivel 1)

**Objetivo:** Comprender y ejecutar la estrategia de respaldo incremental de RMAN, que es la base de la estrategia de respaldo eficiente utilizada en Oracle DBCS para minimizar el tiempo de respaldo y el consumo de almacenamiento.

**Instrucciones:**

1. Ejecutar un respaldo incremental de nivel 0 (base para la estrategia incremental):

   ```bash
   # Respaldo incremental nivel 0 (captura todos los bloques usados)
   # Es la base sobre la que se aplican los incrementales nivel 1
   BACKUP INCREMENTAL LEVEL 0 
     AS COMPRESSED BACKUPSET 
     DATABASE 
     TAG 'INC0_BACKUP_LAB08';

   # Verificar el respaldo incremental nivel 0
   LIST BACKUP TAG 'INC0_BACKUP_LAB08';
   ```

2. Simular actividad en la base de datos (para que el incremental nivel 1 tenga cambios que capturar):

   ```bash
   # Ejecutar comandos SQL desde RMAN usando HOST o SQL
   SQL "ALTER SESSION SET CONTAINER = LAB08_PDB";
   
   SQL "INSERT INTO LAB_USER.CLOUD_CONCEPTS (MODELO, DESCRIPCION, PROVEEDOR) 
        VALUES (''Híbrida'', ''Combina nube publica y privada'', ''Oracle Cloud@Customer'')";
   
   SQL "INSERT INTO LAB_USER.CLOUD_CONCEPTS (MODELO, DESCRIPCION, PROVEEDOR)
        VALUES (''Multicloud'', ''Multiples proveedores simultaneos'', ''OCI + AWS'')";
   
   SQL "COMMIT";
   
   SQL "ALTER SESSION SET CONTAINER = CDB\$ROOT";

   # Forzar un switch de log para generar archive logs
   SQL "ALTER SYSTEM SWITCH LOGFILE";
   SQL "ALTER SYSTEM ARCHIVE LOG CURRENT";
   ```

3. Ejecutar el respaldo incremental de nivel 1 (captura solo los bloques modificados desde el nivel 0):

   ```bash
   # Respaldo incremental nivel 1 diferencial
   # Solo captura bloques modificados desde el último nivel 0 o nivel 1
   BACKUP INCREMENTAL LEVEL 1 
     AS COMPRESSED BACKUPSET 
     DATABASE PLUS ARCHIVELOG 
     TAG 'INC1_BACKUP_LAB08'
     DELETE INPUT;

   # Ver el resumen completo de todos los respaldos
   LIST BACKUP SUMMARY;

   # Ver estadísticas de los respaldos incrementales
   SELECT BS_KEY, BACKUP_TYPE, INCREMENTAL_LEVEL, 
          COMPLETION_TIME, ELAPSED_SECONDS, OUTPUT_BYTES_DISPLAY
   FROM V$BACKUP_SET_DETAILS
   ORDER BY COMPLETION_TIME;
   ```

**Salida Esperada:**

```
Starting backup at 15-JUN-2024 10:45:00
channel ORA_DISK_1: starting compressed incremental level 0 datafile backup set
...
Finished backup at 15-JUN-2024 10:46:30

Starting backup at 15-JUN-2024 10:48:00
channel ORA_DISK_1: starting compressed incremental level 1 datafile backup set
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:45
Finished backup at 15-JUN-2024 10:48:45

List of Backups
===============
Key  TY LV S Device Type Completion Time    #Pieces Tag
---  -- -- - ----------- ------------------ ------- -------------------
1    B  F  A DISK        15-JUN-24          1       FULL_BACKUP_LAB08
3    B  0  A DISK        15-JUN-24          1       INC0_BACKUP_LAB08
4    B  1  A DISK        15-JUN-24          1       INC1_BACKUP_LAB08
```

**Verificación:**

- El respaldo incremental nivel 1 (`LV = 1`) debe ser significativamente más pequeño que el nivel 0 (`LV = 0`)
- Ambos respaldos deben aparecer con estado `A` (Available) en `LIST BACKUP SUMMARY`

---

### Paso 8: Simulación de Pérdida de Datafile y Recuperación con RMAN

**Objetivo:** Simular la pérdida de un datafile de la PDB LAB08_PDB y ejecutar una recuperación completa utilizando los comandos RESTORE y RECOVER de RMAN, replicando el proceso de recuperación que Oracle DBCS puede ejecutar automáticamente ante fallos de almacenamiento.

**Instrucciones:**

1. Desde SQL*Plus, identificar los datafiles de LAB08_PDB para seleccionar el que se simulará como perdido:

   ```bash
   # Abrir una segunda sesión de terminal o salir de RMAN temporalmente
   # Conectarse a SQL*Plus
   sqlplus / as sysdba
   ```

   ```sql
   -- Identificar datafiles de LAB08_PDB
   SELECT FILE#, NAME, STATUS, BYTES/1024/1024 AS SIZE_MB
   FROM V$DATAFILE
   WHERE CON_ID = (SELECT CON_ID FROM V$PDBS WHERE NAME = 'LAB08_PDB');

   -- Tomar nota del FILE# y la ruta del datafile de LAB08_PDB
   -- (generalmente será el último datafile creado)
   ```

2. Cerrar la PDB LAB08_PDB antes de simular la pérdida del datafile:

   ```sql
   -- Cerrar la PDB para poder simular la pérdida del datafile
   ALTER PLUGGABLE DATABASE LAB08_PDB CLOSE IMMEDIATE;

   -- Verificar que está cerrada
   SELECT NAME, OPEN_MODE FROM V$PDBS WHERE NAME = 'LAB08_PDB';
   ```

3. Desde el sistema operativo, renombrar el datafile para simular su pérdida (NO eliminar permanentemente):

   ```bash
   # Salir de SQL*Plus
   EXIT
   ```

   ```bash
   # Identificar la ruta exacta del datafile de LAB08_PDB
   ls -la /u01/app/oracle/oradata/ORCL/LAB08_PDB/

   # Renombrar el datafile para simular pérdida (ajustar el nombre según el output anterior)
   mv /u01/app/oracle/oradata/ORCL/LAB08_PDB/system01.dbf \
      /home/oracle/system01.dbf.PERDIDO

   # Verificar que el archivo ya no está en su ubicación original
   ls -la /u01/app/oracle/oradata/ORCL/LAB08_PDB/
   ```

4. Intentar abrir la PDB para confirmar el error de archivo faltante:

   ```bash
   sqlplus / as sysdba
   ```

   ```sql
   -- Intentar abrir la PDB (debe fallar con error de datafile)
   ALTER PLUGGABLE DATABASE LAB08_PDB OPEN;
   -- Se espera: ORA-01157 o similar indicando archivo no encontrado
   ```

5. Ejecutar la recuperación completa con RMAN:

   ```bash
   -- Salir de SQL*Plus
   EXIT

   -- Conectarse a RMAN
   rman target /
   ```

   ```bash
   # Restaurar y recuperar la PDB completa
   # Este proceso simula la recuperación automática de Oracle DBCS
   
   # Primero, restaurar el datafile faltante desde el respaldo
   RESTORE PLUGGABLE DATABASE LAB08_PDB;

   # Luego, aplicar los archive logs para llevar la PDB al estado actual
   RECOVER PLUGGABLE DATABASE LAB08_PDB;

   # Abrir la PDB después de la recuperación con RESETLOGS si es necesario
   ALTER PLUGGABLE DATABASE LAB08_PDB OPEN RESETLOGS;
   ```

6. Verificar la integridad de los datos después de la recuperación:

   ```bash
   # Conectarse a SQL*Plus para verificar los datos
   sqlplus / as sysdba
   ```

   ```sql
   -- Verificar que la PDB está abierta correctamente
   SELECT NAME, OPEN_MODE FROM V$PDBS WHERE NAME = 'LAB08_PDB';

   -- Cambiar al contenedor LAB08_PDB
   ALTER SESSION SET CONTAINER = LAB08_PDB;

   -- Verificar que los datos de la tabla CLOUD_CONCEPTS están intactos
   SELECT COUNT(*) AS TOTAL_REGISTROS FROM LAB_USER.CLOUD_CONCEPTS;
   SELECT ID, MODELO, DESCRIPCION FROM LAB_USER.CLOUD_CONCEPTS ORDER BY ID;

   -- Regresar al CDB$ROOT
   ALTER SESSION SET CONTAINER = CDB$ROOT;
   ```

**Salida Esperada:**

```sql
-- Antes de la recuperación (error esperado):
ORA-01157: cannot identify/lock data file 7 - see DBWR trace file
ORA-01110: data file 7: '/u01/app/oracle/oradata/ORCL/LAB08_PDB/system01.dbf'
```

```
-- Durante la recuperación RMAN:
Starting restore at 15-JUN-2024 11:00:00
using channel ORA_DISK_1
channel ORA_DISK_1: restoring datafile 00007 to /u01/app/oracle/oradata/ORCL/LAB08_PDB/system01.dbf
channel ORA_DISK_1: reading from backup piece /home/oracle/backup_rman/...
channel ORA_DISK_1: piece handle=/home/oracle/backup_rman/... tag=INC0_BACKUP_LAB08
channel ORA_DISK_1: restored backup piece 1
Finished restore at 15-JUN-2024 11:01:30

Starting recover at 15-JUN-2024 11:01:30
starting media recovery
media recovery complete, elapsed time: 00:00:15
Finished recover at 15-JUN-2024 11:01:45

Pluggable database opened.
```

```sql
-- Verificación post-recuperación:
NAME        OPEN_MODE
----------- ----------
LAB08_PDB   READ WRITE

TOTAL_REGISTROS
---------------
              6

ID  MODELO    DESCRIPCION
--  --------  ------------------------------------------
 1  IaaS      El DBA gestiona SO, Oracle DB y parches
 2  PaaS      Oracle gestiona SO; DBA gestiona esquemas...
 3  SaaS      Oracle gestiona todo el stack tecnologico
 4  PaaS      Afinacion automatica de SQL e indices
 5  Híbrida   Combina nube publica y privada
 6  Multicloud Multiples proveedores simultaneos
```

**Verificación:**

- La PDB `LAB08_PDB` debe estar en `OPEN_MODE = READ WRITE` después de la recuperación
- La tabla `CLOUD_CONCEPTS` debe contener todos los registros, incluyendo los insertados después del respaldo incremental nivel 0
- No debe haber errores ORA- activos en el alert log

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] La base de datos Oracle está en modo `ARCHIVELOG` confirmado por `ARCHIVE LOG LIST`
- [ ] Se identificaron correctamente los modelos IaaS, PaaS y SaaS en el contexto de Oracle Cloud
- [ ] La consulta a `V$PDBS` y `CDB_PDBS` muestra al menos 3 contenedores: `PDB$SEED`, `ORCLPDB` y `LAB08_PDB`
- [ ] La nueva PDB `LAB08_PDB` fue creada exitosamente y se abre en modo `READ WRITE`
- [ ] Se demostró la diferencia entre usuario común (`C##DBA_CLOUD`) y usuario local (`LAB_USER`)
- [ ] El respaldo completo con tag `FULL_BACKUP_LAB08` aparece en `LIST BACKUP SUMMARY` con estado `Available`
- [ ] Los respaldos incrementales nivel 0 y nivel 1 fueron ejecutados exitosamente
- [ ] La PDB `LAB08_PDB` fue recuperada exitosamente después de la simulación de pérdida del datafile
- [ ] La tabla `CLOUD_CONCEPTS` contiene todos los registros originales después de la recuperación

### Procedimiento de Pruebas

1. Verificar el estado completo de la base de datos y las PDBs:

   ```sql
   SELECT NAME, LOG_MODE, OPEN_MODE, CDB FROM V$DATABASE;
   SELECT CON_ID, NAME, OPEN_MODE FROM V$PDBS ORDER BY CON_ID;
   ```
   **Resultado Esperado:** `LOG_MODE = ARCHIVELOG`, `CDB = YES`, las tres PDBs en `READ WRITE` (excepto `PDB$SEED` en `READ ONLY`)

2. Verificar el inventario completo de respaldos RMAN:

   ```bash
   rman target /
   LIST BACKUP SUMMARY;
   CROSSCHECK BACKUP;
   ```
   **Resultado Esperado:** Al menos 4 backupsets con estado `AVAILABLE`; ninguno en estado `EXPIRED`

3. Verificar la integridad de datos post-recuperación:

   ```sql
   ALTER SESSION SET CONTAINER = LAB08_PDB;
   SELECT COUNT(*) FROM LAB_USER.CLOUD_CONCEPTS;
   SELECT USERNAME, COMMON FROM CDB_USERS WHERE USERNAME IN ('LAB_USER', 'C##DBA_CLOUD');
   ```
   **Resultado Esperado:** `COUNT(*) = 6`, `LAB_USER` con `COMMON = NO`, `C##DBA_CLOUD` con `COMMON = YES`

4. Verificar que no hay errores recientes en el alert log:

   ```bash
   tail -50 /u01/app/oracle/diag/rdbms/orcl/ORCL/trace/alert_ORCL.log | grep -i "ORA-"
   ```
   **Resultado Esperado:** Sin errores ORA- críticos relacionados con el laboratorio

---

## Solución de Problemas

### Problema 1: La Base de Datos No Está en Modo ARCHIVELOG

**Síntomas:**
- `ARCHIVE LOG LIST` muestra `Archive Mode: No Archive Mode`
- RMAN muestra error: `RMAN-06149: cannot BACKUP DATABASE in NOARCHIVELOG mode`
- Los comandos de respaldo RMAN fallan inmediatamente

**Causa:**
La base de datos fue instalada con el modo NOARCHIVELOG predeterminado. El modo ARCHIVELOG es necesario para respaldos online con RMAN y para recuperación point-in-time.

**Solución:**

```sql
-- Conectarse como SYSDBA
sqlplus / as sysdba

-- Apagar la base de datos limpiamente
SHUTDOWN IMMEDIATE;

-- Montar sin abrir
STARTUP MOUNT;

-- Habilitar ARCHIVELOG
ALTER DATABASE ARCHIVELOG;

-- Abrir la base de datos
ALTER DATABASE OPEN;

-- Abrir todas las PDBs
ALTER PLUGGABLE DATABASE ALL OPEN;

-- Verificar el cambio
SELECT LOG_MODE FROM V$DATABASE;
ARCHIVE LOG LIST;
```

---

### Problema 2: Error ORA-65096 al Crear Usuario Común sin Prefijo C##

**Síntomas:**
- `ORA-65096: invalid common user or role name` al intentar `CREATE USER ADMIN_CLOUD`
- El error ocurre cuando se intenta crear un usuario sin el prefijo `C##` desde CDB$ROOT

**Causa:**
En Oracle 12c y versiones posteriores, los usuarios comunes creados en el CDB$ROOT deben usar el prefijo `C##` obligatoriamente para distinguirlos de los usuarios locales de las PDBs.

**Solución:**

```sql
-- Opción 1: Usar el prefijo C## (recomendado)
CREATE USER C##ADMIN_CLOUD IDENTIFIED BY Oracle123 CONTAINER = ALL;

-- Opción 2: Cambiar al contenedor de la PDB para crear usuario local
ALTER SESSION SET CONTAINER = LAB08_PDB;
CREATE USER ADMIN_LOCAL IDENTIFIED BY Oracle123;
-- Este usuario solo existirá en LAB08_PDB

-- Opción 3 (NO recomendado en producción): Deshabilitar la restricción temporalmente
ALTER SESSION SET "_oracle_script" = TRUE;
CREATE USER ADMIN_CLOUD IDENTIFIED BY Oracle123;
ALTER SESSION SET "_oracle_script" = FALSE;
```

---

### Problema 3: RMAN No Encuentra los Archivos de Respaldo (EXPIRED)

**Síntomas:**
- `CROSSCHECK BACKUP` muestra backupsets en estado `EXPIRED`
- `LIST BACKUP` muestra respaldos pero RMAN no puede usarlos para recuperación
- Error: `RMAN-06026: some targets not found - aborting restore`

**Causa:**
Los archivos físicos de respaldo fueron movidos, eliminados o el directorio `/home/oracle/backup_rman/` no tiene los permisos correctos para el usuario `oracle`.

**Solución:**

```bash
# Verificar que los archivos físicos existen
ls -la /home/oracle/backup_rman/

# Verificar permisos del directorio
ls -ld /home/oracle/backup_rman/

# Corregir permisos si es necesario
chmod 755 /home/oracle/backup_rman/
chown oracle:oinstall /home/oracle/backup_rman/

# Si los archivos fueron movidos, catalogarlos nuevamente en RMAN
rman target /
```

```bash
# Dentro de RMAN, catalogar los archivos desde su nueva ubicación
CATALOG START WITH '/home/oracle/backup_rman/';

# Actualizar el estado de los backupsets
CROSSCHECK BACKUP;

# Eliminar registros de respaldos que ya no existen físicamente
DELETE EXPIRED BACKUP;

# Ejecutar un nuevo respaldo completo si los archivos se perdieron definitivamente
BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG TAG 'RECOVERY_BACKUP';
```

---

### Problema 4: Error al Crear LAB08_PDB por Ruta de Archivo Incorrecta

**Síntomas:**
- `ORA-27040: file create error` o `ORA-01119: error in creating database file` al ejecutar `CREATE PLUGGABLE DATABASE`
- El directorio destino no existe o el usuario `oracle` no tiene permisos de escritura

**Causa:**
La ruta especificada en `FILE_NAME_CONVERT` no existe o el usuario oracle no tiene permisos para crear archivos en esa ubicación.

**Solución:**

```bash
# Verificar y crear el directorio destino
mkdir -p /u01/app/oracle/oradata/ORCL/LAB08_PDB/
chown oracle:oinstall /u01/app/oracle/oradata/ORCL/LAB08_PDB/
chmod 755 /u01/app/oracle/oradata/ORCL/LAB08_PDB/

# Verificar la ruta real de PDB$SEED para usar en FILE_NAME_CONVERT
```

```sql
-- Verificar la ruta correcta de los datafiles de PDB$SEED
SELECT FILE#, NAME FROM V$DATAFILE WHERE CON_ID = 2;

-- Usar la ruta correcta en la creación de la PDB
-- Ajustar la ruta según lo que muestre la consulta anterior
CREATE PLUGGABLE DATABASE LAB08_PDB
  ADMIN USER pdb_admin IDENTIFIED BY Oracle123
  ROLES = (DBA)
  FILE_NAME_CONVERT = (
    '/u01/app/oracle/oradata/ORCL/pdbseed/',
    '/u01/app/oracle/oradata/ORCL/LAB08_PDB/'
  );
```

---

## Limpieza

```sql
-- Conectarse a SQL*Plus como SYSDBA para limpiar los objetos del laboratorio
sqlplus / as sysdba

-- Cerrar la PDB LAB08_PDB antes de eliminarla
ALTER PLUGGABLE DATABASE LAB08_PDB CLOSE IMMEDIATE;

-- Eliminar la PDB LAB08_PDB y sus datafiles
DROP PLUGGABLE DATABASE LAB08_PDB INCLUDING DATAFILES;

-- Eliminar el usuario común creado en el laboratorio
DROP USER C##DBA_CLOUD CASCADE;

-- Verificar que la PDB fue eliminada
SELECT CON_ID, NAME FROM V$PDBS ORDER BY CON_ID;

EXIT
```

```bash
# Conectarse a RMAN para limpiar los respaldos del laboratorio
rman target /
```

```bash
# Eliminar todos los respaldos del laboratorio (opcional, libera espacio en disco)
# PRECAUCIÓN: Solo ejecutar si no se necesitan estos respaldos para recuperación futura
DELETE BACKUP TAG 'FULL_BACKUP_LAB08';
DELETE BACKUP TAG 'INC0_BACKUP_LAB08';
DELETE BACKUP TAG 'INC1_BACKUP_LAB08';

# Confirmar eliminación cuando RMAN lo solicite (ingresar YES)
EXIT
```

```bash
# Limpiar archivos temporales del sistema operativo
# Restaurar el datafile que fue renombrado durante la simulación de pérdida
# (si no fue restaurado por RMAN automáticamente)
ls -la /home/oracle/*.PERDIDO 2>/dev/null && \
  echo "Archivo de simulación encontrado - verificar si RMAN ya lo restauró"

# Limpiar el directorio de respaldos si se eliminaron desde RMAN
rm -f /home/oracle/backup_rman/*.bkp 2>/dev/null
rm -f /home/oracle/backup_rman/cf_* 2>/dev/null
ls -la /home/oracle/backup_rman/
```

> ⚠️ **Advertencia:** El comando `DROP PLUGGABLE DATABASE ... INCLUDING DATAFILES` elimina permanentemente los datafiles del sistema de archivos. Asegurarse de que no se necesita la PDB `LAB08_PDB` antes de ejecutar este comando. En un entorno de producción, siempre tomar un respaldo previo antes de eliminar cualquier PDB.

> ⚠️ **Advertencia RMAN:** Los comandos `DELETE BACKUP` eliminan permanentemente los archivos de respaldo. Si se está usando este laboratorio como base para prácticas posteriores (módulos 8.2 en adelante), conservar al menos el respaldo `FULL_BACKUP_LAB08` para posibles recuperaciones futuras.

---

## Resumen

### Lo Que Lograste

- **Análisis de modelos cloud:** Identificaste y clasificaste los modelos IaaS, PaaS y SaaS en el contexto de Oracle Cloud Infrastructure, relacionando cada modelo con el nivel de responsabilidad del DBA y ejemplos concretos de servicios Oracle (OCI Compute, DBCS, Autonomous Database, ERP Cloud)
- **Exploración Multitenant:** Consultaste las vistas `V$PDBS` y `CDB_PDBS` para obtener información del estado de las Pluggable Databases, y ejecutaste operaciones de apertura, cierre y creación de PDBs desde el CDB$ROOT
- **Gestión de usuarios:** Demostraste la diferencia fundamental entre usuarios comunes (prefijo `C##`, visibles en todos los contenedores) y usuarios locales (sin prefijo, limitados a su PDB)
- **Configuración RMAN:** Estableciste los parámetros fundamentales de RMAN: canal de respaldo, política de retención de 7 días, autobackup del control file y compresión
- **Respaldos completos e incrementales:** Ejecutaste exitosamente un respaldo completo (`BACKUP DATABASE PLUS ARCHIVELOG`) y una estrategia incremental nivel 0 + nivel 1, verificando su integridad con `CROSSCHECK` y `VALIDATE`
- **Recuperación ante desastres:** Simulaste la pérdida de un datafile crítico y ejecutaste la recuperación completa con `RESTORE PLUGGABLE DATABASE` y `RECOVER PLUGGABLE DATABASE`, verificando la integridad de los datos post-recuperación

### Conceptos Clave

- **Oracle DBCS como PaaS:** En Oracle Database Cloud Service, Oracle gestiona el sistema operativo y los parches, mientras que el DBA se enfoca en la gestión de esquemas, usuarios y rendimiento — exactamente las mismas operaciones que practicaste en este laboratorio
- **CDB/PDB como base del cloud:** La arquitectura Multitenant permite a Oracle DBCS alojar múltiples bases de datos en una sola instancia, reduciendo costos y simplificando la administración a escala
- **RMAN en la nube:** Oracle DBCS utiliza RMAN internamente para sus respaldos automáticos hacia Oracle Object Storage; comprender RMAN localmente es la base para gestionar políticas de respaldo en DBCS
- **Recuperación es práctica:** La habilidad de recuperación no debe aprenderse por primera vez durante un incidente real; practicar el ciclo completo respaldo-pérdida-recuperación es una responsabilidad fundamental del DBA

### Próximos Pasos

- Explorar la arquitectura CDB/PDB con mayor profundidad en la Lección 8.2, incluyendo PDB cloning, PDB relocation y la vista `CDB_*` completa
- Investigar Oracle Autonomous Database y cómo automatiza las tareas de RMAN que ejecutaste manualmente en este laboratorio
- Revisar la documentación de Oracle DBCS en OCI Free Tier para aprovisionar un DB System real en la nube y comparar la experiencia con la administración local practicada hoy
- Configurar una estrategia de respaldo incremental acumulativo (nivel 1 acumulativo) como alternativa a la estrategia diferencial practicada

---

## Recursos Adicionales

- **Oracle Database Backup and Recovery User's Guide 19c** — Referencia completa de todos los comandos RMAN, estrategias de respaldo y procedimientos de recuperación. Disponible en: [https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/](https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/)
- **Oracle Multitenant Administrator's Guide 19c** — Guía oficial para administración de CDB y PDB, incluyendo creación, clonación, conexión y gestión de usuarios en arquitecturas Multitenant. Disponible en: [https://docs.oracle.com/en/database/oracle/oracle-database/19/multi/](https://docs.oracle.com/en/database/oracle/oracle-database/19/multi/)
- **Oracle Cloud Infrastructure – Database Cloud Service Documentation** — Documentación oficial del servicio DBCS en OCI, incluyendo aprovisionamiento, respaldo automático y configuración de alta disponibilidad. Disponible en: [https://docs.oracle.com/en-us/iaas/Content/Database/home.htm](https://docs.oracle.com/en-us/iaas/Content/Database/home.htm)
- **NIST SP 800-145: The NIST Definition of Cloud Computing** — Definición formal y estándar de los modelos de servicio (IaaS, PaaS, SaaS) y modelos de implementación (pública, privada, híbrida) utilizados como referencia en este laboratorio. Disponible en: [https://csrc.nist.gov/publications/detail/sp/800-145/final](https://csrc.nist.gov/publications/detail/sp/800-145/final)
- **Oracle University – Oracle Database 19c: Backup and Recovery Workshop** — Curso oficial de Oracle que profundiza en todas las capacidades de RMAN incluyendo recuperación de medios, catalog RMAN y configuraciones avanzadas. Disponible en: [https://education.oracle.com](https://education.oracle.com)


