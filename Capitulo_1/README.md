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

# 🧪 Lab 01-00-01 (Windows)

## Exploración de la Arquitectura Fundamental de Oracle Database 19c

---

## 📌 Metadatos

| Propiedad          | Valor                                      |
| ------------------ | ------------------------------------------ |
| Duración           | 45 minutos                                 |
| Complejidad        | Principiante                               |
| Nivel Bloom        | Aplicar                                    |
| Módulo             | 1.1 -- Conceptos Fundamentales             |

---

## 📖 Descripción General

En esta práctica explorarás los componentes arquitectónicos de una instancia **Oracle Database 19c** en ejecución, utilizando:

- Vistas dinámicas (V$)
- Comandos del sistema operativo Windows (CMD/PowerShell)

---

## 🎯 Objetivos

Al finalizar podrás:

- Consultar **V$INSTANCE** y **V$DATABASE**
- Analizar memoria (SGA/PGA)
- Identificar procesos Oracle en Windows
- Examinar archivos físicos (datafiles, control files, redo logs)
- Consultar parámetros con SHOW PARAMETER

---

## ⚙️ Prerrequisitos

**Conocimientos**

- SQL básico
- Conceptos de instancia vs base de datos
- Uso básico de CMD o PowerShell

**Acceso**

- Usuario con privilegios DBA
- Acceso a **SQL\*Plus**
- Instancia ORCL en estado OPEN

---

## 🖥️ Configuración Inicial (Windows)

Abrir **CMD o PowerShell como administrador**

**🔹 Verificar variables de entorno**

```cmd
echo %ORACLE_HOME%
echo %ORACLE_SID%
echo %PATH%
🔹 Definir variables (si no existen)

cmd
set ORACLE_BASE=C:\app\oracle\
set ORACLE_HOME=C:\app\oracle\product\19.3.0\dbhome_1\
set ORACLE_SID=ORCL
set PATH=%ORACLE_HOME%\bin;%PATH%
🔹 Verificar listener

cmd
lsnrctl status
🔹 Verificar procesos Oracle

cmd
tasklist | findstr ORCL
✔ Resultado esperado:

cmd
oracle.exe XXXX Console
⚠️ Si no ves procesos Oracle → la instancia no está activa

🚀 Paso 1: Conectarse a Oracle
sql
sqlplus / as sysdba
Configuración

sql
SET LINESIZE 120
SET PAGESIZE 50
SET COLSEP ' | '
Consultas

sql
SELECT INSTANCE_NAME, STATUS, DATABASE_STATUS FROM V$INSTANCE;
SELECT NAME, OPEN_MODE, LOG_MODE FROM V$DATABASE;
✔ Verificación

STATUS → OPEN

OPEN_MODE → READ WRITE

🧠 Paso 2: Memoria SGA
sql
SELECT NAME, VALUE, ROUND(VALUE/1024/1024,2) MB
FROM V$SGA
ORDER BY VALUE DESC;

SELECT NAME, BYTES, ROUND(BYTES/1024/1024,2) MB, RESIZEABLE
FROM V$SGAINFO;
Parámetros

sql
SHOW PARAMETER sga_target
SHOW PARAMETER pga_aggregate_target
⚙️ Paso 3: Procesos de Fondo
sql
SELECT NAME, DESCRIPTION FROM V$BGPROCESS WHERE PADDR <> '00';

SELECT p.SPID, b.NAME
FROM V$BGPROCESS b
JOIN V$PROCESS p ON b.PADDR = p.ADDR;
🔹 En Windows

cmd
tasklist | findstr oracle
O en PowerShell:

powershell
Get-Process | Where-Object {$_.ProcessName -like "*ora*"}
✔ Procesos clave

PMON

SMON

DBW0

LGWR

CKPT

💾 Paso 4: Archivos Físicos
sql
SELECT FILE#, NAME FROM V$DATAFILE;
SELECT NAME FROM V$CONTROLFILE;
SELECT GROUP#, STATUS FROM V$LOG;
🔹 Verificar en Windows

cmd
dir C:\app\oracle\oradata\ORCL\*.dbf
dir C:\app\oracle\oradata\ORCL\*.ctl
dir C:\app\oracle\oradata\ORCL\*.log
⚙️ Paso 5: Parámetros
sql
SHOW PARAMETER db_name
SHOW PARAMETER diagnostic_dest

SELECT NAME, VALUE FROM V$PARAMETER
WHERE NAME IN ('db_name','processes','sessions');
🔹 Logs (Windows)

cmd
dir C:\app\oracle\diag\rdbms\orcl\ORCL\trace
type C:\app\oracle\diag\rdbms\orcl\ORCL\trace\alert_ORCL.log
✅ Validación Final
sql
SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
SELECT COUNT(*) FROM V$BH WHERE STATUS != 'free';
SELECT COUNT(*) FROM V$DATAFILE WHERE STATUS NOT IN ('SYSTEM','ONLINE');
🛠️ Troubleshooting (Windows)
❌ ORA-01034

sql
sqlplus / as sysdba
STARTUP;
❌ Variables no definidas

cmd
set ORACLE_HOME=...
set ORACLE_SID=ORCL
❌ No aparecen procesos

cmd
tasklist | findstr oracle
🧹 Limpieza
sql
EXIT;
cmd
tasklist | findstr sqlplus
🧠 Resumen
✔ Lo aprendido

Conexión como SYSDBA

Uso de vistas V$

Análisis de memoria, procesos y almacenamiento

Relación Oracle ↔ Windows

📌 Conceptos Clave
Instancia = Memoria + Procesos

Base de datos = Archivos físicos

Vistas V$ = introspección en tiempo real

Procesos Oracle = procesos del SO

⏭️ Siguientes pasos
SGA en detalle

Procesos internos

Estructura lógica

Backup con RMAN

Si quieres, en el siguiente paso puedo:

✅ Convertir este lab a PowerShell avanzado (más realista para admins Windows)
✅ O generar versión descargable en Word/PDF profesional
✅ O integrarlo en tu curso completo con slides tipo certificación

## Recursos Adicionales

- **Oracle Database Concepts 19c – Chapter 1 (Introduction to Oracle Database):** Documentación oficial que describe la arquitectura fundamental. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/)
- **Oracle Database Reference 19c – V$ Views:** Referencia completa de todas las vistas dinámicas `V$` con descripción de cada columna. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)
- **Oracle Database 2 Day DBA 19c – Getting Started:** Guía práctica para administradores con ejemplos de consultas a vistas `V$`. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admqs/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admqs/)
- **Oracle Learning Explorer – Oracle Database Foundations:** Curso introductorio gratuito sobre arquitectura Oracle. Disponible en [education.oracle.com](https://education.oracle.com)
- **Oracle Database Administrator's Guide 19c – Managing Processes:** Documentación detallada sobre procesos de fondo y su gestión. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admin/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/)
