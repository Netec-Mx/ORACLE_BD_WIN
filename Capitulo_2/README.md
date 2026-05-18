
## 🧪 Práctica 2. Manejo de los Servicios de Red de la Base de Datos Oracle

Arquitectura de Conectividad Oracle Net Services — Configuración y Diagnóstico

## 📌 Metadatos

## 📖 Descripción General
En este laboratorio configurarás la capa de conectividad de Oracle Net Services en Oracle Database 19c, incluyendo:
listener.ora (servidor)
tnsnames.ora (cliente)
sqlnet.ora (parámetros globales)

## 🎯 Objetivos
Al finalizar podrás:
Configurar y administrar el Listener
Crear aliases TNS
Conectarte usando 3 métodos
Diagnosticar errores ORA comunes
Ajustar sqlnet.ora

## ⚙️ Prerrequisitos
Práctica 01 completada
SQL básico
CMD / PowerShell
Variables Oracle configuradas

## 🖥️ Configuración Inicial (Windows)
Abrir CMD o PowerShell como administrador
🔹 Verificar variables
```ini
set|findstr ORA

Si las variables no están definidas, llama al programa `ambiente.bat` creado en la práctica anterior.
```
🔹 Verificar instancia
```ini
sqlplus / as sysdba
SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
```
🔹 Verificar que existen los archivos de red en la siguieten carpeta:
```ini
dir %ORACLE_HOME%\network\admin
```

## 🚀 Paso 1: Estado del Listener
```ini
lsnrctl status
```
**Verificación:**

- Confirma que el Listener está activo y escuchando en el puerto 1521
- Verifica que el servicio `ORCL` (o el nombre de tu instancia) aparece como registrado
- Anota la ruta del archivo de log del Listener para consultas futuras
---

Ver y configurar los archivos de red de Oracle.
```ini
dir %ORACLE_HOME%\network\admin
```
Crea una carpeta `backup_lab02` y haz un respaldo de los archivos de configuración de red originales.
```ini
mkdir %ORACLE_HOME%\network\admin\backup_lab02
copy %ORACLE_HOME%\network\admin\listener.ora %ORACLE_HOME%\network\admin\backup_lab02\listener.ora.bak
copy %ORACLE_HOME%\network\admin\tnsnames.ora %ORACLE_HOME%\network\admin\backup_lab02\tnsnames.ora.bak
copy %ORACLE_HOME%\network\admin\sqlnet.ora %ORACLE_HOME%\network\admin\backup_lab02\sqlnet.ora.bak
```

## ⚙️ Paso 2: Configurar listener.ora
Detener Listener
```ini
lsnrctl stop

- Borra los archivos originales que respaldaste:
del *.ora
```
Crear archivos
Ruta:
```ini
notepad %ORACLE_HOME%\network\admin\listener.ora
```
Contenido:
```ini
# Registro estático de la instancia de base de datos
# (Complementa el auto-registro dinámico del proceso LREG)
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME = CLRExtProc)
      (ORACLE_HOME = C:\app\oracle\product\19.3.0\dbhome_1)
      (PROGRAM = extproc)
      (ENVS = "EXTPROC_DLLS=ONLY:C:\app\oracle\product\19.3.0\dbhome_1\bin\oraclr19.dll")
    )
  )

# Definición del Listener principal
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = Localhost)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
```
Iniciar el LISTENER y checar su STATUS.
```ini
lsnrctl start
lsnrctl status
```

**Verificación:**

- El Listener debe aparecer como activo con los dos endpoints (TCP e IPC)
- El servicio `ORCL` debe estar listado en la sección "Services Summary"
- No deben aparecer mensajes de error durante el inicio
---

## 🔗 Paso 3: Configurar tnsnames.ora
*********************
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
*********************
Ruta:
```ini
notepad %ORACLE_HOME%\network\admin\tnsnames.ora
```
Contenido:
```ini
# Alias principal para la base de datos ORCL
# Conexión estándar mediante TCP/IP
ORCL = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521)) (CONNECT_DATA = (SERVICE_NAME = ORCL) ) )
# Alias alternativo con nombre descriptivo para aplicaciones
ORACLE_DB_LAB = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 127.0.0.1)(PORT = 1521)) (CONNECT_DATA = (SERVICE_NAME = ORCL) ) )
# Alias con parámetros de timeout para conexiones críticas
ORCL_TIMEOUT = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521)) (CONNECT_DATA = (SERVICE_NAME = ORCL) ) (CONNECT_TIMEOUT = 30) )
```

🔹 Probar conectividad
```ini
# Probar alias ORCL
tnsping ORCL

# Probar alias alternativo
tnsping ORACLE_DB_LAB

# Probar alias con timeout
tnsping ORCL_TIMEOUT
```
✔ Resultado esperado:

**Verificación:**

- Cada `tnsping` debe responder con `OK` y un tiempo de respuesta en milisegundos
- Si aparece el mensaje `TNS-12541: TNS:no listener`, verifica que el Listener esté activo
- Anota los tiempos de respuesta de cada alias para comparación
---

## ⚙️ Paso 4: Configurar sqlnet.ora
Ruta:
```ini
notepad %ORACLE_HOME%\network\admin\sqlnet.ora
```
Contenido:
```ini
# Autenticación del sistema operativo
NAMES.DIRECTORY_PATH = (TNSNAMES, EZCONNECT)
# Permite conexión como "/ as sysdba" sin contraseña
SQLNET.AUTHENTICATION_SERVICES = (NTS)
# Tiempo de espera para operaciones de red (segundos)
SQLNET.INBOUND_CONNECT_TIMEOUT = 60
# Tiempo de espera para operaciones de red (segundos)
SQLNET.SEND_TIMEOUT = 60
SQLNET.RECV_TIMEOUT = 60
# Registro de trazas del cliente (para diagnóstico)
# Niveles: OFF, USER, ADMIN, SUPPORT
TRACE_LEVEL_CLIENT = OFF
# Expiración de conexiones inactivas (minutos)
# 0 = sin expiración (no recomendado en producción)
SQLNET.EXPIRE_TIME = 10
```
Recargar el LISTENER
```ini
lsnrctl reload
```

**Verificación:**

- El comando `lsnrctl reload` debe completarse sin errores
- El estado del Listener debe seguir mostrando los servicios registrados
---

## 🔌 Paso 5: Pruebas de Conexión
**Instrucciones:**

1. **Método 1: Conexión directa (autenticación del sistema operativo)**
2. **Método 2: Conexión usando alias TNS (tnsnames.ora)**
3. **Método 3: Conexión usando Easy Connect (sin tnsnames.ora)**
4. **Método 4: Conexión usando el alias alternativo ORACLE_DB_LAB**

🔹 Método 1:  Conexión directa (autenticación del sistema operativo)
```ini
sqlplus / as sysdba
```
🔹 Método 2: Conexión usando alias TNS (tnsnames.ora)
```ini
sqlplus sqlplus system/oracle_4U@orcl
show user
```
🔹 Método 3: Conexión usando Easy Connect (sin tnsnames.ora)
```ini
sqlplus system/oracle_4U@localhost:1521/ORCL
show user
```
🔹 Método 4: Conexión usando el alias alternativo ORACLE_DB_LAB
```ini
sqlplus sys/oracle_4U@oracle_db_lab as sysdba
show user
```
🔹 Ver sesiones
```ini
SELECT USERNAME, MACHINE, PROGRAM FROM V$SESSION WHERE USERNAME IS NOT NULL;
```

## 🧪 Paso 6: Diagnóstico

❌ Escenario 1: Listener caído
```ini
lsnrctl stop
sqlplus system/oracle_4U@ORCL
```
✔ Error esperado:
```ini
ORA-12541: TNS:no listener
```
Diagnóstico Windows
```ini
- Checa que Oracle y el Listener esten activos desde el Task Manager.
- Checa que el puerto 1521 este escuchando peticiones de conexión.
netstat -ano | findstr 1521
```

❌ Escenario 2: Servicio incorrecto
Agregar en tnsnames.ora:
ORCL_ERROR = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521)) (CONNECT_DATA = (SERVICE_NAME = ERROR) ) )
```ini
tnsping ORCL_ERROR sqlplus system/Oracle_1234@ORCL_ERROR
```
✔ Error esperado:
```ini
ORA-12514
```

## 📄 Logs del Listener
Ruta típica Windows:
```ini
dir %ORACLE_BASE%\diag\tnslsnr
```
Ver log:
```ini
type log.xml
```

## 📊 Paso 7: Información desde Oracle
```ini
SELECT USERNAME, PROGRAM, MACHINE FROM V$SESSION;
SELECT NAME FROM V$ACTIVE_SERVICES;
SHOW PARAMETER local_listener;
```

## ✅ Validación Final
```ini
lsnrctl status
lsnrctl services
tnsping ORCL
sqlplus system/oracle_4U@localhost:1521/ORCL
```

## 🛠️ Troubleshooting
```ini
ORA-12541
lsnrctl start
```

```ini
ORA-12514
ALTER SYSTEM REGISTER;
```

```ini
ORA-12154
```
Verificar:
```ini
tnsping ORCL
echo %TNS_ADMIN%
```

## 🧹 Limpieza
Eliminar alias erróneo:
Editar tnsnames.ora manualmente

## 🧠 Resumen
✔ Logros
Configuración completa de Oracle Net
Uso de Listener
Conexiones por múltiples métodos
Diagnóstico de errores reales

## 📌 Conceptos Clave
Listener = punto de entrada
TNS = abstracción de conexión
Easy Connect = conexión directa
LREG = registro automático

## ⏭️ Siguientes pasos
Shared Server
Connection Manager
Alta concurrencia

## Recursos Adicionales

- **Oracle Net Services Administrator's Guide 19c** — Documentación oficial completa sobre configuración de `listener.ora`, `tnsnames.ora`, `sqlnet.ora` y Oracle Connection Manager. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/
- **Oracle Database Net Services Reference 19c** — Referencia exhaustiva de todos los parámetros disponibles en los archivos de configuración de Oracle Net. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/
- **MOS Note 1074510.1: Troubleshooting Oracle Net Services** — Guía de diagnóstico oficial de Oracle para problemas de conectividad, disponible con cuenta de My Oracle Support (support.oracle.com)
- **Oracle LiveSQL** — Plataforma online para practicar consultas a vistas del diccionario de datos sin instalación local: https://livesql.oracle.com
