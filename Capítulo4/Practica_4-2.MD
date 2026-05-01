# Práctica 4.2. Tecnologías Flashback

<br/><br/>

## Objetivos

Al finalizar esta práctica, serás capaz de **utilizar las tecnologías Flashback en Oracle 19c para recuperar datos ante errores lógicos y revertir una base de datos completa a un punto anterior garantizado**, aplicando los mecanismos **Flashback Query** y **Flashback Database**.

<br/> <br/>

## Tiempo estimado

- 60 minutos

<br/><br/>

## Tabla de ayuda

| Concepto / Comando                        | Descripción                                                                                                         |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Flashback Query**                       | Permite consultar el estado anterior de los datos en una tabla a partir de una marca de tiempo (`AS OF TIMESTAMP`). |
| **Flashback Versions Query (VERSIONS BETWEEN)**                       | Permite consultar el estado de un reglon en un periodo de tiempo (`AS OF TIMESTAMP`). |
| **Flashback Drop**                       | Permite recuperar una tabla eliminada desde el RECYCLEBIN. |
| **Flashback Table**                       | Permite regresar una tabla a un estado anterior en el tiempo. |
| **Flashback Transaction Query / Transaction**                       | Permite deshacer una transacción pasada. |
| **Flashback Database**                    | Restaura toda la base de datos a un punto anterior garantizado mediante *restore points*.                           |
| **Restore Point**                         | Punto de restauración en el tiempo al que puede volver la base de datos usando Flashback.                           |
| **FRA (Fast Recovery Area)**              | Área donde se almacenan los archivos de recuperación, redo logs y flashback logs.                                   |
| **`AS OF TIMESTAMP`**                     | Cláusula SQL que permite ver el estado pasado de los datos según una marca de tiempo.                               |
| **`ALTER DATABASE FLASHBACK ON`**         | Habilita el registro de Flashback Logs en la FRA.                                                                   |
| **`FLASHBACK DATABASE TO RESTORE POINT`** | Comando RMAN para devolver la base de datos a un punto de restauración garantizado.                                 |
| **`RESETLOGS`**                           | Reabre la base de datos tras la operación de Flashback o recuperación.                                              |
| **incarnation**   | Una incarnation representa una versión lógica de la base de datos dentro del historial de RMAN.  |


<br/> <br/>

## Instrucciones

### Tarea 1. Crear una nueva pluggable database.

#### **Paso 1.** Crea la pluggable database PDB_VENTAS

1. Abre una sesión en **SQL*Plus** o **SQL Developer**.
<br/>

2.  Ejecuta el siguiente script para crear la base de datos pluggable:

   ```sql
   CREATE PLUGGABLE DATABASE PDB_VENTAS 
   ADMIN USER pdbadmin IDENTIFIED BY oracle_4U
   FILE_NAME_CONVERT = ('/pdbseed/', '/pdb_ventas/')
   DEFAULT TABLESPACE users DATAFILE '/u01/app/oracle/oradata/ORCL/pdb_ventas/users01.dbf' SIZE 250M;
   -- AUTOSAVE STATE;
   
   Verificación y Apertura:
   Asegúrate de que la nueva PDB esté abierta.

   ALTER PLUGGABLE DATABASE PDB_VENTAS OPEN;
   SELECT NAME, OPEN_MODE FROM V$PDBS;
   
   ```
<br/>

3. Crea el usuario FLASH_USER

   ```sql
   -- 1. Crear el usuario
   CREATE USER FLASH_USER IDENTIFIED BY oracle_4U DEFAULT TABLESPACE users;

   -- 2. Otorgar privilegios de conexión y acceso al diccionario
   GRANT CREATE SESSION, RESOURCE TO FLASH_USER;
   GRANT SELECT ANY DICTIONARY TO FLASH_USER;
   ALTER USER FLASH_USER QUOTA 500M ON USERS;

   -- 3. Otorgar el privilegio clave de Flashback a nivel de tabla
   GRANT FLASHBACK ANY TABLE TO FLASH_USER;

   ```


4. Ejecuta el siguiente script para crear una tabla:

   ```sql
   Asunción: Usaremos una tabla llamada VENTAS para la mayoría de los ejemplos.

   -- Script de preparación de la tabla
   CREATE TABLE VENTAS (
    ID NUMBER PRIMARY KEY,
    PRODUCTO VARCHAR2(50),
    CANTIDAD NUMBER,
    FECHA_VENTA TIMESTAMP
   );

   -- Carga los datos iniciales.

   INSERT INTO VENTAS VALUES (1, 'Laptop', 10, SYSTIMESTAMP);
   INSERT INTO VENTAS VALUES (2, 'Mouse', 50, SYSTIMESTAMP);
   COMMIT;

   -- Verifica los registros cargados.
   SELECT * FROM VENTAS;
   
   ```
<br/><br/>


### Tarea 2. Flashback Query (AS OF TIMESTAMP)
Nivel de Aplicación: PDB
 

#### **Paso 1.** Registra el tiempo actual (T0).

Antes de simular el error lógico, obtén la marca de tiempo actual.

```sql
SELECT SYSTIMESTAMP FROM DUAL;

```

**Anota la hora** obtenida (por ejemplo: `02-NOV-25 06.27.20.376933 PM -07:00`).

<br/>

#### **Paso 2.** Simula un error lógico.

Ejecuta la siguiente instrucción para modificar erróneamente los datos:

```sql

1. Simular un Error: 

    Ejecuta 
    
    UPDATE VENTAS SET CANTIDAD = 0; 
    COMMIT; 
    
```

Verifica el error.

```sql
SELECT * FROM VENTAS;
```

<br/>

#### **Paso 3.** Consulta los datos anteriores con Flashback Query.

Usa la marca de tiempo registrada en el paso 2 para visualizar los valores antes del error.

```sql

SELECT * FROM VENTAS 
AS OF TIMESTAMP 
    TO_TIMESTAMP_TZ('02-NOV-25 06.27.20.376933 PM -07:00', 'DD-MON-RR HH.MI.SS.FF6 PM TZR');
```

<br/>

#### **Paso 4.** Corrige los datos usando Flashback Query (recomendado).

```sql
UPDATE VENTAS v 
SET CANTIDAD = (
    SELECT CANTIDAD 
    FROM VENTAS AS OF TIMESTAMP TO_TIMESTAMP('02-NOV-25 06.27.20.376933 PM -07:00', 'DD-MON-RR HH.MI.SS.FF6 PM TZR') original 
    WHERE original.ID = v.ID
);
COMMIT;
```
<br/>

Verifica que los salarios hayan vuelto a sus valores originales.

```sql
SELECT * FROM VENTAS;
```

<br/><br/>

### Tarea 3. Flashback Versions Query (VERSIONS BETWEEN)
Nivel de Aplicación: PDB
 

#### **Paso 1.** Modificación 1

Ejecuta 
```sql
UPDATE VENTAS SET CANTIDAD = 15 WHERE ID = 1;
COMMIT
```

#### **Paso 2.** Modificación 2 
```sql
UPDATE VENTAS SET CANTIDAD = 20 WHERE ID = 1;
COMMIT;.
```
3.	Consultar Versiones: Ejecuta 
```sql
SELECT VERSIONS_STARTSCN, VERSIONS_ENDSCN, VERSIONS_XID, CANTIDAD 
FROM VENTAS 
VERSIONS BETWEEN TIMESTAMP MINVALUE AND MAXVALUE WHERE ID = 1;
```
<br/>

### Tarea 4. Flashback Drop
Nivel de Aplicación: PDB

#### **Paso 1.** Eliminar Tabla: Ejecuta 
```sql
-- Mientras estás conectado a PDB_VENTAS elimina la tabla.
    DROP TABLE VENTAS;
```

#### **Paso 2.** Ejecuta 
```sql
-- Muestra la basurera de reciclaje,para ver el nombre interno de la tabla eliminada.

    SHOW RECYCLEBIN; 
```

#### **Paso 3.** Recuperar Tabla 

```sql
FLASHBACK TABLE VENTAS TO BEFORE DROP;
SELECT * FROM VENTAS;
```

<br/><br/>

### Tarea 5. Flashback Table

Nivel de Aplicación: PDB

#### **Paso 1.** Prerrequisito: Conéctate a la PDB y ejecuta 
```sql
ALTER TABLE VENTAS ENABLE ROW MOVEMENT;.
```

#### **Paso 2.** Registrar T0: Ejecuta 
```sql
SELECT SYSTIMESTAMP FROM DUAL; y anota la hora (T0).
```

#### **Paso 3.** Simular Error: Ejecuta 
```sql
DELETE FROM VENTAS WHERE CANTIDAD < 50; seguido de COMMIT;
```

#### **Paso 4.** Ejecutar Flashback: Ejecuta 
```sql
FLASHBACK TABLE VENTAS TO TIMESTAMP TO_TIMESTAMP('T0', 'YYYY-MM-DD HH24:MI:SS');
```

<br/><br/>

### Tarea 6. Flashback Transaction Query / Transaction

Nivel de Aplicación: PDB


#### **Paso 1.** Ejecutar Transacción: Ejecuta 
```sql
DELETE FROM VENTAS WHERE ID = 2;
COMMIT;
```

#### **Paso 2.** Obtener Transaction ID: Ejecuta 
```sql
SELECT VERSIONS_XID FROM VENTAS VERSIONS BETWEEN TIMESTAMP MINVALUE AND MAXVALUE WHERE ID = 2;
--- para obtener el XID (ej. 0A001F00C7F90000).
```

#### **Paso 3.** Generar Script de UNDO: Ejecuta 
```sql
EXEC DBMS_FLASHBACK.GET_UNDO_SQL('0A001F00C7F90000'); --- en la PDB para generar el SQL de reversión.
```

#### **Paso 4.** Ejecutar el UNDO: Copia y ejecuta el script INSERT generado para deshacer la transacción.

<br/><br/>

### Tarea 7. Flashback Database (Recuperación física de la base de datos)

#### **Paso 1.** Verifica/Habilita Flashback.

```sql
-- Conectate al contenedor y verifica si FLASHBACK esta en ON.

SELECT FLASHBACK_ON FROM V$DATABASE;

-- Si el query devuelve NO habilita el flashback a nivel contenedor.

SHUTDOWN IMMEDIATE
STARTUP MOUNT
ALTER DATABASE FLASHBACK ON;
ALTER DATABASE OPEN;

```

Esto activa la generación de **Flashback Logs** en la **FRA**.

<br/>

#### **Paso 2.** Crea un punto de restauración garantizado.

```sql

CREATE RESTORE POINT RP_PRE_ERROR GUARANTEE FLASHBACK DATABASE;
```
<br/>

Crea un *restore point* llamado **RP_PRE_ERROR** que permitirá revertir la base de datos a ese momento.

<br/>

#### **Paso 3.** Simula un error fatal.

```sql
DROP TABLESPACE USERS INCLUDING CONTENTS AND DATAFILES;
COMMIT;
```

Esto elimina un tablespace crítico de la base de datos.

<br/>

#### **Paso 4.** Prepara la base de datos para hacer el  Flashback.

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
```

La base de datos debe estar **montada** para ejecutar la operación de Flashback.

<br/>

#### **Paso 5.** Ejecuta el Flashback.

Conéctate a RMAN y ejecuta:

```bash
RMAN TARGET /
FLASHBACK DATABASE TO RESTORE POINT RP_PRE_ERROR;
```

Este comando **rebobina** la base de datos al punto garantizado, utilizando los **Flashback Logs** y **Archived Logs**.

<br/>

#### **Paso 6.** Abre la base de datos.

```sql
ALTER DATABASE OPEN RESETLOGS;
```

Abre la base de datos en una nueva *incarnation*, lista para operar.


<br/><br/>


## Resultado esperado

Al finalizar esta práctica:

* Habrás comprendido y aplicado los mecanismos **Flashback Query** y **Flashback Database**.
* Lograrás recuperar datos eliminados o modificados por error, consultando el estado de la tabla a un instante anterior.
* Podrás **revertir la base de datos completa** a un punto seguro, utilizando un *restore point* garantizado.
* La base de datos deberá abrirse correctamente tras el comando:

  ```sql
  ALTER DATABASE OPEN RESETLOGS;
  ```
* Confirmando que la recuperación fue exitosa.
  <br/>

  ```
  Nota Importante sobre FLASHBACK DATABASE:
  El privilegio para ejecutar el comando FLASHBACK DATABASE (que opera a nivel de CDB).
  No puede ser otorgado a un usuario creado solo en la PDB. 
  Este privilegio requiere el rol SYSDBA o SYSBACKUP y solo puede ser ejecutado por un usuario que se conecte al CDB (Contenedor) en modo MOUNT.

