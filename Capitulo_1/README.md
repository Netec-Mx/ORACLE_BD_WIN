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

# 🧪 Lab 01-00-01 (Windows)
## Exploración de la Arquitectura Fundamental de Oracle Database 19c

### 📌 Metadatos

| Propiedad | Valor |
| :--- | :--- |
| **Duración** | 45 minutos |
| **Complejidad** | Principiante |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 1.1 – Conceptos Fundamentales |

---

### 📖 Descripción General
En esta práctica explorarás los componentes arquitectónicos de una instancia **Oracle Database 19c** en ejecución, utilizando:
* Vistas dinámicas (`V$`)
* Comandos del sistema operativo Windows (`CMD` / `PowerShell`)

### 🎯 Objetivos
Al finalizar este laboratorio podrás:
* Consultar `V$INSTANCE` y `V$DATABASE`.
* Analizar la estructura de memoria (`SGA` / `PGA`).
* Identificar los procesos de Oracle dentro del entorno de Windows.
* Examinar los archivos físicos de la base de datos (*datafiles*, *control files*, *redo logs*).
* Consultar parámetros de configuración con el comando `SHOW PARAMETER`.

---

### ⚙️ Prerrequisitos

#### Conocimientos
* SQL básico.
* Comprensión conceptual de Instancia vs. Base de Datos.
* Uso básico de la consola de comandos (`CMD` o `PowerShell`).

#### Acceso
* Usuario con privilegios de Administrador de Base de Datos (`DBA`).
* Acceso a la herramienta interactiva `SQL*Plus`.
* Instancia `ORCL` levantada y en estado `OPEN`.

---

### 🖥️ Configuración Inicial (Windows)

1. Abre el **CMD** o **PowerShell** con privilegios de **Administrador**.

2. 🔹 **Verificar variables de entorno actuales:**
   ```cmd
   echo %ORACLE_HOME%
   echo %ORACLE_SID%
   echo %PATH%
🔹 Definir variables de entorno (en caso de que no existan):

DOS

set ORACLE_BASE=C:\app\oracle
set ORACLE_HOME=C:\app\oracle\product\19.3.0\dbhome_1
set ORACLE_SID=ORCL
set PATH=%ORACLE_HOME%\bin;%PATH%
🔹 Verificar el estado del Listener:

DOS

lsnrctl status
🔹 Verificar la ejecución de los procesos de Oracle:

DOS

tasklist | findstr ORCL
✔ Resultado esperado:

Plaintext

oracle.exe        XXXX Console
⚠️ Si no visualizas ningún proceso de Oracle, significa que la instancia no se encuentra activa.

🚀 Paso 1: Conectarse a Oracle
Conéctate a la base de datos utilizando autenticación del sistema operativo:

DOS

sqlplus / as sysdba
Configuración de Entorno en SQL*Plus
Para mejorar la legibilidad de la salida de datos, ejecuta los siguientes comandos de formato:

SQL

SET LINESIZE 120
SET PAGESIZE 50
SET COLSEP ' | '
Consultas de Verificación
SQL

SELECT INSTANCE_NAME, STATUS, DATABASE_STATUS FROM V$INSTANCE;
SELECT NAME, OPEN_MODE, LOG_MODE FROM V$DATABASE;
✔ Verificación: El valor de STATUS debe reflejar OPEN y el de OPEN_MODE debe ser READ WRITE.

🧠 Paso 2: Memoria SGA y PGA
Ejecuta las siguientes consultas para analizar la distribución y dimensionamiento de la memoria:

SQL

SELECT NAME, VALUE, ROUND(VALUE/1024/1024,2) AS MB 
FROM V$SGA 
ORDER BY VALUE DESC;
SQL

SELECT NAME, BYTES, ROUND(BYTES/1024/1024,2) AS MB, RESIZEABLE 
FROM V$SGAINFO;
Consulta de Parámetros de Memoria
SQL

SHOW PARAMETER sga_target
SHOW PARAMETER pga_aggregate_target
⚙️ Paso 3: Procesos de Fondo (Background Processes)
Identifica los procesos en segundo plano que interactúan con la instancia:

SQL

SELECT NAME, DESCRIPTION FROM V$BGPROCESS WHERE PADDR <> '00';
SQL

SELECT p.SPID, b.NAME
FROM V$BGPROCESS b
JOIN V$PROCESS p ON b.PADDR = p.ADDR;
🔹 Verificación desde el Sistema Operativo
Puedes validar estos procesos sin salir de la terminal.

En CMD:

DOS

tasklist | findstr oracle
En PowerShell:

PowerShell

Get-Process | Where-Object {$_.ProcessName -like "*ora*"}
📌 Procesos clave a identificar: PMON, SMON, DBW0, LGWR y CKPT.

💾 Paso 4: Archivos Físicos
Consulta la ubicación y el estado de las estructuras físicas de almacenamiento:

SQL

SELECT FILE#, NAME FROM V$DATAFILE;
SELECT NAME FROM V$CONTROLFILE;
SELECT GROUP#, STATUS FROM V$LOG;
🔹 Verificar la existencia en Windows
Abre una consola alternativa del sistema operativo y confirma las rutas físicas:

DOS

dir C:\app\oracle\oradata\ORCL\*.dbf
dir C:\app\oracle\oradata\ORCL\*.ctl
dir C:\app\oracle\oradata\ORCL\*.log
⚙️ Paso 5: Parámetros y Logs de Diagnóstico
Consulta de Parámetros Generales
SQL

SHOW PARAMETER db_name
SHOW PARAMETER diagnostic_dest
SQL

SELECT NAME, VALUE FROM V$PARAMETER
WHERE NAME IN ('db_name','processes','sessions');
🔹 Inspección del Alert Log (Windows)
Localiza y visualiza las últimas líneas del archivo de registro de alertas:

DOS

dir C:\app\oracle\diag\rdbms\orcl\ORCL\trace
type C:\app\oracle\diag\rdbms\orcl\ORCL\trace\alert_ORCL.log
✅ Validación Final
Asegúrate de que el entorno conserve la estabilidad e integridad ejecutando:

SQL

SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
SELECT COUNT(*) FROM V$BH WHERE STATUS != 'free';
SELECT COUNT(*) FROM V$DATAFILE WHERE STATUS NOT IN ('SYSTEM','ONLINE');
🛠️ Troubleshooting (Resolución de Problemas en Windows)
❌ Error ORA-01034 (Oracle not available):

DOS

sqlplus / as sysdba
SQL

STARTUP;
❌ Variables de entorno no definidas o incorrectas:

DOS

set ORACLE_HOME=C:\app\oracle\product\19.3.0\dbhome_1
set ORACLE_SID=ORCL
❌ No aparecen los procesos activos del motor:

DOS

tasklist | findstr oracle
🧹 Limpieza del Entorno
Finaliza tus sesiones de forma ordenada:

SQL

EXIT;
Confirma el cierre en la terminal del sistema operativo:

DOS

tasklist | findstr sqlplus
🧠 Resumen
✔ Lo aprendido
Conexión segura mediante autenticación SYSDBA.

Uso estratégico de vistas de rendimiento dinámico (V$).

Análisis granular de estructuras de memoria, procesos internos y almacenamiento.

Interrelación operativa entre la capa de Oracle y el sistema operativo Windows.

📌 Conceptos Clave
Instancia = Estructuras de Memoria (SGA) + Procesos de Fondo (Background Processes).

Base de datos = Archivos físicos reales (Datafiles, Control files, Redo logs).

Vistas V$ = Mecanismos de introspección del diccionario de datos en tiempo real.

Procesos Oracle = Mapeados directamente como subprocesos/hilos dentro del S.O.

⏭️ Siguientes pasos
Análisis detallado de la arquitectura de la SGA.

Comportamiento y gestión de procesos internos.

Estructuras lógicas de almacenamiento (Tablespaces, Segments, Extents).

Estrategias de Backup y Recuperación utilizando RMAN.
## Recursos Adicionales

- **Oracle Database Concepts 19c – Chapter 1 (Introduction to Oracle Database):** Documentación oficial que describe la arquitectura fundamental. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/)
- **Oracle Database Reference 19c – V$ Views:** Referencia completa de todas las vistas dinámicas `V$` con descripción de cada columna. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)
- **Oracle Database 2 Day DBA 19c – Getting Started:** Guía práctica para administradores con ejemplos de consultas a vistas `V$`. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admqs/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admqs/)
- **Oracle Learning Explorer – Oracle Database Foundations:** Curso introductorio gratuito sobre arquitectura Oracle. Disponible en [education.oracle.com](https://education.oracle.com)
- **Oracle Database Administrator's Guide 19c – Managing Processes:** Documentación detallada sobre procesos de fondo y su gestión. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admin/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/)
