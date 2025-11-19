# Práctica 3.1. Realizando respaldos en Oracle 19c

<br/><br/>

## Objetivos 
Al finalizar la práctica, serás capaz de:
- Configurar y ejecutar respaldos completos, incrementales y de PDB en Oracle 19c utilizando RMAN, asegurando la correcta administración del ciclo de vida de los respaldos.

<br/><br/>

## Tiempo estimado
- 120 minutos.

<br/><br/>

## Prerrequisitos

| **Prerrequisito**                 | **Descripción**                                                                                 |
| --------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Instancia de Oracle 19c**       | Debe estar instalada y ejecutándose correctamente.                                              |
| **Arquitectura**                  | Se requiere la arquitectura **Multitenant (CDB/PDB)**, predeterminada en Oracle 19c.              |
| **Usuario del sistema operativo** | Debe estar conectado como el usuario **`oracle`**, propietario de la base de datos.               |
| **Variables de entorno**          | Las variables **`ORACLE_HOME`** y **`ORACLE_SID`** deben estar configuradas y apuntar a la **CDB**. |

<br/><br/>

### Simulación de cambio de datos

> Utiliza estos comandos para **simular actividad en la base de datos** después de un respaldo de **Nivel 0** y antes de ejecutar un respaldo de **Nivel 1**. Esto permite que **RMAN detecte los bloques modificados** y los incluya en los respaldos incrementales.

**Ejemplos de comandos para generar cambios a la base de datos:**

```bash
$ sqlplus RMAN_USER/Oracle19c@<Nombre_Servicio_PDB_TEST>
SQL> INSERT INTO TEST_DATA VALUES (1, 'Primer cambio para L1');
SQL> COMMIT;
SQL> ALTER SYSTEM ARCHIVE LOG CURRENT;
SQL> exit
```

<br/><br/>

## Instrucciones

### Tarea 1. Creación del ambiente de prueba.

> En esta tarea, usarás comandos **SQL** y **SQL*Plus** para crear una **PDB**, un **tablespace** y un **usuario**, con el objetivo de que **RMAN** pueda registrar y respaldar los cambios realizados en la base de datos.


**A. Verifica y crea una nueva PDB.**

> Si tu base de datos **CDB** no cuenta con una **PDB de pruebas**, créala antes de continuar.
> Recuerda que **el respaldo de una CDB incluye automáticamente todas sus PDB**.


1. Conéctate a la base de datos como SYSDBA:

   ```bash
   $ sqlplus / as sysdba
   ```

2. Crea una nueva PDB de pruebas:

   ```sql
   CREATE PLUGGABLE DATABASE PDB_TEST ADMIN USER PDBADMIN IDENTIFIED BY Oracle19c FILE_NAME_CONVERT = ('/pdbseed/', '/pdb_test/');
   ```

3. Abre la PDB:

   ```sql
   ALTER PLUGGABLE DATABASE PDB_TEST OPEN;
   report schema;
   exit
   ```

4. Verifica que esté en modo **READ WRITE**:

   ```sql
   sqlplus / as sysdba
   SHOW PDBS;
   ```

**B. Creación del tablespace y de un usuario de prueba.**

> La PDB, el tablespace y el usuario creados permitirán realizar cambios en los datos que **RMAN** podrá detectar y registrar en los **respaldos incrementales**.


1. Cambia al contenedor de la PDB:

   ```sql
   ALTER SESSION SET CONTAINER = PDB_TEST;
   ```

2. Crea el tablespace de pruebas:

   ```sql
   CREATE TABLESPACE TS_LAB DATAFILE 'ts_lab01.dbf' SIZE 10M AUTOEXTEND ON NEXT 5M MAXSIZE 100M;
   ```
3. Crea un usuario de respaldo:

   ```sql
   CREATE USER RMAN_USER IDENTIFIED BY Oracle19c DEFAULT TABLESPACE TS_LAB;
   ```
4. Asigna los permisos al usuario:

   ```sql
   GRANT CONNECT, RESOURCE TO RMAN_USER;
   GRANT UNLIMITED TABLESPACE TO RMAN_USER;
   ```
5. Crea una tabla de datos de prueba:

   ```sql
   CREATE TABLE RMAN_USER.TEST_DATA (ID NUMBER, DESCRIPCION VARCHAR2(50)) TABLESPACE TS_LAB;
   ```
6. Sal de SQL*Plus:

   ```sql
   exit
   ```
<br/><br/>

### Tarea 2. Preparación del entorno y verificación del modo `ARCHIVELOG`.

1. Conéctate como SYSDBA:

   ```bash
   $ sqlplus / as sysdba
   ```
2. Verifica el modo de registro:

   ```sql
   ARCHIVE LOG LIST;
   ```

> **Nota:** al ejecutar el comando, la salida debe mostrar la línea **`Database log mode Archive Mode`**, lo que confirma que la base de datos está operando en **modo `ARCHIVELOG`**.


3. Si no está en modo `ARCHIVELOG`, habilitarlo:

   ```sql
   SHUTDOWN IMMEDIATE;
   STARTUP MOUNT;
   ALTER DATABASE ARCHIVELOG;
   ALTER DATABASE OPEN;
   ARCHIVE LOG LIST;
   ```

   ```sql
   exit
   ```

4. Conéctate a RMAN:

   ```bash
   $ rman target /
   ```

   > **Nota:** al conectarte a RMAN, debe mostrarse el mensaje **`connected to target database: <SID> (DBID=...)`**, indicando que la conexión con la base de datos objetivo se realizó correctamente.

5. Verifica la configuración base:

   ```rman
   SHOW ALL;
   ```

<br/> <br/>

### Tarea 3. Configuración persistente básica de RMAN.

1. Configura la política de retención:

   ```rman
   CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
   ```

2. Activa el autorrespaldo del control file:

   ```rman
   CONFIGURE CONTROLFILE AUTOBACKUP ON;
   ```

3. Define el formato del autorespaldo:

   ```rman
   CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '%F';
   ```

4. Habilita la compresión para backupsets:

   ```rman
   CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO COMPRESSED BACKUPSET;
   ```

5. Verifica la configuración:

   ```rman
   SHOW ALL;
   ```

### Tarea 4. Respaldo completo y respaldo de Archivelogs.

1. Ejecuta un respaldo completo de Nivel 0:

   ```rman
   BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG TAG 'FULL_BASE_L0';
   ```

2. Da de alta la base de datos pluggable en el archivo `tnsnames.ora`:
   ```
   cd $ORACLE_HOME/network/admin
   vi tnsnames.ora
   # Agrega la entrada de la PDB_TEST
   :wq!
   ```
   
3. Simula actividad y archivar logs:
   
   
```bash
$ sqlplus RMAN_USER/Oracle19c@PDB_TEST
SQL> INSERT INTO TEST_DATA VALUES (1, 'Primer cambio para L1');
SQL> INSERT INTO TEST_DATA VALUES (2, 'Primer cambio para L1');
SQL> INSERT INTO TEST_DATA VALUES (3, 'Primer cambio para L1');
SQL> COMMIT;
SQL> exit
```

   ```rman
   rman target /
   RMAN> ALTER SYSTEM ARCHIVE LOG CURRENT;
   ```

4. Respalda Archivelogs y elimina los originales:

   ```rman
   BACKUP ARCHIVELOG ALL DELETE INPUT;
   ```

5. Verifica el resumen de respaldos:

   ```rman
   LIST BACKUP SUMMARY;
   exit
   ```

> **Nota:** verifica que en el listado aparezcan **dos backupsets** identificados con las **etiquetas correspondientes** a los respaldos realizados.

<br/><br/>

### Tarea 5. Implementación de la estrategia incremental.

1. Simula cambios en datos:
   
```bash
$ sqlplus RMAN_USER/Oracle19c@PDB_TEST
SQL> INSERT INTO TEST_DATA VALUES (1, 'Segundo cambio para L1');
SQL> INSERT INTO TEST_DATA VALUES (2, 'Segundo cambio para L1');
SQL> INSERT INTO TEST_DATA VALUES (3, 'Segundo cambio para L1');
SQL> COMMIT;
SQL> exit
```
2. Ejecuta el respaldo incremental Nivel 1:

   ```rman
   rman target /

   BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG TAG 'DAILY_INCR_L1';
   ```

3. Realiza más cambios y archiva logs:

```sql
$ sqlplus RMAN_USER/Oracle19c@PDB_TEST
SQL> INSERT INTO TEST_DATA VALUES (1, 'Tercer cambio para L1');
SQL> INSERT INTO TEST_DATA VALUES (2, 'Tercer cambio para L1');
SQL> INSERT INTO TEST_DATA VALUES (3, 'Tercer cambio para L1');
SQL> COMMIT;
SQL> exit
```

   ```rman
   rman target /
   RMAN> ALTER SYSTEM ARCHIVE LOG CURRENT;
   ```

4. Ejecuta un segundo respaldo incremental:

   ```rman
   BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG TAG 'SECOND_INCR_L1';
   ```

5. Compara tamaños de respaldos:

```rman
   LIST BACKUP SUMMARY;
   ```

> **Nota:** verifica que en el listado aparezcan **dos backupsets** identificados con las **etiquetas correspondientes** a los respaldos realizados.

<br/><br/>

### Tarea 6. Mantenimiento y gestión del ciclo de vida.

1. Simula la eliminación física de un respaldo:

   ```bash
   $ cd $ORACLE_BASE/fast_recovery_area/ORCL/backupset/DIR_FECHA_DE_RESPALDO
   $ rm -f un_archivo_backup.bkp
   $ rman target /
   ```

2. Verifica los respaldos cruzados:

   ```rman
   CROSSCHECK BACKUP;
   ```

> **Nota:** el comando **`CROSSCHECK BACKUP`** compara los **registros de respaldos almacenados en el repositorio de RMAN** con los **archivos físicos** en disco.
> Si un respaldo ya no existe en el sistema de archivos, RMAN lo marcará como **EXPIRED**.


3. Elimina los respaldos expirados:

   ```rman
   DELETE EXPIRED BACKUP;
   ```

   > **Nota:** el comando **`DELETE EXPIRED BACKUP`** elimina de la **metadata de RMAN** las referencias a respaldos que ya no existen físicamente en disco.


4. Reporta los respaldos obsoletos:

   ```rman
   REPORT OBSOLETE;
   ```

   > **Nota:** el comando **`REPORT OBSOLETE`** analiza los respaldos existentes y muestra cuáles **pueden eliminarse** según la **política de retención configurada** (por ejemplo, 7 días).


5. Elimina los respaldos obsoletos:

   ```rman
   DELETE OBSOLETE;
   exit
   ```

> **Nota:** el comando **`DELETE OBSOLETE`** **elimina físicamente** los respaldos que RMAN considera **obsoletos**, liberando espacio en el almacenamiento.

<br/><br/>

### Tarea 7. Respaldo de CDB y PDB específica.


> **Nota:** antes de ejecutar los siguientes comandos, **asegúrate de estar conectado a la CDB**, ya que las operaciones de respaldo a nivel contenedor deben realizarse desde este contexto.


1. Verifica las PDB disponibles:

   ```sql
   sqlplus / as sysdba
   SQL>SHOW PDBS;
   exit
   ```

2. Realiza un respaldo completo de toda la base de datos contenedora CDB:

   ```rman
   rman target /
   BACKUP DATABASE PLUS ARCHIVELOG TAG 'CDB_TOTAL';
   ```

3. Realiza un respaldo individual de PDB:

   ```rman
   BACKUP PLUGGABLE DATABASE PDB_TEST PLUS ARCHIVELOG TAG 'PDB_UNITARIO';
   ```

4. Verifica los backupsets generados:

   ```rman
   LIST BACKUP SUMMARY;
   ```

> **Nota:** verifica que en el resumen de respaldos aparezcan **dos backupsets**, uno correspondiente a **toda la CDB** y otro **más pequeño**, generado para la **PDB específica**.

<br/><br/>


### Tarea 8. Copia del control file y respaldo con PDB en modo `OPEN/CLOSE`.

> **Nota:** antes de ejecutar los siguientes comandos, **asegúrate de estar conectado a la CDB**, ya que las operaciones de respaldo a nivel contenedor deben realizarse desde este contexto.

1. Cierra la PDB:

   ```rman
   ALTER PLUGGABLE DATABASE PDB_TEST CLOSE;
   ```

2. Respalda la PDB cerrada:

   ```rman
   BACKUP PLUGGABLE DATABASE PDB_TEST TAG 'PDB_CERRADA_CONSISTENTE';
   LIST BACKUP SUMMARY;
   ```

3. Abre nuevamente la PDB:

   ```rman
   ALTER PLUGGABLE DATABASE PDB_TEST OPEN;
   ```

4. Genera un autorespaldo del control file:

   ```rman
   BACKUP CURRENT CONTROLFILE;
   ```

> **Nota:** este comando **genera un autorespaldo inmediato** del **Control File** y del **SPFILE**, elementos esenciales para la **recuperación de la base de datos** en caso de fallo.


5. Lista los autorespaldos del control file:

   ```rman
   LIST BACKUP OF CONTROLFILE SUMMARY;
   ```
   
> **Nota:** localiza el **archivo de respaldo del Control File**, ya que es **fundamental para realizar una recuperación** cuando **no se dispone de un catálogo de RMAN**.

6. Sal de RMAN:

   ```rman
   EXIT;
   ```

<br/><br/>

## Resultado esperado

Al finalizar esta práctica, el participante habrá configurado correctamente el entorno de respaldo y recuperación en Oracle 19c, comprendiendo el uso de **RMAN** como herramienta central para la administración del ciclo de vida de los respaldos.

En particular, se espera que el alumno haya logrado:

* Verificar que la base de datos se encuentra en **modo ARCHIVELOG** y que los respaldos incluyen los archivos necesarios para una recuperación consistente.
* Aplicar políticas de **retención y compresión** en RMAN, además de habilitar el **autorespaldo del control file**.
* Comprobar el resultado de los respaldos mediante los comandos:

  ```rman
  LIST BACKUP SUMMARY;
  SHOW ALL;
  ```

* Confirmar la creación de los backupsets con sus respectivas etiquetas.
* Gestionar el ciclo de vida de los respaldos utilizando `CROSSCHECK`, `REPORT OBSOLETE` y `DELETE OBSOLETE`, manteniendo la metadata de RMAN sincronizada con los archivos físicos.
* Generar y validarrespaldos tanto a nivel de **CDB** como de **PDB específica** en distintos estados (abierto y cerrado).
