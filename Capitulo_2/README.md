# Lab 02-00-01: Arquitectura de Conectividad Oracle Net Services — Configuración y Diagnóstico

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 60 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 2.1 — Arquitectura de Conectividad |

---

## Descripción General

En este laboratorio configurarás de forma completa la capa de conectividad Oracle Net Services, incluyendo el archivo `listener.ora` para registrar servicios de base de datos, el archivo `tnsnames.ora` para resolución de nombres TNS, y el archivo `sqlnet.ora` para parámetros de autenticación y comportamiento del cliente. Simularás un escenario real cliente-servidor donde deberás configurar tanto el lado del servidor (Listener) como el lado del cliente (aliases TNS), y practicarás el diagnóstico de problemas comunes de conectividad utilizando herramientas como `lsnrctl`, `tnsping` y `sqlplus`.

Este laboratorio refleja situaciones cotidianas en entornos empresariales: desde la configuración inicial del Listener en un nuevo servidor Oracle hasta la resolución de incidentes críticos donde los usuarios no pueden conectarse a la base de datos. Dominar estos conceptos es indispensable para cualquier Administrador de Bases de Datos Oracle.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Configurar y gestionar el Oracle Listener mediante el archivo `listener.ora` y la herramienta `lsnrctl`
- [ ] Crear y validar entradas de conexión en el archivo `tnsnames.ora` para resolución de nombres TNS
- [ ] Establecer conexiones a la base de datos usando tres métodos distintos: Easy Connect, TNS alias y conexión directa
- [ ] Diagnosticar y resolver problemas comunes de conectividad Oracle utilizando `lsnrctl status`, `tnsping` y análisis de logs
- [ ] Configurar parámetros de sesión y comportamiento del cliente mediante el archivo `sqlnet.ora`

---

## Prerrequisitos

### Conocimientos Requeridos

- Conceptos básicos de redes TCP/IP: puertos, direcciones IP, protocolos de transporte
- Familiaridad con la instancia Oracle 19c (Lab 01-00-01 completado)
- Manejo básico de la línea de comandos Linux (navegación de directorios, edición de archivos con `vi` o `nano`)
- Conocimiento de los conceptos teóricos de Oracle Net Services cubiertos en la Lección 2.1

### Acceso Requerido

- Usuario del sistema operativo: `oracle` con acceso a la instancia de base de datos
- Variables de entorno configuradas: `ORACLE_HOME`, `ORACLE_SID`, `ORACLE_BASE`, `PATH`
- Permisos de lectura/escritura sobre `$ORACLE_HOME/network/admin/`
- Acceso a SQL*Plus con credenciales `SYSTEM` o `SYS AS SYSDBA`

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| Procesador | Intel Core i5/i7 o AMD Ryzen 5/7 de 64 bits, mínimo 4 núcleos |
| Memoria RAM | Mínimo 8 GB (recomendado 16 GB) |
| Almacenamiento | Mínimo 80 GB libres en disco |
| Red | Adaptador de red funcional con acceso a localhost (127.0.0.1) |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3.0) Enterprise Edition | Motor principal de base de datos |
| Oracle SQL*Plus | 19c (incluido con Oracle DB) | Herramienta de línea de comandos para conexiones |
| Oracle Linux / CentOS | 7.9 / 8.x | Sistema operativo base |
| Editor de texto | `vi`, `nano` o VS Code | Edición de archivos de configuración Oracle Net |
| Terminal SSH | PuTTY 0.79+ (Windows) / Terminal nativo | Acceso a la VM Oracle |

### Configuración Inicial

Antes de comenzar el laboratorio, verifica que el entorno esté correctamente configurado ejecutando los siguientes comandos como usuario `oracle`:

```bash
# Verificar variables de entorno Oracle
echo "ORACLE_HOME: $ORACLE_HOME"
echo "ORACLE_SID:  $ORACLE_SID"
echo "ORACLE_BASE: $ORACLE_BASE"
echo "PATH: $PATH"

# Si las variables no están configuradas, cargar el perfil
source ~/.bash_profile

# Verificar que la instancia Oracle esté activa
sqlplus -s / as sysdba <<EOF
SELECT INSTANCE_NAME, STATUS, DATABASE_STATUS FROM V\$INSTANCE;
EXIT;
EOF

# Verificar el directorio de configuración de red Oracle
ls -la $ORACLE_HOME/network/admin/
```

**Resultado esperado de la verificación:**

```
ORACLE_HOME: /u01/app/oracle/product/19.3.0/dbhome_1
ORACLE_SID:  ORCL
ORACLE_BASE: /u01/app/oracle
PATH: /u01/app/oracle/product/19.3.0/dbhome_1/bin:/usr/local/bin:/bin:/usr/bin

INSTANCE_NAME    STATUS       DATABASE_STATUS
---------------- ------------ -----------------
ORCL             OPEN         ACTIVE
```

---

## Instrucciones Paso a Paso

### Paso 1: Inspección del Estado Actual del Listener y Archivos de Configuración

**Objetivo:** Familiarizarse con el estado actual del Listener y la estructura de los archivos de configuración de Oracle Net Services antes de realizar modificaciones.

**Instrucciones:**

1. Abre una terminal como usuario `oracle` y verifica el estado actual del Listener:

   ```bash
   lsnrctl status
   ```

2. Examina el contenido actual del directorio de configuración de red Oracle:

   ```bash
   ls -la $ORACLE_HOME/network/admin/
   ```

3. Crea copias de respaldo de los archivos de configuración existentes antes de modificarlos:

   ```bash
   # Crear directorio de respaldo
   mkdir -p $ORACLE_HOME/network/admin/backup_lab02

   # Respaldar archivos existentes (si existen)
   cp $ORACLE_HOME/network/admin/listener.ora \
      $ORACLE_HOME/network/admin/backup_lab02/listener.ora.bak 2>/dev/null || \
      echo "listener.ora no existe aún - se creará desde cero"

   cp $ORACLE_HOME/network/admin/tnsnames.ora \
      $ORACLE_HOME/network/admin/backup_lab02/tnsnames.ora.bak 2>/dev/null || \
      echo "tnsnames.ora no existe aún - se creará desde cero"

   cp $ORACLE_HOME/network/admin/sqlnet.ora \
      $ORACLE_HOME/network/admin/backup_lab02/sqlnet.ora.bak 2>/dev/null || \
      echo "sqlnet.ora no existe aún - se creará desde cero"

   echo "Respaldo completado en: $ORACLE_HOME/network/admin/backup_lab02/"
   ```

4. Visualiza el contenido actual de `listener.ora` (si existe):

   ```bash
   cat $ORACLE_HOME/network/admin/listener.ora
   ```

**Resultado Esperado:**

```
LSNRCTL for Linux: Version 19.0.0.0.0 - Production on DD-MON-YYYY HH:MM:SS

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                DD-MON-YYYY HH:MM:SS
Uptime                    X days X hr. X min. X sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/19.3.0/dbhome_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/hostname/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
Services Summary...
Service "ORCL" has 1 instance(s).
  Instance "ORCL", status READY, has 1 handler(s) for this service...
The command completed successfully
```

**Verificación:**

- Confirma que el Listener está activo y escuchando en el puerto 1521
- Verifica que el servicio `ORCL` (o el nombre de tu instancia) aparece como registrado
- Anota la ruta del archivo de log del Listener para consultas futuras

---

### Paso 2: Configuración del Archivo listener.ora

**Objetivo:** Crear y configurar el archivo `listener.ora` con una definición explícita del Listener, incluyendo dirección de escucha y registro estático de la base de datos.

**Instrucciones:**

1. Detén el Listener actual para realizar la configuración:

   ```bash
   lsnrctl stop
   ```

2. Crea el archivo `listener.ora` con la configuración completa. Utiliza el siguiente comando para crear el archivo con el contenido correcto:

   ```bash
   cat > $ORACLE_HOME/network/admin/listener.ora << 'EOF'
   # ============================================================
   # Oracle Listener Configuration File
   # Archivo: listener.ora
   # Ubicación: $ORACLE_HOME/network/admin/
   # Propósito: Configuración del proceso Oracle Listener
   # Lab: 02-00-01 - Arquitectura de Conectividad
   # ============================================================

   # Definición del Listener principal
   LISTENER =
     (DESCRIPTION_LIST =
       (DESCRIPTION =
         (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
         (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
       )
     )

   # Registro estático de la instancia de base de datos
   # (Complementa el auto-registro dinámico del proceso LREG)
   SID_LIST_LISTENER =
     (SID_LIST =
       (SID_DESC =
         (GLOBAL_DBNAME = ORCL)
         (ORACLE_HOME = /u01/app/oracle/product/19.3.0/dbhome_1)
         (SID_NAME = ORCL)
       )
     )

   # Parámetros de administración del Listener
   # Tiempo de espera para conexiones entrantes (segundos)
   INBOUND_CONNECT_TIMEOUT_LISTENER = 60

   # Registro de trazas para diagnóstico (OFF en producción)
   TRACE_LEVEL_LISTENER = OFF
   EOF
   ```

3. Verifica que el archivo fue creado correctamente:

   ```bash
   cat $ORACLE_HOME/network/admin/listener.ora
   ```

4. Inicia el Listener con la nueva configuración:

   ```bash
   lsnrctl start
   ```

5. Verifica el estado del Listener después del inicio:

   ```bash
   lsnrctl status
   ```

**Resultado Esperado:**

```
LSNRCTL for Linux: Version 19.0.0.0.0 - Production on DD-MON-YYYY HH:MM:SS

Copyright (c) 1991, 2019, Oracle.  All rights reserved.

Starting /u01/app/oracle/product/19.3.0/dbhome_1/bin/tnslsnr: please wait...

TNSLSNR for Linux: Version 19.0.0.0.0 - Production
System parameter file is /u01/app/oracle/product/19.3.0/dbhome_1/network/admin/listener.ora
Log messages written to /u01/app/oracle/diag/tnslsnr/hostname/listener/alert/log.xml
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521)))
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Services Summary...
Service "ORCL" has 1 instance(s).
  Instance "ORCL", status READY, has 1 handler(s) for this service...
The command completed successfully
```

**Verificación:**

- El Listener debe aparecer como activo con los dos endpoints (TCP e IPC)
- El servicio `ORCL` debe estar listado en la sección "Services Summary"
- No deben aparecer mensajes de error durante el inicio

---

### Paso 3: Configuración del Archivo tnsnames.ora

**Objetivo:** Crear el archivo `tnsnames.ora` con múltiples aliases TNS que permitan conectarse a la base de datos mediante nombres lógicos, facilitando la portabilidad y el mantenimiento de las aplicaciones.

**Instrucciones:**

1. Crea el archivo `tnsnames.ora` con varios aliases de conexión:

   ```bash
   cat > $ORACLE_HOME/network/admin/tnsnames.ora << 'EOF'
   # ============================================================
   # Oracle TNS Names Configuration File
   # Archivo: tnsnames.ora
   # Ubicación: $ORACLE_HOME/network/admin/
   # Propósito: Resolución de nombres de servicios Oracle (aliases TNS)
   # Lab: 02-00-01 - Arquitectura de Conectividad
   # ============================================================

   # Alias principal para la base de datos ORCL
   # Conexión estándar mediante TCP/IP
   ORCL =
     (DESCRIPTION =
       (ADDRESS_LIST =
         (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
       )
       (CONNECT_DATA =
         (SERVER = DEDICATED)
         (SERVICE_NAME = ORCL)
       )
     )

   # Alias alternativo con nombre descriptivo para aplicaciones
   ORACLE_DB_LAB =
     (DESCRIPTION =
       (ADDRESS_LIST =
         (ADDRESS = (PROTOCOL = TCP)(HOST = 127.0.0.1)(PORT = 1521))
       )
       (CONNECT_DATA =
         (SERVER = DEDICATED)
         (SERVICE_NAME = ORCL)
       )
     )

   # Alias para conexión con servidor compartido (Shared Server)
   # Nota: Requiere que Shared Server esté configurado en la instancia
   ORCL_SHARED =
     (DESCRIPTION =
       (ADDRESS_LIST =
         (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
       )
       (CONNECT_DATA =
         (SERVER = SHARED)
         (SERVICE_NAME = ORCL)
       )
     )

   # Alias con parámetros de timeout para conexiones críticas
   ORCL_TIMEOUT =
     (DESCRIPTION =
       (ADDRESS_LIST =
         (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
       )
       (CONNECT_DATA =
         (SERVER = DEDICATED)
         (SERVICE_NAME = ORCL)
       )
       (CONNECT_TIMEOUT = 30)
       (RETRY_COUNT = 3)
     )
   EOF
   ```

2. Verifica la sintaxis del archivo creado:

   ```bash
   cat $ORACLE_HOME/network/admin/tnsnames.ora
   ```

3. Verifica la conectividad de cada alias usando `tnsping`:

   ```bash
   # Probar alias ORCL
   tnsping ORCL

   # Probar alias alternativo
   tnsping ORACLE_DB_LAB

   # Probar alias con timeout
   tnsping ORCL_TIMEOUT
   ```

4. Verifica que `tnsping` puede resolver los aliases con múltiples intentos:

   ```bash
   # tnsping con 5 intentos para medir latencia
   tnsping ORCL 5
   ```

**Resultado Esperado:**

```
TNS Ping Utility for Linux: Version 19.0.0.0.0 - Production on DD-MON-YYYY HH:MM:SS

Copyright (c) 1997, 2019, Oracle.  All rights reserved.

Used parameter files:
/u01/app/oracle/product/19.3.0/dbhome_1/network/admin/sqlnet.ora

Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))(CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=ORCL)))
OK (10 msec)
```

**Verificación:**

- Cada `tnsping` debe responder con `OK` y un tiempo de respuesta en milisegundos
- Si aparece el mensaje `TNS-12541: TNS:no listener`, verifica que el Listener esté activo
- Anota los tiempos de respuesta de cada alias para comparación

---

### Paso 4: Configuración del Archivo sqlnet.ora

**Objetivo:** Configurar el archivo `sqlnet.ora` para establecer parámetros globales del cliente y servidor Oracle Net, incluyendo el orden de resolución de nombres, timeouts y parámetros de seguridad básicos.

**Instrucciones:**

1. Crea el archivo `sqlnet.ora` con la configuración de parámetros de red:

   ```bash
   cat > $ORACLE_HOME/network/admin/sqlnet.ora << 'EOF'
   # ============================================================
   # Oracle SQL*Net Configuration File
   # Archivo: sqlnet.ora
   # Ubicación: $ORACLE_HOME/network/admin/
   # Propósito: Parámetros globales de Oracle Net Services
   # Lab: 02-00-01 - Arquitectura de Conectividad
   # ============================================================

   # Orden de resolución de nombres de servicio
   # TNSNAMES: busca en tnsnames.ora
   # EZCONNECT: permite sintaxis Easy Connect (host:port/service)
   # HOSTNAME: usa resolución DNS del sistema operativo
   NAMES.DIRECTORY_PATH = (TNSNAMES, EZCONNECT, HOSTNAME)

   # Autenticación del sistema operativo
   # Permite conexión como "/ as sysdba" sin contraseña
   SQLNET.AUTHENTICATION_SERVICES = (NTS)

   # Tiempo máximo de espera para establecer una conexión (segundos)
   SQLNET.INBOUND_CONNECT_TIMEOUT = 60

   # Tiempo de espera para operaciones de red (segundos)
   SQLNET.SEND_TIMEOUT = 60
   SQLNET.RECV_TIMEOUT = 60

   # Registro de trazas del cliente (para diagnóstico)
   # Niveles: OFF, USER, ADMIN, SUPPORT
   TRACE_LEVEL_CLIENT = OFF

   # Directorio para archivos de traza del cliente
   TRACE_DIRECTORY_CLIENT = /u01/app/oracle/diag/clients

   # Registro de eventos de red
   LOG_DIRECTORY_CLIENT = /u01/app/oracle/diag/clients

   # Expiración de conexiones inactivas (minutos)
   # 0 = sin expiración (no recomendado en producción)
   SQLNET.EXPIRE_TIME = 10
   EOF
   ```

2. Verifica el contenido del archivo:

   ```bash
   cat $ORACLE_HOME/network/admin/sqlnet.ora
   ```

3. Crea el directorio para logs del cliente si no existe:

   ```bash
   mkdir -p /u01/app/oracle/diag/clients
   chmod 755 /u01/app/oracle/diag/clients
   ```

4. Recarga el Listener para que tome los nuevos parámetros:

   ```bash
   lsnrctl reload
   ```

5. Verifica el estado del Listener después de la recarga:

   ```bash
   lsnrctl status
   ```

**Resultado Esperado:**

```
LSNRCTL for Linux: Version 19.0.0.0.0 - Production on DD-MON-YYYY HH:MM:SS

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))
The command completed successfully
```

**Verificación:**

- El comando `lsnrctl reload` debe completarse sin errores
- El estado del Listener debe seguir mostrando los servicios registrados
- Verifica que el archivo `sqlnet.ora` es accesible con los permisos correctos: `ls -la $ORACLE_HOME/network/admin/`

---

### Paso 5: Prueba de Conexiones con Diferentes Métodos

**Objetivo:** Establecer conexiones exitosas a la base de datos Oracle utilizando los tres métodos principales: conexión directa con `/ as sysdba`, alias TNS y sintaxis Easy Connect, para validar la configuración realizada.

**Instrucciones:**

1. **Método 1: Conexión directa (autenticación del sistema operativo)**

   ```bash
   sqlplus / as sysdba <<EOF
   SELECT 'CONEXION DIRECTA EXITOSA' AS metodo,
          INSTANCE_NAME,
          STATUS
   FROM V\$INSTANCE;
   EXIT;
   EOF
   ```

2. **Método 2: Conexión usando alias TNS (tnsnames.ora)**

   ```bash
   sqlplus system/Oracle_1234@ORCL <<EOF
   SELECT 'CONEXION TNS EXITOSA' AS metodo,
          SYS_CONTEXT('USERENV','SESSION_USER') AS usuario,
          SYS_CONTEXT('USERENV','DB_NAME') AS base_datos,
          SYS_CONTEXT('USERENV','HOST') AS host_servidor
   FROM DUAL;
   EXIT;
   EOF
   ```

3. **Método 3: Conexión usando Easy Connect (sin tnsnames.ora)**

   ```bash
   sqlplus system/Oracle_1234@localhost:1521/ORCL <<EOF
   SELECT 'CONEXION EASY CONNECT EXITOSA' AS metodo,
          SYS_CONTEXT('USERENV','SESSION_USER') AS usuario,
          SYS_CONTEXT('USERENV','NETWORK_PROTOCOL') AS protocolo
   FROM DUAL;
   EXIT;
   EOF
   ```

4. **Método 4: Conexión usando el alias alternativo ORACLE_DB_LAB**

   ```bash
   sqlplus system/Oracle_1234@ORACLE_DB_LAB <<EOF
   SELECT 'CONEXION ALIAS ALTERNATIVO EXITOSA' AS metodo,
          SYS_CONTEXT('USERENV','DB_NAME') AS base_datos,
          TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS') AS fecha_conexion
   FROM DUAL;
   EXIT;
   EOF
   ```

5. Verifica las sesiones activas en la base de datos para confirmar las conexiones:

   ```bash
   sqlplus / as sysdba <<EOF
   SELECT SID,
          SERIAL#,
          USERNAME,
          MACHINE,
          PROGRAM,
          STATUS,
          TO_CHAR(LOGON_TIME, 'DD/MM/YYYY HH24:MI:SS') AS LOGON_TIME
   FROM V\$SESSION
   WHERE USERNAME IS NOT NULL
   ORDER BY LOGON_TIME DESC;
   EXIT;
   EOF
   ```

**Resultado Esperado:**

```
METODO                          INSTANCE_NAME    STATUS
------------------------------- ---------------- ------
CONEXION DIRECTA EXITOSA        ORCL             OPEN

METODO                    USUARIO  BASE_DATOS  HOST_SERVIDOR
------------------------- -------- ----------- -------------
CONEXION TNS EXITOSA      SYSTEM   ORCL        hostname

METODO                         USUARIO  PROTOCOLO
------------------------------ -------- ---------
CONEXION EASY CONNECT EXITOSA  SYSTEM   tcp

METODO                              BASE_DATOS  FECHA_CONEXION
----------------------------------- ----------- -------------------
CONEXION ALIAS ALTERNATIVO EXITOSA  ORCL        15/01/2024 10:30:45
```

**Verificación:**

- Los cuatro métodos de conexión deben completarse sin errores `ORA-`
- La consulta a `V$SESSION` debe mostrar las sesiones activas de usuario `SYSTEM`
- Anota la diferencia en el campo `PROGRAM` entre los diferentes métodos de conexión

---

### Paso 6: Simulación y Diagnóstico de Problemas de Conectividad

**Objetivo:** Practicar el diagnóstico de los errores de conectividad más comunes en entornos Oracle: Listener caído, alias TNS incorrecto y puerto bloqueado.

**Instrucciones:**

1. **Escenario 1: Diagnóstico con Listener detenido**

   Detén el Listener y observa el comportamiento:

   ```bash
   # Detener el Listener
   lsnrctl stop

   # Intentar conectar (debe fallar con ORA-12541)
   sqlplus system/Oracle_1234@ORCL 2>&1 | head -5

   # Intentar tnsping (debe fallar)
   tnsping ORCL 2>&1

   # Diagnóstico: verificar si el proceso listener está corriendo
   ps -ef | grep tnslsnr | grep -v grep

   # Diagnóstico: verificar si el puerto 1521 está en uso
   netstat -tlnp | grep 1521 2>/dev/null || ss -tlnp | grep 1521

   # Restaurar el Listener
   lsnrctl start
   ```

2. **Escenario 2: Alias TNS con nombre de servicio incorrecto**

   Agrega temporalmente un alias con error y observa el diagnóstico:

   ```bash
   # Agregar alias con servicio incorrecto al final del tnsnames.ora
   cat >> $ORACLE_HOME/network/admin/tnsnames.ora << 'EOF'

   # Alias con error intencional para diagnóstico
   ORCL_ERROR =
     (DESCRIPTION =
       (ADDRESS_LIST =
         (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
       )
       (CONNECT_DATA =
         (SERVER = DEDICATED)
         (SERVICE_NAME = SERVICIO_INEXISTENTE)
       )
     )
   EOF

   # tnsping resuelve el alias pero la conexión fallará
   tnsping ORCL_ERROR

   # Intentar conexión (debe fallar con ORA-12514)
   sqlplus system/Oracle_1234@ORCL_ERROR 2>&1 | head -5

   # Diagnóstico: verificar servicios registrados en el Listener
   lsnrctl services

   # Verificar servicios desde la base de datos
   sqlplus / as sysdba <<EOF2
   SELECT NAME, NETWORK_NAME, CREATION_DATE
   FROM V\$SERVICES
   ORDER BY NAME;
   EXIT;
   EOF2
   ```

3. **Escenario 3: Activar y revisar el log del Listener**

   ```bash
   # Habilitar logging en el Listener para diagnóstico
   lsnrctl set log_status on

   # Realizar una conexión exitosa para generar entrada en el log
   sqlplus system/Oracle_1234@ORCL <<EOF
   SELECT 'Conexion para generar log' FROM DUAL;
   EXIT;
   EOF

   # Revisar el log del Listener (últimas 20 líneas)
   tail -20 /u01/app/oracle/diag/tnslsnr/$(hostname)/listener/alert/log.xml 2>/dev/null || \
   find /u01/app/oracle/diag -name "log.xml" -path "*/listener/*" 2>/dev/null | \
   head -1 | xargs tail -20
   ```

4. **Escenario 4: Verificar el registro dinámico del proceso LREG**

   ```bash
   sqlplus / as sysdba <<EOF
   -- Verificar el proceso LREG activo
   SELECT PNAME, DESCRIPTION
   FROM V\$BGPROCESS
   WHERE PNAME = 'LREG';

   -- Forzar re-registro del servicio con el Listener
   ALTER SYSTEM REGISTER;

   -- Verificar los servicios después del re-registro
   SELECT NAME, NETWORK_NAME
   FROM V\$ACTIVE_SERVICES
   ORDER BY NAME;
   EXIT;
   EOF

   # Verificar que el servicio aparece en el Listener
   lsnrctl status
   ```

**Resultado Esperado:**

```
-- Escenario 1: Listener detenido
ERROR:
ORA-12541: TNS:no listener

TNS-12541: TNS:no listener
 TNS-12560: TNS:protocol adapter error
  TNS-00511: No listener

-- Escenario 2: Servicio incorrecto
OK (10 msec)   <-- tnsping resuelve el alias pero no verifica el servicio

ERROR:
ORA-12514: TNS:listener does not currently know of service requested in connect descriptor

-- Escenario 4: Proceso LREG
PNAME  DESCRIPTION
------ ------------------------------------
LREG   Listener Registration

System altered.
```

**Verificación:**

- Documenta el código de error exacto para cada escenario (`ORA-12541`, `ORA-12514`)
- Confirma que `ALTER SYSTEM REGISTER` fuerza el re-registro correctamente
- Verifica que `lsnrctl services` muestra información detallada de los handlers

---

### Paso 7: Consulta de Información de Conectividad desde la Base de Datos

**Objetivo:** Utilizar vistas del diccionario de datos y vistas de rendimiento dinámico para obtener información sobre las conexiones activas, el estado de los servicios y los parámetros de red de la instancia.

**Instrucciones:**

1. Consulta las sesiones activas y su información de conexión:

   ```bash
   sqlplus / as sysdba <<EOF
   -- Encabezado del reporte
   PROMPT ==========================================
   PROMPT REPORTE DE SESIONES ACTIVAS
   PROMPT ==========================================

   -- Información detallada de sesiones activas
   SELECT s.SID,
          s.SERIAL#,
          s.USERNAME,
          s.STATUS,
          s.MACHINE,
          s.PROGRAM,
          s.MODULE,
          s.SERVER,
          TO_CHAR(s.LOGON_TIME, 'DD/MM/YYYY HH24:MI:SS') AS LOGON_TIME,
          n.SERVICE_NAME
   FROM V\$SESSION s
   JOIN V\$SESSION_CONNECT_INFO n ON s.SID = n.SID
   WHERE s.USERNAME IS NOT NULL
   ORDER BY s.LOGON_TIME DESC;
   EXIT;
   EOF
   ```

2. Consulta los servicios activos y su configuración:

   ```bash
   sqlplus / as sysdba <<EOF
   PROMPT ==========================================
   PROMPT SERVICIOS REGISTRADOS EN LA INSTANCIA
   PROMPT ==========================================

   SELECT NAME,
          NETWORK_NAME,
          GOAL,
          DTP,
          ENABLED
   FROM V\$ACTIVE_SERVICES
   ORDER BY NAME;

   PROMPT ==========================================
   PROMPT PARAMETROS DE RED DE LA INSTANCIA
   PROMPT ==========================================

   SELECT NAME, VALUE, DESCRIPTION
   FROM V\$PARAMETER
   WHERE NAME IN (
     'local_listener',
     'remote_listener',
     'service_names',
     'db_name',
     'db_domain',
     'dispatchers',
     'shared_servers',
     'max_shared_servers'
   )
   ORDER BY NAME;
   EXIT;
   EOF
   ```

3. Verifica el parámetro `LOCAL_LISTENER` y ajústalo si es necesario:

   ```bash
   sqlplus / as sysdba <<EOF
   -- Verificar el parámetro local_listener
   SHOW PARAMETER local_listener;

   -- Si está vacío, configurarlo para apuntar al listener en puerto 1521
   -- (Esto ayuda al proceso LREG a registrarse correctamente)
   ALTER SYSTEM SET LOCAL_LISTENER = '(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521))' SCOPE=BOTH;

   -- Forzar re-registro después del cambio
   ALTER SYSTEM REGISTER;

   -- Verificar el cambio
   SHOW PARAMETER local_listener;
   EXIT;
   EOF

   # Confirmar en el Listener
   lsnrctl status
   ```

**Resultado Esperado:**

```
==========================================
REPORTE DE SESIONES ACTIVAS
==========================================

SID  SERIAL# USERNAME STATUS  MACHINE    PROGRAM          SERVER    LOGON_TIME
---- ------- -------- ------- ---------- ---------------- --------- -------------------
  1      123 SYSTEM   ACTIVE  hostname   sqlplus@hostname DEDICATED 15/01/2024 10:30:45

==========================================
SERVICIOS REGISTRADOS EN LA INSTANCIA
==========================================

NAME    NETWORK_NAME  GOAL  DTP  ENABLED
------- ------------- ----- ---- -------
ORCL    ORCL          NONE  N    YES
SYS$BG  SYS$BG        NONE  N    YES
SYS$UM  SYS$UM        NONE  N    YES

==========================================
PARAMETROS DE RED DE LA INSTANCIA
==========================================

NAME              VALUE                                              DESCRIPTION
----------------- -------------------------------------------------- -----------
db_name           ORCL                                               database name
local_listener    (ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)) listener
service_names     ORCL                                               service names
shared_servers    0                                                  shared servers
```

**Verificación:**

- El parámetro `local_listener` debe apuntar al puerto 1521
- Los servicios `ORCL` debe aparecer en `V$ACTIVE_SERVICES`
- El campo `SERVER` en `V$SESSION` debe mostrar `DEDICATED` para las conexiones actuales

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] El archivo `listener.ora` está correctamente configurado y el Listener inicia sin errores
- [ ] El archivo `tnsnames.ora` contiene al menos 3 aliases TNS válidos y probados con `tnsping`
- [ ] El archivo `sqlnet.ora` está configurado con el orden de resolución de nombres correcto
- [ ] Las conexiones mediante los tres métodos (directa, TNS alias, Easy Connect) se establecen exitosamente
- [ ] Se identificaron y documentaron correctamente los errores `ORA-12541` y `ORA-12514`
- [ ] El comando `ALTER SYSTEM REGISTER` fuerza el re-registro del servicio con el Listener
- [ ] Las vistas `V$SESSION`, `V$ACTIVE_SERVICES` y `V$PARAMETER` retornan información coherente con la configuración

### Procedimiento de Pruebas

1. **Prueba integral del Listener:**

   ```bash
   # Verificación completa del estado del Listener
   lsnrctl status
   lsnrctl services
   ```
   **Resultado Esperado:** El servicio `ORCL` aparece con estado `READY` y al menos 1 handler disponible.

2. **Prueba de todos los aliases TNS:**

   ```bash
   for alias in ORCL ORACLE_DB_LAB ORCL_TIMEOUT; do
     echo -n "Probando alias $alias: "
     result=$(tnsping $alias 2>&1 | grep -E "OK|TNS-")
     echo $result
   done
   ```
   **Resultado Esperado:** Cada alias responde con `OK (XX msec)`.

3. **Prueba de conexión Easy Connect:**

   ```bash
   sqlplus system/Oracle_1234@localhost:1521/ORCL -S <<EOF
   SELECT 'Easy Connect OK' AS resultado FROM DUAL;
   EXIT;
   EOF
   ```
   **Resultado Esperado:** `Easy Connect OK`

4. **Prueba de re-registro automático:**

   ```bash
   sqlplus / as sysdba -S <<EOF
   ALTER SYSTEM REGISTER;
   SELECT NAME FROM V\$ACTIVE_SERVICES WHERE NAME NOT LIKE 'SYS%';
   EXIT;
   EOF
   lsnrctl status | grep "Service\|Instance"
   ```
   **Resultado Esperado:** El servicio `ORCL` aparece tanto en `V$ACTIVE_SERVICES` como en la salida de `lsnrctl status`.

5. **Prueba de parámetros sqlnet.ora:**

   ```bash
   # Verificar que NAMES.DIRECTORY_PATH está siendo usado
   tnsping ORCL | grep "parameter files"
   ```
   **Resultado Esperado:** La salida debe indicar el uso del archivo `sqlnet.ora` en `$ORACLE_HOME/network/admin/`.

---

## Solución de Problemas

### Problema 1: ORA-12541 — TNS: no listener

**Síntomas:**
- `sqlplus` retorna `ORA-12541: TNS:no listener`
- `tnsping` retorna `TNS-12541: TNS:no listener`
- No es posible establecer ninguna conexión remota a la base de datos

**Causa:**
El proceso Oracle Listener no está en ejecución, o está escuchando en un puerto diferente al configurado en `tnsnames.ora`.

**Solución:**

```bash
# Paso 1: Verificar si el proceso listener está corriendo
ps -ef | grep tnslsnr | grep -v grep

# Paso 2: Verificar qué puerto está usando el listener
netstat -tlnp 2>/dev/null | grep 1521 || ss -tlnp | grep 1521

# Paso 3: Iniciar el listener si está detenido
lsnrctl start

# Paso 4: Si el listener no inicia, revisar el log de errores
tail -50 /u01/app/oracle/diag/tnslsnr/$(hostname)/listener/alert/log.xml

# Paso 5: Verificar permisos del archivo listener.ora
ls -la $ORACLE_HOME/network/admin/listener.ora

# Paso 6: Confirmar que el listener está activo
lsnrctl status
```

---

### Problema 2: ORA-12514 — TNS: listener does not currently know of service

**Síntomas:**
- `tnsping` responde con `OK` (el alias TNS es válido)
- `sqlplus` retorna `ORA-12514: TNS:listener does not currently know of service requested in connect descriptor`
- El Listener está activo pero no reconoce el servicio solicitado

**Causa:**
El nombre del servicio en `tnsnames.ora` no coincide con los servicios registrados en el Listener. Esto ocurre cuando el proceso LREG aún no ha registrado la instancia, o cuando el `SERVICE_NAME` en el alias TNS tiene un error tipográfico.

**Solución:**

```bash
# Paso 1: Verificar los servicios actualmente registrados en el Listener
lsnrctl services

# Paso 2: Verificar el nombre exacto del servicio en la base de datos
sqlplus / as sysdba <<EOF
SELECT NAME, NETWORK_NAME FROM V\$ACTIVE_SERVICES;
SHOW PARAMETER service_names;
EXIT;
EOF

# Paso 3: Forzar el re-registro del servicio con el Listener
sqlplus / as sysdba <<EOF
ALTER SYSTEM REGISTER;
EXIT;
EOF

# Paso 4: Esperar 30 segundos y verificar el Listener
sleep 30
lsnrctl status

# Paso 5: Si el servicio sigue sin aparecer, verificar LOCAL_LISTENER
sqlplus / as sysdba <<EOF
SHOW PARAMETER local_listener;
ALTER SYSTEM SET LOCAL_LISTENER='(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521))' SCOPE=BOTH;
ALTER SYSTEM REGISTER;
EXIT;
EOF
```

---

### Problema 3: ORA-12154 — TNS: could not resolve the connect identifier specified

**Síntomas:**
- `sqlplus system/Oracle_1234@ORCL` retorna `ORA-12154: TNS:could not resolve the connect identifier specified`
- `tnsping ORCL` también falla con el mismo error

**Causa:**
Oracle Net no puede encontrar el alias `ORCL` en ninguna de las fuentes de resolución de nombres configuradas. El archivo `tnsnames.ora` no existe, está en una ubicación incorrecta, o el alias tiene un error de sintaxis.

**Solución:**

```bash
# Paso 1: Verificar la variable TNS_ADMIN (si está definida, tiene prioridad)
echo "TNS_ADMIN: $TNS_ADMIN"

# Paso 2: Verificar la ubicación del tnsnames.ora que Oracle está usando
tnsping ORCL | grep "parameter files"

# Paso 3: Verificar que el archivo tnsnames.ora existe y tiene contenido
ls -la $ORACLE_HOME/network/admin/tnsnames.ora
cat $ORACLE_HOME/network/admin/tnsnames.ora | grep -A5 "^ORCL"

# Paso 4: Verificar la sintaxis del archivo tnsnames.ora
# Buscar errores comunes: paréntesis desbalanceados
grep -c "(" $ORACLE_HOME/network/admin/tnsnames.ora
grep -c ")" $ORACLE_HOME/network/admin/tnsnames.ora

# Paso 5: Verificar el orden de resolución en sqlnet.ora
grep "NAMES.DIRECTORY_PATH" $ORACLE_HOME/network/admin/sqlnet.ora

# Paso 6: Probar con Easy Connect para aislar si el problema es TNS
sqlplus system/Oracle_1234@localhost:1521/ORCL
```

---

### Problema 4: El Listener inicia pero la instancia no aparece en los servicios

**Síntomas:**
- `lsnrctl start` completa exitosamente
- `lsnrctl status` muestra el Listener activo pero sin servicios registrados
- `tnsping ORCL` responde con `OK` pero la conexión falla con `ORA-12514`

**Causa:**
El proceso LREG de la instancia Oracle no ha tenido tiempo de registrarse, o la instancia no está activa.

**Solución:**

```bash
# Paso 1: Verificar que la instancia Oracle está activa
sqlplus / as sysdba <<EOF
SELECT INSTANCE_NAME, STATUS FROM V\$INSTANCE;
EXIT;
EOF

# Paso 2: Si la instancia está activa, forzar el re-registro
sqlplus / as sysdba <<EOF
ALTER SYSTEM REGISTER;
EXIT;
EOF

# Paso 3: Esperar al proceso LREG (puede tardar hasta 60 segundos)
sleep 60
lsnrctl status

# Paso 4: Verificar el parámetro LOCAL_LISTENER
sqlplus / as sysdba <<EOF
SHOW PARAMETER local_listener;
SHOW PARAMETER service_names;
EXIT;
EOF

# Paso 5: Si la instancia no está activa, iniciarla
sqlplus / as sysdba <<EOF
STARTUP;
EXIT;
EOF
```

---

## Limpieza

Ejecuta los siguientes comandos para limpiar el entorno después de completar el laboratorio, manteniendo la configuración funcional para labs posteriores:

```bash
# Eliminar el alias con error intencional del tnsnames.ora
# (El alias ORCL_ERROR creado en el Paso 6)
grep -n "ORCL_ERROR" $ORACLE_HOME/network/admin/tnsnames.ora

# Crear una versión limpia del tnsnames.ora sin el alias de error
grep -v "ORCL_ERROR\|SERVICIO_INEXISTENTE\|Alias con error intencional" \
  $ORACLE_HOME/network/admin/tnsnames.ora > /tmp/tnsnames_clean.ora

# Revisar el archivo limpio antes de reemplazar
cat /tmp/tnsnames_clean.ora

# Reemplazar el archivo original con la versión limpia
cp /tmp/tnsnames_clean.ora $ORACLE_HOME/network/admin/tnsnames.ora
rm /tmp/tnsnames_clean.ora

# Verificar que el Listener está activo y los servicios están registrados
lsnrctl status

# Confirmar que las conexiones siguen funcionando
sqlplus system/Oracle_1234@ORCL -S <<EOF
SELECT 'Configuracion final OK' AS estado,
       SYS_CONTEXT('USERENV','DB_NAME') AS base_datos
FROM DUAL;
EXIT;
EOF

# Listar los archivos de configuración finales
echo "=== Archivos de configuración Oracle Net ==="
ls -la $ORACLE_HOME/network/admin/*.ora
echo ""
echo "=== Respaldos del laboratorio ==="
ls -la $ORACLE_HOME/network/admin/backup_lab02/
```

> ⚠️ **Advertencia:** No elimines los archivos `listener.ora`, `tnsnames.ora` y `sqlnet.ora` configurados en este laboratorio. Son necesarios para los labs posteriores (Lab 02-00-02 en adelante). Los respaldos en `backup_lab02/` pueden conservarse como referencia. Si necesitas reiniciar el laboratorio, restaura los respaldos con: `cp $ORACLE_HOME/network/admin/backup_lab02/*.bak $ORACLE_HOME/network/admin/` y renombra los archivos eliminando la extensión `.bak`.

---

## Resumen

### Lo que Lograste

- Configuraste el archivo `listener.ora` con registro estático y dinámico de la instancia Oracle, estableciendo el Listener como punto de entrada para todas las conexiones remotas
- Creaste el archivo `tnsnames.ora` con múltiples aliases TNS que permiten conectarse a la base de datos mediante nombres lógicos en lugar de parámetros de red directos
- Configuraste el archivo `sqlnet.ora` con el orden de resolución de nombres, timeouts y parámetros de seguridad básicos que gobiernan el comportamiento de Oracle Net Services
- Estableciste conexiones exitosas usando los tres métodos principales: autenticación del SO, alias TNS y sintaxis Easy Connect
- Simulaste y diagnosticaste los errores de conectividad más comunes en entornos Oracle: `ORA-12541`, `ORA-12514` y `ORA-12154`

### Conceptos Clave Aprendidos

- **Oracle Net Services** opera como capa de sesión/presentación sobre TCP/IP, proporcionando transparencia de ubicación mediante el uso de Service Names y aliases TNS
- El **Oracle Listener** es el único punto de entrada para conexiones remotas; su caída implica la interrupción total de la conectividad para todos los usuarios remotos
- El proceso **LREG** registra automáticamente los servicios de la instancia con el Listener; el comando `ALTER SYSTEM REGISTER` fuerza este registro de inmediato
- Los tres archivos de configuración (`listener.ora`, `tnsnames.ora`, `sqlnet.ora`) tienen responsabilidades distintas y complementarias dentro de la arquitectura de conectividad
- La diferencia entre `ORA-12541` (Listener caído) y `ORA-12514` (servicio no registrado) permite un diagnóstico más rápido y preciso de los problemas de conectividad

### Próximos Pasos

- Explora la configuración avanzada del Listener para entornos con múltiples instancias o múltiples listeners en diferentes puertos
- Investiga la configuración de Oracle Connection Manager (CMAN) para escenarios de proxy y multiplexación de conexiones en entornos de alta concurrencia
- Practica la configuración del modelo de **Shared Server** modificando los parámetros `DISPATCHERS` y `SHARED_SERVERS` en la instancia para entornos OLTP con muchas conexiones concurrentes

---

## Recursos Adicionales

- **Oracle Net Services Administrator's Guide 19c** — Documentación oficial completa sobre configuración de `listener.ora`, `tnsnames.ora`, `sqlnet.ora` y Oracle Connection Manager. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/
- **Oracle Database Net Services Reference 19c** — Referencia exhaustiva de todos los parámetros disponibles en los archivos de configuración de Oracle Net. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/
- **MOS Note 1074510.1: Troubleshooting Oracle Net Services** — Guía de diagnóstico oficial de Oracle para problemas de conectividad, disponible con cuenta de My Oracle Support (support.oracle.com)
- **Oracle LiveSQL** — Plataforma online para practicar consultas a vistas del diccionario de datos sin instalación local: https://livesql.oracle.com
