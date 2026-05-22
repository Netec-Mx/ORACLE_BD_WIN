## Práctica 7. Tipos de datos LOB y su Manejo en Oracle

## Metadatos

| Propiedad | Valor |
|-----------|-------|
| **Duración** | 90 minutos |
| **Complejidad** | Intermedio |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 7 — Large Objects (LOBs) |
| **Versión Oracle** | 19c (19.3+) |

---

## Descripción General

En este laboratorio explorarás de forma práctica los cuatro tipos de datos LOB disponibles en Oracle Database: `BLOB`, `CLOB`, `NCLOB` y `BFILE`. Crearás tablas con columnas LOB configurando parámetros de almacenamiento avanzados, insertarás y recuperarás datos utilizando el paquete `DBMS_LOB`, y evaluarás las diferencias entre el almacenamiento BasicFiles y SecureFiles. Al finalizar, habrás construido un sistema de gestión documental simplificado que demuestra el uso real de LOBs en aplicaciones empresariales, comprendiendo cómo Oracle organiza internamente estos objetos de gran tamaño dentro de sus tablespaces.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Crear tablas Oracle con columnas de tipo `BLOB`, `CLOB` y `NCLOB` especificando cláusulas `LOB STORE AS` con parámetros de almacenamiento personalizados (`CHUNK`, `CACHE`, `STORAGE IN ROW`).
- [ ] Insertar, actualizar y recuperar datos LOB utilizando operaciones SQL estándar y el paquete `DBMS_LOB` (`WRITE`, `READ`, `APPEND`, `SUBSTR`, `GETLENGTH`).
- [ ] Implementar operaciones avanzadas sobre CLOBs incluyendo búsqueda de contenido con `DBMS_LOB.INSTR` y lectura parcial con `DBMS_LOB.SUBSTR`.
- [ ] Configurar objetos `DIRECTORY` de Oracle y utilizar `BFILENAME` con `DBMS_LOB.LOADFROMFILE` para cargar contenido desde el sistema de archivos.
- [ ] Comparar las características de almacenamiento BasicFiles vs SecureFiles y crear LOBs con deduplicación y compresión habilitadas.
- [ ] Consultar vistas del diccionario de datos para monitorear segmentos LOB y su consumo de espacio en tablespaces.

---

## Prerrequisitos

### Conocimiento Requerido

- Conocimiento de PL/SQL básico: bloques anónimos, variables, cursores y manejo de excepciones con `EXCEPTION WHEN`.
- Comprensión de tipos de datos Oracle estándar (`VARCHAR2`, `NUMBER`, `DATE`) y DDL para creación y modificación de tablas.
- Familiaridad con el concepto de tablespaces, segmentos y extents en Oracle Database.
- Conocimiento básico de cómo invocar procedimientos y funciones de un paquete PL/SQL (notación `PAQUETE.PROCEDIMIENTO`).
- Haber completado los laboratorios de los módulos anteriores (01 al 06) con la instancia Oracle operativa.

### Acceso Requerido

- Conexión a Oracle Database 19c como usuario `PRACTICA_USER` con privilegios `CREATE TABLE`, `CREATE DIRECTORY` (o que el DBA cree el directorio), `UNLIMITED TABLESPACE` o cuota en el tablespace de práctica.
- Acceso como `SYSDBA` (usuario `SYS`) para crear tablespaces dedicados y objetos `DIRECTORY`.
- Acceso RDP o SSH a la máquina virtual Oracle Windows donde reside la base de datos.
- SQL*Plus o SQL Developer disponible para ejecutar los scripts del laboratorio.

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Especificación |
|------------|----------------|
| CPU | Intel Core i5 8va gen o AMD Ryzen 5 (mínimo) |
| RAM | 16 GB mínimo (recomendado 32 GB) |
| Almacenamiento | 100 GB libres en disco (SSD recomendado) |
| Red | Adaptador de red funcional con loopback |

### Requisitos de Software

| Software | Versión | Propósito |
|----------|---------|-----------|
| Oracle Database | 19c (19.3+) | Motor de base de datos principal |
| SQL*Plus | Incluido con 19c | Ejecución de scripts SQL y PL/SQL |
| SQL Developer | 23.1+ | Visualización de contenido LOB y resultados |
| Oracle Windows | 11.x o Server 19+ | Sistema operativo huésped |
| PuTTY / Terminal SSH | 0.79+ / nativo | Acceso remoto a la VM |

### Configuración Inicial

Antes de comenzar el laboratorio, verifica que la instancia Oracle esté activa y establece las variables de entorno necesarias:

```bash
# Conectarse a la VM Oracle Windows via RDP o SSH
# Verificar que el listener esté activo y muestre el servicio `orcl`.
lsnrctl status

# Establece el ambiente de variables ORACLE llamando al programa `ambiente.bat`
set | findstr ORA
 
# O en su caso establece las variables de entorno Oracle
export ORACLE_BASE=C:\app\oracle
export ORACLE_HOME=C:\app\oracle\product\19.3.0\dbhome_1
export ORACLE_SID=ORCL
export PATH=$ORACLE_HOME\bin:$PATH

# Verificar conectividad a la base de datos
sqlplus  / as sysdba
SELECT instance_name, status FROM v$instance;
EXIT;
```

  Crear directorio de archivos LOB
mkdir C:\oracle\lob_files
  Crear archivos de prueba

echo CONTRATO DE SERVICIOS PROFESIONALES > C:\oracle\lob_files\contrato_001.txt
echo Empresa ABC >> C:\oracle\lob_files\contrato_001.txt
echo Consultora XYZ >> C:\oracle\lob_files\contrato_001.txt

echo MANUAL TECNICO > C:\oracle\lob_files\manual_tecnico.txt
echo Oracle Database 19c >> C:\oracle\lob_files\manual_tecnico.txt
dir C:\oracle\lob_files
________________________________________
Paso 1: Crear Tablespace y DIRECTORY
sqlplus / as sysdba
CREATE TABLESPACE lob_data_ts
DATAFILE 'C:\app\oracle\oradata\ORCL\lob_data01.dbf'
SIZE 200M
AUTOEXTEND ON NEXT 50M MAXSIZE 2G
EXTENT MANAGEMENT LOCAL
SEGMENT SPACE MANAGEMENT AUTO;
CREATE OR REPLACE DIRECTORY dir_lob_files AS 'C:\oracle\lob_files';

GRANT READ, WRITE ON DIRECTORY dir_lob_files TO practica_user;
ALTER USER practica_user QUOTA UNLIMITED ON lob_data_ts;
GRANT CREATE ANY DIRECTORY TO practica_user;
________________________________________
Paso 2: Crear Tablas con LOB
CONNECT practica_user/Oracle123@ORCL
CREATE TABLE lab07_contratos (
    id_contrato NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    numero_contrato VARCHAR2(20),
    cliente VARCHAR2(200),
    texto_contrato CLOB,
    notas_internas NCLOB
)
LOB (texto_contrato) STORE AS BASICFILE (
    TABLESPACE lob_data_ts
    ENABLE STORAGE IN ROW
)
LOB (notas_internas) STORE AS BASICFILE (
    TABLESPACE lob_data_ts
    DISABLE STORAGE IN ROW
);
CREATE TABLE lab07_multimedia (
    id_archivo NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre_archivo VARCHAR2(255),
    contenido_blob BLOB
)
LOB (contenido_blob) STORE AS SECUREFILE (
    TABLESPACE lob_data_ts
);
CREATE TABLE lab07_manuales_ext (
    id_manual NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    codigo_manual VARCHAR2(20),
    archivo_externo BFILE
);
________________________________________
Paso 3: Insertar CLOB
INSERT INTO lab07_contratos (
    numero_contrato,
    cliente,
    texto_contrato,
    notas_internas
) VALUES (
    'CONT-001',
    'Empresa Alpha',
    TO_CLOB('Contrato de servicios Oracle'),
    TO_NCLOB('Notas internas')
);

COMMIT;
________________________________________
DECLARE
    v_clob CLOB;
BEGIN
    DBMS_LOB.CREATETEMPORARY(v_clob, TRUE);

    DBMS_LOB.WRITEAPPEND(v_clob, 30, 'Contrato generado con DBMS_LOB');

    INSERT INTO lab07_contratos (
        numero_contrato,
        cliente,
        texto_contrato
    ) VALUES (
        'CONT-002',
        'Empresa Beta',
        v_clob
    );

    COMMIT;
END;
/
________________________________________
Paso 4: Lectura y Búsqueda CLOB
SELECT
    numero_contrato,
    DBMS_LOB.SUBSTR(texto_contrato, 100, 1)
FROM lab07_contratos;
________________________________________
SELECT
    numero_contrato,
    DBMS_LOB.INSTR(texto_contrato, 'Contrato', 1, 1)
FROM lab07_contratos;
________________________________________
SET SERVEROUTPUT ON;

DECLARE
    v_clob    CLOB;
    v_buffer  VARCHAR2(200);
    v_amount  BINARY_INTEGER := 50; -- Declaramos e inicializamos la variable aquí
BEGIN
    SELECT texto_contrato INTO v_clob
    FROM lab07_contratos
    WHERE numero_contrato = 'CONT-002';

    -- Ahora le pasamos la variable v_amount en lugar del número estático
    DBMS_LOB.READ(v_clob, v_amount, 1, v_buffer);

    DBMS_OUTPUT.PUT_LINE('--- Contenido leído (' || v_amount || ' caracteres) ---');
    DBMS_OUTPUT.PUT_LINE(v_buffer);
END;
/
________________________________________
SET SERVEROUTPUT ON;

DECLARE
    v_clob   CLOB;
    v_extra  CLOB;
    v_texto  VARCHAR2(20) := ' ADENDA';
BEGIN
    -- 1. Bloqueamos la fila para actualización
    SELECT texto_contrato INTO v_clob
    FROM lab07_contratos
    WHERE numero_contrato = 'CONT-001'
    FOR UPDATE;

    -- 2. Creamos el LOB temporal
    DBMS_LOB.CREATETEMPORARY(v_extra, TRUE);

    -- 3. Corregido: Usamos LENGTH para pasar el tamaño exacto (7)
    DBMS_LOB.WRITEAPPEND(v_extra, LENGTH(v_texto), v_texto);

    -- 4. Anexamos el LOB temporal al CLOB real de la tabla
    DBMS_LOB.APPEND(v_clob, v_extra);

    -- 5. Liberamos el LOB temporal (Buena práctica para no dejar basura en TEMP)
    DBMS_LOB.FREETEMPORARY(v_extra);

    -- 6. Consolidamos cambios y liberamos el bloqueo del FOR UPDATE
    COMMIT;
    
    DBMS_OUTPUT.PUT_LINE('Adenda anexada exitosamente.');
END;
/
________________________________________
Paso 5: BLOB desde archivos
echo PNG_SIMULADO > C:\oracle\lob_files\imagen_producto_001.png
echo PDF_SIMULADO > C:\oracle\lob_files\manual_tecnico_v2.pdf
________________________________________
SET SERVEROUTPUT ON;

DECLARE
    v_blob       BLOB;
    v_bfile      BFILE;
    
    -- Variables necesarias que hacían falta:
    v_dest_off   INTEGER := 1; -- Posición inicial en el BLOB destino
    v_src_off    INTEGER := 1; -- Posición inicial en el BFILE origen
    v_amount     INTEGER;      -- Almacenará el tamaño total a cargar
BEGIN
    -- 1. Insertamos y recuperamos el localizador del BLOB vacío
    INSERT INTO lab07_multimedia (
        nombre_archivo,
        contenido_blob
    ) VALUES (
        'imagen_producto_001.png',
        EMPTY_BLOB()
    )
    RETURNING contenido_blob INTO v_blob;

    -- 2. Inicializamos el puntero al archivo físico en el Sistema Operativo
    v_bfile := BFILENAME('DIR_LOB_FILES', 'imagen_producto_001.png');

    -- 3. Abrimos el archivo origen en modo solo lectura
    DBMS_LOB.OPEN(v_bfile, DBMS_LOB.LOB_READONLY);

    -- 4. Guardamos el tamaño exacto del archivo en nuestra variable v_amount
    v_amount := DBMS_LOB.GETLENGTH(v_bfile);

    -- 5. Corregido: Llamamos al procedimiento pasando TODOS los parámetros requeridos
    DBMS_LOB.LOADBLOBFROMFILE(
        dest_lob    => v_blob,
        src_bfile   => v_bfile,
        amount      => v_amount,
        dest_offset => v_dest_off,
        src_offset  => v_src_off
    );

    -- 6. Cerramos el archivo para liberar recursos del Sistema Operativo
    DBMS_LOB.CLOSE(v_bfile);

    -- 7. Consolidamos los cambios de manera definitiva
    COMMIT;
    
    DBMS_OUTPUT.PUT_LINE('Archivo cargado exitosamente en el BLOB.');
EXCEPTION
    WHEN OTHERS THEN
        -- Asegurar el cierre del archivo si algo falla a mitad del proceso
        IF DBMS_LOB.ISOPEN(v_bfile) = 1 THEN
            DBMS_LOB.CLOSE(v_bfile);
        END IF;
        RAISE;
END;
/
________________________________________
INSERT INTO lab07_manuales_ext (
    codigo_manual,
    archivo_externo
) VALUES (
    'MAN-001',
    BFILENAME('DIR_LOB_FILES', 'manual_tecnico.txt')
);

COMMIT;
________________________________________
SELECT
    codigo_manual,
    DBMS_LOB.FILEEXISTS(archivo_externo) AS existe
FROM lab07_manuales_ext;
________________________________________
SET SERVEROUTPUT ON;

DECLARE
    v_bfile   BFILE;
    v_buffer  RAW(200);
    v_amount  BINARY_INTEGER := 100; -- Declaramos e inicializamos la variable aquí
BEGIN
    SELECT archivo_externo INTO v_bfile
    FROM lab07_manuales_ext
    -- Nota: Asegúrate de que esta tabla solo devuelva una fila o añade un WHERE 
    -- para evitar un ORA-01422 (exactamente una fila por SELECT INTO).
    FETCH FIRST 1 ROWS ONLY; 

    -- 1. Abrimos el archivo externo en modo solo lectura
    DBMS_LOB.OPEN(v_bfile, DBMS_LOB.LOB_READONLY);

    -- 2. Corregido: Pasamos la variable v_amount en lugar del número directo
    DBMS_LOB.READ(v_bfile, v_amount, 1, v_buffer);

    -- 3. Cerramos el archivo para liberar el descriptor en el sistema operativo
    DBMS_LOB.CLOSE(v_bfile);

    -- 4. Convertimos los bytes crudos a texto plano e imprimimos
    DBMS_OUTPUT.PUT_LINE('--- Datos leídos del archivo externo ---');
    DBMS_OUTPUT.PUT_LINE(UTL_RAW.CAST_TO_VARCHAR2(v_buffer));
EXCEPTION
    WHEN OTHERS THEN
        -- Garantizar el cierre seguro del archivo ante cualquier error intermedio
        IF DBMS_LOB.ISOPEN(v_bfile) = 1 THEN
            DBMS_LOB.CLOSE(v_bfile);
        END IF;
        RAISE;
END;
/
________________________________________
Paso 6: Validación final
SELECT COUNT(*) FROM lab07_contratos;
SELECT COUNT(*) FROM lab07_multimedia;
SELECT COUNT(*) FROM lab07_manuales_ext;
________________________________________
SELECT
    table_name,
    column_name,
    securefile
FROM user_lobs;
 
________________________________________
Resultado esperado
•	Inserciones exitosas con IDs generados 
•	Búsquedas retornan coincidencias reales 
•	Extractos muestran contexto alrededor del término 
________________________________________
Lo importante que integra este paso
•	Uso combinado de CLOB + BLOB + BFILE 
•	Diferenciación SecureFiles vs BasicFiles en un mismo modelo 
•	Manipulación real con DBMS_LOB 
•	Encapsulación lógica en procedimientos y funciones 
•	Búsqueda dentro de documentos (base para motores tipo ECM/DMS) 



## Recursos Adicionales

- **Oracle Database SecureFiles and Large Objects Developer's Guide 19c** — Documentación oficial completa sobre todos los tipos LOB, arquitectura de almacenamiento, API DBMS_LOB y mejores prácticas. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/adlob/
- **Oracle Database PL/SQL Packages and Types Reference 19c — DBMS_LOB** — Referencia completa de todos los procedimientos y funciones del paquete `DBMS_LOB` con ejemplos detallados. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_LOB.html
- **Oracle Database SQL Language Reference 19c — LOB Data Types** — Sección dedicada a los tipos de datos LOB en SQL, incluyendo DDL para `LOB STORE AS`, restricciones y comportamiento en DML. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/
- **Oracle Database Administrator's Guide 19c — Managing LOBs** — Guía para administradores sobre gestión de segmentos LOB, monitoreo de espacio y estrategias de mantenimiento. Disponible en: https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/
- **Vista USER_LOBS** — Consulta `SELECT * FROM user_lobs` o `SELECT * FROM dba_lobs` para inspeccionar todas las propiedades de los segmentos LOB en tu esquema o en toda la base de datos, incluyendo `SECUREFILE`, `COMPRESS`, `DEDUPLICATE`, `ENCRYPT`, `CACHE`, `LOGGING` y `IN_ROW`.
