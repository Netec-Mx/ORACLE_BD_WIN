
🧪 Práctica 1. Revisión de la Configuración y Arquitectura de la Base de Datos Oracle 19c 

Exploración de la Arquitectura Fundamental de Oracle Database 19c

## 📌 Metadatos

## 📖 Descripción General
En esta práctica explorarás los componentes arquitectónicos de una instancia Oracle Database 19c en ejecución, utilizando:
Vistas dinámicas (V$)
Comandos del sistema operativo Windows (CMD/PowerShell)

## 🎯 Objetivos
Al finalizar podrás:
Consultar V$INSTANCE y V$DATABASE
Analizar memoria (SGA/PGA)
Identificar procesos Oracle en Windows
Examinar archivos físicos (datafiles, control files, redo logs)
Consultar parámetros con SHOW PARAMETER

## ⚙️ Prerrequisitos
Conocimientos
SQL básico
Conceptos de instancia vs base de datos
Uso básico de CMD o PowerShell
Acceso
Usuario con privilegios DBA
Acceso a SQL*Plus
Instancia ORCL en estado OPEN

## 🖥️ Configuración Inicial (Windows)
Abrir CMD o PowerShell como administrador
🔹 Verificar variables de entorno
```sql
echo %ORACLE_HOME%
echo %ORACLE_SID%
echo %PATH%
```
🔹 Definir variables (si no existen), estas variables no son obligatorias en Windows pero a veces son utiles para ciertos casos.
```sql
set ORACLE_BASE=C:\app\oracle
set ORACLE_HOME=C:\app\oracle\product\19.3.0\dbhome_1
set ORACLE_SID=ORCL
set PATH=%ORACLE_HOME%\bin;%PATH%
```
🔹 Verificar listener
```sql
lsnrctl status
```
🔹 Verificar procesos Oracle
```sql
Abre el `Task Manager` y asegurate que ORACLE RDBS y TNSLSNR estén en ejecución.
```
✔ Resultado esperado:
⚠️ Si no ves procesos Oracle → la instancia no está activa

## 🚀 Paso 1: Conectarse a Oracle
```sql
sqlplus / as sysdba
```
Configuración del ancho de linea, número de lineas por página y separador.
```sql
SET LINESIZE 120
SET PAGESIZE 50
SET COLSEP ' | '
```
Consultas básicas de la INSTANCIA y de la BASE DE DATOS
```sql
SELECT INSTANCE_NAME, STATUS, DATABASE_STATUS FROM V$INSTANCE;
SELECT NAME, OPEN_MODE, LOG_MODE FROM V$DATABASE;
```
✔ Verificación
STATUS → OPEN
OPEN_MODE → READ WRITE

## 🧠 Paso 2: Memoria SGA
```sql
SELECT NAME, VALUE, ROUND(VALUE/1024/1024,2) MB FROM V$SGA ORDER BY VALUE DESC;
SELECT NAME, BYTES, ROUND(BYTES/1024/1024,2) MB, RESIZEABLE FROM V$SGAINFO;
```
Parámetros
```sql
SHOW PARAMETER sga_target SHOW PARAMETER pga_aggregate_target
```

## ⚙️ Paso 3: Procesos de Fondo
```sql
SELECT NAME, DESCRIPTION FROM V$BGPROCESS WHERE PADDR <> '00';
SELECT p.SPID, b.NAME FROM V$BGPROCESS b JOIN V$PROCESS p ON b.PADDR = p.ADDR;
```
🔹 En Windows
```sql
tasklist | findstr oracle
```
O en PowerShell:
```sql
Get-Process | Where-Object {$_.ProcessName -like "*ora*"}
```
✔ Procesos clave
PMON
SMON
DBW0
LGWR
CKPT

## 💾 Paso 4: Archivos Físicos
```sql
SELECT FILE#, NAME FROM V$DATAFILE;
SELECT NAME FROM V$CONTROLFILE;
SELECT GROUP#, STATUS FROM V$LOG;
```

🔹 Verificar en Windows
```sql
dir C:\app\oracle\oradata\ORCL\*.dbf dir C:\app\oracle\oradata\ORCL\*.ctl dir C:\app\oracle\oradata\ORCL\*.log
```

## ⚙️ Paso 5: Parámetros
```sql
SHOW PARAMETER db_name SHOW PARAMETER diagnostic_dest
SELECT NAME, VALUE FROM V$PARAMETER WHERE NAME IN ('db_name','processes','sessions');
```

🔹 Logs (Windows)
```sql
dir C:\app\oracle\diag\rdbms\orcl\ORCL\trace
type C:\app\oracle\diag\rdbms\orcl\ORCL\trace\alert_ORCL.log
```

## ✅ Validación Final
```sql
SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
SELECT COUNT(*) FROM V$BH WHERE STATUS != 'free';
SELECT COUNT(*) FROM V$DATAFILE WHERE STATUS NOT IN ('SYSTEM','ONLINE');
```

## 🛠️ Troubleshooting (Windows)
❌ ORA-01034
```sql
sqlplus / as sysdba
STARTUP;
```

❌ Variables no definidas
```sql
set ORACLE_HOME=... set ORACLE_SID=ORCL
```

❌ No aparecen procesos
```sql
tasklist | findstr oracle
```

## 🧹 Limpieza
```sql
EXIT;
tasklist | findstr sqlplus
```

## 🧠 Resumen
✔ Lo aprendido
Conexión como SYSDBA
Uso de vistas V$
Análisis de memoria, procesos y almacenamiento
Relación Oracle ↔ Windows

## 📌 Conceptos Clave
Instancia = Memoria + Procesos
Base de datos = Archivos físicos
Vistas V$ = introspección en tiempo real
Procesos Oracle = procesos del SO


## Recursos Adicionales

- **Oracle Database Concepts 19c – Chapter 1 (Introduction to Oracle Database):** Documentación oficial que describe la arquitectura fundamental. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/)
- **Oracle Database Reference 19c – V$ Views:** Referencia completa de todas las vistas dinámicas `V$` con descripción de cada columna. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)
- **Oracle Database 2 Day DBA 19c – Getting Started:** Guía práctica para administradores con ejemplos de consultas a vistas `V$`. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admqs/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admqs/)
- **Oracle Learning Explorer – Oracle Database Foundations:** Curso introductorio gratuito sobre arquitectura Oracle. Disponible en [education.oracle.com](https://education.oracle.com)
- **Oracle Database Administrator's Guide 19c – Managing Processes:** Documentación detallada sobre procesos de fondo y su gestión. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admin/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/)
