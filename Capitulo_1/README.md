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

# 🧪 Lab 01-00-01 (Windows): Exploración de la Arquitectura Fundamental de Oracle Database 19c

| Propiedad | Valor |
| :--- | :--- |
| **Duración** | 45 minutos |
| **Complejidad** | Principiante |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 1.1 – Conceptos Fundamentales |

En esta práctica explorarás los componentes arquitectónicos de una instancia Oracle Database 19c en ejecución, utilizando vistas dinámicas (V$) y comandos del sistema operativo Windows (CMD/PowerShell). Al finalizar este laboratorio podrás consultar V$INSTANCE y V$DATABASE, analizar la distribución de memoria (SGA/PGA), identificar procesos de Oracle dentro del entorno de Windows, examinar los archivos físicos de la base de datos (datafiles, control files, redo logs), y consultar y modificar parámetros del sistema con el comando SHOW PARAMETER.

Para realizar esta práctica es necesario contar con conocimientos de SQL básico, conceptos teóricos de la diferencia entre Instancia y Base de Datos, uso básico de la consola de comandos de Windows (CMD o PowerShell), poseer un usuario con privilegios de Administrador de Base de Datos (DBA), acceso local o remoto a SQL*Plus, y tener una instancia de Oracle denominada ORCL en estado OPEN.

Para comenzar la configuración inicial en Windows, abra la consola de comandos (CMD o PowerShell) con privilegios de Administrador y verifique las variables de entorno actuales ejecutando:
echo %ORACLE_HOME%
echo %ORACLE_SID%
echo %PATH%

En caso de que no existan, defina las variables en la consola con los siguientes comandos:
set ORACLE_BASE=C:\app\oracle
set ORACLE_HOME=C:\app\oracle\product\19.3.0\dbhome_1
set ORACLE_SID=ORCL
set PATH=%ORACLE_HOME%\bin;%PATH%

A continuación, verifique el estado del Listener con el comando "lsnrctl status" y compruebe los procesos de Oracle activos en el Sistema Operativo con "tasklist | findstr ORCL". El resultado esperado debe mostrar "oracle.exe XXXX Console". Si no visualiza el proceso oracle.exe, significa que la instancia no está activa y debe iniciarla antes de continuar.

Para el Paso 1: Conectarse a Oracle, inicie sesión en la base de datos utilizando autenticación por sistema operativo ejecutando "sqlplus / as sysdba". Para mejorar la legibilidad de las consultas en SQL*Plus, configure los siguientes parámetros de línea:
SET LINESIZE 120
SET PAGESIZE 50
SET COLSEP ' | '

Realice las consultas de verificación inicial ejecutando:
SELECT INSTANCE_NAME, STATUS, DATABASE_STATUS FROM V$INSTANCE;
SELECT NAME, OPEN_MODE, LOG_MODE FROM V$DATABASE;
Como verificación obligatoria, el STATUS de la instancia debe reportar OPEN y el OPEN_MODE de la base de datos debe reportar READ WRITE.

Para el Paso 2: Memoria SGA (System Global Area), ejecute las siguientes consultas para auditar la asignación y componentes de la estructura de memoria global:
SELECT NAME, VALUE, ROUND(VALUE/1024/1024,2) MB FROM V$SGA ORDER BY VALUE DESC;
SELECT NAME, BYTES, ROUND(BYTES/1024/1024,2) MB, RESIZEABLE FROM V$SGAINFO;

Compruebe los límites configurados para la gestión de memoria automática o manual mediante los comandos de verificación de parámetros de memoria:
SHOW PARAMETER sga_target
SHOW PARAMETER pga_aggregate_target

Para el Paso 3: Procesos de Fondo (Background Processes), identifique los procesos internos que mantienen la operatividad de la instancia con las siguientes sentencias SQL:
SELECT NAME, DESCRIPTION FROM V$BGPROCESS WHERE PADDR <> '00';
SELECT p.SPID, b.NAME FROM V$BGPROCESS b JOIN V$PROCESS p ON b.PADDR = p.ADDR;

Para realizar la verificación desde el Sistema Operativo (Windows), abra una ventana de comandos en paralelo para mapear estos procesos. En CMD ejecute "tasklist | findstr oracle", o en PowerShell ejecute "Get-Process | Where-Object {$_.ProcessName -like '*ora*'}". Los procesos clave que debe identificar son PMON (Process Monitor), SMON (System Monitor), DBW0 (Database Writer), LGWR (Log Writer) y CKPT (Checkpoint).

Para el Paso 4: Archivos Físicos, consulte la ubicación exacta en disco de las estructuras físicas que componen la Base de Datos con las siguientes consultas:
SELECT FILE#, NAME FROM V$DATAFILE;
SELECT NAME FROM V$CONTROLFILE;
SELECT GROUP#, STATUS FROM V$LOG;

Para verificar la existencia física en Windows, salga de SQL*Plus temporalmente o abra un CMD para validar que los archivos se encuentran en las rutas indicadas ejecutando:
dir C:\app\oracle\oradata\ORCL\*.dbf
dir C:\app\oracle\oradata\ORCL\*.ctl
dir C:\app\oracle\oradata\ORCL\*.log

Para el Paso 5: Parámetros del Sistema y Logs, realice la consulta de parámetros básicos ejecutando:
SHOW PARAMETER db_name
SHOW PARAMETER diagnostic_dest
SELECT NAME, VALUE FROM V$PARAMETER WHERE NAME IN ('db_name','processes','sessions');

Para la inspección de Logs de Alerta en Windows, verifique las rutas del directorio de diagnóstico y visualice las últimas líneas del archivo de log principal ejecutando en la consola de comandos de Windows:
dir C:\app\oracle\diag\rdbms\orcl\ORCL\trace
type C:\app\oracle\diag\rdbms\orcl\ORCL\trace\alert_ORCL.log

Para la Validación Final, asegúrese de que el estado del laboratorio concluye de manera íntegra y sin inconsistencias ejecutando las siguientes consultas SQL:
SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
SELECT COUNT(*) FROM V$BH WHERE STATUS != 'free';
SELECT COUNT(*) FROM V$DATAFILE WHERE STATUS NOT IN ('SYSTEM','ONLINE');

En caso de requerir Troubleshooting (Resolución de Problemas en Windows): Si se presenta el error "❌ ORA-01034: ORACLE not available", indica que la instancia no ha sido iniciada, por lo que debe ejecutar "sqlplus / as sysdba" seguido del comando "STARTUP;". Si las variables de entorno no están definidas y comandos como sqlplus no son reconocidos, asigne manualmente las variables en la consola actual con "set ORACLE_HOME=C:\app\oracle\product\19.3.0\dbhome_1", "set ORACLE_SID=ORCL" y "set PATH=%ORACLE_HOME%\bin;%PATH%". Si no aparecen procesos activos de Oracle, valide los servicios de Windows asociados ejecutando "tasklist | findstr oracle"; si no arroja resultados, acceda a services.msc e inicie manualmente el servicio OracleServiceORCL.

Para la Limpieza del Entorno, finalice de forma ordenada las herramientas utilizadas durante la sesión ejecutando el comando "EXIT;" en SQL*Plus, y en la consola de Windows valide el cierre de sesiones ejecutando "tasklist | findstr sqlplus".

En resumen, lo aprendido en esta práctica incluye la conexión segura mediante roles administrativos (SYSDBA), el uso de vistas de rendimiento dinámico V$ para auditoría interna, el análisis de la distribución de memoria, procesos de fondo y almacenamiento físico, junto con la comprensión de la interacción nativa entre Oracle Database y el ecosistema Windows. Los conceptos clave a retener son que la Instancia equivale a las Estructuras de Memoria (SGA) más los Procesos de Fondo, la Base de datos es la colección de Archivos físicos en disco (Datafiles, Control files, Redo logs), las Vistas V$ representan tablas virtuales de introspección en tiempo real, y los Procesos Oracle operan como hilos de ejecución dentro del proceso unificado de sistema operativo en Windows. Como siguientes pasos sugeridos se encuentra la profundización en las sub-estructuras del SGA, los ciclos de vida e interacciones de los Procesos Internos, el mapeo de la Estructura Lógica (Tablespaces, Extents, Segments) y la configuración de respaldos integrales con RMAN.


## Recursos Adicionales

- **Oracle Database Concepts 19c – Chapter 1 (Introduction to Oracle Database):** Documentación oficial que describe la arquitectura fundamental. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/)
- **Oracle Database Reference 19c – V$ Views:** Referencia completa de todas las vistas dinámicas `V$` con descripción de cada columna. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)
- **Oracle Database 2 Day DBA 19c – Getting Started:** Guía práctica para administradores con ejemplos de consultas a vistas `V$`. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admqs/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admqs/)
- **Oracle Learning Explorer – Oracle Database Foundations:** Curso introductorio gratuito sobre arquitectura Oracle. Disponible en [education.oracle.com](https://education.oracle.com)
- **Oracle Database Administrator's Guide 19c – Managing Processes:** Documentación detallada sobre procesos de fondo y su gestión. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/admin/](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/)
