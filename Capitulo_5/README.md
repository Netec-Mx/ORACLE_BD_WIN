# Lab 05-00-01: Exportación, Importación y Carga de Datos con Oracle Data Pump, SQL*Loader y Tablas Externas

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 90 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 5.1 – Oracle Data Pump (Export/Import) |

---

## Descripción General

Este laboratorio cubre los tres mecanismos principales de movimiento y carga de datos en Oracle Database 19c: Oracle Data Pump (`expdp`/`impdp`), SQL*Loader (`sqlldr`) y Tablas Externas. Los estudiantes configurarán objetos de directorio Oracle, ejecutarán exportaciones e importaciones de esquemas completos con remapeo, cargarán datos masivos desde archivos CSV mediante archivos de control SQL*Loader, y definirán tablas externas para consultar archivos del sistema operativo directamente desde SQL.

Este laboratorio refleja escenarios reales de administración de bases de datos: migración de esquemas entre ambientes, ingesta de datos desde fuentes externas y acceso eficiente a datos sin cargarlos físicamente en la base de datos. Dominar estas herramientas es una competencia fundamental para cualquier DBA de Oracle en entornos empresariales.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Crear y configurar objetos de directorio Oracle (`DIRECTORY`) con los privilegios adecuados para operaciones de Data Pump
- [ ] Ejecutar exportaciones de esquemas completos con `expdp` usando diferentes modos (`SCHEMAS`, `CONTENT`, `EXCLUDE`) y verificar los archivos generados
- [ ] Importar dump files con `impdp` aplicando remapeo de esquema (`REMAP_SCHEMA`) y remapeo de tablespace (`REMAP_TABLESPACE`)
- [ ] Crear archivos de control SQL*Loader (`.ctl`) para cargar datos desde archivos CSV con manejo de campos nulos, delimitadores y registros de error
- [ ] Definir y consultar Tablas Externas usando el driver `ORACLE_LOADER` para acceder a datos en archivos del sistema operativo sin importarlos físicamente
- [ ] Comparar las ventajas, limitaciones y casos de uso apropiados para cada mecanismo de movimiento de datos

---

## Prerrequisitos

### Conocimiento Requerido

- Conocimiento de DDL y DML básico en Oracle (`CREATE TABLE`, `INSERT`, `SELECT`, `DROP`)
- Familiaridad con el sistema de archivos Linux y permisos de directorios (`chmod`, `chown`, `ls -la`)
- Comprensión básica de la arquitectura Oracle (instancia, esquemas, tablespaces, segmentos)
- Conocimiento del esquema de ejemplo HR (tablas `EMPLOYEES`, `DEPARTMENTS`, `JOBS`)

### Acceso Requerido

- Acceso SSH a la máquina virtual Oracle Linux 8.x con Oracle Database 19c instalado
- Usuario del sistema operativo: `oracle` con acceso al entorno de Oracle (`ORACLE_HOME`, `ORACLE_SID`)
- Acceso a SQL*Plus como `SYSTEM` o `SYS AS SYSDBA` con contraseña conocida
- Privilegios `DATAPUMP_EXP_FULL_DATABASE` y `DATAPUMP_IMP_FULL_DATABASE` para el usuario administrador
- El esquema HR debe estar desbloqueado y activo en la base de datos (o se creará en el Paso 1)

---

## Entorno de Laboratorio

### Hardware Requirements

| Componente | Especificación |
|-----------|---------------|
| CPU | Intel Core i5 8va gen o AMD Ryzen 5 (mínimo), i7/Ryzen 7 recomendado |
| RAM | Mínimo 16 GB (32 GB recomendado para instancias CDB/PDB simultáneas) |
| Almacenamiento | Mínimo 100 GB libres en disco (SSD recomendado) |
| Red | Adaptador de red funcional con soporte loopback |

### Software Requirements

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) o 21c (21.3) | Motor principal de base de datos |
| SQL*Plus | Incluido con Oracle 19c | Administración y ejecución de scripts |
| Oracle Data Pump (`expdp`/`impdp`) | Incluido con Oracle 19c | Exportación e importación de datos |
| SQL*Loader (`sqlldr`) | Incluido con Oracle 19c | Carga masiva desde archivos planos |
| Oracle Linux | 8.x (8.7 o 8.8) | Sistema operativo del entorno de práctica |
| PuTTY / Terminal SSH | PuTTY 0.79+ / Terminal nativo | Acceso remoto a la VM Linux |

### Configuración Inicial

Antes de iniciar el laboratorio, verificar que el entorno Oracle esté correctamente configurado:

```bash
# Conectarse a la VM Oracle Linux como usuario oracle
ssh oracle@192.168.1.100

# Verificar variables de entorno Oracle
echo $ORACLE_HOME
echo $ORACLE_SID
echo $PATH

# Si las variables no están configuradas, cargarlas manualmente
source ~/.bash_profile

# Verificar que la base de datos esté activa
sqlplus -S / as sysdba <<EOF
SELECT instance_name, status FROM v\$instance;
EXIT;
EOF
```

```bash
# Crear los directorios del sistema operativo que usaremos en este laboratorio
mkdir -p /u01/oracle/datapump
mkdir -p /u01/oracle/sqlldr
mkdir -p /u01/oracle/external

# Asignar permisos correctos al usuario oracle
chmod 755 /u01/oracle/datapump
chmod 755 /u01/oracle/sqlldr
chmod 755 /u01/oracle/external

# Verificar que los directorios existen y tienen los permisos correctos
ls -la /u01/oracle/
```

---

## Instrucciones Paso a Paso

### Paso 1: Preparar el Esquema de Práctica y Verificar el Esquema HR

**Objetivo:** Asegurarse de que el esquema HR esté disponible y crear un usuario de práctica (`PRACTICA_USER`) que se usará como destino en la importación posterior.

**Instrucciones:**

1. Conectarse a SQL*Plus como administrador:

   ```bash
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1
   ```

2. Verificar si el esquema HR existe y está desbloqueado:

   ```sql
   SELECT username, account_status, default_tablespace
   FROM dba_users
   WHERE username IN ('HR', 'PRACTICA_USER')
   ORDER BY username;
   ```

3. Si el usuario HR está bloqueado, desbloquearlo y asignarle contraseña:

   ```sql
   ALTER USER hr IDENTIFIED BY Oracle123 ACCOUNT UNLOCK;
   ```

4. Si el esquema HR no existe, crearlo con los objetos básicos necesarios para el laboratorio:

   ```sql
   -- Crear el usuario HR si no existe
   CREATE USER hr IDENTIFIED BY Oracle123
   DEFAULT TABLESPACE users
   TEMPORARY TABLESPACE temp
   QUOTA UNLIMITED ON users;

   GRANT CONNECT, RESOURCE TO hr;
   GRANT CREATE VIEW TO hr;

   -- Crear tablas de ejemplo en el esquema HR
   CONNECT hr/Oracle123@localhost:1521/ORCLPDB1

   CREATE TABLE departments (
       department_id   NUMBER(4) PRIMARY KEY,
       department_name VARCHAR2(30) NOT NULL,
       location_id     NUMBER(4)
   );

   CREATE TABLE employees (
       employee_id    NUMBER(6) PRIMARY KEY,
       first_name     VARCHAR2(20),
       last_name      VARCHAR2(25) NOT NULL,
       email          VARCHAR2(25) NOT NULL UNIQUE,
       hire_date      DATE NOT NULL,
       job_id         VARCHAR2(10) NOT NULL,
       salary         NUMBER(8,2),
       department_id  NUMBER(4) REFERENCES departments(department_id)
   );

   INSERT INTO departments VALUES (10, 'Administration', 1700);
   INSERT INTO departments VALUES (20, 'Marketing', 1800);
   INSERT INTO departments VALUES (30, 'Purchasing', 1700);
   INSERT INTO departments VALUES (60, 'IT', 1400);
   INSERT INTO departments VALUES (90, 'Executive', 1700);

   INSERT INTO employees VALUES (100, 'Steven', 'King', 'SKING', DATE '2003-06-17', 'AD_PRES', 24000, 90);
   INSERT INTO employees VALUES (101, 'Neena', 'Kochar', 'NKOCHAR', DATE '2005-09-21', 'AD_VP', 17000, 90);
   INSERT INTO employees VALUES (102, 'Lex', 'De Haan', 'LDEHAAN', DATE '2001-01-13', 'AD_VP', 17000, 90);
   INSERT INTO employees VALUES (103, 'Alexander', 'Hunold', 'AHUNOLD', DATE '2006-01-03', 'IT_PROG', 9000, 60);
   INSERT INTO employees VALUES (107, 'Diana', 'Lorentz', 'DLORENTZ', DATE '2007-02-07', 'IT_PROG', 4200, 60);

   COMMIT;
   ```

5. Crear el usuario destino `PRACTICA_USER` para la importación posterior:

   ```sql
   -- Reconectarse como SYSTEM
   CONNECT system/Oracle123@localhost:1521/ORCLPDB1

   CREATE USER practica_user IDENTIFIED BY Oracle123
   DEFAULT TABLESPACE users
   TEMPORARY TABLESPACE temp
   QUOTA UNLIMITED ON users;

   GRANT CONNECT, RESOURCE TO practica_user;
   GRANT CREATE VIEW TO practica_user;
   GRANT IMP_FULL_DATABASE TO practica_user;
   ```

**Salida Esperada:**

```
User altered.

Table created.
Table created.

5 rows created.
5 rows created.

Commit complete.

User created.
Grant succeeded.
```

**Verificación:**

```sql
-- Verificar que los objetos HR existen
SELECT object_name, object_type, status
FROM dba_objects
WHERE owner = 'HR'
ORDER BY object_type, object_name;
```

Debe mostrar al menos las tablas `DEPARTMENTS` y `EMPLOYEES` con status `VALID`.

---

### Paso 2: Crear y Configurar el Objeto DIRECTORY de Oracle

**Objetivo:** Crear el objeto de directorio Oracle (`DIRECTORY`) que Data Pump utilizará para leer y escribir archivos dump, y otorgar los privilegios necesarios a los usuarios involucrados.

**Instrucciones:**

1. Conectarse como SYSTEM y crear el objeto de directorio para Data Pump:

   ```sql
   CONNECT system/Oracle123@localhost:1521/ORCLPDB1

   -- Crear el directorio lógico Oracle que apunta al directorio físico del SO
   CREATE OR REPLACE DIRECTORY dp_dir AS '/u01/oracle/datapump';

   -- Verificar que el directorio se creó correctamente
   SELECT directory_name, directory_path
   FROM dba_directories
   WHERE directory_name = 'DP_DIR';
   ```

2. Otorgar privilegios de lectura y escritura sobre el directorio a los usuarios que ejecutarán Data Pump:

   ```sql
   -- Otorgar privilegios al usuario SYSTEM (ya los tiene implícitamente, pero es buena práctica)
   GRANT READ, WRITE ON DIRECTORY dp_dir TO system;

   -- Otorgar privilegios al usuario HR para que pueda exportar su propio esquema
   GRANT READ, WRITE ON DIRECTORY dp_dir TO hr;

   -- Otorgar privilegios al usuario PRACTICA_USER para la importación
   GRANT READ, WRITE ON DIRECTORY dp_dir TO practica_user;
   ```

3. Crear también el directorio para SQL*Loader y tablas externas:

   ```sql
   CREATE OR REPLACE DIRECTORY sqlldr_dir AS '/u01/oracle/sqlldr';
   CREATE OR REPLACE DIRECTORY ext_dir AS '/u01/oracle/external';

   GRANT READ, WRITE ON DIRECTORY sqlldr_dir TO system;
   GRANT READ, WRITE ON DIRECTORY sqlldr_dir TO hr;
   GRANT READ, WRITE ON DIRECTORY ext_dir TO system;
   GRANT READ, WRITE ON DIRECTORY ext_dir TO hr;

   -- Verificar todos los directorios creados
   SELECT directory_name, directory_path
   FROM dba_directories
   WHERE directory_name IN ('DP_DIR', 'SQLLDR_DIR', 'EXT_DIR')
   ORDER BY directory_name;
   ```

4. Verificar desde el sistema operativo que los directorios físicos existen y tienen los permisos correctos:

   ```bash
   # Salir de SQL*Plus y verificar en el SO
   ls -la /u01/oracle/datapump
   ls -la /u01/oracle/sqlldr
   ls -la /u01/oracle/external
   ```

**Salida Esperada:**

```
Directory created.

DIRECTORY_NAME    DIRECTORY_PATH
----------------- ---------------------------
DP_DIR            /u01/oracle/datapump

Grant succeeded.
Grant succeeded.
Grant succeeded.

DIRECTORY_NAME    DIRECTORY_PATH
----------------- ---------------------------
DP_DIR            /u01/oracle/datapump
EXT_DIR           /u01/oracle/external
SQLLDR_DIR        /u01/oracle/sqlldr
```

**Verificación:**

- Confirmar que los tres directorios aparecen en `DBA_DIRECTORIES`
- Confirmar que los directorios físicos en el SO tienen permisos `755` y son propiedad del usuario `oracle`

---

### Paso 3: Exportar el Esquema HR con expdp (Modo Schema)

**Objetivo:** Ejecutar una exportación completa del esquema HR usando `expdp` en modo `SCHEMAS`, generando un dump file con datos y metadatos, y analizar el log de exportación.

**Instrucciones:**

1. Desde la terminal Linux (como usuario `oracle`), ejecutar la exportación completa del esquema HR:

   ```bash
   expdp system/Oracle123@localhost:1521/ORCLPDB1 \
     SCHEMAS=HR \
     DIRECTORY=dp_dir \
     DUMPFILE=hr_full_export.dmp \
     LOGFILE=hr_full_export.log \
     JOB_NAME=export_hr_full
   ```

2. Observar la salida en pantalla y esperar a que el proceso termine. Debería completarse en menos de 2 minutos para un esquema pequeño.

3. Verificar que el archivo dump y el log fueron creados correctamente:

   ```bash
   ls -lh /u01/oracle/datapump/
   ```

4. Revisar el contenido del log de exportación:

   ```bash
   cat /u01/oracle/datapump/hr_full_export.log
   ```

5. Ahora ejecutar una exportación solo de metadatos (sin datos), útil para copiar la estructura a ambientes de desarrollo:

   ```bash
   expdp system/Oracle123@localhost:1521/ORCLPDB1 \
     SCHEMAS=HR \
     DIRECTORY=dp_dir \
     DUMPFILE=hr_metadata_only.dmp \
     LOGFILE=hr_metadata_only.log \
     CONTENT=METADATA_ONLY \
     JOB_NAME=export_hr_meta
   ```

6. Comparar el tamaño de ambos archivos dump:

   ```bash
   ls -lh /u01/oracle/datapump/hr_full_export.dmp
   ls -lh /u01/oracle/datapump/hr_metadata_only.dmp
   ```

7. Ejecutar una exportación excluyendo índices para demostrar el uso del parámetro `EXCLUDE`:

   ```bash
   expdp system/Oracle123@localhost:1521/ORCLPDB1 \
     SCHEMAS=HR \
     DIRECTORY=dp_dir \
     DUMPFILE=hr_noindex.dmp \
     LOGFILE=hr_noindex.log \
     EXCLUDE=INDEX \
     JOB_NAME=export_hr_noindex
   ```

**Salida Esperada:**

```
Export: Release 19.0.0.0.0 - Production on [fecha]
Version 19.3.0.0.0

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0

Starting "SYSTEM"."EXPORT_HR_FULL":  system/********@localhost:1521/ORCLPDB1 SCHEMAS=HR DIRECTORY=dp_dir DUMPFILE=hr_full_export.dmp LOGFILE=hr_full_export.log JOB_NAME=export_hr_full
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
Processing object type SCHEMA_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
Processing object type SCHEMA_EXPORT/STATISTICS/MARKER
Processing object type SCHEMA_EXPORT/PRE_SCHEMA/PROCACT_SCHEMA
Processing object type SCHEMA_EXPORT/TABLE/TABLE
Processing object type SCHEMA_EXPORT/TABLE/CONSTRAINT/CONSTRAINT
Processing object type SCHEMA_EXPORT/TABLE/CONSTRAINT/REF_CONSTRAINT
. . exported "HR"."DEPARTMENTS"                          6.031 KB       5 rows
. . exported "HR"."EMPLOYEES"                            6.507 KB       5 rows
Master table "SYSTEM"."EXPORT_HR_FULL" successfully loaded/unloaded
******************************************************************************
Dump file set for SYSTEM.EXPORT_HR_FULL is:
  /u01/oracle/datapump/hr_full_export.dmp
Job "SYSTEM"."EXPORT_HR_FULL" successfully completed at [hora]
```

**Verificación:**

```bash
# Verificar que los tres archivos dump existen
ls -lh /u01/oracle/datapump/*.dmp

# El archivo con datos debe ser mayor que el de solo metadatos
# Ejemplo esperado:
# hr_full_export.dmp     ~  200 KB  (con datos)
# hr_metadata_only.dmp   ~   80 KB  (sin datos)
# hr_noindex.dmp         ~  180 KB  (sin índices)
```

---

### Paso 4: Importar el Esquema HR con impdp Usando Remapeo de Esquema

**Objetivo:** Importar el dump file del esquema HR hacia el usuario `PRACTICA_USER` usando `impdp` con el parámetro `REMAP_SCHEMA`, demostrando cómo mover datos entre esquemas diferentes.

**Instrucciones:**

1. Antes de importar, verificar que `PRACTICA_USER` no tiene objetos todavía:

   ```sql
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT object_name, object_type
   FROM dba_objects
   WHERE owner = 'PRACTICA_USER';
   EXIT;
   EOF
   ```

2. Ejecutar la importación con remapeo de esquema:

   ```bash
   impdp system/Oracle123@localhost:1521/ORCLPDB1 \
     DIRECTORY=dp_dir \
     DUMPFILE=hr_full_export.dmp \
     LOGFILE=hr_import_remap.log \
     REMAP_SCHEMA=HR:PRACTICA_USER \
     JOB_NAME=import_hr_remap
   ```

3. Revisar el log de importación para verificar que no hubo errores:

   ```bash
   cat /u01/oracle/datapump/hr_import_remap.log
   ```

4. Verificar que los objetos fueron creados correctamente en `PRACTICA_USER`:

   ```sql
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT object_name, object_type, status
   FROM dba_objects
   WHERE owner = 'PRACTICA_USER'
   ORDER BY object_type, object_name;
   EXIT;
   EOF
   ```

5. Verificar que los datos también fueron importados:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT COUNT(*) AS total_empleados FROM employees;
   SELECT COUNT(*) AS total_departamentos FROM departments;
   SELECT first_name, last_name, salary FROM employees ORDER BY employee_id;
   EXIT;
   EOF
   ```

6. Demostrar una importación solo de metadatos (estructura sin datos) en un esquema temporal:

   ```sql
   -- Crear un esquema temporal para la demostración
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   CREATE USER temp_schema IDENTIFIED BY Oracle123
   DEFAULT TABLESPACE users QUOTA UNLIMITED ON users;
   GRANT CONNECT, RESOURCE TO temp_schema;
   EXIT;
   EOF
   ```

   ```bash
   impdp system/Oracle123@localhost:1521/ORCLPDB1 \
     DIRECTORY=dp_dir \
     DUMPFILE=hr_metadata_only.dmp \
     LOGFILE=hr_import_meta.log \
     REMAP_SCHEMA=HR:TEMP_SCHEMA \
     JOB_NAME=import_hr_meta_only
   ```

   ```sql
   -- Verificar que las tablas existen pero sin datos
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT table_name FROM dba_tables WHERE owner = 'TEMP_SCHEMA';
   SELECT COUNT(*) FROM temp_schema.employees;
   EXIT;
   EOF
   ```

**Salida Esperada:**

```
Import: Release 19.0.0.0.0 - Production on [fecha]

Connected to: Oracle Database 19c Enterprise Edition

Starting "SYSTEM"."IMPORT_HR_REMAP":
Processing object type SCHEMA_EXPORT/TABLE/TABLE
Processing object type SCHEMA_EXPORT/TABLE/TABLE_DATA
. . imported "PRACTICA_USER"."DEPARTMENTS"               6.031 KB       5 rows
. . imported "PRACTICA_USER"."EMPLOYEES"                 6.507 KB       5 rows
Processing object type SCHEMA_EXPORT/TABLE/CONSTRAINT/CONSTRAINT
Processing object type SCHEMA_EXPORT/TABLE/CONSTRAINT/REF_CONSTRAINT
Job "SYSTEM"."IMPORT_HR_REMAP" successfully completed at [hora]
```

**Verificación:**

```sql
-- Consulta de verificación final
SELECT 'HR' AS esquema, COUNT(*) AS empleados FROM hr.employees
UNION ALL
SELECT 'PRACTICA_USER', COUNT(*) FROM practica_user.employees;
```

Ambos esquemas deben mostrar el mismo número de empleados (5 filas).

---

### Paso 5: Monitorear Trabajos de Data Pump con Vistas del Diccionario

**Objetivo:** Aprender a consultar las vistas del diccionario de datos para monitorear trabajos de Data Pump activos y completados, y practicar el modo interactivo.

**Instrucciones:**

1. Lanzar una exportación en segundo plano para poder monitorearla (usar `nohup` o simplemente observar mientras ejecuta):

   ```bash
   # Exportar con múltiples archivos para simular una operación más larga
   expdp system/Oracle123@localhost:1521/ORCLPDB1 \
     SCHEMAS=HR \
     DIRECTORY=dp_dir \
     DUMPFILE=hr_monitor_%U.dmp \
     LOGFILE=hr_monitor.log \
     PARALLEL=2 \
     FILESIZE=5M \
     JOB_NAME=export_hr_monitor &
   ```

2. Mientras el trabajo está ejecutando (o inmediatamente después), consultar las vistas de monitoreo:

   ```sql
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   -- Ver todos los trabajos de Data Pump (activos y completados)
   SELECT owner_name, job_name, operation, job_mode, state, degree
   FROM dba_datapump_jobs
   ORDER BY job_name;

   -- Ver detalles adicionales de trabajos activos
   SELECT owner_name, job_name, state,
          attached_sessions, datapump_sessions
   FROM dba_datapump_jobs
   WHERE state = 'EXECUTING';

   EXIT;
   EOF
   ```

3. Consultar la vista de sesiones para ver los worker processes activos:

   ```sql
   sqlplus / as sysdba <<EOF

   -- Ver procesos de Data Pump activos en la instancia
   SELECT sid, serial#, username, program, status
   FROM v\$session
   WHERE program LIKE '%DM%' OR program LIKE '%DW%'
   ORDER BY program;

   EXIT;
   EOF
   ```

4. Demostrar cómo adjuntarse a un trabajo activo (si el trabajo anterior ya terminó, relanzarlo):

   ```bash
   # Si el trabajo anterior ya terminó, lanzar uno nuevo
   expdp system/Oracle123@localhost:1521/ORCLPDB1 \
     SCHEMAS=HR \
     DIRECTORY=dp_dir \
     DUMPFILE=hr_interactive.dmp \
     LOGFILE=hr_interactive.log \
     JOB_NAME=export_interactive

   # Durante la ejecución, presionar Ctrl+C para entrar al modo interactivo
   # En el prompt "Export>" ejecutar:
   # STATUS
   # CONTINUE_CLIENT
   ```

**Salida Esperada:**

```sql
-- Resultado de DBA_DATAPUMP_JOBS
OWNER_NAME  JOB_NAME              OPERATION  JOB_MODE  STATE       DEGREE
----------- --------------------- ---------- --------- ----------- ------
SYSTEM      EXPORT_HR_FULL        EXPORT     SCHEMA    NOT RUNNING  1
SYSTEM      EXPORT_HR_META        EXPORT     SCHEMA    NOT RUNNING  1
SYSTEM      EXPORT_HR_MONITOR     EXPORT     SCHEMA    EXECUTING    2
SYSTEM      IMPORT_HR_REMAP       IMPORT     SCHEMA    NOT RUNNING  1
```

**Verificación:**

```sql
-- Verificar historial de trabajos completados
SELECT job_name, operation, job_mode, state
FROM dba_datapump_jobs
WHERE state = 'NOT RUNNING'
ORDER BY job_name;
```

Deben aparecer al menos los trabajos `EXPORT_HR_FULL`, `EXPORT_HR_META` e `IMPORT_HR_REMAP` con estado `NOT RUNNING` (completados).

---

### Paso 6: Preparar Archivos CSV para SQL*Loader

**Objetivo:** Crear los archivos de datos CSV y el archivo de control SQL*Loader necesarios para cargar registros en una tabla Oracle desde un archivo plano.

**Instrucciones:**

1. Crear la tabla destino en el esquema `PRACTICA_USER` para la carga con SQL*Loader:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   CREATE TABLE empleados_carga (
       emp_id        NUMBER(6),
       nombre        VARCHAR2(50),
       apellido      VARCHAR2(50),
       email         VARCHAR2(50),
       fecha_ingreso DATE,
       salario       NUMBER(10,2),
       departamento  VARCHAR2(30),
       activo        CHAR(1)
   );

   EXIT;
   EOF
   ```

2. Crear el archivo CSV de datos de prueba:

   ```bash
   cat > /u01/oracle/sqlldr/empleados_datos.csv << 'EOF'
   201,Juan,García,jgarcia@empresa.com,15-01-2020,35000.50,Tecnología,S
   202,María,López,mlopez@empresa.com,20-03-2019,42000.00,Finanzas,S
   203,Carlos,Martínez,cmartinez@empresa.com,05-07-2021,28500.75,Marketing,S
   204,Ana,Rodríguez,arodriguez@empresa.com,10-11-2018,55000.00,Dirección,S
   205,Pedro,Sánchez,psanchez@empresa.com,01-06-2022,31000.25,Tecnología,S
   206,Laura,Fernández,lfernandez@empresa.com,15-09-2020,38000.00,RRHH,S
   207,Miguel,Torres,mtorres@empresa.com,22-02-2021,29000.50,Marketing,N
   208,Sofía,Ramírez,sramirez@empresa.com,08-04-2019,47000.00,Finanzas,S
   209,Diego,Morales,dmorales@empresa.com,30-10-2022,26500.00,Tecnología,S
   210,Isabella,Vargas,ivargas@empresa.com,12-08-2020,41000.75,Dirección,S
   EOF
   ```

3. Crear el archivo de control SQL*Loader (`.ctl`):

   ```bash
   cat > /u01/oracle/sqlldr/empleados_carga.ctl << 'EOF'
   -- Archivo de control SQL*Loader para cargar empleados desde CSV
   -- Tabla destino: PRACTICA_USER.EMPLEADOS_CARGA

   LOAD DATA
   INFILE '/u01/oracle/sqlldr/empleados_datos.csv'
   BADFILE '/u01/oracle/sqlldr/empleados_carga.bad'
   DISCARDFILE '/u01/oracle/sqlldr/empleados_carga.dsc'
   APPEND INTO TABLE practica_user.empleados_carga
   FIELDS TERMINATED BY ','
   OPTIONALLY ENCLOSED BY '"'
   TRAILING NULLCOLS
   (
       emp_id        INTEGER EXTERNAL,
       nombre        CHAR,
       apellido      CHAR,
       email         CHAR,
       fecha_ingreso DATE "DD-MM-YYYY",
       salario       DECIMAL EXTERNAL,
       departamento  CHAR,
       activo        CHAR
   )
   EOF
   ```

4. Crear también un archivo CSV con algunos registros intencionalmente incorrectos para demostrar el manejo de errores:

   ```bash
   cat > /u01/oracle/sqlldr/empleados_errores.csv << 'EOF'
   211,Roberto,Castro,rcastro@empresa.com,15-01-2023,32000.00,Tecnología,S
   212,REGISTRO_INVALIDO,,email_invalido,fecha_mala,NO_ES_NUMERO,Ventas,X
   213,Valentina,Cruz,vcruz@empresa.com,20-05-2022,36000.50,Marketing,S
   EOF
   ```

   ```bash
   cat > /u01/oracle/sqlldr/empleados_errores.ctl << 'EOF'
   LOAD DATA
   INFILE '/u01/oracle/sqlldr/empleados_errores.csv'
   BADFILE '/u01/oracle/sqlldr/empleados_errores.bad'
   DISCARDFILE '/u01/oracle/sqlldr/empleados_errores.dsc'
   APPEND INTO TABLE practica_user.empleados_carga
   FIELDS TERMINATED BY ','
   OPTIONALLY ENCLOSED BY '"'
   TRAILING NULLCOLS
   (
       emp_id        INTEGER EXTERNAL,
       nombre        CHAR,
       apellido      CHAR,
       email         CHAR,
       fecha_ingreso DATE "DD-MM-YYYY",
       salario       DECIMAL EXTERNAL,
       departamento  CHAR,
       activo        CHAR
   )
   EOF
   ```

**Salida Esperada:**

```
Table created.
```

```bash
# Verificar los archivos creados
ls -la /u01/oracle/sqlldr/
# Debe mostrar:
# empleados_datos.csv
# empleados_carga.ctl
# empleados_errores.csv
# empleados_errores.ctl
```

**Verificación:**

```bash
# Verificar contenido del archivo CSV
wc -l /u01/oracle/sqlldr/empleados_datos.csv
# Debe mostrar: 10 /u01/oracle/sqlldr/empleados_datos.csv

# Verificar contenido del archivo de control
cat /u01/oracle/sqlldr/empleados_carga.ctl
```

---

### Paso 7: Ejecutar SQL*Loader y Analizar los Resultados

**Objetivo:** Cargar los datos del archivo CSV hacia la tabla Oracle usando `sqlldr`, analizar el log file generado y revisar el manejo de registros con errores.

**Instrucciones:**

1. Ejecutar SQL*Loader con el archivo de control principal:

   ```bash
   sqlldr userid=practica_user/Oracle123@localhost:1521/ORCLPDB1 \
     control=/u01/oracle/sqlldr/empleados_carga.ctl \
     log=/u01/oracle/sqlldr/empleados_carga_sqlldr.log
   ```

2. Revisar el log file generado por SQL*Loader:

   ```bash
   cat /u01/oracle/sqlldr/empleados_carga_sqlldr.log
   ```

3. Verificar que los datos fueron cargados correctamente en la tabla:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT COUNT(*) AS total_registros FROM empleados_carga;
   SELECT emp_id, nombre, apellido, salario, departamento
   FROM empleados_carga
   ORDER BY emp_id;
   EXIT;
   EOF
   ```

4. Ahora ejecutar SQL*Loader con el archivo que contiene errores para ver el manejo de bad records:

   ```bash
   sqlldr userid=practica_user/Oracle123@localhost:1521/ORCLPDB1 \
     control=/u01/oracle/sqlldr/empleados_errores.ctl \
     log=/u01/oracle/sqlldr/empleados_errores_sqlldr.log
   ```

5. Revisar el log del proceso con errores:

   ```bash
   cat /u01/oracle/sqlldr/empleados_errores_sqlldr.log
   ```

6. Verificar el archivo BAD que contiene los registros rechazados:

   ```bash
   # Verificar si el archivo BAD fue creado (solo si hubo errores)
   ls -la /u01/oracle/sqlldr/*.bad 2>/dev/null || echo "No se generaron archivos BAD"
   cat /u01/oracle/sqlldr/empleados_errores.bad 2>/dev/null
   ```

7. Verificar el total de registros después de ambas cargas:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT COUNT(*) AS total_registros FROM empleados_carga;
   SELECT departamento, COUNT(*) AS cantidad, AVG(salario) AS salario_promedio
   FROM empleados_carga
   GROUP BY departamento
   ORDER BY departamento;
   EXIT;
   EOF
   ```

**Salida Esperada:**

```
SQL*Loader: Release 19.0.0.0.0 - Production on [fecha]
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Path used:      Conventional
Commit point reached - logical record count 10

Table PRACTICA_USER.EMPLEADOS_CARGA:
  10 Rows successfully loaded.

Check the log file:
  /u01/oracle/sqlldr/empleados_carga_sqlldr.log
for more information about the load.
```

```
-- Resultado de la consulta de verificación
TOTAL_REGISTROS
---------------
             10

EMP_ID NOMBRE    APELLIDO   SALARIO    DEPARTAMENTO
------ --------- ---------- ---------- -----------
   201 Juan      García     35000.5    Tecnología
   202 María     López      42000      Finanzas
   ...
```

**Verificación:**

```bash
# El log file debe indicar:
grep "Rows successfully loaded" /u01/oracle/sqlldr/empleados_carga_sqlldr.log
# Debe mostrar: 10 Rows successfully loaded.

# Para el archivo con errores:
grep "Rows successfully loaded\|Rows not loaded" /u01/oracle/sqlldr/empleados_errores_sqlldr.log
```

---

### Paso 8: Crear y Consultar Tablas Externas con ORACLE_LOADER

**Objetivo:** Definir una Tabla Externa en Oracle que lea directamente desde un archivo CSV del sistema operativo sin importar los datos físicamente, y ejecutar consultas SQL sobre ella.

**Instrucciones:**

1. Crear el archivo de datos que será leído por la tabla externa:

   ```bash
   cat > /u01/oracle/external/productos_ext.csv << 'EOF'
   1001,Laptop Pro 15,Electrónica,85000.00,150,2024-01-15
   1002,Mouse Inalámbrico,Periféricos,450.50,500,2024-01-20
   1003,Teclado Mecánico,Periféricos,1200.00,300,2024-02-01
   1004,Monitor 4K 27",Electrónica,12500.00,80,2024-02-10
   1005,Auriculares BT,Audio,2800.75,200,2024-02-15
   1006,Webcam HD,Periféricos,850.00,120,2024-03-01
   1007,SSD 1TB,Almacenamiento,2200.50,250,2024-03-05
   1008,Hub USB-C,Periféricos,650.00,400,2024-03-10
   1009,Tablet 10",Electrónica,7500.00,90,2024-03-15
   1010,Impresora Laser,Oficina,4500.00,60,2024-03-20
   EOF
   ```

2. Verificar que el archivo tiene los permisos correctos para que Oracle pueda leerlo:

   ```bash
   chmod 644 /u01/oracle/external/productos_ext.csv
   ls -la /u01/oracle/external/
   ```

3. Crear la Tabla Externa en Oracle que apunta al archivo CSV:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   CREATE TABLE productos_externos (
       producto_id   NUMBER(6),
       nombre        VARCHAR2(50),
       categoria     VARCHAR2(30),
       precio        NUMBER(10,2),
       stock         NUMBER(6),
       fecha_ingreso DATE
   )
   ORGANIZATION EXTERNAL (
       TYPE ORACLE_LOADER
       DEFAULT DIRECTORY ext_dir
       ACCESS PARAMETERS (
           RECORDS DELIMITED BY NEWLINE
           FIELDS TERMINATED BY ','
           OPTIONALLY ENCLOSED BY '"'
           MISSING FIELD VALUES ARE NULL
           (
               producto_id   INTEGER EXTERNAL,
               nombre        CHAR(50),
               categoria     CHAR(30),
               precio        DECIMAL EXTERNAL,
               stock         INTEGER EXTERNAL,
               fecha_ingreso DATE 'YYYY-MM-DD'
           )
       )
       LOCATION ('productos_ext.csv')
   )
   REJECT LIMIT UNLIMITED;

   EXIT;
   EOF
   ```

4. Consultar la tabla externa directamente con SQL:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   -- Consulta básica sobre la tabla externa
   SELECT * FROM productos_externos ORDER BY producto_id;

   -- Consulta con filtro y agregación
   SELECT categoria,
          COUNT(*) AS cantidad_productos,
          AVG(precio) AS precio_promedio,
          SUM(stock) AS stock_total
   FROM productos_externos
   GROUP BY categoria
   ORDER BY categoria;

   -- Consulta con JOIN entre tabla externa y tabla interna
   SELECT e.nombre AS empleado,
          e.departamento,
          p.nombre AS producto,
          p.precio
   FROM empleados_carga e
   CROSS JOIN productos_externos p
   WHERE e.departamento = 'Tecnología'
     AND p.categoria = 'Electrónica'
   ORDER BY e.nombre, p.producto_id;

   EXIT;
   EOF
   ```

5. Verificar los metadatos de la tabla externa en el diccionario de datos:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   -- Ver propiedades de la tabla externa
   SELECT table_name, type_owner, type_name,
          default_directory_name, reject_limit
   FROM user_external_tables;

   -- Ver la ubicación del archivo externo
   SELECT table_name, location
   FROM user_external_locations;

   EXIT;
   EOF
   ```

6. Demostrar que la tabla externa NO almacena datos físicamente en Oracle:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   -- Intentar hacer INSERT en una tabla externa (debe fallar)
   INSERT INTO productos_externos VALUES (9999, 'Test', 'Test', 100, 1, SYSDATE);

   -- Verificar que no hay segmentos de datos para la tabla externa
   SELECT segment_name, segment_type, bytes
   FROM user_segments
   WHERE segment_name = 'PRODUCTOS_EXTERNOS';

   EXIT;
   EOF
   ```

**Salida Esperada:**

```sql
-- Resultado de SELECT * FROM productos_externos
PRODUCTO_ID NOMBRE                CATEGORIA    PRECIO   STOCK FECHA_INGRE
----------- --------------------- ------------ -------- ----- -----------
       1001 Laptop Pro 15         Electrónica  85000    150   15/01/2024
       1002 Mouse Inalámbrico     Periféricos  450.5    500   20/01/2024
       ...
       1010 Impresora Laser       Oficina      4500     60    20/03/2024

10 rows selected.

-- Resultado de la agregación por categoría
CATEGORIA     CANTIDAD_PRODUCTOS PRECIO_PROMEDIO STOCK_TOTAL
------------- ------------------ --------------- -----------
Audio                          1         2800.75         200
Almacenamiento                 1         2200.5          250
Electrónica                    3        35000           320
Oficina                        1         4500            60
Periféricos                    4          787.625       1320

-- Intento de INSERT en tabla externa (debe fallar)
INSERT INTO productos_externos VALUES (9999, 'Test', 'Test', 100, 1, SYSDATE)
*
ERROR at line 1:
ORA-30657: operation not supported on external organized table
```

**Verificación:**

```sql
-- Verificar que la tabla externa existe en el diccionario
SELECT table_name, type_name, default_directory_name
FROM user_external_tables;
-- Debe mostrar: PRODUCTOS_EXTERNOS, ORACLE_LOADER, EXT_DIR
```

---

### Paso 9: Comparar los Mecanismos de Movimiento de Datos

**Objetivo:** Ejecutar un análisis comparativo entre Data Pump, SQL*Loader y Tablas Externas para consolidar la comprensión de cuándo usar cada herramienta.

**Instrucciones:**

1. Crear una tabla de resumen con los resultados del laboratorio:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   -- Resumen de objetos creados en este laboratorio
   SELECT object_name, object_type, created, last_ddl_time
   FROM user_objects
   ORDER BY object_type, object_name;

   EXIT;
   EOF
   ```

2. Verificar el volumen de datos en cada tabla:

   ```sql
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   SELECT 'EMPLOYEES (importado con Data Pump)' AS origen,
          COUNT(*) AS registros
   FROM employees
   UNION ALL
   SELECT 'DEPARTMENTS (importado con Data Pump)',
          COUNT(*)
   FROM departments
   UNION ALL
   SELECT 'EMPLEADOS_CARGA (cargado con SQL*Loader)',
          COUNT(*)
   FROM empleados_carga
   UNION ALL
   SELECT 'PRODUCTOS_EXTERNOS (tabla externa - sin carga física)',
          COUNT(*)
   FROM productos_externos;

   EXIT;
   EOF
   ```

3. Verificar los archivos generados durante el laboratorio:

   ```bash
   echo "=== Archivos Data Pump ==="
   ls -lh /u01/oracle/datapump/*.dmp 2>/dev/null
   ls -lh /u01/oracle/datapump/*.log 2>/dev/null

   echo ""
   echo "=== Archivos SQL*Loader ==="
   ls -lh /u01/oracle/sqlldr/ 2>/dev/null

   echo ""
   echo "=== Archivos Tablas Externas ==="
   ls -lh /u01/oracle/external/ 2>/dev/null
   ```

4. Registrar las observaciones clave en un script de análisis:

   ```sql
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF

   -- Verificar todos los directorios Oracle configurados
   SELECT directory_name, directory_path
   FROM dba_directories
   WHERE directory_name IN ('DP_DIR', 'SQLLDR_DIR', 'EXT_DIR')
   ORDER BY directory_name;

   -- Verificar los privilegios sobre los directorios
   SELECT grantee, granted_role, privilege, directory_name
   FROM (
       SELECT tp.grantee, NULL AS granted_role, tp.privilege, tp.table_name AS directory_name
       FROM dba_tab_privs tp
       WHERE tp.table_name IN ('DP_DIR', 'SQLLDR_DIR', 'EXT_DIR')
   )
   ORDER BY directory_name, grantee;

   EXIT;
   EOF
   ```

**Salida Esperada:**

```
=== Resumen de registros por mecanismo ===
ORIGEN                                               REGISTROS
---------------------------------------------------- ---------
EMPLOYEES (importado con Data Pump)                          5
DEPARTMENTS (importado con Data Pump)                        5
EMPLEADOS_CARGA (cargado con SQL*Loader)                    12
PRODUCTOS_EXTERNOS (tabla externa - sin carga física)       10
```

**Verificación:**

- Data Pump: Archivos `.dmp` y `.log` presentes en `/u01/oracle/datapump/`
- SQL*Loader: Datos cargados en `EMPLEADOS_CARGA`, archivos `.log` y posiblemente `.bad` en `/u01/oracle/sqlldr/`
- Tabla Externa: `PRODUCTOS_EXTERNOS` consultable pero sin segmentos físicos en Oracle

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El objeto `DIRECTORY` `DP_DIR` fue creado y apunta a `/u01/oracle/datapump` con privilegios READ/WRITE asignados a HR, SYSTEM y PRACTICA_USER
- [ ] Se generaron al menos 3 archivos dump: `hr_full_export.dmp`, `hr_metadata_only.dmp` y `hr_noindex.dmp`
- [ ] La importación con `REMAP_SCHEMA=HR:PRACTICA_USER` creó correctamente las tablas `EMPLOYEES` y `DEPARTMENTS` en el esquema `PRACTICA_USER` con sus datos
- [ ] SQL*Loader cargó exitosamente 10 registros en la tabla `PRACTICA_USER.EMPLEADOS_CARGA` desde el archivo CSV
- [ ] Los registros con errores del segundo CSV fueron rechazados y registrados en el archivo `.bad`
- [ ] La tabla externa `PRODUCTOS_EXTERNOS` fue creada y permite consultas SQL que retornan 10 filas
- [ ] Un intento de `INSERT` en la tabla externa resultó en el error `ORA-30657`
- [ ] La vista `DBA_DATAPUMP_JOBS` muestra los trabajos completados del laboratorio

### Procedimiento de Pruebas

1. Verificar la exportación completa:

   ```bash
   # El archivo dump debe existir y tener tamaño mayor a 0
   test -f /u01/oracle/datapump/hr_full_export.dmp && \
     echo "PASS: Archivo dump existe" || echo "FAIL: Archivo dump no encontrado"

   # Verificar tamaño del archivo
   du -h /u01/oracle/datapump/hr_full_export.dmp
   ```
   **Resultado Esperado:** `PASS: Archivo dump existe` y tamaño mayor a 100 KB

2. Verificar la importación con remapeo:

   ```bash
   sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT COUNT(*) AS tablas_importadas
   FROM dba_tables
   WHERE owner = 'PRACTICA_USER';
   EXIT;
   EOF
   ```
   **Resultado Esperado:** Al menos 2 tablas (`DEPARTMENTS`, `EMPLOYEES`)

3. Verificar la carga con SQL*Loader:

   ```bash
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT COUNT(*) FROM empleados_carga;
   EXIT;
   EOF
   ```
   **Resultado Esperado:** 12 registros (10 del primer CSV + 2 válidos del segundo)

4. Verificar la tabla externa:

   ```bash
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT COUNT(*) FROM productos_externos;
   EXIT;
   EOF
   ```
   **Resultado Esperado:** 10 filas

5. Verificar que la tabla externa no tiene segmentos físicos:

   ```bash
   sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
   SELECT COUNT(*) AS segmentos_fisicos
   FROM user_segments
   WHERE segment_name = 'PRODUCTOS_EXTERNOS';
   EXIT;
   EOF
   ```
   **Resultado Esperado:** 0 segmentos físicos

---

## Solución de Problemas

### Problema 1: Error ORA-39002 / ORA-39070 – No se puede abrir el archivo de log de Data Pump

**Síntomas:**
- Al ejecutar `expdp` o `impdp`, aparece el error:
  ```
  ORA-39002: invalid operation
  ORA-39070: Unable to open the log file.
  ORA-29283: invalid file operation
  ORA-06512: at "SYS.UTL_FILE", line 536
  ```

**Causa:**
El directorio físico del sistema operativo no existe, no tiene los permisos correctos, o el usuario del sistema operativo que ejecuta Oracle (`oracle`) no tiene permiso de escritura sobre él.

**Solución:**

```bash
# Verificar que el directorio existe
ls -la /u01/oracle/datapump

# Si no existe, crearlo
mkdir -p /u01/oracle/datapump

# Asignar permisos correctos (propietario oracle, grupo dba)
chown oracle:dba /u01/oracle/datapump
chmod 755 /u01/oracle/datapump

# Verificar que el objeto DIRECTORY Oracle apunta a la ruta correcta
sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
SELECT directory_name, directory_path FROM dba_directories WHERE directory_name = 'DP_DIR';
EOF

# Si la ruta es incorrecta, recrear el directorio
sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
CREATE OR REPLACE DIRECTORY dp_dir AS '/u01/oracle/datapump';
EOF
```

---

### Problema 2: Error ORA-39006 / ORA-39065 – Privilegios insuficientes para Data Pump

**Síntomas:**
- Al ejecutar `expdp` con un usuario que no es SYSTEM/SYS, aparece:
  ```
  ORA-39006: internal error
  ORA-39065: unexpected master process exception in DISPATCH
  ORA-01031: insufficient privileges
  ```

**Causa:**
El usuario no tiene el privilegio `DATAPUMP_EXP_FULL_DATABASE` (para exportación full) o no tiene privilegios `READ`/`WRITE` sobre el objeto `DIRECTORY` utilizado.

**Solución:**

```sql
-- Conectarse como SYSTEM y otorgar los privilegios necesarios
sqlplus system/Oracle123@localhost:1521/ORCLPDB1

-- Para exportación de esquema propio (no requiere privilegio full)
GRANT READ, WRITE ON DIRECTORY dp_dir TO hr;

-- Para exportación de esquemas ajenos o full database
GRANT DATAPUMP_EXP_FULL_DATABASE TO hr;

-- Para importación con remapeo
GRANT DATAPUMP_IMP_FULL_DATABASE TO practica_user;

-- Verificar privilegios del usuario
SELECT privilege FROM dba_sys_privs WHERE grantee = 'HR'
UNION ALL
SELECT privilege FROM dba_sys_privs WHERE grantee = 'PRACTICA_USER';
```

---

### Problema 3: SQL*Loader – Error "SQL*Loader-500: Unable to open file" o registros rechazados inesperadamente

**Síntomas:**
- `sqlldr` termina con error:
  ```
  SQL*Loader-500: Unable to open file /u01/oracle/sqlldr/empleados_datos.csv
  ```
- O bien, todos los registros van al archivo `.bad` sin cargarse.

**Causa:**
El archivo de datos no existe en la ruta especificada en el archivo de control, o el archivo de control tiene errores de sintaxis en la definición de campos (delimitadores incorrectos, formato de fecha equivocado).

**Solución:**

```bash
# Verificar que el archivo de datos existe y tiene contenido
ls -la /u01/oracle/sqlldr/empleados_datos.csv
wc -l /u01/oracle/sqlldr/empleados_datos.csv

# Verificar que el archivo tiene permisos de lectura
chmod 644 /u01/oracle/sqlldr/empleados_datos.csv

# Revisar el archivo de control para detectar errores de sintaxis
cat /u01/oracle/sqlldr/empleados_carga.ctl

# Si hay registros en el archivo BAD, revisar el motivo del rechazo
cat /u01/oracle/sqlldr/empleados_carga.bad

# Revisar el log de SQL*Loader para el mensaje de error detallado
cat /u01/oracle/sqlldr/empleados_carga_sqlldr.log | grep -A5 "error\|Error\|ERROR"

# Ejecutar SQL*Loader con el parámetro ERRORS=999 para ver todos los errores
sqlldr userid=practica_user/Oracle123@localhost:1521/ORCLPDB1 \
  control=/u01/oracle/sqlldr/empleados_carga.ctl \
  log=/u01/oracle/sqlldr/debug_load.log \
  errors=999
```

---

### Problema 4: Error ORA-29913 al consultar la Tabla Externa

**Síntomas:**
- Al ejecutar `SELECT * FROM productos_externos`, aparece:
  ```
  ORA-29913: error in executing ODCIEXTTABLEOPEN callout
  ORA-29400: data cartridge error
  KUP-04040: file productos_ext.csv in EXT_DIR not found
  ```

**Causa:**
El archivo CSV referenciado en la tabla externa no existe en el directorio físico apuntado por el objeto `DIRECTORY`, o el nombre del archivo en la cláusula `LOCATION` no coincide exactamente con el nombre del archivo en el sistema operativo (incluyendo mayúsculas/minúsculas).

**Solución:**

```bash
# Verificar que el archivo existe en el directorio correcto
ls -la /u01/oracle/external/

# Si el archivo no existe, crearlo nuevamente
cat > /u01/oracle/external/productos_ext.csv << 'EOF'
1001,Laptop Pro 15,Electrónica,85000.00,150,2024-01-15
1002,Mouse Inalámbrico,Periféricos,450.50,500,2024-01-20
EOF

# Verificar permisos del archivo (debe ser legible por el usuario oracle)
chmod 644 /u01/oracle/external/productos_ext.csv

# Verificar que el objeto DIRECTORY apunta al directorio correcto
sqlplus system/Oracle123@localhost:1521/ORCLPDB1 <<EOF
SELECT directory_name, directory_path
FROM dba_directories
WHERE directory_name = 'EXT_DIR';
EOF

# Verificar el nombre exacto del archivo en la tabla externa
sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
SELECT table_name, location FROM user_external_locations;
EOF
```

---

### Problema 5: Error ORA-31684 – El objeto ya existe durante la importación con impdp

**Síntomas:**
- Durante la importación, aparecen mensajes como:
  ```
  ORA-31684: Object type TABLE:"PRACTICA_USER"."EMPLOYEES" already exists
  ```
- El trabajo termina con estado `COMPLETED WITH ERRORS`.

**Causa:**
Los objetos que se intenta importar ya existen en el esquema destino de una ejecución anterior del laboratorio.

**Solución:**

```sql
-- Opción 1: Eliminar los objetos existentes antes de reimportar
sqlplus practica_user/Oracle123@localhost:1521/ORCLPDB1 <<EOF
DROP TABLE employees CASCADE CONSTRAINTS PURGE;
DROP TABLE departments CASCADE CONSTRAINTS PURGE;
EOF

-- Opción 2: Usar el parámetro TABLE_EXISTS_ACTION en impdp
-- Valores posibles: SKIP (ignorar), APPEND (agregar datos), TRUNCATE (truncar y cargar), REPLACE (recrear)
```

```bash
impdp system/Oracle123@localhost:1521/ORCLPDB1 \
  DIRECTORY=dp_dir \
  DUMPFILE=hr_full_export.dmp \
  LOGFILE=hr_import_replace.log \
  REMAP_SCHEMA=HR:PRACTICA_USER \
  TABLE_EXISTS_ACTION=REPLACE \
  JOB_NAME=import_hr_replace
```

---

## Limpieza

Ejecutar los siguientes comandos para eliminar los objetos y archivos creados durante el laboratorio:

```sql
-- Conectarse como SYSTEM para limpiar los objetos creados
sqlplus system/Oracle123@localhost:1521/ORCLPDB1

-- Eliminar el esquema temporal de demostración
DROP USER temp_schema CASCADE;

-- Eliminar objetos creados en PRACTICA_USER (si se desea limpiar)
-- NOTA: Mantener PRACTICA_USER para prácticas posteriores del curso
-- Solo limpiar las tablas específicas de este laboratorio
DROP TABLE practica_user.empleados_carga PURGE;
DROP TABLE practica_user.productos_externos;

-- Eliminar los directorios Oracle (solo si no se usarán en prácticas posteriores)
-- PRECAUCIÓN: Comentar estas líneas si las prácticas posteriores los necesitan
-- DROP DIRECTORY dp_dir;
-- DROP DIRECTORY sqlldr_dir;
-- DROP DIRECTORY ext_dir;

EXIT;
```

```bash
# Limpiar archivos temporales del sistema operativo
# PRECAUCIÓN: Conservar los dump files si se necesitan para prácticas posteriores

# Limpiar solo los archivos de log de SQL*Loader y archivos BAD
rm -f /u01/oracle/sqlldr/*.log
rm -f /u01/oracle/sqlldr/*.bad
rm -f /u01/oracle/sqlldr/*.dsc

# Limpiar los archivos de datos CSV de SQL*Loader (se pueden regenerar)
rm -f /u01/oracle/sqlldr/*.csv
rm -f /u01/oracle/sqlldr/*.ctl

# Conservar los dump files de Data Pump para referencia
# Si se desea liberar espacio, ejecutar:
# rm -f /u01/oracle/datapump/*.dmp
# rm -f /u01/oracle/datapump/*.log

# Verificar el espacio liberado
df -h /u01/
```

> ⚠️ **Advertencia:** No eliminar los objetos de directorio Oracle (`DP_DIR`, `SQLLDR_DIR`, `EXT_DIR`) ni al usuario `PRACTICA_USER` si el curso continúa con prácticas posteriores que dependen de ellos. Las prácticas de módulos 6, 7 y 8 pueden requerir estos objetos. Verificar la **Secuencia Obligatoria** del curso antes de ejecutar la limpieza completa.

> ⚠️ **Advertencia:** Los archivos dump de Data Pump son binarios propietarios de Oracle y no pueden abrirse con editores de texto. Si necesitas inspeccionar su contenido, usa el parámetro `SQLFILE` de `impdp` para extraer el DDL en formato SQL legible: `impdp system/Oracle123 DIRECTORY=dp_dir DUMPFILE=hr_full_export.dmp SQLFILE=hr_ddl.sql`.

---

## Resumen

### Lo que Lograste

- **Configuración de DIRECTORY Objects:** Creaste tres objetos de directorio Oracle (`DP_DIR`, `SQLLDR_DIR`, `EXT_DIR`) que vinculan rutas del sistema operativo con objetos Oracle, y otorgaste los privilegios `READ`/`WRITE` necesarios a los usuarios participantes
- **Exportación con Data Pump:** Ejecutaste tres variantes de exportación con `expdp`: exportación completa con datos, exportación solo de metadatos (`CONTENT=METADATA_ONLY`) y exportación excluyendo índices (`EXCLUDE=INDEX`), comprendiendo el impacto de cada parámetro en el tamaño del dump
- **Importación con remapeo:** Importaste el esquema HR hacia `PRACTICA_USER` usando `REMAP_SCHEMA`, demostrando cómo mover datos entre esquemas diferentes en la misma o diferente base de datos
- **Carga masiva con SQL*Loader:** Creaste archivos de control `.ctl` con definición de campos, formatos de fecha y manejo de errores, y ejecutaste cargas exitosas con `sqlldr`, analizando los archivos de log y bad records
- **Tablas Externas:** Definiste una tabla con `ORGANIZATION EXTERNAL` usando el driver `ORACLE_LOADER`, ejecutaste consultas SQL incluyendo `JOIN` con tablas internas, y verificaste que las tablas externas no almacenan datos físicamente en Oracle

### Conceptos Clave Aprendidos

- **Data Pump es del lado del servidor:** A diferencia de `exp/imp`, toda la operación ocurre dentro del motor Oracle, lo que permite mayor velocidad y el uso de paralelismo con múltiples worker processes
- **El DIRECTORY Object es obligatorio:** Data Pump no acepta rutas del SO directamente; requiere un objeto de directorio Oracle que actúa como una capa de abstracción entre Oracle y el sistema de archivos
- **Cada herramienta tiene su caso de uso óptimo:** Data Pump para mover datos entre bases de datos Oracle; SQL*Loader para ingestar datos masivos desde fuentes externas (CSV, archivos planos); Tablas Externas para acceso de solo lectura a archivos externos sin necesidad de carga física
- **El monitoreo es parte del proceso:** La vista `DBA_DATAPUMP_JOBS` y el modo interactivo de Data Pump permiten controlar trabajos en tiempo real, lo que es crítico para operaciones en bases de datos de producción de gran tamaño
- **Los archivos de control SQL*Loader son la clave:** La flexibilidad de SQL*Loader reside en los archivos `.ctl`, que permiten transformaciones de datos, manejo de múltiples delimitadores, formatos de fecha personalizados y control granular del proceso de carga

### Próximos Pasos

- **Práctica 5.2 – SQL*Loader Avanzado:** Explorar el modo Direct Path de SQL*Loader para cargas de alto rendimiento, el uso de expresiones SQL en los archivos de control y la carga de múltiples tablas desde un solo archivo
- **Explorar REMAP_TABLESPACE:** En entornos donde el tablespace destino tiene un nombre diferente al origen, usar `REMAP_TABLESPACE=origen:destino` en `impdp` para redirigir los objetos al tablespace correcto
- **Data Pump con Transportable Tablespaces:** Investigar el modo `TRANSPORT_TABLESPACES` para mover tablespaces completos entre bases de datos de forma más eficiente que la exportación/importación convencional
- **Automatización con scripts Shell:** Crear scripts `.sh` que automaticen las exportaciones periódicas con Data Pump, incorporando rotación de archivos dump y notificaciones por email, para implementar respaldos lógicos automatizados

---

## Recursos Adicionales

- **Oracle Data Pump Export Guide 19c** – Documentación oficial con todos los parámetros de `expdp`, ejemplos avanzados y consideraciones de rendimiento: [docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-data-pump-export-utility.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-data-pump-export-utility.html)
- **Oracle Data Pump Import Guide 19c** – Guía oficial para `impdp` incluyendo transformaciones de importación, manejo de errores y opciones de remapeo: [docs.oracle.com/en/database/oracle/oracle-database/19/sutil/datapump-import-utility.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/datapump-import-utility.html)
- **SQL*Loader Reference 19c** – Referencia completa de SQL*Loader con sintaxis de archivos de control, modos de carga (conventional vs. direct path) y parámetros de línea de comandos: [docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- **Oracle External Tables 19c** – Documentación del driver `ORACLE_LOADER` y `ORACLE_DATAPUMP` para tablas externas, incluyendo casos de uso para ETL y reporting: [docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tables.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tables.html)
- **DBA_DATAPUMP_JOBS View** – Referencia de la vista del diccionario para monitoreo de trabajos Data Pump: [docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_DATAPUMP_JOBS.html](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_DATAPUMP_JOBS.html)
