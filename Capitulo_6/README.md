# Lab 06-00-01: Metodología de Monitoreo Proactivo y Afinación de Rendimiento en Oracle Database 19c

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 105 minutos |
| **Complejidad** | Avanzado |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 6 – Monitoreo, Diagnóstico y Afinación de Rendimiento |
| **Versión Oracle** | 19c (19.3+) |

---

## Descripción General

Este laboratorio guía al estudiante a través del ciclo completo de monitoreo proactivo y afinación de rendimiento en Oracle Database 19c. Partiendo de la metodología Observar → Analizar → Actuar → Documentar, el estudiante consultará vistas dinámicas de rendimiento (`V$`), generará y analizará reportes AWR y ADDM, optimizará sentencias SQL problemáticas con SQL Tuning Advisor y planes de ejecución, y configurará la gestión automática de memoria (AMM/ASMM).

El laboratorio simula un escenario real de degradación de rendimiento que el estudiante debe diagnosticar y resolver de forma sistemática, aplicando las mismas herramientas y técnicas que utiliza un DBA profesional en entornos de producción. Al finalizar, el estudiante habrá construido una rutina de monitoreo reproducible y documentada.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Consultar e interpretar vistas dinámicas de rendimiento (`V$SESSION`, `V$SQL`, `V$WAIT_CLASS`, `V$SYSSTAT`, `V$SGASTAT`) para establecer una línea base de rendimiento
- [ ] Generar snapshots AWR manualmente y producir reportes AWR en formato texto para identificar cuellos de botella
- [ ] Ejecutar ADDM y analizar sus recomendaciones automáticas para priorizar acciones de optimización
- [ ] Capturar sentencias SQL con alto consumo de recursos desde `V$SQL` y generar planes de ejecución con `EXPLAIN PLAN` y `DBMS_XPLAN`
- [ ] Utilizar SQL Tuning Advisor (`DBMS_SQLTUNE`) para obtener recomendaciones de optimización de consultas específicas
- [ ] Crear índices B-Tree y compuestos estratégicos para mejorar el rendimiento de consultas identificadas
- [ ] Configurar `MEMORY_TARGET` para AMM y ajustar `SGA_TARGET` / `PGA_AGGREGATE_TARGET` según las necesidades de la carga de trabajo
- [ ] Aplicar el ciclo de monitoreo completo (Observar → Analizar → Actuar → Documentar) en un escenario integrador de diagnóstico

---

## Prerrequisitos

### Conocimiento Requerido

- Conocimiento sólido de SQL incluyendo joins, subconsultas y funciones de agregación
- Comprensión de la arquitectura de memoria Oracle (SGA: buffer cache, shared pool, redo log buffer; PGA)
- Familiaridad con índices B-Tree y su impacto en planes de ejecución
- Haber completado el laboratorio 05-00-01 o tener un esquema de prueba con datos suficientes para generar carga
- Comprensión de la metodología de monitoreo proactivo (Lección 6.1): ciclo Observar → Analizar → Actuar → Documentar

### Acceso Requerido

- Acceso a Oracle Database 19c con privilegios `DBA` (usuario `SYS` o equivalente)
- Licencia Oracle Diagnostics Pack habilitada (requerida para AWR/ADDM en Enterprise Edition)
- Acceso SSH a la máquina virtual Oracle Linux 8.x
- Acceso a SQL*Plus y/o SQL Developer
- Usuario de práctica `PRACTICA_USER` creado con privilegios suficientes (ver sección de configuración inicial)

---

## Entorno de Laboratorio

### Hardware Requirements

| Componente | Especificación |
|------------|----------------|
| CPU | Intel Core i7/i9 o AMD Ryzen 7/9 con virtualización habilitada |
| RAM | Mínimo 16 GB (recomendado 32 GB para Oracle 19c + VM) |
| Almacenamiento | Mínimo 100 GB libres en SSD |
| Red | Adaptador de red funcional con soporte loopback |

### Software Requirements

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) | Motor principal de base de datos |
| Oracle Linux | 8.x (8.7 u 8.8) | Sistema operativo huésped |
| SQL*Plus | Incluido con Oracle 19c | Ejecución de comandos SQL y PL/SQL |
| Oracle SQL Developer | 23.1 o superior | IDE gráfico para visualización de planes de ejecución |
| Oracle Enterprise Manager DB Express | Incluido con Oracle 19c | Monitoreo visual de KPIs y memoria |
| PuTTY / Terminal SSH | 0.79+ / Nativo | Acceso remoto a la VM Oracle Linux |

### Configuración Inicial

Antes de comenzar el laboratorio, ejecuta los siguientes comandos para preparar el entorno. Conéctate a la VM Oracle Linux mediante SSH y verifica que la instancia esté operativa:

```bash
# Conectarse a la VM Oracle Linux via SSH
ssh oracle@192.168.56.101

# Verificar que la instancia Oracle esté activa
export ORACLE_SID=ORCL
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export PATH=$ORACLE_HOME/bin:$PATH

sqlplus / as sysdba
```

```sql
-- Verificar estado de la instancia
SELECT instance_name, status, database_status FROM v$instance;

-- Verificar que el Diagnostics Pack esté habilitado
SELECT name, value FROM v$parameter WHERE name = 'control_management_pack_access';

-- Si el resultado NO es 'DIAGNOSTIC+TUNING', ejecutar:
-- ALTER SYSTEM SET control_management_pack_access = 'DIAGNOSTIC+TUNING' SCOPE=BOTH;

-- Verificar modo ARCHIVELOG (necesario para AWR completo)
SELECT log_mode FROM v$database;

EXIT;
```

```bash
# Crear el usuario de práctica si no existe
sqlplus / as sysdba << 'EOF'
-- Crear usuario de práctica para el laboratorio
CREATE USER practica_user IDENTIFIED BY Oracle123
  DEFAULT TABLESPACE users
  TEMPORARY TABLESPACE temp;

-- Otorgar privilegios necesarios
GRANT CREATE SESSION TO practica_user;
GRANT CREATE TABLE TO practica_user;
GRANT CREATE INDEX TO practica_user;
GRANT CREATE SEQUENCE TO practica_user;
GRANT CREATE PROCEDURE TO practica_user;
GRANT UNLIMITED TABLESPACE TO practica_user;
GRANT SELECT ANY DICTIONARY TO practica_user;
GRANT SELECT_CATALOG_ROLE TO practica_user;
GRANT ADVISOR TO practica_user;
GRANT EXECUTE ON DBMS_SQLTUNE TO practica_user;

-- Crear directorio de trabajo para el laboratorio
CREATE OR REPLACE DIRECTORY lab_dir AS '/home/oracle/lab06';

COMMIT;
EXIT;
EOF

# Crear directorio de trabajo en el sistema operativo
mkdir -p /home/oracle/lab06
chmod 755 /home/oracle/lab06
echo "Entorno de laboratorio listo."
```

---

## Instrucciones Paso a Paso

### Paso 1: Establecer la Línea Base de Rendimiento con Vistas V$

**Objetivo:** Consultar las vistas dinámicas de rendimiento más importantes para obtener una fotografía del estado actual de la base de datos y establecer la línea base (baseline) de KPIs fundamentales.

**Instrucciones:**

1. Conéctate a SQL*Plus como DBA y ejecuta el script de observación inicial:

   ```bash
   sqlplus / as sysdba
   ```

2. Consulta las sesiones activas y sus eventos de espera actuales:

   ```sql
   -- ============================================================
   -- KPI 1: Sesiones activas y sus eventos de espera
   -- Etapa del ciclo: OBSERVAR
   -- ============================================================
   SET LINESIZE 200
   SET PAGESIZE 50
   COLUMN username FORMAT A20
   COLUMN event FORMAT A40
   COLUMN status FORMAT A10
   COLUMN wait_class FORMAT A20

   SELECT
       s.sid,
       s.serial#,
       s.username,
       s.status,
       s.event,
       s.wait_class,
       s.seconds_in_wait,
       s.sql_id
   FROM v$session s
   WHERE s.type = 'USER'
     AND s.status IN ('ACTIVE', 'WAITING')
   ORDER BY s.seconds_in_wait DESC;
   ```

3. Consulta las clases de espera del sistema para identificar dónde se está gastando el tiempo:

   ```sql
   -- ============================================================
   -- KPI 2: Distribución de tiempo de espera por clase
   -- Permite identificar si el cuello de botella es CPU, I/O, red, etc.
   -- ============================================================
   COLUMN wait_class FORMAT A25
   COLUMN total_waits FORMAT 999,999,999
   COLUMN total_timeouts FORMAT 999,999,999
   COLUMN time_waited_secs FORMAT 999,999.99

   SELECT
       wait_class,
       total_waits,
       total_timeouts,
       ROUND(time_waited / 100, 2) AS time_waited_secs,
       ROUND(time_waited / NULLIF(total_waits, 0) / 100, 4) AS avg_wait_secs
   FROM v$system_event
   WHERE wait_class != 'Idle'
   ORDER BY time_waited DESC
   FETCH FIRST 15 ROWS ONLY;
   ```

4. Verifica el hit ratio del buffer cache (KPI de memoria más importante):

   ```sql
   -- ============================================================
   -- KPI 3: Buffer Cache Hit Ratio
   -- Umbral saludable: > 95%
   -- Si está por debajo, puede indicar SGA insuficiente
   -- ============================================================
   SELECT
       name,
       physical_reads,
       db_block_gets + consistent_gets AS logical_reads,
       ROUND(
           (1 - (physical_reads / NULLIF(db_block_gets + consistent_gets, 0))) * 100,
           2
       ) AS hit_ratio_pct,
       CASE
           WHEN (1 - (physical_reads / NULLIF(db_block_gets + consistent_gets, 0))) * 100 >= 95
           THEN 'SALUDABLE'
           WHEN (1 - (physical_reads / NULLIF(db_block_gets + consistent_gets, 0))) * 100 >= 85
           THEN 'ATENCION'
           ELSE 'CRITICO'
       END AS estado
   FROM v$buffer_pool_statistics
   WHERE name = 'DEFAULT';
   ```

5. Consulta el uso de memoria de la SGA por componente:

   ```sql
   -- ============================================================
   -- KPI 4: Distribución de memoria SGA por componente
   -- ============================================================
   COLUMN pool FORMAT A25
   COLUMN name FORMAT A35
   COLUMN mb_usados FORMAT 999,999.99

   SELECT
       pool,
       name,
       ROUND(bytes / 1024 / 1024, 2) AS mb_usados
   FROM v$sgastat
   WHERE pool IS NOT NULL
   ORDER BY bytes DESC
   FETCH FIRST 20 ROWS ONLY;
   ```

6. Revisa las estadísticas del sistema más relevantes:

   ```sql
   -- ============================================================
   -- KPI 5: Estadísticas clave del sistema (V$SYSSTAT)
   -- ============================================================
   COLUMN stat_name FORMAT A45
   COLUMN value FORMAT 999,999,999,999

   SELECT
       name AS stat_name,
       value
   FROM v$sysstat
   WHERE name IN (
       'physical reads',
       'physical writes',
       'db block gets',
       'consistent gets',
       'redo writes',
       'redo size',
       'parse count (total)',
       'parse count (hard)',
       'execute count',
       'user commits',
       'user rollbacks',
       'sorts (memory)',
       'sorts (disk)'
   )
   ORDER BY name;
   ```

7. Guarda los resultados como línea base en un archivo:

   ```bash
   # Salir de SQL*Plus temporalmente
   EXIT;
   ```

   ```bash
   # Generar script de línea base y guardarlo en archivo
   sqlplus / as sysdba << 'EOF' > /home/oracle/lab06/baseline_inicial.txt
   SET LINESIZE 200
   SET PAGESIZE 100
   SET FEEDBACK ON
   SPOOL /home/oracle/lab06/baseline_kpis.txt

   PROMPT ============================================================
   PROMPT LINEA BASE DE RENDIMIENTO - $(date)
   PROMPT ============================================================

   PROMPT --- Buffer Cache Hit Ratio ---
   SELECT name,
          ROUND((1 - (physical_reads / NULLIF(db_block_gets + consistent_gets, 0))) * 100, 2) AS hit_ratio_pct
   FROM v$buffer_pool_statistics WHERE name = 'DEFAULT';

   PROMPT --- Top 10 Eventos de Espera (No Idle) ---
   SELECT wait_class, ROUND(time_waited/100,2) AS secs_waited
   FROM v$system_event
   WHERE wait_class != 'Idle'
   ORDER BY time_waited DESC
   FETCH FIRST 10 ROWS ONLY;

   PROMPT --- Uso de Memoria SGA ---
   SELECT pool, name, ROUND(bytes/1024/1024,2) AS mb
   FROM v$sgastat WHERE pool IS NOT NULL
   ORDER BY bytes DESC FETCH FIRST 10 ROWS ONLY;

   SPOOL OFF
   EXIT;
   EOF

   echo "Línea base guardada en /home/oracle/lab06/baseline_kpis.txt"
   cat /home/oracle/lab06/baseline_kpis.txt
   ```

**Salida Esperada:**

```
Buffer Cache Hit Ratio:
NAME    HIT_RATIO_PCT  ESTADO
------- -------------- ----------
DEFAULT         97.43  SALUDABLE

Top 10 Eventos de Espera (No Idle):
WAIT_CLASS               SECS_WAITED
------------------------ -----------
User I/O                     1245.32
System I/O                    892.10
Concurrency                   234.56
...

Línea base guardada en /home/oracle/lab06/baseline_kpis.txt
```

**Verificación:**

- El archivo `/home/oracle/lab06/baseline_kpis.txt` debe existir y contener datos
- El hit ratio del buffer cache debe ser consultable sin errores
- Todas las vistas `V$` deben responder sin errores `ORA-`

---

### Paso 2: Crear Datos de Prueba y Simular Carga de Trabajo Degradada

**Objetivo:** Crear un esquema de prueba con datos suficientes y simular una carga de trabajo con consultas ineficientes que generen eventos de espera y consumo de recursos, para tener un escenario real de diagnóstico.

**Instrucciones:**

1. Crea el esquema de prueba con tablas e índices:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET ECHO ON
   SET FEEDBACK ON

   -- ============================================================
   -- Crear tabla principal de ventas (sin índices óptimos)
   -- ============================================================
   DROP TABLE ventas_lab PURGE;
   DROP TABLE clientes_lab PURGE;
   DROP TABLE productos_lab PURGE;

   CREATE TABLE clientes_lab (
       cliente_id    NUMBER(10)    PRIMARY KEY,
       nombre        VARCHAR2(100) NOT NULL,
       email         VARCHAR2(150),
       ciudad        VARCHAR2(50),
       segmento      VARCHAR2(20),
       fecha_registro DATE DEFAULT SYSDATE
   );

   CREATE TABLE productos_lab (
       producto_id   NUMBER(10)    PRIMARY KEY,
       nombre        VARCHAR2(100) NOT NULL,
       categoria     VARCHAR2(50),
       precio        NUMBER(10,2),
       stock         NUMBER(10)
   );

   CREATE TABLE ventas_lab (
       venta_id      NUMBER(10)    PRIMARY KEY,
       cliente_id    NUMBER(10),
       producto_id   NUMBER(10),
       fecha_venta   DATE,
       cantidad      NUMBER(5),
       monto_total   NUMBER(12,2),
       estado        VARCHAR2(20),
       region        VARCHAR2(30)
   );

   -- Secuencias
   CREATE SEQUENCE seq_cliente START WITH 1 INCREMENT BY 1;
   CREATE SEQUENCE seq_producto START WITH 1 INCREMENT BY 1;
   CREATE SEQUENCE seq_venta START WITH 1 INCREMENT BY 1;

   COMMIT;
   EXIT;
   EOF
   ```

2. Pobla las tablas con datos de prueba usando PL/SQL:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET ECHO OFF
   SET FEEDBACK OFF
   SET SERVEROUTPUT ON SIZE UNLIMITED

   DECLARE
       v_ciudades   SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST(
           'Ciudad de Mexico','Guadalajara','Monterrey','Puebla',
           'Tijuana','Leon','Juarez','Torreon','Queretaro','Merida'
       );
       v_segmentos  SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST(
           'Premium','Estandar','Basico','VIP','Corporativo'
       );
       v_categorias SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST(
           'Electronica','Ropa','Hogar','Deportes','Alimentos',
           'Tecnologia','Libros','Juguetes','Automotriz','Salud'
       );
       v_estados    SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST(
           'Completada','Pendiente','Cancelada','En proceso'
       );
       v_regiones   SYS.ODCIVARCHAR2LIST := SYS.ODCIVARCHAR2LIST(
           'Norte','Sur','Centro','Este','Oeste'
       );
   BEGIN
       -- Insertar 5,000 clientes
       DBMS_OUTPUT.PUT_LINE('Insertando clientes...');
       FOR i IN 1..5000 LOOP
           INSERT INTO clientes_lab VALUES (
               seq_cliente.NEXTVAL,
               'Cliente_' || LPAD(i, 5, '0'),
               'cliente' || i || '@email.com',
               v_ciudades(MOD(i, 10) + 1),
               v_segmentos(MOD(i, 5) + 1),
               SYSDATE - DBMS_RANDOM.VALUE(0, 1095)
           );
       END LOOP;
       COMMIT;
       DBMS_OUTPUT.PUT_LINE('5,000 clientes insertados.');

       -- Insertar 1,000 productos
       DBMS_OUTPUT.PUT_LINE('Insertando productos...');
       FOR i IN 1..1000 LOOP
           INSERT INTO productos_lab VALUES (
               seq_producto.NEXTVAL,
               'Producto_' || LPAD(i, 4, '0'),
               v_categorias(MOD(i, 10) + 1),
               ROUND(DBMS_RANDOM.VALUE(10, 5000), 2),
               TRUNC(DBMS_RANDOM.VALUE(0, 500))
           );
       END LOOP;
       COMMIT;
       DBMS_OUTPUT.PUT_LINE('1,000 productos insertados.');

       -- Insertar 100,000 ventas (volumen suficiente para generar estadísticas)
       DBMS_OUTPUT.PUT_LINE('Insertando ventas (puede tomar 1-2 minutos)...');
       FOR i IN 1..100000 LOOP
           INSERT INTO ventas_lab VALUES (
               seq_venta.NEXTVAL,
               TRUNC(DBMS_RANDOM.VALUE(1, 5001)),
               TRUNC(DBMS_RANDOM.VALUE(1, 1001)),
               SYSDATE - DBMS_RANDOM.VALUE(0, 365),
               TRUNC(DBMS_RANDOM.VALUE(1, 20)),
               ROUND(DBMS_RANDOM.VALUE(50, 10000), 2),
               v_estados(MOD(i, 4) + 1),
               v_regiones(MOD(i, 5) + 1)
           );
           IF MOD(i, 10000) = 0 THEN
               COMMIT;
               DBMS_OUTPUT.PUT_LINE('  ' || i || ' ventas insertadas...');
           END IF;
       END LOOP;
       COMMIT;
       DBMS_OUTPUT.PUT_LINE('100,000 ventas insertadas. Datos de prueba listos.');
   END;
   /

   -- Recopilar estadísticas de las tablas
   EXEC DBMS_STATS.GATHER_TABLE_STATS('PRACTICA_USER', 'VENTAS_LAB', CASCADE => TRUE);
   EXEC DBMS_STATS.GATHER_TABLE_STATS('PRACTICA_USER', 'CLIENTES_LAB', CASCADE => TRUE);
   EXEC DBMS_STATS.GATHER_TABLE_STATS('PRACTICA_USER', 'PRODUCTOS_LAB', CASCADE => TRUE);

   EXIT;
   EOF
   ```

3. Simula consultas ineficientes para generar carga y eventos de espera:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET ECHO OFF
   SET FEEDBACK OFF
   SET SERVEROUTPUT ON

   DECLARE
       v_count NUMBER;
       v_monto NUMBER;
   BEGIN
       DBMS_OUTPUT.PUT_LINE('Simulando carga de trabajo ineficiente...');

       -- Consulta 1: Full Table Scan en tabla grande (sin índice en fecha_venta)
       FOR i IN 1..5 LOOP
           SELECT COUNT(*), SUM(monto_total)
           INTO v_count, v_monto
           FROM ventas_lab
           WHERE TO_CHAR(fecha_venta, 'YYYY') = TO_CHAR(SYSDATE, 'YYYY')
             AND estado = 'Completada';
           DBMS_OUTPUT.PUT_LINE('Iteracion ' || i || ': ' || v_count || ' ventas, monto=' || ROUND(v_monto,2));
       END LOOP;

       -- Consulta 2: Join sin índice en columna de join
       FOR i IN 1..3 LOOP
           SELECT COUNT(*)
           INTO v_count
           FROM ventas_lab v
           JOIN clientes_lab c ON v.cliente_id = c.cliente_id
           JOIN productos_lab p ON v.producto_id = p.producto_id
           WHERE c.segmento = 'Premium'
             AND p.categoria = 'Electronica'
             AND v.monto_total > 1000;
           DBMS_OUTPUT.PUT_LINE('Join iteration ' || i || ': ' || v_count || ' registros');
       END LOOP;

       -- Consulta 3: Función en columna indexable (previene uso de índice)
       FOR i IN 1..3 LOOP
           SELECT COUNT(*)
           INTO v_count
           FROM clientes_lab
           WHERE UPPER(ciudad) = 'GUADALAJARA';
           DBMS_OUTPUT.PUT_LINE('UPPER scan ' || i || ': ' || v_count || ' clientes');
       END LOOP;

       DBMS_OUTPUT.PUT_LINE('Carga simulada completada.');
   END;
   /

   EXIT;
   EOF
   ```

**Salida Esperada:**

```
Insertando clientes...
5,000 clientes insertados.
Insertando productos...
1,000 productos insertados.
Insertando ventas (puede tomar 1-2 minutos)...
  10000 ventas insertadas...
  20000 ventas insertadas...
  ...
  100000 ventas insertadas. Datos de prueba listos.

Simulando carga de trabajo ineficiente...
Iteracion 1: 25432 ventas, monto=127654321.50
...
Carga simulada completada.
```

**Verificación:**

- Las tres tablas deben existir con datos: `SELECT COUNT(*) FROM ventas_lab;` debe retornar 100,000
- Las estadísticas deben estar actualizadas: `SELECT last_analyzed FROM dba_tables WHERE owner='PRACTICA_USER';`

---

### Paso 3: Generar Snapshots AWR y Reportes de Diagnóstico

**Objetivo:** Generar snapshots AWR manualmente, crear un reporte AWR en formato texto y ejecutar ADDM para analizar automáticamente el período de carga simulada.

**Instrucciones:**

1. Toma el snapshot AWR inicial (antes de la carga de trabajo):

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON
   SET LINESIZE 200

   -- Verificar la configuración actual de AWR
   SELECT snap_interval, retention FROM dba_hist_wr_control;

   -- Verificar los últimos snapshots disponibles
   SELECT snap_id, begin_interval_time, end_interval_time
   FROM dba_hist_snapshot
   ORDER BY snap_id DESC
   FETCH FIRST 5 ROWS ONLY;

   EXIT;
   EOF
   ```

2. Genera el snapshot AWR de inicio del período de análisis:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON

   DECLARE
       v_snap_id NUMBER;
   BEGIN
       -- Tomar snapshot AWR manualmente
       v_snap_id := DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
       DBMS_OUTPUT.PUT_LINE('Snapshot AWR creado con ID: ' || v_snap_id);
   END;
   /

   -- Confirmar que el snapshot fue creado
   SELECT snap_id, TO_CHAR(end_interval_time, 'DD-MON-YYYY HH24:MI:SS') AS snap_time
   FROM dba_hist_snapshot
   ORDER BY snap_id DESC
   FETCH FIRST 3 ROWS ONLY;

   EXIT;
   EOF
   ```

3. Ejecuta nuevamente la carga de trabajo para generar diferencia entre snapshots:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET ECHO OFF
   SET FEEDBACK OFF
   SET SERVEROUTPUT ON

   DECLARE
       v_count NUMBER;
       v_monto NUMBER;
       v_nombre VARCHAR2(100);
   BEGIN
       DBMS_OUTPUT.PUT_LINE('Ejecutando carga de trabajo para AWR...');

       -- Carga intensiva 1: Consultas de agregación sin índices
       FOR i IN 1..10 LOOP
           SELECT COUNT(*), SUM(monto_total), AVG(monto_total)
           INTO v_count, v_monto, v_monto
           FROM ventas_lab v
           JOIN clientes_lab c ON v.cliente_id = c.cliente_id
           WHERE v.fecha_venta BETWEEN SYSDATE - 365 AND SYSDATE
             AND v.estado != 'Cancelada'
             AND c.segmento IN ('Premium', 'VIP');
       END LOOP;
       DBMS_OUTPUT.PUT_LINE('Carga 1 completada: ' || v_count || ' registros procesados');

       -- Carga intensiva 2: Ordenamiento sin índice
       FOR i IN 1..5 LOOP
           SELECT COUNT(*)
           INTO v_count
           FROM (
               SELECT v.cliente_id, SUM(v.monto_total) AS total_compras
               FROM ventas_lab v
               GROUP BY v.cliente_id
               ORDER BY total_compras DESC
           ) subq
           WHERE ROWNUM <= 100;
       END LOOP;
       DBMS_OUTPUT.PUT_LINE('Carga 2 completada');

       -- Carga intensiva 3: Búsquedas de texto sin índice de función
       FOR i IN 1..8 LOOP
           SELECT COUNT(*)
           INTO v_count
           FROM clientes_lab
           WHERE LOWER(nombre) LIKE '%cliente_001%'
              OR LOWER(email) LIKE '%@email.com';
       END LOOP;
       DBMS_OUTPUT.PUT_LINE('Carga 3 completada: ' || v_count || ' clientes encontrados');

       DBMS_OUTPUT.PUT_LINE('Carga para AWR completada exitosamente.');
   END;
   /

   EXIT;
   EOF
   ```

4. Toma el snapshot AWR final y genera el reporte:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON
   SET LINESIZE 200

   DECLARE
       v_snap_id_fin NUMBER;
       v_snap_id_ini NUMBER;
       v_dbid        NUMBER;
       v_inst_num    NUMBER;
   BEGIN
       -- Tomar snapshot final
       v_snap_id_fin := DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
       DBMS_OUTPUT.PUT_LINE('Snapshot final AWR creado con ID: ' || v_snap_id_fin);

       -- Obtener el snapshot anterior (inicio)
       SELECT snap_id INTO v_snap_id_ini
       FROM (
           SELECT snap_id FROM dba_hist_snapshot
           ORDER BY snap_id DESC
       )
       WHERE ROWNUM = 2;

       DBMS_OUTPUT.PUT_LINE('Rango de análisis: Snap ' || v_snap_id_ini || ' a ' || v_snap_id_fin);

       -- Obtener DBID e instancia
       SELECT dbid, instance_number
       INTO v_dbid, v_inst_num
       FROM v$database, v$instance;

       DBMS_OUTPUT.PUT_LINE('DBID: ' || v_dbid || ' | Instancia: ' || v_inst_num);
       DBMS_OUTPUT.PUT_LINE('Para generar reporte AWR ejecute:');
       DBMS_OUTPUT.PUT_LINE('@$ORACLE_HOME/rdbms/admin/awrrpt.sql');
   END;
   /

   -- Mostrar los últimos 5 snapshots para confirmar
   SELECT snap_id,
          TO_CHAR(begin_interval_time, 'HH24:MI:SS') AS inicio,
          TO_CHAR(end_interval_time, 'HH24:MI:SS') AS fin
   FROM dba_hist_snapshot
   ORDER BY snap_id DESC
   FETCH FIRST 5 ROWS ONLY;

   EXIT;
   EOF
   ```

5. Genera el reporte AWR en formato texto automáticamente:

   ```bash
   # Obtener snap_ids para el reporte
   SNAP_IDS=$(sqlplus -S / as sysdba << 'EOF'
   SET PAGESIZE 0 FEEDBACK OFF HEADING OFF
   SELECT snap_id FROM (SELECT snap_id FROM dba_hist_snapshot ORDER BY snap_id DESC) WHERE ROWNUM <= 2;
   EXIT;
   EOF
   )

   SNAP_FIN=$(echo $SNAP_IDS | awk '{print $1}')
   SNAP_INI=$(echo $SNAP_IDS | awk '{print $2}')

   echo "Generando reporte AWR: Snap $SNAP_INI -> $SNAP_FIN"

   sqlplus / as sysdba << EOF
   SET LINESIZE 200
   SET PAGESIZE 50
   SPOOL /home/oracle/lab06/reporte_awr.txt

   -- Generar reporte AWR usando DBMS_WORKLOAD_REPOSITORY
   SELECT output FROM TABLE(
       DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_TEXT(
           (SELECT dbid FROM v\$database),
           (SELECT instance_number FROM v\$instance),
           ${SNAP_INI},
           ${SNAP_FIN}
       )
   );

   SPOOL OFF
   EXIT;
   EOF

   echo "Reporte AWR generado en /home/oracle/lab06/reporte_awr.txt"
   echo "Primeras 100 líneas del reporte:"
   head -100 /home/oracle/lab06/reporte_awr.txt
   ```

6. Ejecuta ADDM para análisis automático y obtén recomendaciones:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON SIZE UNLIMITED
   SET LINESIZE 200
   SET PAGESIZE 100
   COLUMN findings FORMAT A80
   COLUMN type FORMAT A20

   DECLARE
       v_task_name  VARCHAR2(30) := 'ADDM_LAB06_ANALISIS';
       v_snap_ini   NUMBER;
       v_snap_fin   NUMBER;
       v_dbid       NUMBER;
       v_inst       NUMBER;
   BEGIN
       -- Obtener parámetros
       SELECT dbid INTO v_dbid FROM v$database;
       SELECT instance_number INTO v_inst FROM v$instance;

       SELECT MIN(snap_id), MAX(snap_id)
       INTO v_snap_ini, v_snap_fin
       FROM (SELECT snap_id FROM dba_hist_snapshot ORDER BY snap_id DESC FETCH FIRST 2 ROWS ONLY);

       -- Eliminar tarea previa si existe
       BEGIN
           DBMS_ADVISOR.DELETE_TASK(v_task_name);
       EXCEPTION WHEN OTHERS THEN NULL;
       END;

       -- Crear tarea ADDM
       DBMS_ADVISOR.CREATE_TASK(
           advisor_name => 'ADDM',
           task_name    => v_task_name
       );

       -- Configurar parámetros de la tarea
       DBMS_ADVISOR.SET_TASK_PARAMETER(v_task_name, 'START_SNAPSHOT', v_snap_ini);
       DBMS_ADVISOR.SET_TASK_PARAMETER(v_task_name, 'END_SNAPSHOT', v_snap_fin);
       DBMS_ADVISOR.SET_TASK_PARAMETER(v_task_name, 'INSTANCE', v_inst);
       DBMS_ADVISOR.SET_TASK_PARAMETER(v_task_name, 'DB_ID', v_dbid);

       -- Ejecutar análisis ADDM
       DBMS_ADVISOR.EXECUTE_TASK(v_task_name);
       DBMS_OUTPUT.PUT_LINE('Tarea ADDM ejecutada: ' || v_task_name);
   END;
   /

   -- Consultar los hallazgos de ADDM
   SPOOL /home/oracle/lab06/reporte_addm.txt

   PROMPT ============================================================
   PROMPT REPORTE ADDM - HALLAZGOS Y RECOMENDACIONES
   PROMPT ============================================================

   SELECT
       f.type,
       f.impact_type,
       ROUND(f.impact, 2) AS impacto_pct,
       SUBSTR(f.message, 1, 80) AS hallazgo
   FROM dba_advisor_findings f
   JOIN dba_advisor_tasks t ON f.task_id = t.task_id
   WHERE t.task_name = 'ADDM_LAB06_ANALISIS'
   ORDER BY f.impact DESC;

   PROMPT --- Recomendaciones ADDM ---
   SELECT
       r.type,
       ROUND(r.benefit, 2) AS beneficio_pct,
       SUBSTR(r.message, 1, 80) AS recomendacion
   FROM dba_advisor_recommendations r
   JOIN dba_advisor_tasks t ON r.task_id = t.task_id
   WHERE t.task_name = 'ADDM_LAB06_ANALISIS'
   ORDER BY r.benefit DESC;

   SPOOL OFF

   EXIT;
   EOF

   echo "Reporte ADDM guardado en /home/oracle/lab06/reporte_addm.txt"
   cat /home/oracle/lab06/reporte_addm.txt
   ```

**Salida Esperada:**

```
Snapshot AWR creado con ID: 142
Rango de análisis: Snap 141 a 142
DBID: 1539870386 | Instancia: 1

Reporte AWR generado en /home/oracle/lab06/reporte_awr.txt

REPORTE ADDM - HALLAZGOS Y RECOMENDACIONES
TYPE         IMPACT_TYPE  IMPACTO_PCT  HALLAZGO
------------ ------------ ----------- ----------------------------------------
FINDING      PROBLEM           45.23  SQL statements consuming significant ...
FINDING      PROBLEM           32.10  Hard parse due to literal usage ...
FINDING      SYMPTOM           18.75  Wait class "User I/O" was consuming ...

--- Recomendaciones ADDM ---
TYPE         BENEFICIO_PCT  RECOMENDACION
------------ -------------- ----------------------------------------
SQL_TUNING         45.23    Run SQL Tuning Advisor on statement ...
SCHEMA             32.10    Consider creating an index on column ...
```

**Verificación:**

- El archivo `/home/oracle/lab06/reporte_awr.txt` debe tener más de 100 líneas
- El archivo `/home/oracle/lab06/reporte_addm.txt` debe contener hallazgos y recomendaciones
- La consulta a `dba_advisor_findings` debe retornar al menos 1 fila

---

### Paso 4: Identificar y Analizar Sentencias SQL Problemáticas

**Objetivo:** Usar `V$SQL` para identificar las sentencias SQL con mayor consumo de recursos, generar sus planes de ejecución con `EXPLAIN PLAN` y `DBMS_XPLAN`, e interpretar los resultados para identificar oportunidades de optimización.

**Instrucciones:**

1. Identifica las sentencias SQL con mayor consumo de recursos desde `V$SQL`:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 250
   SET PAGESIZE 50
   COLUMN sql_id FORMAT A15
   COLUMN elapsed_secs FORMAT 999,999.99
   COLUMN cpu_secs FORMAT 999,999.99
   COLUMN buffer_gets FORMAT 999,999,999
   COLUMN disk_reads FORMAT 999,999,999
   COLUMN executions FORMAT 999,999
   COLUMN sql_text_short FORMAT A60

   SPOOL /home/oracle/lab06/top_sql.txt

   PROMPT ============================================================
   PROMPT TOP 10 SENTENCIAS SQL POR CONSUMO DE RECURSOS
   PROMPT Fuente: V$SQL - Etapa OBSERVAR del ciclo de monitoreo
   PROMPT ============================================================

   -- Top SQL por tiempo de CPU
   PROMPT --- TOP SQL por CPU ---
   SELECT
       sql_id,
       executions,
       ROUND(elapsed_time / 1000000, 2) AS elapsed_secs,
       ROUND(cpu_time / 1000000, 2) AS cpu_secs,
       buffer_gets,
       disk_reads,
       ROUND(buffer_gets / NULLIF(executions, 0), 0) AS gets_per_exec,
       SUBSTR(sql_text, 1, 60) AS sql_text_short
   FROM v$sql
   WHERE executions > 0
     AND parsing_schema_name = 'PRACTICA_USER'
   ORDER BY cpu_time DESC
   FETCH FIRST 10 ROWS ONLY;

   -- Top SQL por lecturas físicas (I/O intensivo)
   PROMPT --- TOP SQL por Lecturas Fisicas (I/O) ---
   SELECT
       sql_id,
       executions,
       disk_reads,
       ROUND(disk_reads / NULLIF(executions, 0), 0) AS reads_per_exec,
       buffer_gets,
       ROUND(elapsed_time / 1000000, 2) AS elapsed_secs,
       SUBSTR(sql_text, 1, 60) AS sql_text_short
   FROM v$sql
   WHERE executions > 0
     AND parsing_schema_name = 'PRACTICA_USER'
     AND disk_reads > 0
   ORDER BY disk_reads DESC
   FETCH FIRST 10 ROWS ONLY;

   SPOOL OFF
   EXIT;
   EOF

   cat /home/oracle/lab06/top_sql.txt
   ```

2. Obtén el `sql_id` de la consulta más costosa y genera su plan de ejecución:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 100
   SET ECHO ON

   -- ============================================================
   -- Generar plan de ejecución para la consulta problemática #1
   -- Full Table Scan con función en columna
   -- ============================================================

   -- Limpiar tabla de planes previos
   DELETE FROM plan_table WHERE statement_id = 'CONSULTA_PROBLEMATICA_1';
   COMMIT;

   -- Generar plan de ejecución
   EXPLAIN PLAN
   SET STATEMENT_ID = 'CONSULTA_PROBLEMATICA_1'
   FOR
   SELECT COUNT(*), SUM(monto_total)
   FROM ventas_lab
   WHERE TO_CHAR(fecha_venta, 'YYYY') = TO_CHAR(SYSDATE, 'YYYY')
     AND estado = 'Completada';

   -- Visualizar el plan con DBMS_XPLAN
   SELECT * FROM TABLE(
       DBMS_XPLAN.DISPLAY(
           'PLAN_TABLE',
           'CONSULTA_PROBLEMATICA_1',
           'ALL'
       )
   );

   EXIT;
   EOF
   ```

3. Genera el plan de ejecución para la consulta de join problemática:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 100
   SET ECHO ON

   -- ============================================================
   -- Plan de ejecución para JOIN sin índices en columnas de join
   -- ============================================================
   DELETE FROM plan_table WHERE statement_id = 'CONSULTA_JOIN_SIN_INDICE';
   COMMIT;

   EXPLAIN PLAN
   SET STATEMENT_ID = 'CONSULTA_JOIN_SIN_INDICE'
   FOR
   SELECT v.venta_id, c.nombre, p.nombre AS producto, v.monto_total
   FROM ventas_lab v
   JOIN clientes_lab c ON v.cliente_id = c.cliente_id
   JOIN productos_lab p ON v.producto_id = p.producto_id
   WHERE c.segmento = 'Premium'
     AND p.categoria = 'Electronica'
     AND v.monto_total > 1000
   ORDER BY v.monto_total DESC;

   -- Mostrar plan con formato detallado
   SELECT * FROM TABLE(
       DBMS_XPLAN.DISPLAY(
           'PLAN_TABLE',
           'CONSULTA_JOIN_SIN_INDICE',
           'TYPICAL +ROWS +BYTES +COST'
       )
   );

   EXIT;
   EOF
   ```

4. Analiza el plan real de ejecución usando `V$SQL_PLAN`:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 100
   COLUMN operation FORMAT A30
   COLUMN options FORMAT A20
   COLUMN object_name FORMAT A30
   COLUMN cost FORMAT 999,999
   COLUMN cardinality FORMAT 999,999,999

   PROMPT ============================================================
   PROMPT PLANES DE EJECUCION REALES DESDE V$SQL_PLAN
   PROMPT (Para sentencias ya ejecutadas por PRACTICA_USER)
   PROMPT ============================================================

   -- Obtener sql_ids de las consultas ejecutadas por practica_user
   SELECT sql_id, ROUND(elapsed_time/1000000,2) AS elapsed_secs,
          executions, SUBSTR(sql_text,1,80) AS sql_text
   FROM v$sql
   WHERE parsing_schema_name = 'PRACTICA_USER'
     AND elapsed_time > 0
   ORDER BY elapsed_time DESC
   FETCH FIRST 5 ROWS ONLY;

   EXIT;
   EOF
   ```

5. Usa `DBMS_XPLAN.DISPLAY_CURSOR` para ver el plan de la última ejecución real:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 100

   -- Ejecutar la consulta problemática para capturar su plan real
   SELECT /*+ GATHER_PLAN_STATISTICS */ COUNT(*), SUM(monto_total)
   FROM ventas_lab v
   JOIN clientes_lab c ON v.cliente_id = c.cliente_id
   WHERE c.segmento = 'Premium'
     AND v.fecha_venta > SYSDATE - 90;

   -- Capturar el plan real de la última consulta ejecutada en esta sesión
   SELECT * FROM TABLE(
       DBMS_XPLAN.DISPLAY_CURSOR(
           NULL,   -- NULL = última sentencia de la sesión actual
           NULL,
           'ALLSTATS LAST'
       )
   );

   EXIT;
   EOF
   ```

**Salida Esperada:**

```
TOP 10 SENTENCIAS SQL POR CONSUMO DE RECURSOS:
SQL_ID          EXECUTIONS  ELAPSED_SECS  CPU_SECS  BUFFER_GETS  SQL_TEXT_SHORT
--------------- ----------  ------------ --------- ------------ --------------------
8fk2m9p3qr7t1         10      12.45       11.23    2,345,678   SELECT COUNT(*), SUM...
...

Plan hash value: 1234567890
----------------------------------------------------------
| Id | Operation          | Name       | Rows | Bytes | Cost |
----------------------------------------------------------
|  0 | SELECT STATEMENT   |            |      |       |  4521|
|  1 |  SORT AGGREGATE    |            |    1 |    26 |      |
|* 2 |   TABLE ACCESS FULL| VENTAS_LAB | 25000|   650K|  4521|
----------------------------------------------------------
Predicate Information:
  2 - filter(TO_CHAR("FECHA_VENTA",'YYYY')=TO_CHAR(SYSDATE@!,'YYYY')
             AND "ESTADO"='Completada')
```

**Verificación:**

- Los planes de ejecución deben mostrar `TABLE ACCESS FULL` para las consultas sin índices
- El archivo `/home/oracle/lab06/top_sql.txt` debe contener sentencias de `PRACTICA_USER`
- `DBMS_XPLAN.DISPLAY` debe retornar filas sin errores

---

### Paso 5: Optimizar SQL con SQL Tuning Advisor y Creación de Índices

**Objetivo:** Utilizar `DBMS_SQLTUNE` (SQL Tuning Advisor) para obtener recomendaciones automáticas de optimización, implementar las recomendaciones creando índices estratégicos, y verificar la mejora en los planes de ejecución.

**Instrucciones:**

1. Ejecuta SQL Tuning Advisor para la consulta más costosa:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON SIZE UNLIMITED
   SET LINESIZE 200

   DECLARE
       v_task_name VARCHAR2(30) := 'STA_VENTAS_LAB_01';
       v_sql_text  CLOB;
   BEGIN
       -- Eliminar tarea previa si existe
       BEGIN
           DBMS_SQLTUNE.DROP_TUNING_TASK(v_task_name);
       EXCEPTION WHEN OTHERS THEN NULL;
       END;

       -- Definir el SQL a analizar
       v_sql_text := 'SELECT COUNT(*), SUM(monto_total) ' ||
                     'FROM ventas_lab ' ||
                     'WHERE TO_CHAR(fecha_venta, ''YYYY'') = TO_CHAR(SYSDATE, ''YYYY'') ' ||
                     'AND estado = ''Completada''';

       -- Crear la tarea de SQL Tuning
       v_task_name := DBMS_SQLTUNE.CREATE_TUNING_TASK(
           sql_text     => v_sql_text,
           user_name    => 'PRACTICA_USER',
           scope        => 'COMPREHENSIVE',
           time_limit   => 60,
           task_name    => v_task_name,
           description  => 'Analisis de consulta con funcion en columna fecha'
       );

       DBMS_OUTPUT.PUT_LINE('Tarea SQL Tuning creada: ' || v_task_name);

       -- Ejecutar el análisis
       DBMS_SQLTUNE.EXECUTE_TUNING_TASK(task_name => v_task_name);
       DBMS_OUTPUT.PUT_LINE('Análisis completado.');
   END;
   /

   -- Ver el reporte de recomendaciones
   SPOOL /home/oracle/lab06/sql_tuning_reporte.txt

   PROMPT ============================================================
   PROMPT REPORTE SQL TUNING ADVISOR - STA_VENTAS_LAB_01
   PROMPT ============================================================

   SELECT DBMS_SQLTUNE.REPORT_TUNING_TASK('STA_VENTAS_LAB_01') AS reporte
   FROM DUAL;

   SPOOL OFF

   EXIT;
   EOF

   echo "Reporte SQL Tuning Advisor guardado."
   head -80 /home/oracle/lab06/sql_tuning_reporte.txt
   ```

2. Ejecuta SQL Tuning Advisor para la consulta de join:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON SIZE UNLIMITED

   DECLARE
       v_task_name VARCHAR2(30) := 'STA_JOIN_LAB_02';
       v_sql_text  CLOB;
   BEGIN
       BEGIN
           DBMS_SQLTUNE.DROP_TUNING_TASK(v_task_name);
       EXCEPTION WHEN OTHERS THEN NULL;
       END;

       v_sql_text :=
           'SELECT v.venta_id, c.nombre, p.nombre AS producto, v.monto_total ' ||
           'FROM ventas_lab v ' ||
           'JOIN clientes_lab c ON v.cliente_id = c.cliente_id ' ||
           'JOIN productos_lab p ON v.producto_id = p.producto_id ' ||
           'WHERE c.segmento = :1 ' ||
           'AND p.categoria = :2 ' ||
           'AND v.monto_total > :3 ' ||
           'ORDER BY v.monto_total DESC';

       v_task_name := DBMS_SQLTUNE.CREATE_TUNING_TASK(
           sql_text     => v_sql_text,
           bind_list    => sql_binds(anydata.ConvertVarchar2('Premium'),
                                    anydata.ConvertVarchar2('Electronica'),
                                    anydata.ConvertNumber(1000)),
           user_name    => 'PRACTICA_USER',
           scope        => 'COMPREHENSIVE',
           time_limit   => 60,
           task_name    => v_task_name,
           description  => 'Analisis de join sin indices en columnas de join'
       );

       DBMS_SQLTUNE.EXECUTE_TUNING_TASK(task_name => v_task_name);
       DBMS_OUTPUT.PUT_LINE('Análisis de JOIN completado: ' || v_task_name);
   END;
   /

   -- Ver reporte
   SELECT DBMS_SQLTUNE.REPORT_TUNING_TASK('STA_JOIN_LAB_02') FROM DUAL;

   EXIT;
   EOF
   ```

3. Implementa los índices recomendados (aplicando las recomendaciones del advisor):

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET ECHO ON
   SET FEEDBACK ON
   SET TIMING ON

   -- ============================================================
   -- IMPLEMENTAR RECOMENDACIONES - Etapa ACTUAR del ciclo
   -- ============================================================

   -- Índice 1: En fecha_venta para evitar full scan con filtro de fecha
   -- NOTA: Índice basado en función para soportar TO_CHAR(fecha_venta, 'YYYY')
   PROMPT Creando indice en fecha_venta...
   CREATE INDEX idx_ventas_fecha ON ventas_lab(fecha_venta);

   -- Índice 2: Índice compuesto en (estado, fecha_venta) para la consulta combinada
   PROMPT Creando indice compuesto estado + fecha...
   CREATE INDEX idx_ventas_estado_fecha ON ventas_lab(estado, fecha_venta, monto_total);

   -- Índice 3: Índice basado en función para TO_CHAR(fecha_venta, 'YYYY')
   PROMPT Creando indice de funcion para TO_CHAR...
   CREATE INDEX idx_ventas_anio ON ventas_lab(TO_CHAR(fecha_venta, 'YYYY'));

   -- Índice 4: En cliente_id para acelerar el JOIN
   PROMPT Creando indice en cliente_id de ventas...
   CREATE INDEX idx_ventas_cliente_id ON ventas_lab(cliente_id);

   -- Índice 5: En producto_id para acelerar el JOIN
   PROMPT Creando indice en producto_id de ventas...
   CREATE INDEX idx_ventas_producto_id ON ventas_lab(producto_id);

   -- Índice 6: En segmento de clientes para el filtro
   PROMPT Creando indice en segmento de clientes...
   CREATE INDEX idx_clientes_segmento ON clientes_lab(segmento);

   -- Índice 7: En categoria de productos
   PROMPT Creando indice en categoria de productos...
   CREATE INDEX idx_productos_categoria ON productos_lab(categoria);

   -- Índice 8: Índice de función para búsqueda UPPER(ciudad)
   PROMPT Creando indice de funcion UPPER(ciudad)...
   CREATE INDEX idx_clientes_ciudad_upper ON clientes_lab(UPPER(ciudad));

   -- Recopilar estadísticas actualizadas con los nuevos índices
   EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'VENTAS_LAB', CASCADE => TRUE);
   EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'CLIENTES_LAB', CASCADE => TRUE);
   EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'PRODUCTOS_LAB', CASCADE => TRUE);

   PROMPT Todos los indices creados y estadisticas actualizadas.

   EXIT;
   EOF
   ```

4. Verifica la mejora en los planes de ejecución después de crear los índices:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 100
   SET TIMING ON

   -- ============================================================
   -- VERIFICAR MEJORA - Comparar planes ANTES y DESPUÉS
   -- ============================================================

   -- Plan DESPUÉS de crear índices - Consulta con función en fecha
   DELETE FROM plan_table WHERE statement_id = 'CONSULTA_OPTIMIZADA_1';
   COMMIT;

   EXPLAIN PLAN
   SET STATEMENT_ID = 'CONSULTA_OPTIMIZADA_1'
   FOR
   SELECT COUNT(*), SUM(monto_total)
   FROM ventas_lab
   WHERE TO_CHAR(fecha_venta, 'YYYY') = TO_CHAR(SYSDATE, 'YYYY')
     AND estado = 'Completada';

   PROMPT --- Plan OPTIMIZADO (con indice de funcion) ---
   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY('PLAN_TABLE', 'CONSULTA_OPTIMIZADA_1', 'TYPICAL'));

   -- Plan DESPUÉS - Consulta de JOIN optimizada
   DELETE FROM plan_table WHERE statement_id = 'JOIN_OPTIMIZADO';
   COMMIT;

   EXPLAIN PLAN
   SET STATEMENT_ID = 'JOIN_OPTIMIZADO'
   FOR
   SELECT v.venta_id, c.nombre, p.nombre AS producto, v.monto_total
   FROM ventas_lab v
   JOIN clientes_lab c ON v.cliente_id = c.cliente_id
   JOIN productos_lab p ON v.producto_id = p.producto_id
   WHERE c.segmento = 'Premium'
     AND p.categoria = 'Electronica'
     AND v.monto_total > 1000
   ORDER BY v.monto_total DESC;

   PROMPT --- Plan JOIN OPTIMIZADO (con indices en columnas de join) ---
   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY('PLAN_TABLE', 'JOIN_OPTIMIZADO', 'TYPICAL +ROWS +COST'));

   -- Prueba de rendimiento: medir tiempo de ejecución real
   PROMPT --- Midiendo tiempo de ejecucion real ---
   SET TIMING ON

   SELECT COUNT(*), SUM(monto_total)
   FROM ventas_lab
   WHERE TO_CHAR(fecha_venta, 'YYYY') = TO_CHAR(SYSDATE, 'YYYY')
     AND estado = 'Completada';

   SELECT COUNT(*)
   FROM ventas_lab v
   JOIN clientes_lab c ON v.cliente_id = c.cliente_id
   JOIN productos_lab p ON v.producto_id = p.producto_id
   WHERE c.segmento = 'Premium'
     AND p.categoria = 'Electronica'
     AND v.monto_total > 1000;

   EXIT;
   EOF
   ```

5. Documenta la mejora de rendimiento (Etapa DOCUMENTAR del ciclo):

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 50
   SPOOL /home/oracle/lab06/documentacion_mejoras.txt

   PROMPT ============================================================
   PROMPT DOCUMENTACION DE MEJORAS - ETAPA DOCUMENTAR
   PROMPT Fecha: $(date)
   PROMPT ============================================================

   PROMPT --- Indices creados en el esquema PRACTICA_USER ---
   SELECT index_name, table_name, index_type, status,
          ROUND(leaf_blocks * 8 / 1024, 2) AS size_mb
   FROM dba_indexes
   WHERE owner = 'PRACTICA_USER'
   ORDER BY table_name, index_name;

   PROMPT --- Comparativa de rendimiento (ultimas ejecuciones en V$SQL) ---
   SELECT sql_id,
          executions,
          ROUND(elapsed_time/1000000/NULLIF(executions,0),4) AS avg_elapsed_secs,
          ROUND(cpu_time/1000000/NULLIF(executions,0),4) AS avg_cpu_secs,
          ROUND(buffer_gets/NULLIF(executions,0),0) AS avg_buffer_gets,
          SUBSTR(sql_text,1,60) AS sql_text
   FROM v$sql
   WHERE parsing_schema_name = 'PRACTICA_USER'
     AND executions > 0
   ORDER BY elapsed_time DESC
   FETCH FIRST 10 ROWS ONLY;

   SPOOL OFF
   EXIT;
   EOF

   echo "Documentación guardada en /home/oracle/lab06/documentacion_mejoras.txt"
   cat /home/oracle/lab06/documentacion_mejoras.txt
   ```

**Salida Esperada:**

```
REPORTE SQL TUNING ADVISOR - STA_VENTAS_LAB_01
GENERAL INFORMATION
Task Name: STA_VENTAS_LAB_01
Scope: COMPREHENSIVE
...
FINDINGS SECTION
1- Index Finding (see explain plans section below)
   The execution plan of this statement can be improved by creating one or more indices.
   Recommendation: Consider running the Access Advisor to improve the physical schema design
   for this query.
   Rationale: Creating the recommended indices significantly improves the execution plan...

Plan OPTIMIZADO (con indice de funcion):
Plan hash value: 987654321
------------------------------------------------------------------
| Id | Operation                   | Name               | Rows  |
------------------------------------------------------------------
|  0 | SELECT STATEMENT            |                    |     1 |
|  1 |  SORT AGGREGATE             |                    |     1 |
|* 2 |   INDEX RANGE SCAN          | IDX_VENTAS_ANIO    | 25000 |
------------------------------------------------------------------
```

**Verificación:**

- Los planes de ejecución OPTIMIZADOS deben mostrar `INDEX RANGE SCAN` en lugar de `TABLE ACCESS FULL`
- El archivo `/home/oracle/lab06/sql_tuning_reporte.txt` debe contener recomendaciones de índices
- El tiempo de ejecución de las consultas debe haber disminuido (verificar con `SET TIMING ON`)

---

### Paso 6: Configurar Gestión Automática de Memoria (AMM y ASMM)

**Objetivo:** Revisar la configuración actual de memoria de la instancia Oracle, configurar `MEMORY_TARGET` para AMM (Automatic Memory Management) y ajustar `SGA_TARGET` / `PGA_AGGREGATE_TARGET` para ASMM, analizando el impacto en el rendimiento.

**Instrucciones:**

1. Revisa la configuración actual de memoria:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 50
   COLUMN name FORMAT A35
   COLUMN value FORMAT A20
   COLUMN description FORMAT A50

   SPOOL /home/oracle/lab06/memoria_config_inicial.txt

   PROMPT ============================================================
   PROMPT CONFIGURACION ACTUAL DE MEMORIA - ESTADO INICIAL
   PROMPT ============================================================

   -- Parámetros de memoria principales
   SELECT name, value, description
   FROM v$parameter
   WHERE name IN (
       'memory_target',
       'memory_max_target',
       'sga_target',
       'sga_max_size',
       'pga_aggregate_target',
       'pga_aggregate_limit',
       'shared_pool_size',
       'db_cache_size',
       'large_pool_size',
       'java_pool_size',
       'streams_pool_size'
   )
   ORDER BY name;

   PROMPT --- Uso actual de SGA por componente ---
   SELECT pool, name, ROUND(bytes/1024/1024, 2) AS mb_usados
   FROM v$sgastat
   WHERE pool IS NOT NULL
   ORDER BY bytes DESC
   FETCH FIRST 15 ROWS ONLY;

   PROMPT --- Resumen de PGA ---
   SELECT name, ROUND(value/1024/1024, 2) AS mb
   FROM v$pgastat
   WHERE name IN (
       'total PGA inuse',
       'total PGA allocated',
       'maximum PGA allocated',
       'total freeable PGA memory',
       'cache hit percentage'
   );

   SPOOL OFF
   EXIT;
   EOF

   cat /home/oracle/lab06/memoria_config_inicial.txt
   ```

2. Consulta el Memory Advisor para obtener recomendaciones de tamaño óptimo:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 50
   COLUMN memory_size_mb FORMAT 999,999
   COLUMN estd_db_time_factor FORMAT 999.99
   COLUMN version FORMAT 999

   PROMPT ============================================================
   PROMPT MEMORY ADVISOR - RECOMENDACIONES DE TAMAÑO OPTIMO
   PROMPT ============================================================

   -- SGA Advisor: relación entre tamaño de SGA y tiempo de base de datos
   PROMPT --- SGA Size Advisor ---
   SELECT
       ROUND(sga_size / 1024, 0) AS sga_size_mb,
       sga_size_factor,
       ROUND(estd_db_time_factor, 3) AS estd_db_time_factor,
       estd_physical_reads
   FROM v$sga_target_advice
   ORDER BY sga_size;

   -- PGA Advisor: relación entre tamaño de PGA y rendimiento
   PROMPT --- PGA Size Advisor ---
   SELECT
       ROUND(pga_target_for_estimate / 1024 / 1024, 0) AS pga_target_mb,
       pga_target_factor,
       ROUND(estd_pga_cache_hit_percentage, 1) AS cache_hit_pct,
       estd_overalloc_count
   FROM v$pga_target_advice
   ORDER BY pga_target_for_estimate;

   EXIT;
   EOF
   ```

3. Configura ASMM (Automatic Shared Memory Management) ajustando `SGA_TARGET`:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON
   SET LINESIZE 200

   -- ============================================================
   -- CONFIGURAR ASMM (Automatic Shared Memory Management)
   -- SGA_TARGET permite a Oracle gestionar automáticamente los
   -- componentes internos de la SGA dentro del límite definido
   -- ============================================================

   -- Verificar configuración actual antes del cambio
   SHOW PARAMETER sga_target;
   SHOW PARAMETER memory_target;

   -- Nota: Si MEMORY_TARGET > 0, Oracle usa AMM (gestiona SGA+PGA)
   -- Si MEMORY_TARGET = 0 y SGA_TARGET > 0, Oracle usa ASMM (solo SGA)

   -- Paso 1: Asegurarse de que AMM está desactivado para configurar ASMM
   -- (Si memory_target > 0, primero desactivarlo)
   DECLARE
       v_mem_target NUMBER;
   BEGIN
       SELECT TO_NUMBER(value) INTO v_mem_target
       FROM v$parameter WHERE name = 'memory_target';

       IF v_mem_target > 0 THEN
           EXECUTE IMMEDIATE 'ALTER SYSTEM SET memory_target = 0 SCOPE=SPFILE';
           DBMS_OUTPUT.PUT_LINE('MEMORY_TARGET desactivado. Requiere reinicio.');
       ELSE
           DBMS_OUTPUT.PUT_LINE('MEMORY_TARGET ya está en 0. Continuando con ASMM.');
       END IF;
   END;
   /

   -- Configurar SGA_TARGET (ajustar según la RAM disponible)
   -- En un sistema con 16 GB RAM, asignar ~4 GB a SGA es razonable
   ALTER SYSTEM SET sga_target = 1536M SCOPE=BOTH;
   ALTER SYSTEM SET sga_max_size = 2048M SCOPE=SPFILE;

   -- Configurar PGA_AGGREGATE_TARGET
   ALTER SYSTEM SET pga_aggregate_target = 512M SCOPE=BOTH;

   -- Verificar cambios aplicados
   SHOW PARAMETER sga_target;
   SHOW PARAMETER pga_aggregate_target;

   PROMPT ============================================================
   PROMPT ASMM configurado. Oracle gestionará automáticamente:
   PROMPT - Buffer Cache
   PROMPT - Shared Pool
   PROMPT - Large Pool
   PROMPT - Java Pool
   PROMPT - Streams Pool
   PROMPT dentro del límite de SGA_TARGET
   PROMPT ============================================================

   EXIT;
   EOF
   ```

4. Configura AMM (Automatic Memory Management) con `MEMORY_TARGET`:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET SERVEROUTPUT ON
   SET LINESIZE 200

   -- ============================================================
   -- CONFIGURAR AMM (Automatic Memory Management)
   -- MEMORY_TARGET permite a Oracle gestionar SGA + PGA juntos
   -- Oracle redistribuye memoria entre SGA y PGA según la carga
   -- ============================================================

   PROMPT Verificando prerequisito: /dev/shm debe ser suficiente para AMM

   -- Verificar si hay suficiente espacio en /dev/shm (tmpfs)
   -- Desde SQL*Plus no podemos ejecutar comandos OS, pero documentamos el requerimiento

   DECLARE
       v_sga_max   NUMBER;
       v_pga_max   NUMBER;
       v_mem_total NUMBER;
   BEGIN
       -- Obtener valores actuales para calcular MEMORY_TARGET apropiado
       SELECT TO_NUMBER(value) INTO v_sga_max FROM v$parameter WHERE name = 'sga_max_size';
       SELECT TO_NUMBER(value) INTO v_pga_max FROM v$parameter WHERE name = 'pga_aggregate_target';

       v_mem_total := v_sga_max + v_pga_max;
       DBMS_OUTPUT.PUT_LINE('SGA_MAX_SIZE actual: ' || ROUND(v_sga_max/1024/1024,0) || ' MB');
       DBMS_OUTPUT.PUT_LINE('PGA_AGGREGATE_TARGET actual: ' || ROUND(v_pga_max/1024/1024,0) || ' MB');
       DBMS_OUTPUT.PUT_LINE('MEMORY_TARGET sugerido (SGA+PGA): ' || ROUND(v_mem_total/1024/1024,0) || ' MB');
   END;
   /

   -- Configurar MEMORY_TARGET para AMM
   -- MEMORY_TARGET debe ser <= MEMORY_MAX_TARGET
   ALTER SYSTEM SET memory_max_target = 2048M SCOPE=SPFILE;
   ALTER SYSTEM SET memory_target = 1792M SCOPE=SPFILE;

   -- Con AMM activo, estos parámetros son opcionales (son límites mínimos)
   -- Oracle puede asignar más si lo necesita, hasta MEMORY_TARGET
   ALTER SYSTEM SET sga_target = 0 SCOPE=SPFILE;
   ALTER SYSTEM SET pga_aggregate_target = 0 SCOPE=SPFILE;

   PROMPT ============================================================
   PROMPT AMM configurado en SPFILE. Requiere reinicio para activarse.
   PROMPT Para reiniciar: SHUTDOWN IMMEDIATE; STARTUP;
   PROMPT ============================================================

   PROMPT Configuracion actual (antes del reinicio):
   SHOW PARAMETER memory_target;
   SHOW PARAMETER memory_max_target;

   EXIT;
   EOF
   ```

5. Monitorea el estado de la memoria después de la configuración:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 50
   SPOOL /home/oracle/lab06/memoria_monitoreo.txt

   PROMPT ============================================================
   PROMPT MONITOREO DE MEMORIA POST-CONFIGURACION
   PROMPT ============================================================

   -- Estado actual de componentes SGA
   PROMPT --- Componentes SGA actuales ---
   SELECT component,
          ROUND(current_size/1024/1024, 2) AS current_mb,
          ROUND(min_size/1024/1024, 2) AS min_mb,
          ROUND(max_size/1024/1024, 2) AS max_mb,
          last_oper_type,
          last_oper_mode
   FROM v$sga_dynamic_components
   ORDER BY current_size DESC;

   -- Estadísticas de PGA
   PROMPT --- Estadisticas PGA ---
   SELECT name, ROUND(value/1024/1024, 2) AS mb
   FROM v$pgastat
   WHERE name IN (
       'total PGA inuse',
       'total PGA allocated',
       'maximum PGA allocated',
       'cache hit percentage',
       'total freeable PGA memory'
   );

   -- Historial de cambios automáticos de memoria
   PROMPT --- Historial de cambios de memoria automaticos (ultimos 10) ---
   SELECT component,
          TO_CHAR(start_time, 'DD-MON HH24:MI') AS inicio,
          ROUND(initial_size/1024/1024, 2) AS inicial_mb,
          ROUND(final_size/1024/1024, 2) AS final_mb,
          oper_type,
          status
   FROM v$memory_resize_ops
   ORDER BY start_time DESC
   FETCH FIRST 10 ROWS ONLY;

   SPOOL OFF
   EXIT;
   EOF

   echo "Monitoreo de memoria guardado."
   cat /home/oracle/lab06/memoria_monitoreo.txt
   ```

**Salida Esperada:**

```
CONFIGURACION ACTUAL DE MEMORIA - ESTADO INICIAL
NAME                              VALUE
--------------------------------- --------------------
db_cache_size                     0
memory_max_target                 2G
memory_target                     1792M
pga_aggregate_limit               2G
pga_aggregate_target              512M
sga_max_size                      2G
sga_target                        1536M

SGA Size Advisor:
SGA_SIZE_MB  SGA_SIZE_FACTOR  ESTD_DB_TIME_FACTOR  ESTD_PHYSICAL_READS
----------- ---------------- -------------------- -------------------
        512             0.33                 2.45           8,234,567
       1024             0.67                 1.32           3,456,789
       1536             1.00                 1.00           1,234,567
       2048             1.33                 0.98           1,198,234
```

**Verificación:**

- Los parámetros `SGA_TARGET` y `PGA_AGGREGATE_TARGET` deben reflejar los nuevos valores
- `V$SGA_DYNAMIC_COMPONENTS` debe mostrar los componentes con sus tamaños actuales
- No debe haber errores `ORA-` al ejecutar los comandos `ALTER SYSTEM`

---

### Paso 7: Escenario Integrador – Diagnóstico Completo de Rendimiento

**Objetivo:** Aplicar el ciclo completo de monitoreo (Observar → Analizar → Actuar → Documentar) en un escenario integrador que simula un problema real de degradación de rendimiento, consolidando todos los conocimientos del laboratorio.

**Instrucciones:**

1. Simula el escenario de degradación: una consulta de reporte ejecutada sin índices:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET ECHO OFF
   SET FEEDBACK OFF
   SET SERVEROUTPUT ON
   SET TIMING ON

   PROMPT ============================================================
   PROMPT ESCENARIO INTEGRADOR: Reporte de ventas degradado
   PROMPT ============================================================

   -- Simular la consulta problemática de un reporte de negocio
   -- Esta consulta representa un caso típico de un reporte de cierre de mes
   DECLARE
       v_count NUMBER;
       v_total NUMBER;
   BEGIN
       DBMS_OUTPUT.PUT_LINE('Ejecutando reporte de ventas mensual...');

       -- Consulta de reporte que usa funciones en columnas indexadas
       -- y realiza múltiples joins sin aprovechar índices óptimos
       SELECT
           COUNT(DISTINCT v.cliente_id),
           SUM(v.monto_total),
           AVG(v.monto_total)
       INTO v_count, v_total, v_total
       FROM ventas_lab v
       JOIN clientes_lab c ON v.cliente_id = c.cliente_id
       JOIN productos_lab p ON v.producto_id = p.producto_id
       WHERE EXTRACT(YEAR FROM v.fecha_venta) = EXTRACT(YEAR FROM SYSDATE)
         AND EXTRACT(MONTH FROM v.fecha_venta) = EXTRACT(MONTH FROM SYSDATE)
         AND v.estado IN ('Completada', 'En proceso')
         AND c.ciudad IN ('Ciudad de Mexico', 'Guadalajara', 'Monterrey');

       DBMS_OUTPUT.PUT_LINE('Clientes únicos: ' || v_count);
       DBMS_OUTPUT.PUT_LINE('Total ventas: $' || ROUND(v_total, 2));
   END;
   /

   EXIT;
   EOF
   ```

2. Etapa OBSERVAR: Captura el estado del sistema durante/después de la carga:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 50
   SPOOL /home/oracle/lab06/escenario_observar.txt

   PROMPT ============================================================
   PROMPT ETAPA 1: OBSERVAR - Estado del sistema
   PROMPT ============================================================

   -- Sesiones activas y sus esperas
   PROMPT --- Sesiones activas ---
   SELECT s.sid, s.username, s.status, s.event,
          s.wait_class, s.seconds_in_wait, s.sql_id
   FROM v$session s
   WHERE s.type = 'USER'
   ORDER BY s.seconds_in_wait DESC;

   -- Top eventos de espera recientes (última hora)
   PROMPT --- Eventos de espera recientes (ultima hora) ---
   SELECT event, count(*) AS ocurrencias,
          ROUND(AVG(wait_time_micro)/1000, 2) AS avg_wait_ms,
          wait_class
   FROM v$active_session_history
   WHERE sample_time > SYSDATE - 1/24
     AND session_type = 'FOREGROUND'
   GROUP BY event, wait_class
   ORDER BY ocurrencias DESC
   FETCH FIRST 10 ROWS ONLY;

   -- SQL más reciente con alto consumo
   PROMPT --- SQL reciente con alto consumo ---
   SELECT sql_id, executions,
          ROUND(elapsed_time/1000000, 2) AS elapsed_secs,
          ROUND(cpu_time/1000000, 2) AS cpu_secs,
          buffer_gets, disk_reads,
          SUBSTR(sql_text, 1, 70) AS sql_text
   FROM v$sql
   WHERE parsing_schema_name = 'PRACTICA_USER'
     AND last_active_time > SYSDATE - 1/24
   ORDER BY elapsed_time DESC
   FETCH FIRST 5 ROWS ONLY;

   SPOOL OFF
   EXIT;
   EOF

   cat /home/oracle/lab06/escenario_observar.txt
   ```

3. Etapa ANALIZAR: Genera plan de ejecución y compara con la línea base:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 100
   SPOOL /home/oracle/lab06/escenario_analizar.txt

   PROMPT ============================================================
   PROMPT ETAPA 2: ANALIZAR - Plan de ejecución del reporte degradado
   PROMPT ============================================================

   DELETE FROM plan_table WHERE statement_id = 'REPORTE_DEGRADADO';
   COMMIT;

   EXPLAIN PLAN
   SET STATEMENT_ID = 'REPORTE_DEGRADADO'
   FOR
   SELECT COUNT(DISTINCT v.cliente_id), SUM(v.monto_total), AVG(v.monto_total)
   FROM ventas_lab v
   JOIN clientes_lab c ON v.cliente_id = c.cliente_id
   JOIN productos_lab p ON v.producto_id = p.producto_id
   WHERE EXTRACT(YEAR FROM v.fecha_venta) = EXTRACT(YEAR FROM SYSDATE)
     AND EXTRACT(MONTH FROM v.fecha_venta) = EXTRACT(MONTH FROM SYSDATE)
     AND v.estado IN ('Completada', 'En proceso')
     AND c.ciudad IN ('Ciudad de Mexico', 'Guadalajara', 'Monterrey');

   SELECT * FROM TABLE(
       DBMS_XPLAN.DISPLAY('PLAN_TABLE', 'REPORTE_DEGRADADO', 'ALL')
   );

   SPOOL OFF
   EXIT;
   EOF

   cat /home/oracle/lab06/escenario_analizar.txt
   ```

4. Etapa ACTUAR: Implementa la solución optimizada:

   ```bash
   sqlplus practica_user/Oracle123 << 'EOF'
   SET ECHO ON
   SET TIMING ON
   SPOOL /home/oracle/lab06/escenario_actuar.txt

   PROMPT ============================================================
   PROMPT ETAPA 3: ACTUAR - Implementar optimizacion del reporte
   PROMPT ============================================================

   -- Crear índice compuesto para la consulta de reporte
   -- Cubre: filtros de fecha (EXTRACT), estado y ciudad
   CREATE INDEX idx_ventas_reporte ON ventas_lab(
       EXTRACT(YEAR FROM fecha_venta),
       EXTRACT(MONTH FROM fecha_venta),
       estado,
       cliente_id,
       monto_total
   );

   CREATE INDEX idx_clientes_ciudad ON clientes_lab(ciudad, cliente_id);

   -- Actualizar estadísticas
   EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'VENTAS_LAB', CASCADE => TRUE);
   EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'CLIENTES_LAB', CASCADE => TRUE);

   -- Verificar nuevo plan de ejecución
   DELETE FROM plan_table WHERE statement_id = 'REPORTE_OPTIMIZADO';
   COMMIT;

   EXPLAIN PLAN
   SET STATEMENT_ID = 'REPORTE_OPTIMIZADO'
   FOR
   SELECT COUNT(DISTINCT v.cliente_id), SUM(v.monto_total), AVG(v.monto_total)
   FROM ventas_lab v
   JOIN clientes_lab c ON v.cliente_id = c.cliente_id
   JOIN productos_lab p ON v.producto_id = p.producto_id
   WHERE EXTRACT(YEAR FROM v.fecha_venta) = EXTRACT(YEAR FROM SYSDATE)
     AND EXTRACT(MONTH FROM v.fecha_venta) = EXTRACT(MONTH FROM SYSDATE)
     AND v.estado IN ('Completada', 'En proceso')
     AND c.ciudad IN ('Ciudad de Mexico', 'Guadalajara', 'Monterrey');

   PROMPT --- Plan OPTIMIZADO del reporte ---
   SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY('PLAN_TABLE', 'REPORTE_OPTIMIZADO', 'TYPICAL'));

   -- Ejecutar la consulta optimizada y medir tiempo
   PROMPT --- Ejecucion optimizada ---
   SELECT COUNT(DISTINCT v.cliente_id), SUM(v.monto_total), AVG(v.monto_total)
   FROM ventas_lab v
   JOIN clientes_lab c ON v.cliente_id = c.cliente_id
   JOIN productos_lab p ON v.producto_id = p.producto_id
   WHERE EXTRACT(YEAR FROM v.fecha_venta) = EXTRACT(YEAR FROM SYSDATE)
     AND EXTRACT(MONTH FROM v.fecha_venta) = EXTRACT(MONTH FROM SYSDATE)
     AND v.estado IN ('Completada', 'En proceso')
     AND c.ciudad IN ('Ciudad de Mexico', 'Guadalajara', 'Monterrey');

   SPOOL OFF
   EXIT;
   EOF

   cat /home/oracle/lab06/escenario_actuar.txt
   ```

5. Etapa DOCUMENTAR: Genera el reporte final consolidado del laboratorio:

   ```bash
   sqlplus / as sysdba << 'EOF'
   SET LINESIZE 200
   SET PAGESIZE 100
   SPOOL /home/oracle/lab06/reporte_final_lab06.txt

   PROMPT ============================================================
   PROMPT REPORTE FINAL - LABORATORIO 06-00-01
   PROMPT Metodologia de Monitoreo Proactivo Oracle 19c
   PROMPT ============================================================

   PROMPT
   PROMPT === SECCION 1: INDICES CREADOS ===
   SELECT index_name, table_name, index_type, uniqueness, status,
          ROUND(leaf_blocks * 8 / 1024, 2) AS size_mb
   FROM dba_indexes
   WHERE owner = 'PRACTICA_USER'
   ORDER BY table_name, index_name;

   PROMPT
   PROMPT === SECCION 2: ESTADISTICAS DE TABLAS ===
   SELECT table_name, num_rows, blocks,
          TO_CHAR(last_analyzed, 'DD-MON-YYYY HH24:MI') AS last_analyzed
   FROM dba_tables
   WHERE owner = 'PRACTICA_USER'
   ORDER BY table_name;

   PROMPT
   PROMPT === SECCION 3: TOP SQL FINAL (PRACTICA_USER) ===
   SELECT sql_id,
          executions,
          ROUND(elapsed_time/1000000/NULLIF(executions,0), 4) AS avg_elapsed_secs,
          ROUND(buffer_gets/NULLIF(executions,0), 0) AS avg_buffer_gets,
          SUBSTR(sql_text, 1, 60) AS sql_text
   FROM v$sql
   WHERE parsing_schema_name = 'PRACTICA_USER'
     AND executions > 0
   ORDER BY elapsed_time DESC
   FETCH FIRST 10 ROWS ONLY;

   PROMPT
   PROMPT === SECCION 4: CONFIGURACION DE MEMORIA FINAL ===
   SELECT name, value
   FROM v$parameter
   WHERE name IN ('memory_target','memory_max_target','sga_target',
                  'pga_aggregate_target','sga_max_size')
   ORDER BY name;

   PROMPT
   PROMPT === SECCION 5: SNAPSHOTS AWR GENERADOS ===
   SELECT snap_id,
          TO_CHAR(begin_interval_time, 'DD-MON-YYYY HH24:MI') AS inicio,
          TO_CHAR(end_interval_time, 'DD-MON-YYYY HH24:MI') AS fin
   FROM dba_hist_snapshot
   ORDER BY snap_id DESC
   FETCH FIRST 10 ROWS ONLY;

   PROMPT
   PROMPT === FIN DEL REPORTE ===

   SPOOL OFF
   EXIT;
   EOF

   echo ""
   echo "============================================"
   echo "LABORATORIO 06-00-01 COMPLETADO"
   echo "Archivos generados en /home/oracle/lab06/:"
   ls -la /home/oracle/lab06/
   echo "============================================"
   ```

**Salida Esperada:**

```
============================================================
REPORTE FINAL - LABORATORIO 06-00-01
============================================================

=== SECCION 1: INDICES CREADOS ===
INDEX_NAME                    TABLE_NAME     INDEX_TYPE  STATUS  SIZE_MB
----------------------------- -------------- ----------- ------- -------
IDX_CLIENTES_CIUDAD           CLIENTES_LAB   NORMAL      VALID     0.45
IDX_CLIENTES_CIUDAD_UPPER     CLIENTES_LAB   FUNCTION-BASED VALID  0.38
IDX_CLIENTES_SEGMENTO         CLIENTES_LAB   NORMAL      VALID     0.32
IDX_PRODUCTOS_CATEGORIA       PRODUCTOS_LAB  NORMAL      VALID     0.12
IDX_VENTAS_ANIO               VENTAS_LAB     FUNCTION-BASED VALID  2.34
IDX_VENTAS_CLIENTE_ID         VENTAS_LAB     NORMAL      VALID     3.12
IDX_VENTAS_ESTADO_FECHA       VENTAS_LAB     NORMAL      VALID     4.56
IDX_VENTAS_FECHA              VENTAS_LAB     NORMAL      VALID     3.89
IDX_VENTAS_PRODUCTO_ID        VENTAS_LAB     NORMAL      VALID     2.98
IDX_VENTAS_REPORTE            VENTAS_LAB     FUNCTION-BASED VALID  5.23

=== SECCION 2: ESTADISTICAS DE TABLAS ===
TABLE_NAME      NUM_ROWS  LAST_ANALYZED
--------------- --------- -------------------
CLIENTES_LAB       5,000  15-JAN-2025 14:32
PRODUCTOS_LAB      1,000  15-JAN-2025 14:32
VENTAS_LAB       100,000  15-JAN-2025 14:45

LABORATORIO 06-00-01 COMPLETADO
Archivos generados en /home/oracle/lab06/:
-rw-r--r-- 1 oracle oinstall  45234 Jan 15 14:50 baseline_kpis.txt
-rw-r--r-- 1 oracle oinstall 234567 Jan 15 15:12 reporte_awr.txt
-rw-r--r-- 1 oracle oinstall  12345 Jan 15 15:13 reporte_addm.txt
...
```

**Verificación:**

- El directorio `/home/oracle/lab06/` debe contener al menos 8 archivos de reporte
- El reporte final debe mostrar todos los índices creados con estado `VALID`
- Las estadísticas de tablas deben tener `last_analyzed` con fecha del día actual

---

## Validación y Pruebas

### Criterios de Éxito

- [ ] La línea base de rendimiento fue capturada exitosamente y guardada en `/home/oracle/lab06/baseline_kpis.txt`
- [ ] Las tablas `ventas_lab`, `clientes_lab` y `productos_lab` contienen 100,000, 5,000 y 1,000 registros respectivamente
- [ ] Se generaron al menos 2 snapshots AWR y el reporte AWR en `/home/oracle/lab06/reporte_awr.txt` tiene más de 100 líneas
- [ ] El reporte ADDM en `/home/oracle/lab06/reporte_addm.txt` contiene al menos 1 hallazgo
- [ ] Los planes de ejecución ANTES de crear índices muestran `TABLE ACCESS FULL` en `ventas_lab`
- [ ] Los planes de ejecución DESPUÉS de crear índices muestran `INDEX RANGE SCAN` o `INDEX SKIP SCAN`
- [ ] El reporte SQL Tuning Advisor en `/home/oracle/lab06/sql_tuning_reporte.txt` contiene recomendaciones de índices
- [ ] Se crearon al menos 8 índices en el esquema `PRACTICA_USER` con estado `VALID`
- [ ] Los parámetros de memoria `SGA_TARGET` y `PGA_AGGREGATE_TARGET` fueron modificados sin errores
- [ ] El reporte final consolidado fue generado en `/home/oracle/lab06/reporte_final_lab06.txt`

### Procedimiento de Pruebas

1. Verifica que las tablas tienen los datos correctos:

   ```sql
   -- Conectarse como practica_user
   -- sqlplus practica_user/Oracle123
   SELECT 'ventas_lab' AS tabla, COUNT(*) AS registros FROM ventas_lab
   UNION ALL
   SELECT 'clientes_lab', COUNT(*) FROM clientes_lab
   UNION ALL
   SELECT 'productos_lab', COUNT(*) FROM productos_lab;
   ```

   **Resultado Esperado:**
   ```
   TABLA           REGISTROS
   --------------- ---------
   ventas_lab        100,000
   clientes_lab        5,000
   productos_lab       1,000
   ```

2. Verifica que los índices existen y están válidos:

   ```sql
   -- Conectarse como sysdba
   -- sqlplus / as sysdba
   SELECT COUNT(*) AS total_indices
   FROM dba_indexes
   WHERE owner = 'PRACTICA_USER'
     AND status = 'VALID';
   ```

   **Resultado Esperado:**
   ```
   TOTAL_INDICES
   -------------
              10
   ```

3. Confirma que los snapshots AWR fueron creados:

   ```sql
   SELECT COUNT(*) AS snapshots_hoy
   FROM dba_hist_snapshot
   WHERE TRUNC(end_interval_time) = TRUNC(SYSDATE);
   ```

   **Resultado Esperado:**
   ```
   SNAPSHOTS_HOY
   -------------
               2  (o más)
   ```

4. Verifica que la tarea ADDM fue ejecutada exitosamente:

   ```sql
   SELECT task_name, status, advisor_name
   FROM dba_advisor_tasks
   WHERE task_name = 'ADDM_LAB06_ANALISIS';
   ```

   **Resultado Esperado:**
   ```
   TASK_NAME              STATUS     ADVISOR_NAME
   ---------------------- ---------- ------------
   ADDM_LAB06_ANALISIS    COMPLETED  ADDM
   ```

5. Verifica la mejora en el plan de ejecución:

   ```sql
   -- Como practica_user: verificar que la consulta usa índice
   EXPLAIN PLAN FOR
   SELECT COUNT(*), SUM(monto_total)
   FROM ventas_lab
   WHERE TO_CHAR(fecha_venta, 'YYYY') = TO_CHAR(SYSDATE, 'YYYY')
     AND estado = 'Completada';

   SELECT operation, options, object_name, cost
   FROM plan_table
   WHERE statement_id IS NULL
   ORDER BY id;
   ```

   **Resultado Esperado:**
   ```
   OPERATION              OPTIONS     OBJECT_NAME          COST
   ---------------------- ----------- -------------------- ----
   SELECT STATEMENT                                         234
   SORT               AGGREGATE                              
   INDEX              RANGE SCAN  IDX_VENTAS_ANIO          234
   ```

---

## Solución de Problemas

### Problema 1: Error ORA-13516 al ejecutar ADDM (Diagnostics Pack no habilitado)

**Síntomas:**
- Al ejecutar `DBMS_ADVISOR.CREATE_TASK` con `advisor_name => 'ADDM'` aparece error `ORA-13516: AWR Operation failed`
- O aparece mensaje: `ORA-00439: feature not enabled: Diagnostics Pack`

**Causa:**
El parámetro `CONTROL_MANAGEMENT_PACK_ACCESS` no está configurado para habilitar el Diagnostics Pack, que es requerido para AWR y ADDM en Oracle Enterprise Edition.

**Solución:**

```sql
-- Conectarse como sysdba
sqlplus / as sysdba

-- Verificar el valor actual
SELECT name, value FROM v$parameter WHERE name = 'control_management_pack_access';

-- Habilitar Diagnostics + Tuning Pack
ALTER SYSTEM SET control_management_pack_access = 'DIAGNOSTIC+TUNING' SCOPE=BOTH;

-- Verificar que el cambio fue aplicado
SELECT name, value FROM v$parameter WHERE name = 'control_management_pack_access';
```

---

### Problema 2: Error ORA-04031 al configurar SGA_TARGET (memoria insuficiente)

**Síntomas:**
- Al ejecutar `ALTER SYSTEM SET sga_target = 1536M` aparece error `ORA-04031: unable to allocate X bytes of shared memory`
- O aparece `ORA-02097: parameter cannot be modified because specified value is invalid`

**Causa:**
El valor de `SGA_TARGET` especificado excede `SGA_MAX_SIZE` o la memoria física disponible en la VM.

**Solución:**

```sql
-- Verificar la memoria disponible y el límite máximo de SGA
sqlplus / as sysdba

SHOW PARAMETER sga_max_size;
SHOW PARAMETER sga_target;

-- Verificar cuánta memoria tiene la instancia actualmente
SELECT ROUND(SUM(bytes)/1024/1024, 0) AS sga_actual_mb FROM v$sga;

-- Ajustar a un valor más conservador (50% de la RAM disponible)
-- Si la VM tiene 8 GB de RAM, usar máximo 3 GB para SGA
ALTER SYSTEM SET sga_target = 768M SCOPE=BOTH;
ALTER SYSTEM SET pga_aggregate_target = 256M SCOPE=BOTH;

-- Verificar
SHOW PARAMETER sga_target;
```

---

### Problema 3: Los índices creados no son utilizados por el optimizador

**Síntomas:**
- Después de crear los índices, el plan de ejecución sigue mostrando `TABLE ACCESS FULL`
- `EXPLAIN PLAN` no muestra `INDEX RANGE SCAN` para las consultas optimizadas

**Causa:**
Las estadísticas de las tablas no fueron actualizadas después de crear los índices, o el optimizador considera que el full scan es más eficiente dado el volumen de datos y el factor de selectividad.

**Solución:**

```sql
-- Conectarse como practica_user
sqlplus practica_user/Oracle123

-- Recopilar estadísticas de todas las tablas con CASCADE para incluir índices
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'VENTAS_LAB', CASCADE => TRUE, DEGREE => 4);
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'CLIENTES_LAB', CASCADE => TRUE);
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'PRODUCTOS_LAB', CASCADE => TRUE);

-- Si el problema persiste, verificar la selectividad del índice
SELECT index_name, num_rows, distinct_keys,
       ROUND(num_rows/NULLIF(distinct_keys,0), 2) AS clustering_factor
FROM dba_indexes
WHERE owner = USER
ORDER BY index_name;

-- Forzar el uso del índice con un hint para verificar que funciona
EXPLAIN PLAN FOR
SELECT /*+ INDEX(v IDX_VENTAS_ESTADO_FECHA) */ COUNT(*), SUM(monto_total)
FROM ventas_lab v
WHERE estado = 'Completada'
  AND fecha_venta > SYSDATE - 365;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

---

### Problema 4: Error al generar el reporte AWR (snap_id inválido)

**Síntomas:**
- Al ejecutar `DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_TEXT` aparece error `ORA-13516` o el reporte está vacío
- La función retorna 0 filas

**Causa:**
Los `snap_id` proporcionados no son válidos, no pertenecen al `DBID` correcto, o el intervalo entre snapshots es demasiado corto.

**Solución:**

```sql
-- Verificar los snapshots disponibles
sqlplus / as sysdba

SELECT snap_id, dbid, instance_number,
       TO_CHAR(begin_interval_time, 'DD-MON HH24:MI') AS inicio,
       TO_CHAR(end_interval_time, 'DD-MON HH24:MI') AS fin
FROM dba_hist_snapshot
ORDER BY snap_id DESC
FETCH FIRST 10 ROWS ONLY;

-- Obtener el DBID correcto
SELECT dbid FROM v$database;

-- Generar el reporte con los valores correctos
-- Sustituir SNAP_INI y SNAP_FIN con los IDs obtenidos arriba
SELECT output FROM TABLE(
    DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_TEXT(
        (SELECT dbid FROM v$database),
        (SELECT instance_number FROM v$instance),
        &SNAP_INI,
        &SNAP_FIN
    )
);
```

---

### Problema 5: Error ORA-01031 al ejecutar DBMS_SQLTUNE (privilegios insuficientes)

**Síntomas:**
- Al ejecutar `DBMS_SQLTUNE.CREATE_TUNING_TASK` como `practica_user` aparece `ORA-01031: insufficient privileges`
- O `ORA-00942: table or view does not exist` al consultar `DBA_ADVISOR_TASKS`

**Causa:**
El usuario `practica_user` no tiene el privilegio `ADVISOR` o no tiene `EXECUTE` en `DBMS_SQLTUNE`.

**Solución:**

```sql
-- Conectarse como sysdba y otorgar los privilegios necesarios
sqlplus / as sysdba

GRANT ADVISOR TO practica_user;
GRANT EXECUTE ON DBMS_SQLTUNE TO practica_user;
GRANT SELECT ON DBA_ADVISOR_TASKS TO practica_user;
GRANT SELECT ON DBA_ADVISOR_FINDINGS TO practica_user;
GRANT SELECT ON DBA_ADVISOR_RECOMMENDATIONS TO practica_user;

-- Verificar que los privilegios fueron otorgados
SELECT privilege FROM dba_sys_privs WHERE grantee = 'PRACTICA_USER'
UNION ALL
SELECT privilege FROM dba_tab_privs WHERE grantee = 'PRACTICA_USER' AND owner = 'SYS';
```

---

## Limpieza

Ejecuta los siguientes comandos para limpiar los objetos creados durante el laboratorio. Realiza la limpieza al final del curso o cuando el espacio en disco sea limitado.

```bash
sqlplus / as sysdba << 'EOF'
SET SERVEROUTPUT ON
SET ECHO ON

PROMPT ============================================================
PROMPT LIMPIEZA DEL LABORATORIO 06-00-01
PROMPT ============================================================

-- Eliminar tareas de AWR y Advisor creadas
BEGIN
    -- Eliminar tarea ADDM
    BEGIN
        DBMS_ADVISOR.DELETE_TASK('ADDM_LAB06_ANALISIS');
        DBMS_OUTPUT.PUT_LINE('Tarea ADDM eliminada.');
    EXCEPTION WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Tarea ADDM no encontrada (ya eliminada).');
    END;

    -- Eliminar tareas SQL Tuning Advisor
    BEGIN
        DBMS_SQLTUNE.DROP_TUNING_TASK('STA_VENTAS_LAB_01');
        DBMS_OUTPUT.PUT_LINE('Tarea STA_VENTAS_LAB_01 eliminada.');
    EXCEPTION WHEN OTHERS THEN NULL;
    END;

    BEGIN
        DBMS_SQLTUNE.DROP_TUNING_TASK('STA_JOIN_LAB_02');
        DBMS_OUTPUT.PUT_LINE('Tarea STA_JOIN_LAB_02 eliminada.');
    EXCEPTION WHEN OTHERS THEN NULL;
    END;
END;
/

-- Eliminar objetos del esquema practica_user
DROP TABLE practica_user.ventas_lab PURGE;
DROP TABLE practica_user.clientes_lab PURGE;
DROP TABLE practica_user.productos_lab PURGE;
DROP SEQUENCE practica_user.seq_cliente;
DROP SEQUENCE practica_user.seq_producto;
DROP SEQUENCE practica_user.seq_venta;

-- Limpiar entradas de plan_table de practica_user
DELETE FROM practica_user.plan_table;
COMMIT;

DBMS_OUTPUT.PUT_LINE('Objetos de practica_user eliminados.');

-- Opcional: Eliminar snapshots AWR del laboratorio
-- ADVERTENCIA: Solo eliminar si no se necesitan para análisis posterior
-- EXEC DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE(low_snap_id => &SNAP_INI, high_snap_id => &SNAP_FIN);

PROMPT Limpieza completada.
EXIT;
EOF
```

```bash
# Limpiar archivos de reporte del sistema operativo
# ADVERTENCIA: Esto elimina todos los reportes generados
# Guardar copias antes si son necesarias para entregables

echo "Archivos en /home/oracle/lab06/:"
ls -la /home/oracle/lab06/

read -p "¿Desea eliminar los archivos de reporte? (s/N): " confirm
if [ "$confirm" = "s" ] || [ "$confirm" = "S" ]; then
    rm -rf /home/oracle/lab06/
    echo "Directorio /home/oracle/lab06/ eliminado."
else
    echo "Archivos conservados en /home/oracle/lab06/"
fi
```

> ⚠️ **Advertencia:** No revertir los cambios de configuración de memoria (`SGA_TARGET`, `PGA_AGGREGATE_TARGET`) si los nuevos valores están funcionando correctamente y mejorando el rendimiento. Los cambios de memoria con `SCOPE=SPFILE` requieren reinicio para revertirse. Si necesitas restaurar los valores originales, usa `ALTER SYSTEM SET sga_target = [valor_original] SCOPE=SPFILE;` y reinicia la instancia con `SHUTDOWN IMMEDIATE; STARTUP;`.

> ⚠️ **Advertencia:** Los snapshots AWR ocupan espacio en el tablespace `SYSAUX`. Si el laboratorio generó muchos snapshots, considera limpiarlos con `DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE` para liberar espacio.

---

## Resumen

### Lo que Lograste

- **Estableciste una línea base de rendimiento** consultando vistas dinámicas `V$SESSION`, `V$WAIT_CLASS`, `V$SYSSTAT`, `V$BUFFER_POOL_STATISTICS` y `V$SGASTAT`, aplicando la etapa OBSERVAR del ciclo de monitoreo proactivo
- **Generaste y analizaste reportes AWR y ADDM** tomando snapshots manuales, produciendo reportes en formato texto e interpretando los hallazgos y recomendaciones automáticas de ADDM para identificar cuellos de botella
- **Identificaste sentencias SQL problemáticas** desde `V$SQL` y generaste sus planes de ejecución con `EXPLAIN PLAN`, `DBMS_XPLAN.DISPLAY` y `DBMS_XPLAN.DISPLAY_CURSOR`, confirmando la presencia de full table scans ineficientes
- **Utilizaste SQL Tuning Advisor** (`DBMS_SQLTUNE`) para obtener recomendaciones automáticas de optimización e implementaste esas recomendaciones creando índices B-Tree, compuestos y basados en funciones
- **Configuraste la gestión automática de memoria** ajustando `SGA_TARGET` para ASMM y `MEMORY_TARGET` para AMM, y consultaste el Memory Advisor para determinar los valores óptimos
- **Aplicaste el ciclo completo de monitoreo** (Observar → Analizar → Actuar → Documentar) en un escenario integrador de diagnóstico, generando documentación reproducible del proceso

### Conceptos Clave Aprendidos

- El **monitoreo proactivo** requiere una línea base (*baseline*) preestablecida; sin ella, no es posible distinguir comportamiento normal de anormal
- Las **vistas `V$`** son la fuente de verdad en tiempo real del estado de Oracle; `V$ACTIVE_SESSION_HISTORY` es especialmente valiosa para análisis histórico de esperas
- El **ciclo AWR → ADDM** automatiza gran parte del diagnóstico: AWR captura los datos, ADDM los analiza y genera recomendaciones priorizadas por impacto
- Un **full table scan** no siempre es malo; el optimizador lo elige cuando es más eficiente que un index scan. El contexto (selectividad, volumen, patrón de acceso) determina la estrategia óptima
- Los **índices basados en funciones** son esenciales cuando las consultas aplican funciones (`TO_CHAR`, `UPPER`, `EXTRACT`) sobre columnas en cláusulas `WHERE`
- **AMM** (`MEMORY_TARGET`) simplifica la administración al gestionar SGA y PGA juntos, pero requiere que `/dev/shm` tenga suficiente espacio en Linux
- La etapa **DOCUMENTAR** del ciclo de monitoreo es tan importante como las demás: construye el conocimiento institucional que acelera la resolución de incidentes futuros

### Próximos Pasos

- Explorar **Oracle Enterprise Manager Database Express** para visualizar gráficamente los KPIs estudiados en este laboratorio (acceder en `https://[IP_VM]:5500/em`)
- Profundizar en **Active Session History (ASH)** mediante `V$ACTIVE_SESSION_HISTORY` y `DBA_HIST_ACTIVE_SESS_HISTORY` para análisis de rendimiento histórico más granular
- Investigar **SQL Plan Baselines** (`DBMS_SPM`) para estabilizar los planes de ejecución de consultas críticas y evitar regresiones de rendimiento
- Practicar la **gestión de Undo y Temp** monitoreando `V$UNDOSTAT` y `V$SORT_SEGMENT` para completar la visión integral del rendimiento de almacenamiento

---

## Recursos Adicionales

- **Oracle Database 19c Performance Tuning Guide** – Documentación oficial completa sobre metodologías de diagnóstico, vistas `V$`, AWR, ADDM y afinación de SQL. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/)
- **Oracle Database 19c SQL Tuning Guide** – Referencia específica para `EXPLAIN PLAN`, `DBMS_XPLAN`, SQL Tuning Advisor y SQL Plan Management. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- **Oracle Database 19c Reference (V$ Views)** – Descripción completa de todas las vistas de rendimiento dinámico usadas en este laboratorio. Disponible en [docs.oracle.com/en/database/oracle/oracle-database/19/refrn/](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/)
- **"Oracle Database 12c Performance Tuning Recipes" – Sam Alapati** – Libro de referencia práctica con recetas aplicables directamente a Oracle 19c para diagnóstico y optimización
- **My Oracle Support (MOS)** – Base de conocimiento de Oracle con notas técnicas sobre problemas específicos de rendimiento. Accesible en [support.oracle.com](https://support.oracle.com) con cuenta Oracle
- **Oracle Live SQL** – Entorno gratuito en línea para practicar SQL Oracle sin instalación local. Disponible en [livesql.oracle.com](https://livesql.oracle.com)
