## 🧪 Práctica 3. Configuración y Manejo de la Seguridad de la Base de Datos Oracle 19c

Administración Completa de Seguridad en Oracle Database 19c

## ⚙️ Configuración Inicial

1. Abrir CMD o PowerShell como Administrador
🔹 Verificar variables
```ini
set|findstr ORA

Si las variables no están definidas, llama al programa `ambiente.bat` creado en la práctica anterior.
```
2. Verificar instancia
```ini
sqlplus / as sysdba
SELECT INSTANCE_NAME, STATUS, OPEN_MODE FROM V$INSTANCE;
```
3. Crear directorio de trabajo
```sql
mkdir C:\lab03
cd C:\lab03
```

## 🧱 Paso 1: Tablespace y Datos de Prueba
```sql
CREATE TABLESPACE practice_ts
DATAFILE 'C:\app\oracle\oradata\ORCL\ORCLPDB1\practice_ts01.dbf'
SIZE 100M AUTOEXTEND ON NEXT 10M MAXSIZE 500M;

   -- Verificar creación del tablespace
   SELECT tablespace_name, status, block_size, extent_management
   FROM   dba_tablespaces
   WHERE  tablespace_name = 'PRACTICE_TS';

CREATE TABLE system.empleados_sensibles ( emp_id NUMBER PRIMARY KEY,
nombre VARCHAR2(100),
salario NUMBER(10,2),
departamento VARCHAR2(50),
fecha_ingreso DATE DEFAULT SYSDATE )
TABLESPACE practice_ts;

INSERT INTO system.empleados_sensibles VALUES (1,'Ana García',85000,'Finanzas',SYSDATE);
INSERT INTO system.empleados_sensibles VALUES (2,'Carlos López',72000,'TI',SYSDATE);
INSERT INTO system.empleados_sensibles VALUES (3,'María Torres',95000,'Dirección',SYSDATE);
INSERT INTO system.empleados_sensibles VALUES (4,'Roberto Silva',68000,'RRHH',SYSDATE);
INSERT INTO system.empleados_sensibles VALUES (5,'Laura Mendez',78000,'TI',SYSDATE); COMMIT;
```

## 🔐 Paso 2: Perfiles
```sql
CREATE PROFILE perfil_desarrollador
LIMIT FAILED_LOGIN_ATTEMPTS 5
PASSWORD_LIFE_TIME 90 PASSWORD_REUSE_TIME 365
PASSWORD_REUSE_MAX 10 PASSWORD_LOCK_TIME 1/24
PASSWORD_GRACE_TIME 7 SESSIONS_PER_USER 3 CPU_PER_CALL 3000 CONNECT_TIME 480 IDLE_TIME 60; CREATE PROFILE perfil_readonly LIMIT FAILED_LOGIN_ATTEMPTS 3 PASSWORD_LIFE_TIME 60 PASSWORD_REUSE_TIME 180 PASSWORD_REUSE_MAX 5 PASSWORD_LOCK_TIME 1/12 PASSWORD_GRACE_TIME 3 SESSIONS_PER_USER 2 CPU_PER_CALL 1000 CONNECT_TIME 240 IDLE_TIME 30; ALTER SYSTEM SET RESOURCE_LIMIT = TRUE;
```

## 👤 Paso 3: Usuarios
```sql
CREATE USER dev_juan IDENTIFIED BY "DevJuan#2024" DEFAULT TABLESPACE practice_ts QUOTA 200M ON practice_ts PROFILE perfil_desarrollador; CREATE USER rpt_maria IDENTIFIED BY "RptMaria#2024" DEFAULT TABLESPACE practice_ts QUOTA 0 ON practice_ts PROFILE perfil_readonly; CREATE USER app_erp IDENTIFIED BY "AppErp#2024" DEFAULT TABLESPACE practice_ts QUOTA UNLIMITED ON practice_ts PROFILE perfil_desarrollador;
```

## 🔑 Paso 4: Privilegios de Sistema
```sql
GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE, CREATE TRIGGER TO dev_juan; GRANT CREATE SESSION TO rpt_maria; GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE, CREATE TRIGGER, CREATE TYPE, CREATE DATABASE LINK, ALTER SESSION TO app_erp;
```

## 📊 Paso 5: Privilegios de Objeto
```sql
GRANT SELECT, INSERT, UPDATE ON system.empleados_sensibles TO dev_juan; GRANT SELECT ON system.empleados_sensibles TO rpt_maria; GRANT SELECT, INSERT, UPDATE, DELETE ON system.empleados_sensibles TO app_erp; REVOKE UPDATE ON system.empleados_sensibles FROM dev_juan;
```

## 🧩 Paso 6: Roles
```sql
CREATE ROLE rol_readonly; CREATE ROLE rol_developer; CREATE ROLE rol_app_admin IDENTIFIED BY "RolAdmin#2024"; GRANT CREATE SESSION TO rol_readonly; GRANT SELECT ON system.empleados_sensibles TO rol_readonly; GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE TO rol_developer; GRANT SELECT, INSERT, UPDATE ON system.empleados_sensibles TO rol_developer; GRANT rol_developer TO rol_app_admin; GRANT CREATE TRIGGER, CREATE TYPE, ALTER SESSION TO rol_app_admin; GRANT DELETE ON system.empleados_sensibles TO rol_app_admin; GRANT rol_readonly TO rpt_maria; GRANT rol_developer TO dev_juan; GRANT rol_app_admin TO app_erp;
```

## 🔄 Paso 7: Gestión de Usuarios
```sql
ALTER USER app_erp ACCOUNT LOCK; ALTER USER app_erp ACCOUNT UNLOCK PASSWORD EXPIRE; ALTER USER dev_juan QUOTA 500M ON practice_ts; ALTER USER dev_juan IDENTIFIED BY "DevJuan#NewPass2024";
```

## 🛡️ Paso 8: Auditoría
```sql
CREATE AUDIT POLICY pol_audit_empleados ACTIONS SELECT, INSERT, UPDATE, DELETE ON system.empleados_sensibles; CREATE AUDIT POLICY pol_audit_ddl ACTIONS CREATE TABLE, DROP TABLE, ALTER TABLE; CREATE AUDIT POLICY pol_audit_logon_fail ACTIONS LOGON; AUDIT POLICY pol_audit_empleados; AUDIT POLICY pol_audit_ddl; AUDIT POLICY pol_audit_logon_fail WHENEVER NOT SUCCESSFUL;
```

## 🔍 Consultas de Auditoría
```sql
SELECT dbusername, action_name, object_name, event_timestamp FROM unified_audit_trail ORDER BY event_timestamp DESC;
```

## 📈 Paso 9: Reportes
```sql
SELECT username, account_status, profile FROM dba_users WHERE username IN ('DEV_JUAN','RPT_MARIA','APP_ERP'); SELECT grantee, privilege FROM dba_sys_privs; SELECT * FROM dba_role_privs;
```

## 🧹 Limpieza
```sql
DROP USER dev_juan CASCADE; DROP USER rpt_maria CASCADE; DROP USER app_erp CASCADE; DROP ROLE rol_readonly; DROP ROLE rol_developer; DROP ROLE rol_app_admin; DROP PROFILE perfil_desarrollador CASCADE; DROP PROFILE perfil_readonly CASCADE; DROP TABLE system.empleados_sensibles PURGE; DROP TABLESPACE practice_ts INCLUDING CONTENTS AND DATAFILES;
```

## ✅ Resultado
Con este laboratorio:
Implementas seguridad completa en Oracle 19c
Aplicas mínimo privilegio real
Usas roles como en producción
Configuras auditoría corporativa
Validas todo desde diccionario










