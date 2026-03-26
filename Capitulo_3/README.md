# Lab 03-00-01: Administración Completa de Seguridad en Oracle Database — Usuarios, Privilegios, Roles, Perfiles y Auditoría

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 3.1 — Creación y Gestión de Usuarios |

---

## Descripción General

En esta práctica implementarás un modelo completo de seguridad en Oracle Database 19c, abarcando desde la creación de usuarios con atributos de seguridad avanzados hasta la configuración de auditoría unificada para rastrear operaciones críticas. Trabajarás con los comandos `CREATE USER`, `ALTER USER`, `GRANT`, `REVOKE`, `CREATE ROLE`, `CREATE PROFILE` y las políticas de auditoría de Oracle Unified Auditing.

Esta práctica refleja escenarios reales de administración de bases de datos empresariales donde el principio de mínimo privilegio, la segregación de roles y la trazabilidad de operaciones son requisitos fundamentales de cumplimiento normativo (ISO 27001, SOX, PCI-DSS). Al finalizar, habrás construido una arquitectura de seguridad completa y verificable desde el diccionario de datos.

---

## Objetivos de Aprendizaje

Al completar esta práctica, serás capaz de:

- [ ] Crear usuarios Oracle con atributos completos: tablespace por defecto, tablespace temporal, cuota y perfil asignado
- [ ] Otorgar y revocar privilegios de sistema (`CREATE SESSION`, `CREATE TABLE`, `CREATE VIEW`) y de objeto (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) verificando el principio de mínimo privilegio
- [ ] Crear roles personalizados (`ROL_READONLY`, `ROL_DEVELOPER`) y asignarlos a usuarios específicos
- [ ] Definir perfiles de contraseña con políticas de expiración, intentos fallidos y reutilización
- [ ] Habilitar y consultar Oracle Unified Auditing para registrar operaciones DDL y accesos a tablas sensibles

---

## Prerrequisitos

### Conocimiento Requerido

- Comprensión de conceptos de seguridad: autenticación, autorización y auditoría en bases de datos
- Familiaridad con comandos DDL básicos de Oracle (`CREATE TABLE`, `CREATE VIEW`)
- Conocimiento de la arquitectura CDB/PDB de Oracle Database 19c
- Práctica 02-00-01 completada (conectividad Oracle Net operativa y listener activo)

### Acceso Requerido

- Acceso como DBA (`SYS` o `SYSTEM`) a la base de datos Oracle 19c
- Sesión SSH activa a la máquina virtual Oracle Linux 8.x
- SQL*Plus disponible en el PATH del sistema operativo
- Acceso a una PDB (Pluggable Database) operativa, por ejemplo `ORCLPDB1`

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | Intel Core i5 8va gen o AMD Ryzen 5 (mínimo) |
| RAM | 16 GB mínimo (32 GB recomendado) |
| Almacenamiento | 100 GB libres en disco (SSD recomendado) |
| Red | Adaptador de red funcional con loopback |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) | Motor principal de base de datos |
| Oracle Linux | 8.x (8.7 o 8.8) | Sistema operativo huésped |
| SQL*Plus | Incluido con Oracle 19c | Ejecución de comandos SQL/DDL |
| Oracle SQL Developer | 23.1 o superior | Validación gráfica opcional |
| PuTTY / Terminal SSH | 0.79+ / Nativo | Acceso remoto a la VM |

### Configuración Inicial

Antes de comenzar, verifica que el entorno Oracle esté operativo y establece las variables de entorno necesarias:

```bash
# Conectarse a la VM Oracle Linux via SSH
ssh oracle@192.168.56.101

# Configurar variables de entorno Oracle
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME/bin:$PATH

# Verificar que la instancia esté activa
sqlplus -S / as sysdba <<EOF
SELECT instance_name, status, open_mode FROM v\$instance;
EXIT;
EOF
```

**Salida esperada de verificación:**

```
INSTANCE_NAME    STATUS       OPEN_MODE
---------------- ------------ --------------------
orcl             OPEN         READ WRITE
```

```bash
# Verificar que la PDB esté abierta
sqlplus -S / as sysdba <<EOF
SELECT name, open_mode FROM v\$pdbs;
EXIT;
EOF
```

**Salida esperada:**

```
NAME                           OPEN_MODE
------------------------------ ----------
PDB$SEED                       READ ONLY
ORCLPDB1                       READ WRITE
```

> **Nota:** Si `ORCLPDB1` aparece en modo `MOUNTED`, ejecuta: `ALTER PLUGGABLE DATABASE ORCLPDB1 OPEN;`

```bash
# Crear directorio de trabajo para los scripts de esta práctica
mkdir -p /home/oracle/lab03
cd /home/oracle/lab03
```

---

## Instrucciones Paso a Paso

### Paso 1: Crear el Tablespace de Práctica y Preparar el Entorno

**Objetivo:** Crear un tablespace dedicado `PRACTICE_TS` en la PDB para almacenar los objetos de esta práctica, siguiendo la recomendación de aislamiento del entorno de laboratorio.

**Instrucciones:**

1. Conéctate a SQL*Plus como SYSDBA y cambia al contenedor PDB:

   ```bash
   sqlplus / as sysdba
   ```

2. Dentro de SQL*Plus, abre la PDB y crea el tablespace de práctica:

   ```sql
   -- Cambiar al contenedor PDB
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Verificar el contenedor actual
   SELECT SYS_CONTEXT('USERENV', 'CON_NAME') AS contenedor_actual FROM DUAL;

   -- Crear tablespace dedicado para la práctica
   CREATE TABLESPACE practice_ts
       DATAFILE '/u01/app/oracle/oradata/ORCL/ORCLPDB1/practice_ts01.dbf'
       SIZE 100M
       AUTOEXTEND ON NEXT 10M MAXSIZE 500M
       EXTENT MANAGEMENT LOCAL
       SEGMENT SPACE MANAGEMENT AUTO;

   -- Verificar creación del tablespace
   SELECT tablespace_name, status, block_size, extent_management
   FROM   dba_tablespaces
   WHERE  tablespace_name = 'PRACTICE_TS';
   ```

3. Crea también una tabla de datos sensibles que usaremos para las pruebas de privilegios:

   ```sql
   -- Crear tabla de prueba como SYSTEM en ORCLPDB1
   CREATE TABLE system.empleados_sensibles (
       emp_id      NUMBER PRIMARY KEY,
       nombre      VARCHAR2(100),
       salario     NUMBER(10,2),
       departamento VARCHAR2(50),
       fecha_ingreso DATE DEFAULT SYSDATE
   ) TABLESPACE practice_ts;

   -- Insertar datos de prueba
   INSERT INTO system.empleados_sensibles VALUES (1, 'Ana García', 85000, 'Finanzas', SYSDATE);
   INSERT INTO system.empleados_sensibles VALUES (2, 'Carlos López', 72000, 'TI', SYSDATE);
   INSERT INTO system.empleados_sensibles VALUES (3, 'María Torres', 95000, 'Dirección', SYSDATE);
   INSERT INTO system.empleados_sensibles VALUES (4, 'Roberto Silva', 68000, 'RRHH', SYSDATE);
   INSERT INTO system.empleados_sensibles VALUES (5, 'Laura Mendez', 78000, 'TI', SYSDATE);
   COMMIT;

   -- Verificar datos insertados
   SELECT * FROM system.empleados_sensibles;
   ```

**Salida Esperada:**

```
CONTENEDOR_ACTUAL
-----------------
ORCLPDB1

TABLESPACE_NAME   STATUS    BLOCK_SIZE EXTENT_MAN
----------------- --------- ---------- ----------
PRACTICE_TS       ONLINE          8192 LOCAL

EMP_ID NOMBRE              SALARIO DEPARTAMENTO FECHA_INGRESO
------ ------------------- ------- ------------ -------------
     1 Ana García         85000   Finanzas     15-JAN-24
     2 Carlos López       72000   TI           15-JAN-24
     3 María Torres       95000   Dirección    15-JAN-24
     4 Roberto Silva      68000   RRHH         15-JAN-24
     5 Laura Mendez       78000   TI           15-JAN-24
```

**Verificación:**

- Confirma que `TABLESPACE_NAME = PRACTICE_TS` y `STATUS = ONLINE`
- Confirma que se insertaron 5 filas en `system.empleados_sensibles`

---

### Paso 2: Crear Perfiles de Usuario con Políticas de Contraseña y Recursos

**Objetivo:** Definir dos perfiles (`PERFIL_DESARROLLADOR` y `PERFIL_READONLY`) con políticas de contraseña y límites de recursos diferenciados antes de crear los usuarios, ya que los perfiles deben existir previamente.

**Instrucciones:**

1. Crea el perfil para desarrolladores con políticas estrictas de contraseña:

   ```sql
   -- Perfil para usuarios desarrolladores
   CREATE PROFILE perfil_desarrollador LIMIT
       -- Políticas de contraseña
       FAILED_LOGIN_ATTEMPTS    5
       PASSWORD_LIFE_TIME       90
       PASSWORD_REUSE_TIME      365
       PASSWORD_REUSE_MAX       10
       PASSWORD_LOCK_TIME       1/24
       PASSWORD_GRACE_TIME      7
       -- Límites de recursos de sesión
       SESSIONS_PER_USER        3
       CPU_PER_SESSION          UNLIMITED
       CPU_PER_CALL             3000
       CONNECT_TIME             480
       IDLE_TIME                60
       LOGICAL_READS_PER_SESSION UNLIMITED
       PRIVATE_SGA              10M;
   ```

2. Crea el perfil para usuarios de solo lectura con restricciones más estrictas:

   ```sql
   -- Perfil para usuarios de solo lectura (menor privilegio)
   CREATE PROFILE perfil_readonly LIMIT
       -- Políticas de contraseña más estrictas
       FAILED_LOGIN_ATTEMPTS    3
       PASSWORD_LIFE_TIME       60
       PASSWORD_REUSE_TIME      180
       PASSWORD_REUSE_MAX       5
       PASSWORD_LOCK_TIME       1/12
       PASSWORD_GRACE_TIME      3
       -- Límites de recursos más restrictivos
       SESSIONS_PER_USER        2
       CPU_PER_SESSION          10000
       CPU_PER_CALL             1000
       CONNECT_TIME             240
       IDLE_TIME                30
       LOGICAL_READS_PER_SESSION 100000
       PRIVATE_SGA              5M;
   ```

3. Verifica los perfiles creados en el diccionario de datos:

   ```sql
   -- Consultar los perfiles creados y sus límites
   SELECT profile,
          resource_name,
          resource_type,
          limit
   FROM   dba_profiles
   WHERE  profile IN ('PERFIL_DESARROLLADOR', 'PERFIL_READONLY')
   ORDER  BY profile, resource_type, resource_name;
   ```

4. Activa la verificación de recursos (necesaria para que los límites de CPU/sesión tengan efecto):

   ```sql
   -- Verificar si la gestión de recursos está activa
   SHOW PARAMETER RESOURCE_LIMIT;

   -- Activar límites de recursos si está en FALSE
   ALTER SYSTEM SET RESOURCE_LIMIT = TRUE;
   ```

**Salida Esperada:**

```
PROFILE               RESOURCE_NAME                  RESOURCE_TYPE LIMIT
--------------------- ------------------------------ ------------- ---------
PERFIL_DESARROLLADOR  CONNECT_TIME                   KERNEL        480
PERFIL_DESARROLLADOR  CPU_PER_CALL                   KERNEL        3000
PERFIL_DESARROLLADOR  FAILED_LOGIN_ATTEMPTS          PASSWORD      5
PERFIL_DESARROLLADOR  IDLE_TIME                      KERNEL        60
PERFIL_DESARROLLADOR  PASSWORD_GRACE_TIME            PASSWORD      7
PERFIL_DESARROLLADOR  PASSWORD_LIFE_TIME             PASSWORD      90
PERFIL_DESARROLLADOR  PASSWORD_LOCK_TIME             PASSWORD      .04166667
PERFIL_DESARROLLADOR  PASSWORD_REUSE_MAX             PASSWORD      10
PERFIL_DESARROLLADOR  PASSWORD_REUSE_TIME            PASSWORD      365
PERFIL_DESARROLLADOR  SESSIONS_PER_USER              KERNEL        3
PERFIL_READONLY       FAILED_LOGIN_ATTEMPTS          PASSWORD      3
PERFIL_READONLY       PASSWORD_LIFE_TIME             PASSWORD      60
...

NAME                 TYPE        VALUE
-------------------- ----------- -----
resource_limit       boolean     TRUE
```

**Verificación:**

- Ambos perfiles deben aparecer en `DBA_PROFILES` con sus límites correspondientes
- `RESOURCE_LIMIT` debe ser `TRUE`

---

### Paso 3: Crear Usuarios con Atributos Completos de Seguridad

**Objetivo:** Crear tres usuarios con diferentes configuraciones de seguridad que representen roles empresariales reales: un desarrollador, un usuario de solo lectura y un administrador de aplicación.

**Instrucciones:**

1. Crea el usuario desarrollador con perfil y cuota específica:

   ```sql
   -- Usuario desarrollador de aplicación
   CREATE USER dev_juan
       IDENTIFIED BY "DevJuan#2024"
       DEFAULT TABLESPACE practice_ts
       TEMPORARY TABLESPACE temp
       QUOTA 200M ON practice_ts
       PROFILE perfil_desarrollador
       ACCOUNT UNLOCK;

   -- Verificar creación
   SELECT username,
          account_status,
          default_tablespace,
          temporary_tablespace,
          profile,
          created,
          lock_date,
          expiry_date
   FROM   dba_users
   WHERE  username = 'DEV_JUAN';
   ```

2. Crea el usuario de solo lectura para reportes:

   ```sql
   -- Usuario de solo lectura para reportes
   CREATE USER rpt_maria
       IDENTIFIED BY "RptMaria#2024"
       DEFAULT TABLESPACE practice_ts
       TEMPORARY TABLESPACE temp
       QUOTA 0 ON practice_ts
       PROFILE perfil_readonly
       ACCOUNT UNLOCK;

   -- Verificar creación
   SELECT username,
          account_status,
          default_tablespace,
          profile
   FROM   dba_users
   WHERE  username = 'RPT_MARIA';
   ```

3. Crea el usuario administrador de aplicación con cuota ilimitada:

   ```sql
   -- Usuario administrador de aplicación ERP
   CREATE USER app_erp
       IDENTIFIED BY "AppErp#2024"
       DEFAULT TABLESPACE practice_ts
       TEMPORARY TABLESPACE temp
       QUOTA UNLIMITED ON practice_ts
       PROFILE perfil_desarrollador
       ACCOUNT UNLOCK;

   -- Verificar los tres usuarios creados
   SELECT username,
          account_status,
          default_tablespace,
          temporary_tablespace,
          profile,
          TO_CHAR(created, 'DD-MON-YYYY HH24:MI') AS fecha_creacion
   FROM   dba_users
   WHERE  username IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP')
   ORDER  BY username;
   ```

4. Verifica las cuotas asignadas en el tablespace:

   ```sql
   -- Consultar cuotas de tablespace de los usuarios creados
   SELECT username,
          tablespace_name,
          bytes / 1024 / 1024 AS mb_usados,
          CASE max_bytes
              WHEN -1 THEN 'UNLIMITED'
              ELSE TO_CHAR(max_bytes / 1024 / 1024) || ' MB'
          END AS cuota_maxima
   FROM   dba_ts_quotas
   WHERE  username IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP')
   ORDER  BY username;
   ```

**Salida Esperada:**

```
USERNAME    ACCOUNT_STATUS DEFAULT_TABLESPACE TEMPORARY_TABLESPACE PROFILE
----------- -------------- ------------------ -------------------- --------------------
APP_ERP     OPEN           PRACTICE_TS        TEMP                 PERFIL_DESARROLLADOR
DEV_JUAN    OPEN           PRACTICE_TS        TEMP                 PERFIL_DESARROLLADOR
RPT_MARIA   OPEN           PRACTICE_TS        TEMP                 PERFIL_READONLY

USERNAME    TABLESPACE_NAME  MB_USADOS CUOTA_MAXIMA
----------- ---------------- --------- ------------
APP_ERP     PRACTICE_TS              0 UNLIMITED
DEV_JUAN    PRACTICE_TS              0 200 MB
RPT_MARIA   PRACTICE_TS              0 0 MB
```

**Verificación:**

- Los tres usuarios deben aparecer con `ACCOUNT_STATUS = OPEN`
- `RPT_MARIA` tiene cuota `0 MB` (no puede crear objetos directamente)
- `APP_ERP` tiene cuota `UNLIMITED`

---

### Paso 4: Otorgar Privilegios de Sistema

**Objetivo:** Aplicar el principio de mínimo privilegio otorgando únicamente los privilegios de sistema necesarios para cada rol de usuario, verificando que los privilegios se registran correctamente en el diccionario de datos.

**Instrucciones:**

1. Otorga privilegios de sistema básicos al desarrollador:

   ```sql
   -- Privilegios mínimos para que el desarrollador pueda conectarse y crear objetos
   GRANT CREATE SESSION TO dev_juan;
   GRANT CREATE TABLE TO dev_juan;
   GRANT CREATE VIEW TO dev_juan;
   GRANT CREATE SEQUENCE TO dev_juan;
   GRANT CREATE PROCEDURE TO dev_juan;
   GRANT CREATE TRIGGER TO dev_juan;

   -- Verificar privilegios de sistema otorgados a dev_juan
   SELECT grantee,
          privilege,
          admin_option
   FROM   dba_sys_privs
   WHERE  grantee = 'DEV_JUAN'
   ORDER  BY privilege;
   ```

2. Otorga privilegios mínimos al usuario de reportes:

   ```sql
   -- Solo necesita conectarse, los privilegios de objeto se otorgan por separado
   GRANT CREATE SESSION TO rpt_maria;

   -- Verificar
   SELECT grantee, privilege, admin_option
   FROM   dba_sys_privs
   WHERE  grantee = 'RPT_MARIA';
   ```

3. Otorga privilegios ampliados al administrador de aplicación:

   ```sql
   -- Privilegios para administrador de aplicación ERP
   GRANT CREATE SESSION TO app_erp;
   GRANT CREATE TABLE TO app_erp;
   GRANT CREATE VIEW TO app_erp;
   GRANT CREATE SEQUENCE TO app_erp;
   GRANT CREATE PROCEDURE TO app_erp;
   GRANT CREATE TRIGGER TO app_erp;
   GRANT CREATE TYPE TO app_erp;
   GRANT CREATE DATABASE LINK TO app_erp;
   GRANT ALTER SESSION TO app_erp;

   -- Verificar todos los privilegios de sistema de los tres usuarios
   SELECT grantee,
          privilege,
          admin_option
   FROM   dba_sys_privs
   WHERE  grantee IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP')
   ORDER  BY grantee, privilege;
   ```

4. Prueba que los usuarios pueden conectarse con sus nuevos privilegios:

   ```bash
   # Abrir una segunda terminal SSH para probar conectividad
   # (ejecutar desde el shell de Oracle Linux, NO desde SQL*Plus)
   ```

   ```sql
   -- Desde SQL*Plus actual, probar conexión como dev_juan
   CONNECT dev_juan/"DevJuan#2024"@ORCLPDB1

   -- Verificar sesión activa
   SELECT USER AS usuario_actual,
          SYS_CONTEXT('USERENV', 'CON_NAME') AS contenedor,
          SYS_CONTEXT('USERENV', 'SESSION_USER') AS sesion_usuario
   FROM   DUAL;

   -- Intentar crear una tabla (debe funcionar)
   CREATE TABLE test_dev (
       id NUMBER PRIMARY KEY,
       descripcion VARCHAR2(100)
   );

   -- Confirmar creación
   SELECT table_name FROM user_tables WHERE table_name = 'TEST_DEV';

   -- Volver a conectarse como SYSTEM para continuar
   CONNECT system/"Oracle123"@ORCLPDB1
   ALTER SESSION SET CONTAINER = ORCLPDB1;
   ```

**Salida Esperada:**

```
GRANTEE     PRIVILEGE                ADMIN_OPTION
----------- ------------------------ ------------
APP_ERP     ALTER SESSION            NO
APP_ERP     CREATE DATABASE LINK     NO
APP_ERP     CREATE PROCEDURE         NO
APP_ERP     CREATE SEQUENCE          NO
APP_ERP     CREATE SESSION           NO
APP_ERP     CREATE TABLE             NO
APP_ERP     CREATE TRIGGER           NO
APP_ERP     CREATE TYPE              NO
APP_ERP     CREATE VIEW              NO
DEV_JUAN    CREATE PROCEDURE         NO
DEV_JUAN    CREATE SEQUENCE          NO
DEV_JUAN    CREATE SESSION           NO
DEV_JUAN    CREATE TABLE             NO
DEV_JUAN    CREATE TRIGGER           NO
DEV_JUAN    CREATE VIEW              NO
RPT_MARIA   CREATE SESSION           NO

USUARIO_ACTUAL  CONTENEDOR  SESION_USUARIO
--------------- ----------- ---------------
DEV_JUAN        ORCLPDB1    DEV_JUAN

TABLE_NAME
----------
TEST_DEV
```

**Verificación:**

- `RPT_MARIA` solo tiene `CREATE SESSION`
- `DEV_JUAN` puede crear tablas exitosamente
- La reconexión como `SYSTEM` debe funcionar sin errores

---

### Paso 5: Otorgar y Revocar Privilegios de Objeto

**Objetivo:** Aplicar privilegios de objeto sobre la tabla `empleados_sensibles` a los diferentes usuarios, verificando el principio de mínimo privilegio y el efecto de revocar privilegios.

**Instrucciones:**

1. Otorga privilegios de objeto diferenciados por usuario:

   ```sql
   -- Asegurarse de estar en ORCLPDB1 como SYSTEM o SYS
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- dev_juan puede leer y modificar datos de empleados
   GRANT SELECT, INSERT, UPDATE ON system.empleados_sensibles TO dev_juan;

   -- rpt_maria solo puede leer (mínimo privilegio para reportes)
   GRANT SELECT ON system.empleados_sensibles TO rpt_maria;

   -- app_erp tiene control total sobre la tabla
   GRANT SELECT, INSERT, UPDATE, DELETE ON system.empleados_sensibles TO app_erp;

   -- Verificar privilegios de objeto otorgados
   SELECT grantee,
          owner,
          table_name,
          privilege,
          grantable,
          hierarchy
   FROM   dba_tab_privs
   WHERE  table_name = 'EMPLEADOS_SENSIBLES'
   AND    grantee IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP')
   ORDER  BY grantee, privilege;
   ```

2. Prueba los privilegios de objeto con el usuario `rpt_maria`:

   ```sql
   -- Conectarse como rpt_maria para probar
   CONNECT rpt_maria/"RptMaria#2024"@ORCLPDB1

   -- Esta consulta DEBE funcionar (tiene SELECT)
   SELECT emp_id, nombre, departamento
   FROM   system.empleados_sensibles;

   -- Este INSERT DEBE FALLAR (no tiene privilegio INSERT)
   INSERT INTO system.empleados_sensibles
   VALUES (6, 'Prueba Fallo', 50000, 'Test', SYSDATE);
   ```

3. Prueba la revocación de privilegios:

   ```sql
   -- Volver como SYSTEM
   CONNECT system/"Oracle123"@ORCLPDB1
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Revocar privilegio UPDATE de dev_juan (ya no necesita modificar salarios)
   REVOKE UPDATE ON system.empleados_sensibles FROM dev_juan;

   -- Verificar que UPDATE fue revocado
   SELECT grantee, privilege
   FROM   dba_tab_privs
   WHERE  table_name = 'EMPLEADOS_SENSIBLES'
   AND    grantee = 'DEV_JUAN';

   -- Confirmar que dev_juan ya no puede hacer UPDATE
   CONNECT dev_juan/"DevJuan#2024"@ORCLPDB1

   -- Este UPDATE DEBE FALLAR después de la revocación
   UPDATE system.empleados_sensibles
   SET    salario = 99999
   WHERE  emp_id = 1;

   -- Volver como SYSTEM
   CONNECT system/"Oracle123"@ORCLPDB1
   ALTER SESSION SET CONTAINER = ORCLPDB1;
   ```

**Salida Esperada:**

```
GRANTEE     OWNER   TABLE_NAME            PRIVILEGE  GRANTABLE
----------- ------- --------------------- ---------- ---------
APP_ERP     SYSTEM  EMPLEADOS_SENSIBLES   DELETE     NO
APP_ERP     SYSTEM  EMPLEADOS_SENSIBLES   INSERT     NO
APP_ERP     SYSTEM  EMPLEADOS_SENSIBLES   SELECT     NO
APP_ERP     SYSTEM  EMPLEADOS_SENSIBLES   UPDATE     NO
DEV_JUAN    SYSTEM  EMPLEADOS_SENSIBLES   INSERT     NO
DEV_JUAN    SYSTEM  EMPLEADOS_SENSIBLES   SELECT     NO
DEV_JUAN    SYSTEM  EMPLEADOS_SENSIBLES   UPDATE     NO
RPT_MARIA   SYSTEM  EMPLEADOS_SENSIBLES   SELECT     NO

-- rpt_maria: SELECT funciona
EMP_ID NOMBRE              DEPARTAMENTO
------ ------------------- ------------
     1 Ana García         Finanzas
     2 Carlos López       TI
...

-- rpt_maria: INSERT falla
ERROR at line 1:
ORA-01031: insufficient privileges

-- dev_juan: UPDATE falla después de REVOKE
ERROR at line 1:
ORA-01031: insufficient privileges
```

**Verificación:**

- `rpt_maria` puede hacer `SELECT` pero no `INSERT`
- Después del `REVOKE`, `dev_juan` no puede hacer `UPDATE`
- El diccionario de datos refleja los cambios inmediatamente

---

### Paso 6: Crear y Gestionar Roles Personalizados

**Objetivo:** Crear roles `ROL_READONLY` y `ROL_DEVELOPER` que encapsulen conjuntos de privilegios, simplificando la administración de seguridad cuando hay múltiples usuarios con los mismos requisitos de acceso.

**Instrucciones:**

1. Crea los roles personalizados:

   ```sql
   -- Asegurarse de estar en ORCLPDB1 como SYSTEM
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Crear rol de solo lectura
   CREATE ROLE rol_readonly;

   -- Crear rol de desarrollador
   CREATE ROLE rol_developer;

   -- Crear rol de administrador de aplicación (con contraseña de rol)
   CREATE ROLE rol_app_admin IDENTIFIED BY "RolAdmin#2024";

   -- Verificar creación de roles
   SELECT role,
          password_required,
          authentication_type
   FROM   dba_roles
   WHERE  role IN ('ROL_READONLY', 'ROL_DEVELOPER', 'ROL_APP_ADMIN')
   ORDER  BY role;
   ```

2. Asigna privilegios a los roles:

   ```sql
   -- Privilegios para ROL_READONLY
   GRANT CREATE SESSION TO rol_readonly;
   GRANT SELECT ON system.empleados_sensibles TO rol_readonly;

   -- Privilegios para ROL_DEVELOPER
   GRANT CREATE SESSION TO rol_developer;
   GRANT CREATE TABLE TO rol_developer;
   GRANT CREATE VIEW TO rol_developer;
   GRANT CREATE SEQUENCE TO rol_developer;
   GRANT CREATE PROCEDURE TO rol_developer;
   GRANT SELECT, INSERT, UPDATE ON system.empleados_sensibles TO rol_developer;

   -- Privilegios para ROL_APP_ADMIN (incluye los del desarrollador más permisos adicionales)
   GRANT rol_developer TO rol_app_admin;
   GRANT CREATE TRIGGER TO rol_app_admin;
   GRANT CREATE TYPE TO rol_app_admin;
   GRANT DELETE ON system.empleados_sensibles TO rol_app_admin;
   GRANT ALTER SESSION TO rol_app_admin;

   -- Verificar privilegios asignados a los roles
   SELECT grantee AS rol,
          privilege,
          admin_option
   FROM   dba_sys_privs
   WHERE  grantee IN ('ROL_READONLY', 'ROL_DEVELOPER', 'ROL_APP_ADMIN')
   ORDER  BY grantee, privilege;
   ```

3. Asigna los roles a los usuarios y verifica la herencia de privilegios:

   ```sql
   -- Asignar roles a usuarios
   -- rpt_maria recibe el rol de solo lectura
   GRANT rol_readonly TO rpt_maria;

   -- dev_juan recibe el rol de desarrollador
   GRANT rol_developer TO dev_juan;

   -- app_erp recibe el rol de administrador de aplicación
   GRANT rol_app_admin TO app_erp;

   -- Verificar roles asignados a los usuarios
   SELECT grantee,
          granted_role,
          admin_option,
          default_role
   FROM   dba_role_privs
   WHERE  grantee IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP')
   ORDER  BY grantee, granted_role;

   -- Verificar privilegios efectivos de dev_juan (directos + por rol)
   SELECT 'SISTEMA' AS tipo,
          privilege AS privilegio,
          NULL AS objeto
   FROM   dba_sys_privs
   WHERE  grantee = 'DEV_JUAN'
   UNION ALL
   SELECT 'OBJETO' AS tipo,
          privilege AS privilegio,
          owner || '.' || table_name AS objeto
   FROM   dba_tab_privs
   WHERE  grantee = 'DEV_JUAN'
   UNION ALL
   SELECT 'ROL' AS tipo,
          granted_role AS privilegio,
          NULL AS objeto
   FROM   dba_role_privs
   WHERE  grantee = 'DEV_JUAN'
   ORDER  BY 1, 2;
   ```

4. Prueba que el rol funciona correctamente para un nuevo usuario:

   ```sql
   -- Crear un nuevo usuario que solo recibe el rol (sin privilegios directos)
   CREATE USER rpt_pedro
       IDENTIFIED BY "RptPedro#2024"
       DEFAULT TABLESPACE practice_ts
       TEMPORARY TABLESPACE temp
       QUOTA 0 ON practice_ts
       PROFILE perfil_readonly
       ACCOUNT UNLOCK;

   -- Asignar solo el rol (sin privilegios directos)
   GRANT rol_readonly TO rpt_pedro;

   -- Probar conexión y acceso de rpt_pedro
   CONNECT rpt_pedro/"RptPedro#2024"@ORCLPDB1

   -- Debe poder hacer SELECT por el rol
   SELECT COUNT(*) AS total_empleados
   FROM   system.empleados_sensibles;

   -- Volver como SYSTEM
   CONNECT system/"Oracle123"@ORCLPDB1
   ALTER SESSION SET CONTAINER = ORCLPDB1;
   ```

**Salida Esperada:**

```
ROLE            PASSWORD_REQUIRED AUTHENTICATION_TYPE
--------------- ----------------- -------------------
ROL_APP_ADMIN   YES               PASSWORD
ROL_DEVELOPER   NO                NONE
ROL_READONLY    NO                NONE

GRANTEE     GRANTED_ROLE    ADMIN_OPTION DEFAULT_ROLE
----------- --------------- ------------ ------------
APP_ERP     ROL_APP_ADMIN   NO           YES
DEV_JUAN    ROL_DEVELOPER   NO           YES
RPT_MARIA   ROL_READONLY    NO           YES

TIPO    PRIVILEGIO          OBJETO
------- ------------------- ---------------------------
OBJETO  INSERT              SYSTEM.EMPLEADOS_SENSIBLES
OBJETO  SELECT              SYSTEM.EMPLEADOS_SENSIBLES
ROL     ROL_DEVELOPER
SISTEMA CREATE PROCEDURE
SISTEMA CREATE SEQUENCE
SISTEMA CREATE SESSION
SISTEMA CREATE TABLE
SISTEMA CREATE TRIGGER
SISTEMA CREATE VIEW

TOTAL_EMPLEADOS
---------------
              5
```

**Verificación:**

- Los tres roles deben aparecer en `DBA_ROLES`
- `ROL_APP_ADMIN` debe tener `PASSWORD_REQUIRED = YES`
- `rpt_pedro` puede hacer `SELECT` solo con el rol, sin privilegios directos

---

### Paso 7: Modificar Usuarios — Bloqueo, Expiración y Cambio de Atributos

**Objetivo:** Practicar operaciones de mantenimiento del ciclo de vida de usuarios: bloquear cuentas durante mantenimiento, forzar cambio de contraseña y modificar atributos de almacenamiento.

**Instrucciones:**

1. Simula un escenario de mantenimiento bloqueando una cuenta:

   ```sql
   -- Asegurarse de estar en ORCLPDB1 como SYSTEM
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Escenario: mantenimiento programado - bloquear cuenta de aplicación
   ALTER USER app_erp ACCOUNT LOCK;

   -- Verificar estado bloqueado
   SELECT username,
          account_status,
          lock_date
   FROM   dba_users
   WHERE  username = 'APP_ERP';

   -- Intentar conectarse como app_erp (DEBE FALLAR)
   CONNECT app_erp/"AppErp#2024"@ORCLPDB1
   ```

2. Desbloquea la cuenta y fuerza cambio de contraseña:

   ```sql
   -- Volver como SYSTEM (si la conexión anterior falló, ya estamos como SYSTEM)
   CONNECT system/"Oracle123"@ORCLPDB1
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Fin del mantenimiento - reactivar y forzar cambio de contraseña
   ALTER USER app_erp ACCOUNT UNLOCK PASSWORD EXPIRE;

   -- Verificar estado: debe mostrar EXPIRED & LOCKED o EXPIRED(GRACE)
   SELECT username,
          account_status,
          lock_date,
          expiry_date
   FROM   dba_users
   WHERE  username = 'APP_ERP';
   ```

3. Modifica la cuota y el tablespace por defecto de un usuario:

   ```sql
   -- Ampliar cuota de dev_juan
   ALTER USER dev_juan QUOTA 500M ON practice_ts;

   -- Cambiar contraseña de dev_juan (simulando rotación de contraseñas)
   ALTER USER dev_juan IDENTIFIED BY "DevJuan#NewPass2024";

   -- Verificar cambios
   SELECT username,
          default_tablespace,
          profile,
          account_status
   FROM   dba_users
   WHERE  username = 'DEV_JUAN';

   -- Verificar nueva cuota
   SELECT username,
          tablespace_name,
          CASE max_bytes
              WHEN -1 THEN 'UNLIMITED'
              ELSE TO_CHAR(max_bytes / 1024 / 1024) || ' MB'
          END AS cuota_maxima
   FROM   dba_ts_quotas
   WHERE  username = 'DEV_JUAN';
   ```

4. Consulta el estado general de todas las cuentas de práctica:

   ```sql
   -- Resumen del estado de todas las cuentas de práctica
   SELECT username,
          account_status,
          TO_CHAR(created, 'DD-MON-YYYY') AS fecha_creacion,
          TO_CHAR(lock_date, 'DD-MON-YYYY HH24:MI') AS fecha_bloqueo,
          TO_CHAR(expiry_date, 'DD-MON-YYYY') AS fecha_expiracion,
          profile
   FROM   dba_users
   WHERE  username IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP', 'RPT_PEDRO')
   ORDER  BY username;
   ```

**Salida Esperada:**

```
USERNAME    ACCOUNT_STATUS LOCK_DATE
----------- -------------- -------------------
APP_ERP     LOCKED         15-JAN-2024 10:30

-- Intento de conexión bloqueada:
ERROR:
ORA-28000: the account is locked

-- Después de UNLOCK PASSWORD EXPIRE:
USERNAME    ACCOUNT_STATUS        LOCK_DATE  EXPIRY_DATE
----------- --------------------- ---------- -----------
APP_ERP     EXPIRED & LOCKED                 15-JAN-2024

-- Cuota actualizada:
USERNAME    TABLESPACE_NAME  CUOTA_MAXIMA
----------- ---------------- ------------
DEV_JUAN    PRACTICE_TS      500 MB

-- Estado general:
USERNAME    ACCOUNT_STATUS    FECHA_CREACION FECHA_BLOQUEO FECHA_EXPIRACION PROFILE
----------- ----------------- -------------- ------------- ---------------- --------------------
APP_ERP     EXPIRED & LOCKED  15-JAN-2024                  15-JAN-2024      PERFIL_DESARROLLADOR
DEV_JUAN    OPEN              15-JAN-2024                  15-APR-2024      PERFIL_DESARROLLADOR
RPT_MARIA   OPEN              15-JAN-2024                  15-MAR-2024      PERFIL_READONLY
RPT_PEDRO   OPEN              15-JAN-2024                  15-MAR-2024      PERFIL_READONLY
```

**Verificación:**

- `APP_ERP` debe mostrar `EXPIRED & LOCKED` después de `UNLOCK PASSWORD EXPIRE`
- La cuota de `DEV_JUAN` debe ser `500 MB`
- Todos los usuarios deben tener fechas de expiración según sus perfiles

---

### Paso 8: Configurar Oracle Unified Auditing

**Objetivo:** Habilitar Oracle Unified Auditing para registrar operaciones críticas sobre la tabla de empleados y operaciones DDL, consultando `UNIFIED_AUDIT_TRAIL` para verificar los registros generados.

**Instrucciones:**

1. Verifica el estado actual de Unified Auditing:

   ```sql
   -- Asegurarse de estar en ORCLPDB1 como SYSTEM
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Verificar si Unified Auditing está habilitado
   SELECT value
   FROM   v$option
   WHERE  parameter = 'Unified Auditing';

   -- Verificar políticas de auditoría existentes
   SELECT policy_name,
          enabled_opt,
          user_name,
          success,
          failure
   FROM   audit_unified_enabled_policies
   ORDER  BY policy_name;
   ```

2. Crea una política de auditoría para operaciones DML sobre datos sensibles:

   ```sql
   -- Política para auditar accesos a la tabla de empleados sensibles
   CREATE AUDIT POLICY pol_audit_empleados
       ACTIONS SELECT ON system.empleados_sensibles,
                INSERT ON system.empleados_sensibles,
                UPDATE ON system.empleados_sensibles,
                DELETE ON system.empleados_sensibles
       WHEN 'SYS_CONTEXT(''USERENV'', ''SESSION_USER'') != ''SYSTEM'''
       EVALUATE PER SESSION;

   -- Política para auditar operaciones DDL de los usuarios de práctica
   CREATE AUDIT POLICY pol_audit_ddl
       ACTIONS CREATE TABLE,
               DROP TABLE,
               ALTER TABLE,
               CREATE VIEW,
               DROP VIEW,
               CREATE PROCEDURE,
               DROP PROCEDURE
       WHEN 'SYS_CONTEXT(''USERENV'', ''SESSION_USER'') IN (''DEV_JUAN'', ''APP_ERP'')'
       EVALUATE PER STATEMENT;

   -- Política para auditar intentos de inicio de sesión fallidos
   CREATE AUDIT POLICY pol_audit_logon_fail
       ACTIONS LOGON;
   ```

3. Habilita las políticas de auditoría:

   ```sql
   -- Habilitar política de empleados para todos los usuarios
   AUDIT POLICY pol_audit_empleados;

   -- Habilitar política DDL
   AUDIT POLICY pol_audit_ddl;

   -- Habilitar política de logon solo para fallos
   AUDIT POLICY pol_audit_logon_fail WHENEVER NOT SUCCESSFUL;

   -- Verificar políticas habilitadas
   SELECT policy_name,
          enabled_opt,
          user_name,
          success,
          failure
   FROM   audit_unified_enabled_policies
   WHERE  policy_name LIKE 'POL_AUDIT%'
   ORDER  BY policy_name;
   ```

4. Genera actividad para poblar el trail de auditoría:

   ```sql
   -- Conectarse como dev_juan para generar eventos auditados
   CONNECT dev_juan/"DevJuan#NewPass2024"@ORCLPDB1

   -- Operación SELECT auditada
   SELECT * FROM system.empleados_sensibles WHERE departamento = 'TI';

   -- Operación INSERT auditada
   INSERT INTO system.empleados_sensibles
   VALUES (6, 'Pedro Nuevo', 71000, 'TI', SYSDATE);
   COMMIT;

   -- Crear tabla (DDL auditado)
   CREATE TABLE dev_juan.proyectos (
       proy_id NUMBER PRIMARY KEY,
       nombre  VARCHAR2(100)
   );

   -- Conectarse como rpt_maria para generar eventos
   CONNECT rpt_maria/"RptMaria#2024"@ORCLPDB1

   SELECT nombre, departamento FROM system.empleados_sensibles;

   -- Intentar login fallido para generar evento de fallo
   CONNECT usuario_inexistente/"WrongPass123"@ORCLPDB1

   -- Volver como SYSTEM para consultar el trail
   CONNECT system/"Oracle123"@ORCLPDB1
   ALTER SESSION SET CONTAINER = ORCLPDB1;
   ```

5. Consulta el `UNIFIED_AUDIT_TRAIL` para verificar los registros:

   ```sql
   -- Consultar eventos de auditoría generados en esta práctica
   SELECT dbusername,
          action_name,
          object_schema,
          object_name,
          unified_audit_policies,
          TO_CHAR(event_timestamp, 'DD-MON-YYYY HH24:MI:SS') AS fecha_evento,
          return_code
   FROM   unified_audit_trail
   WHERE  unified_audit_policies LIKE '%POL_AUDIT%'
   AND    event_timestamp > SYSDATE - 1/24
   ORDER  BY event_timestamp DESC;

   -- Resumen de eventos por usuario y acción
   SELECT dbusername,
          action_name,
          COUNT(*) AS total_eventos,
          MIN(return_code) AS cod_min,
          MAX(return_code) AS cod_max
   FROM   unified_audit_trail
   WHERE  unified_audit_policies LIKE '%POL_AUDIT%'
   AND    event_timestamp > SYSDATE - 1/24
   GROUP  BY dbusername, action_name
   ORDER  BY dbusername, action_name;

   -- Identificar intentos de acceso fallidos
   SELECT dbusername,
          action_name,
          return_code,
          TO_CHAR(event_timestamp, 'DD-MON-YYYY HH24:MI:SS') AS fecha_intento
   FROM   unified_audit_trail
   WHERE  return_code != 0
   AND    event_timestamp > SYSDATE - 1/24
   ORDER  BY event_timestamp DESC;
   ```

**Salida Esperada:**

```
VALUE
-----
TRUE

POLICY_NAME              ENABLED_OPT USER_NAME SUCCESS FAILURE
------------------------ ----------- --------- ------- -------
POL_AUDIT_DDL            BY                    YES     YES
POL_AUDIT_EMPLEADOS      BY                    YES     YES
POL_AUDIT_LOGON_FAIL     BY                    NO      YES

DBUSERNAME  ACTION_NAME OBJECT_SCHEMA OBJECT_NAME          UNIFIED_AUDIT_POLICIES  FECHA_EVENTO             RETURN_CODE
----------- ----------- ------------- -------------------- ----------------------- ------------------------ -----------
DEV_JUAN    SELECT      SYSTEM        EMPLEADOS_SENSIBLES  POL_AUDIT_EMPLEADOS     15-JAN-2024 10:45:30               0
DEV_JUAN    INSERT      SYSTEM        EMPLEADOS_SENSIBLES  POL_AUDIT_EMPLEADOS     15-JAN-2024 10:45:35               0
DEV_JUAN    CREATE TABLE DEV_JUAN     PROYECTOS            POL_AUDIT_DDL           15-JAN-2024 10:45:40               0
RPT_MARIA   SELECT      SYSTEM        EMPLEADOS_SENSIBLES  POL_AUDIT_EMPLEADOS     15-JAN-2024 10:45:50               0

DBUSERNAME          ACTION_NAME TOTAL_EVENTOS COD_MIN COD_MAX
------------------- ----------- ------------- ------- -------
DEV_JUAN            CREATE TABLE            1       0       0
DEV_JUAN            INSERT                  1       0       0
DEV_JUAN            SELECT                  1       0       0
RPT_MARIA           SELECT                  1       0       0
USUARIO_INEXISTENTE LOGON                   1    1017    1017
```

**Verificación:**

- Las tres políticas deben aparecer en `AUDIT_UNIFIED_ENABLED_POLICIES`
- Los eventos de `DEV_JUAN` y `RPT_MARIA` deben aparecer en `UNIFIED_AUDIT_TRAIL`
- El intento de login fallido debe tener `RETURN_CODE != 0` (código 1017 = usuario inválido)

---

### Paso 9: Consultas Avanzadas del Diccionario de Datos

**Objetivo:** Consolidar el conocimiento del diccionario de datos generando reportes completos del estado de seguridad de la base de datos, como lo haría un DBA en un entorno de producción.

**Instrucciones:**

1. Genera un reporte completo de usuarios y sus privilegios:

   ```sql
   -- Asegurarse de estar en ORCLPDB1 como SYSTEM
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Reporte completo: usuarios con sus roles y privilegios directos
   SELECT u.username,
          u.account_status,
          u.profile,
          u.default_tablespace,
          (SELECT COUNT(*)
           FROM   dba_sys_privs sp
           WHERE  sp.grantee = u.username) AS privs_sistema,
          (SELECT COUNT(*)
           FROM   dba_tab_privs tp
           WHERE  tp.grantee = u.username) AS privs_objeto,
          (SELECT COUNT(*)
           FROM   dba_role_privs rp
           WHERE  rp.grantee = u.username) AS roles_asignados
   FROM   dba_users u
   WHERE  u.username IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP', 'RPT_PEDRO')
   ORDER  BY u.username;
   ```

2. Identifica usuarios con privilegios excesivos (análisis de seguridad):

   ```sql
   -- Identificar usuarios con privilegio DBA (potencialmente peligroso)
   SELECT grantee, granted_role, admin_option
   FROM   dba_role_privs
   WHERE  granted_role = 'DBA'
   AND    grantee NOT IN ('SYS', 'SYSTEM', 'DBA')
   ORDER  BY grantee;

   -- Identificar usuarios que pueden otorgar privilegios (ADMIN OPTION)
   SELECT grantee, privilege, admin_option
   FROM   dba_sys_privs
   WHERE  admin_option = 'YES'
   AND    grantee NOT IN ('SYS', 'SYSTEM', 'DBA', 'DATAPUMP_IMP_FULL_DATABASE',
                          'DATAPUMP_EXP_FULL_DATABASE', 'IMP_FULL_DATABASE',
                          'EXP_FULL_DATABASE')
   ORDER  BY grantee, privilege;
   ```

3. Genera un reporte de cuotas y uso de almacenamiento:

   ```sql
   -- Reporte de cuotas y uso de almacenamiento por usuario
   SELECT q.username,
          q.tablespace_name,
          ROUND(q.bytes / 1024 / 1024, 2) AS mb_usados,
          CASE q.max_bytes
              WHEN -1 THEN 'UNLIMITED'
              ELSE ROUND(q.max_bytes / 1024 / 1024, 2) || ' MB'
          END AS cuota_maxima,
          CASE q.max_bytes
              WHEN -1 THEN 0
              WHEN 0  THEN 100
              ELSE ROUND((q.bytes / q.max_bytes) * 100, 1)
          END AS pct_uso
   FROM   dba_ts_quotas q
   WHERE  q.username IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP', 'RPT_PEDRO')
   ORDER  BY q.username, q.tablespace_name;

   -- Verificar usuarios con cuentas no OPEN (bloqueadas o expiradas)
   SELECT username,
          account_status,
          TO_CHAR(lock_date, 'DD-MON-YYYY HH24:MI') AS fecha_bloqueo,
          TO_CHAR(expiry_date, 'DD-MON-YYYY') AS fecha_expiracion
   FROM   dba_users
   WHERE  account_status != 'OPEN'
   AND    username NOT LIKE 'ORACLE%'
   AND    username NOT LIKE 'XDB%'
   AND    username NOT IN ('SYS', 'SYSTEM', 'DBSNMP', 'APPQOSSYS',
                           'DBSFWUSER', 'GGSYS', 'ANONYMOUS', 'CTXSYS',
                           'DVSYS', 'DVF', 'GSMADMIN_INTERNAL', 'MDSYS',
                           'OLAPSYS', 'ORDPLUGINS', 'ORDSYS', 'SI_INFORMTN_SCHEMA',
                           'WMSYS', 'OUTLN', 'LBACSYS')
   ORDER  BY account_status, username;
   ```

**Salida Esperada:**

```
USERNAME    ACCOUNT_STATUS PROFILE              DEFAULT_TABLESPACE PRIVS_SISTEMA PRIVS_OBJETO ROLES_ASIGNADOS
----------- -------------- -------------------- ------------------ ------------- ------------ ---------------
APP_ERP     EXPIRED&LOCKED PERFIL_DESARROLLADOR PRACTICE_TS                    9            1               1
DEV_JUAN    OPEN           PERFIL_DESARROLLADOR PRACTICE_TS                    6            2               1
RPT_MARIA   OPEN           PERFIL_READONLY      PRACTICE_TS                    1            1               1
RPT_PEDRO   OPEN           PERFIL_READONLY      PRACTICE_TS                    0            0               1

-- No deben aparecer usuarios de práctica con rol DBA
no rows selected

USERNAME    TABLESPACE_NAME  MB_USADOS CUOTA_MAXIMA PCT_USO
----------- ---------------- --------- ------------ -------
APP_ERP     PRACTICE_TS              0 UNLIMITED          0
DEV_JUAN    PRACTICE_TS              0 500 MB             0
RPT_MARIA   PRACTICE_TS              0 0 MB             100
RPT_PEDRO   PRACTICE_TS              0 0 MB             100

USERNAME    ACCOUNT_STATUS    FECHA_BLOQUEO        FECHA_EXPIRACION
----------- ----------------- -------------------- ----------------
APP_ERP     EXPIRED & LOCKED                       15-JAN-2024
```

**Verificación:**

- El reporte de usuarios muestra la cantidad correcta de privilegios por usuario
- `RPT_PEDRO` tiene `0` privilegios directos pero `1` rol asignado
- Solo `APP_ERP` debe aparecer como no `OPEN`

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El tablespace `PRACTICE_TS` fue creado exitosamente y está en estado `ONLINE`
- [ ] Los perfiles `PERFIL_DESARROLLADOR` y `PERFIL_READONLY` existen en `DBA_PROFILES` con los límites correctos
- [ ] Los cuatro usuarios (`DEV_JUAN`, `RPT_MARIA`, `APP_ERP`, `RPT_PEDRO`) existen en `DBA_USERS`
- [ ] `RPT_MARIA` solo tiene `CREATE SESSION` como privilegio de sistema directo
- [ ] Los roles `ROL_READONLY`, `ROL_DEVELOPER` y `ROL_APP_ADMIN` existen en `DBA_ROLES`
- [ ] `RPT_PEDRO` puede hacer `SELECT` en `empleados_sensibles` únicamente a través del rol
- [ ] Las tres políticas de auditoría están habilitadas en `AUDIT_UNIFIED_ENABLED_POLICIES`
- [ ] `UNIFIED_AUDIT_TRAIL` contiene registros de las operaciones realizadas durante la práctica
- [ ] `APP_ERP` está en estado `EXPIRED & LOCKED` después del paso 7

### Procedimiento de Prueba

1. Verifica el estado completo del entorno de seguridad:

   ```sql
   -- Conectarse como SYSTEM a ORCLPDB1
   ALTER SESSION SET CONTAINER = ORCLPDB1;

   -- Prueba 1: Verificar todos los usuarios de práctica
   SELECT username, account_status, profile
   FROM   dba_users
   WHERE  username IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP', 'RPT_PEDRO')
   ORDER  BY username;
   ```

   **Resultado Esperado:** 4 filas con los usuarios y sus perfiles correctos

2. Verifica la integridad de los roles:

   ```sql
   -- Prueba 2: Verificar roles y sus asignaciones
   SELECT r.role,
          COUNT(rp.grantee) AS usuarios_con_rol
   FROM   dba_roles r
   LEFT JOIN dba_role_privs rp ON rp.granted_role = r.role
   WHERE  r.role IN ('ROL_READONLY', 'ROL_DEVELOPER', 'ROL_APP_ADMIN')
   GROUP  BY r.role
   ORDER  BY r.role;
   ```

   **Resultado Esperado:**

   ```
   ROLE            USUARIOS_CON_ROL
   --------------- ----------------
   ROL_APP_ADMIN                  1
   ROL_DEVELOPER                  1
   ROL_READONLY                   3
   ```

3. Verifica la auditoría activa:

   ```sql
   -- Prueba 3: Contar eventos de auditoría generados
   SELECT COUNT(*) AS total_eventos_auditados
   FROM   unified_audit_trail
   WHERE  unified_audit_policies LIKE '%POL_AUDIT%'
   AND    event_timestamp > SYSDATE - 1/24;
   ```

   **Resultado Esperado:** Al menos 5 eventos auditados

4. Prueba el principio de mínimo privilegio:

   ```sql
   -- Prueba 4: Verificar que rpt_pedro no tiene privilegios directos
   SELECT COUNT(*) AS privs_directos
   FROM   dba_sys_privs
   WHERE  grantee = 'RPT_PEDRO';
   ```

   **Resultado Esperado:** `PRIVS_DIRECTOS = 0`

---

## Solución de Problemas

### Problema 1: ORA-01031 al Intentar Crear Objetos con un Usuario Nuevo

**Síntomas:**
- El usuario puede conectarse (`CREATE SESSION` funciona)
- Al ejecutar `CREATE TABLE` aparece `ORA-01031: insufficient privileges`
- El usuario tiene el privilegio `CREATE TABLE` otorgado

**Causa:**
El usuario tiene el privilegio `CREATE TABLE` pero no tiene cuota en ningún tablespace. En Oracle, tener el privilegio de crear una tabla no otorga automáticamente espacio para almacenarla. La cuota debe ser otorgada explícitamente sobre el tablespace por defecto.

**Solución:**

```sql
-- Verificar si el usuario tiene cuota
SELECT username, tablespace_name, max_bytes
FROM   dba_ts_quotas
WHERE  username = 'NOMBRE_USUARIO';

-- Si max_bytes = 0, otorgar cuota
ALTER USER nombre_usuario QUOTA 100M ON practice_ts;

-- Alternativa: cuota ilimitada en el tablespace específico
ALTER USER nombre_usuario QUOTA UNLIMITED ON practice_ts;

-- O bien, otorgar el privilegio de sistema UNLIMITED TABLESPACE
-- (solo recomendado para DBA, no para usuarios de aplicación)
GRANT UNLIMITED TABLESPACE TO nombre_usuario;
```

---

### Problema 2: ORA-28000 — La Cuenta Está Bloqueada

**Síntomas:**
- Al intentar conectarse aparece `ORA-28000: the account is locked`
- El usuario estaba funcionando previamente
- Puede haber ocurrido después de varios intentos fallidos de login

**Causa:**
La cuenta fue bloqueada automáticamente por el perfil de usuario al exceder el número máximo de intentos fallidos (`FAILED_LOGIN_ATTEMPTS`), o fue bloqueada manualmente con `ALTER USER ... ACCOUNT LOCK`.

**Solución:**

```sql
-- Verificar el estado de la cuenta
SELECT username,
       account_status,
       lock_date,
       profile
FROM   dba_users
WHERE  username = 'NOMBRE_USUARIO';

-- Verificar el límite de intentos del perfil
SELECT profile, resource_name, limit
FROM   dba_profiles
WHERE  profile = 'NOMBRE_PERFIL'
AND    resource_name = 'FAILED_LOGIN_ATTEMPTS';

-- Desbloquear la cuenta
ALTER USER nombre_usuario ACCOUNT UNLOCK;

-- Si también está expirada, resetear contraseña
ALTER USER nombre_usuario IDENTIFIED BY "NuevaContrasena#2024" ACCOUNT UNLOCK;
```

---

### Problema 3: Los Eventos No Aparecen en UNIFIED_AUDIT_TRAIL

**Síntomas:**
- Las políticas de auditoría están habilitadas en `AUDIT_UNIFIED_ENABLED_POLICIES`
- Se ejecutaron operaciones que deberían ser auditadas
- `UNIFIED_AUDIT_TRAIL` no muestra registros recientes

**Causa:**
Oracle Unified Auditing puede usar un modo de escritura asíncrona (queued-write mode) donde los registros se escriben al trail periódicamente, no inmediatamente. Además, puede que Unified Auditing no esté completamente habilitado en modo puro.

**Solución:**

```sql
-- Verificar el modo de auditoría
SELECT value FROM v$option WHERE parameter = 'Unified Auditing';

-- Forzar flush del buffer de auditoría al trail
EXEC DBMS_AUDIT_MGMT.FLUSH_UNIFIED_AUDIT_TRAIL;

-- Verificar si hay registros pendientes
SELECT COUNT(*) FROM unified_audit_trail
WHERE  event_timestamp > SYSDATE - 1;

-- Si el problema persiste, verificar que las políticas estén habilitadas
SELECT policy_name, enabled_opt, success, failure
FROM   audit_unified_enabled_policies
WHERE  policy_name LIKE 'POL_AUDIT%';

-- Verificar que no haya filtros de usuario que excluyan las operaciones
-- Revisar la cláusula WHEN de las políticas creadas
SELECT policy_name, entity_name, entity_type, enabled_opt
FROM   audit_unified_enabled_policies
WHERE  policy_name LIKE 'POL_AUDIT%';
```

---

### Problema 4: ORA-00959 — Tablespace No Existe al Crear Usuario

**Síntomas:**
- Al ejecutar `CREATE USER` aparece `ORA-00959: tablespace 'PRACTICE_TS' does not exist`
- El tablespace fue creado en un paso anterior

**Causa:**
El tablespace fue creado en un contenedor diferente (CDB root en lugar de la PDB), o la sesión actual no está conectada al contenedor correcto.

**Solución:**

```bash
# Verificar el contenedor actual desde SQL*Plus
```

```sql
-- Verificar contenedor actual
SELECT SYS_CONTEXT('USERENV', 'CON_NAME') AS contenedor FROM DUAL;

-- Si estás en CDB$ROOT, cambiar a la PDB
ALTER SESSION SET CONTAINER = ORCLPDB1;

-- Verificar que el tablespace existe en la PDB
SELECT tablespace_name, status
FROM   dba_tablespaces
WHERE  tablespace_name = 'PRACTICE_TS';

-- Si no existe, crear el tablespace en la PDB correcta
CREATE TABLESPACE practice_ts
    DATAFILE '/u01/app/oracle/oradata/ORCL/ORCLPDB1/practice_ts01.dbf'
    SIZE 100M
    AUTOEXTEND ON NEXT 10M MAXSIZE 500M;
```

---

### Problema 5: ORA-02149 — Tablespace Especificado No Disponible para Cuota

**Síntomas:**
- Al ejecutar `ALTER USER ... QUOTA ... ON tablespace_name` aparece error
- O al intentar crear objetos el usuario recibe error de cuota

**Causa:**
El tablespace especificado en la cuota no es el tablespace por defecto del usuario, o el tablespace está en estado offline o read-only.

**Solución:**

```sql
-- Verificar estado del tablespace
SELECT tablespace_name, status, contents
FROM   dba_tablespaces
WHERE  tablespace_name = 'PRACTICE_TS';

-- Verificar tablespace por defecto del usuario
SELECT username, default_tablespace
FROM   dba_users
WHERE  username = 'NOMBRE_USUARIO';

-- Si el tablespace está offline, ponerlo online
ALTER TABLESPACE practice_ts ONLINE;

-- Otorgar cuota en el tablespace correcto
ALTER USER nombre_usuario QUOTA 200M ON practice_ts;

-- Verificar cuota otorgada
SELECT username, tablespace_name, max_bytes
FROM   dba_ts_quotas
WHERE  username = 'NOMBRE_USUARIO';
```

---

## Limpieza

Ejecuta los siguientes comandos para eliminar todos los objetos creados durante esta práctica. Realiza la limpieza al finalizar la sesión o cuando necesites liberar espacio.

```sql
-- Conectarse como SYSTEM a ORCLPDB1
ALTER SESSION SET CONTAINER = ORCLPDB1;

-- Paso 1: Deshabilitar y eliminar políticas de auditoría
NOAUDIT POLICY pol_audit_empleados;
NOAUDIT POLICY pol_audit_ddl;
NOAUDIT POLICY pol_audit_logon_fail;

DROP AUDIT POLICY pol_audit_empleados;
DROP AUDIT POLICY pol_audit_ddl;
DROP AUDIT POLICY pol_audit_logon_fail;

-- Paso 2: Eliminar usuarios con todos sus objetos (CASCADE)
DROP USER dev_juan CASCADE;
DROP USER rpt_maria CASCADE;
DROP USER app_erp CASCADE;
DROP USER rpt_pedro CASCADE;

-- Paso 3: Eliminar roles personalizados
DROP ROLE rol_readonly;
DROP ROLE rol_developer;
DROP ROLE rol_app_admin;

-- Paso 4: Eliminar perfiles personalizados
DROP PROFILE perfil_desarrollador CASCADE;
DROP PROFILE perfil_readonly CASCADE;

-- Paso 5: Eliminar tabla de datos de prueba
DROP TABLE system.empleados_sensibles PURGE;

-- Paso 6: Eliminar el tablespace de práctica
DROP TABLESPACE practice_ts
    INCLUDING CONTENTS AND DATAFILES
    CASCADE CONSTRAINTS;

-- Paso 7: Verificar limpieza completa
SELECT COUNT(*) AS usuarios_restantes
FROM   dba_users
WHERE  username IN ('DEV_JUAN', 'RPT_MARIA', 'APP_ERP', 'RPT_PEDRO');

SELECT COUNT(*) AS roles_restantes
FROM   dba_roles
WHERE  role IN ('ROL_READONLY', 'ROL_DEVELOPER', 'ROL_APP_ADMIN');

SELECT COUNT(*) AS tablespaces_restantes
FROM   dba_tablespaces
WHERE  tablespace_name = 'PRACTICE_TS';
```

**Salida esperada de verificación de limpieza:**

```
USUARIOS_RESTANTES
------------------
                 0

ROLES_RESTANTES
---------------
              0

TABLESPACES_RESTANTES
---------------------
                    0
```

```bash
# Limpiar archivos de trabajo del laboratorio
rm -rf /home/oracle/lab03
```

> ⚠️ **Advertencia:** `DROP USER ... CASCADE` elimina **permanentemente** todos los objetos del esquema del usuario (tablas, vistas, procedimientos, secuencias, etc.). Esta operación es irreversible. En entornos de producción, siempre realiza un respaldo completo con RMAN o Data Pump antes de ejecutar este comando.

> ⚠️ **Advertencia:** `DROP TABLESPACE ... INCLUDING CONTENTS AND DATAFILES` elimina el datafile físico del disco. Asegúrate de que no haya datos importantes en el tablespace antes de ejecutarlo.

> ⚠️ **Nota sobre Auditoría:** Los registros en `UNIFIED_AUDIT_TRAIL` generados durante esta práctica permanecerán en el trail incluso después de la limpieza (son registros históricos). Esto es el comportamiento correcto desde el punto de vista de seguridad. Si necesitas limpiarlos en un entorno de laboratorio, un DBA puede usar `DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL`.

---

## Resumen

### Lo que Lograste

- **Creaste un tablespace dedicado** `PRACTICE_TS` para aislar los objetos de práctica, siguiendo las mejores prácticas de administración Oracle
- **Definiste perfiles de usuario** (`PERFIL_DESARROLLADOR`, `PERFIL_READONLY`) con políticas de contraseña diferenciadas (intentos fallidos, tiempo de vida, reutilización) y límites de recursos de sesión
- **Creaste cuatro usuarios** con atributos completos: tablespace por defecto, tablespace temporal, cuota específica y perfil asignado, representando roles empresariales reales
- **Aplicaste el principio de mínimo privilegio** otorgando solo los privilegios de sistema necesarios para cada rol (`CREATE SESSION`, `CREATE TABLE`, `CREATE VIEW`, etc.)
- **Gestionaste privilegios de objeto** sobre `empleados_sensibles`, otorgando `SELECT`, `INSERT`, `UPDATE`, `DELETE` de forma selectiva y verificando el efecto de `REVOKE`
- **Creaste roles personalizados** (`ROL_READONLY`, `ROL_DEVELOPER`, `ROL_APP_ADMIN`) que encapsulan conjuntos de privilegios, demostrando cómo simplifican la administración cuando hay múltiples usuarios con los mismos requisitos
- **Ejecutaste operaciones del ciclo de vida** de usuarios: bloqueo de cuenta, desbloqueo con expiración forzada y modificación de cuotas con `ALTER USER`
- **Configuraste Oracle Unified Auditing** con tres políticas diferenciadas (DML sobre tabla sensible, DDL de desarrolladores, intentos de login fallidos) y verificaste los registros en `UNIFIED_AUDIT_TRAIL`
- **Consultaste el diccionario de datos** usando `DBA_USERS`, `DBA_SYS_PRIVS`, `DBA_TAB_PRIVS`, `DBA_ROLES`, `DBA_ROLE_PRIVS`, `DBA_PROFILES`, `DBA_TS_QUOTAS` y `UNIFIED_AUDIT_TRAIL`

### Conceptos Clave Aprendidos

- **Separación de autenticación y autorización:** Crear un usuario solo permite la autenticación. Los privilegios (autorización) deben otorgarse explícitamente; Oracle no asume ningún acceso por defecto
- **Cuota vs. Tablespace por defecto:** Asignar un tablespace por defecto a un usuario NO otorga cuota automáticamente. Son dos operaciones independientes que deben realizarse por separado
- **Roles como contenedores de privilegios:** Los roles simplifican la administración agrupando privilegios. Un cambio en el rol afecta automáticamente a todos los usuarios que lo tienen asignado
- **Perfiles como políticas de seguridad:** Los perfiles aplican políticas de contraseña y límites de recursos de forma centralizada; cambiar el perfil de un usuario cambia inmediatamente sus políticas
- **Unified Auditing como herramienta de trazabilidad:** Las políticas de auditoría permiten registrar operaciones específicas de forma granular, esencial para cumplimiento normativo (SOX, PCI-DSS, ISO 27001)
- **Irreversibilidad de DROP CASCADE:** Las operaciones de eliminación con `CASCADE` son permanentes; el ciclo de vida de los usuarios debe gestionarse con precaución en producción

### Próximos Pasos

- **Lección 3.2 — Privilegios de Sistema y de Objeto:** Profundizar en la jerarquía completa de privilegios Oracle, `GRANT WITH ADMIN OPTION`, `GRANT WITH GRANT OPTION` y las implicaciones de seguridad de cada uno
- **Lección 3.3 — Roles Avanzados:** Explorar los roles predefinidos de Oracle (`DBA`, `CONNECT`, `RESOURCE`), roles seguros con contraseña y roles de aplicación
- **Lección 3.4 — Auditoría Avanzada:** Configurar políticas de auditoría basadas en condiciones complejas, gestión del ciclo de vida del trail y alertas automáticas con Oracle Audit Vault
- **Práctica recomendada:** Revisar la documentación oficial de Oracle Database 19c Security Guide, especialmente los capítulos sobre "Configuring Privilege and Role Authorization" y "Monitoring Database Activity with Auditing"

---

## Recursos Adicionales

- **Oracle Database 19c Security Guide** — Documentación oficial completa sobre gestión de usuarios, privilegios, roles, perfiles y auditoría. Disponible en: [https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/)
- **Oracle Database SQL Language Reference 19c** — Referencia completa de la sintaxis de `CREATE USER`, `ALTER USER`, `DROP USER`, `GRANT`, `REVOKE`, `CREATE ROLE`, `CREATE PROFILE` y `CREATE AUDIT POLICY`. Disponible en: [https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- **Oracle Database Reference 19c (Vistas V$ y DBA_)** — Descripción detallada de todas las vistas del diccionario de datos utilizadas en esta práctica (`DBA_USERS`, `DBA_SYS_PRIVS`, `DBA_TAB_PRIVS`, `DBA_ROLES`, `DBA_PROFILES`, `UNIFIED_AUDIT_TRAIL`). Disponible en: [https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)
- **Oracle Live SQL** — Entorno gratuito en línea para practicar comandos SQL de Oracle sin instalación local. Incluye scripts de ejemplo sobre seguridad. Disponible en: [https://livesql.oracle.com](https://livesql.oracle.com)
- **Oracle Database Vault Administrator's Guide 19c** — Para escenarios avanzados de separación de privilegios y control de acceso a datos del DBA. Disponible en: [https://docs.oracle.com/en/database/oracle/oracle-database/19/dvadm/](https://docs.oracle.com/en/database/oracle/oracle-database/19/dvadm/)
