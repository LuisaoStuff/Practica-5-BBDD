# Alumno 1 (Luis Vázquez Alejo)

## ORACLE

**1. Muestra los espacios de tablas existentes en tu base de datos y la ruta de los ficheros que los componen. ¿Están las extensiones gestionadas localmente o por diccionario?**

Para visualizar los _tablespaces_ del sistema, tendremos que ejcutar el siguiente comando.

```
SQL> col FILE_NAME form A40;
SQL> select FILE_NAME, TABLESPACE_NAME from dba_data_files UNION select FILE_NAME, TABLESPACE_NAME from dba_temp_files;

FILE_NAME				 TABLESPACE_NAME
---------------------------------------- ------------------------------
/opt/oracle/oradata/orcl/sysaux01.dbf	 SYSAUX
/opt/oracle/oradata/orcl/system01.dbf	 SYSTEM
/opt/oracle/oradata/orcl/temp01.dbf	 TEMP
/opt/oracle/oradata/orcl/undotbs01.dbf	 UNDOTBS1
/opt/oracle/oradata/orcl/users01.dbf	 USERS
```

En mi caso, como en la instalación no cambié la opción para poder hacer uso del parámetro _storage_ a la hora de crear tablas, están gestionadas **localmente**.

**2. Usa la vista del diccionario de datos v$datafile para mirar cuando fue la última vez que se ejecutó el proceso CKPT en tu base de datos.**

```
select max(CHECKPOINT_TIME) from v$datafile;

MAX(CHEC
--------
24/01/20
```

**3. Intenta crear el tablespace TS1 con un fichero de 2M en tu disco que crezca automáticamente cuando sea necesario. ¿Puedes hacer que la gestión de extensiones sea por diccionario? Averigua la razón.**

```
SQL> create tablespace ts1
  2  datafile 'c:\app\Oracle\oradata\orcl\ts1.dbf'
  3  size 2M
  4  autoextend on;

Tablespace creado.
```

Dependiendiendo de cómo se instaló _Oracle_. Es decir, si durante la instalación, se especificó una gestión manual del almacenamiento,

**4. Averigua el tamaño de un bloque de datos en tu base de datos. Cámbialo al doble del valor que tenga.**

```
select distinct bytes/blocks from user_segments;

BYTES/BLOCKS
------------
	8192

```

En principio _Oracle_ no permite la modificación del tamaño de bloque de un tablespace ya creado, por lo que nos disponemos a crear uno nuevo. Antes de crear dicho tablespace, vamos a modificar un valor del sistema para establecer el nuevo tamaño de bloque.

```
ALTER SYSTEM SET DB_16k_CACHE_SIZE=100M;
```
Después reiniciamos la base de datos para que se cargue la variable que hemos redefinido y creamos este nuevo _tablespace_.

```
shutdown inmediate;
startup
create tablespace prueba datafile '/home/oracle/test.img' size 1M blocksize 16K;
```
Y a continuación ejecutamos una consulta para verificar el tamaño de bloque.

```
SQL> select tablespace_name, block_size from dba_tablespaces where tablespace_name='PRUEBA';

TABLESPACE_NAME 	       BLOCK_SIZE
------------------------------ ----------
PRUEBA				    16384
```

**5. Realiza un procedimiento MostrarObjetosdeUsuarioenTS que reciba el nombre de un tablespace y el de un usuario y muestre qué objetos tiene el usuario en dicho tablespace y qué tamaño tiene cada uno de ellos.**

```
create or replace procedure MostrarObjetosdeUsuarioenTS(p_tsname VARCHAR2,
							p_user VARCHAR2)
is

	cursor c_userobjectsTS
	is
	select segment_name object, bytes
	from dba_segments
	where tablespace_name=p_tsname
	and owner=p_user;

begin
	dbms_output.put_line('OBJETOS USUARIO '||p_user||chr(9)||'TAMAÑO (BYTES)');
	for i in c_userobjectsTS loop
		dbms_output.put_line(i.object||chr(9)||chr(9)||i.bytes);
	end loop;
end;
/
```

_Comprobación_
```
SQL> exec MostrarObjetosdeUsuarioenTS('USERS','LUIS');
OBJETOS USUARIO LUIS	TAMAÑO (BYTES)
OPERACIONES		65536
CLINICAS		65536
CLIENTES		65536
ANIMALES		65536
VACUNAS			65536
VIRUS			65536
VETERINARIOS		65536
VACUNACIONES		65536
SALAS			65536
QUIROFANOS		65536
CITASPOSIBLES		65536
CITASPREVIAS		65536
PK_OPERACION		65536
PK_CLINICA		65536
PK_DNI			65536
PK_ANIMAL		65536
PK_VACUNA		65536
PK_VIRUS		65536
PK_VETERINARIO		65536
PK_VACUNAS		65536
PK_SALA_CLINICA		65536
PK_QUIROFANO		65536
PK_CITA			65536
PK_CITAS		65536
SYS_C007423		65536

Procedimiento PL/SQL terminado correctamente.
```

**6. Realiza un procedimiento llamado MostrarUsrsCuotaIlimitada que muestre los usuarios que puedan escribir de forma ilimitada en más de uno de los tablespaces que cuentan con ficheros en la unidad C:**

Para saber qué usuarios tienen una cuota ilimitada, debemos observar dos campos; el número máximos de bloques o **max_blocks** y el número máximo de _bytes_ o **max_bytes**. Para que el usuario tenga una cuota ilimitada, ambos campos deben de tener valor igual a **-1**.

```
create or replace procedure MostrarUsrsCuotaIlimitada
is
	cursor c_UsuarioSinCuota
	is
	select username nombre
	from DBA_TS_QUOTAS
	where max_blocks = -1
	and max_bytes = -1
	and tablespace_name in (select tablespace_name
				  from DBA_DATA_FILES
				  where substr(file_name,1,12) = '/home/oracle')
	group by username
	having count(tablespace_name) > 1;

begin
	dbms_output.put_line('Usuarios con cuota ilimitada');
	for usuario in c_UsuarioSinCuota loop
		dbms_output.put_line(usuario.nombre);
	end loop;
end;
/
```


## POSTGRES:
       
**7. Averigua si existe el concepto de tablespace en Postgres, en qué consiste y las diferencias con los tablespaces de ORACLE.**

Los _tablespaces_ si que existen en _Postgres_, aunque como veremos más adelante presentan una funcionalidad bastante reducida respecto a la versión de _Oracle_. Sin embargo, los _tablespaces_ en _Postgres_ si que aportan una mejora real al sistema, ya que permiten la segmentación y distribución física de los datos al poder ubicarlos en diversos directorios. Un ejemplo práctico sería crear un _tablespace_ con los datos de acceso más concurridos y ubicarlo en una unidad de disco estado sólido.

* Sintaxis en Oracle

```
CREATE [TEMPORARY|UNDO] TABLESPACE nombrets
[DATAFILE|TEMPFILE] ruta_absoluta_fichero [SIZE nº [K|M]][REUSE][AUTOEXTEND ON [MAXSIZE n M]]
… (podemos añadir más ficheros separando con comas)...
[LOGGING|NOLOGGING]
[PERMANENT|TEMPORARY]
[DEFAULT STORAGE
(clausulas de almacenamiento)]
[OFFLINE];
```

* Sintaxis en Postgres

```
CREATE TABLESPACE tablespace_name
    [ OWNER { new_owner | CURRENT_USER | SESSION_USER } ]
    LOCATION 'directory'
    [ WITH ( tablespace_option = value [, ... ] ) ]
```

* Referencias: 
1. [Crear tablespace en Oralce](https://docs.oracle.com/cd/B19306_01/server.102/b14200/statements_7003.htm)
2. Documentación oficial de Postgresql; [Definición de tablespace](https://www.postgresql.org/docs/10/manage-ag-tablespaces.html), [Creación de tablespaces](https://www.postgresql.org/docs/10/sql-createtablespace.html)


### Diferencias principales

* En Oracle se usan ficheros (_datafiles_) y en Postgres usamos directorios.
* Mientras que Oracle permite una definición más precisa de estos _tablespaces_, Postgres se queda un poco corto ya que ni siquiera podemos definir un tamaño máximo.

## MySQL:

**8. Averigua si pueden establecerse claúsulas de almacenamiento para las tablas o los espacios de tablas en MySQL.**

A la hora de crear tablas podemos usar una serie de parámetros para definir aspectos como el número máximo de registros, la ruta donde se ubicará el fichero contenedor de la tabla o si queremos encriptar el contenido o no. Estos son todos los parámetros:

```
MariaDB [(none)]> help create table
...

table_options:
    table_option [[,] table_option] ...

table_option:
    ENGINE [=] engine_name
  | AUTO_INCREMENT [=] value
  | AVG_ROW_LENGTH [=] value
  | [DEFAULT] CHARACTER SET [=] charset_name
  | CHECKSUM [=] {0 | 1}
  | [DEFAULT] COLLATE [=] collation_name
  | COMMENT [=] &apos;string&apos;
  | CONNECTION [=] &apos;connect_string&apos;
  | DATA DIRECTORY [=] &apos;absolute path to directory&apos;
  | DELAY_KEY_WRITE [=] {0 | 1}
  | INDEX DIRECTORY [=] &apos;absolute path to directory&apos;
  | INSERT_METHOD [=] { NO | FIRST | LAST }
  | KEY_BLOCK_SIZE [=] value
  | MAX_ROWS [=] value
  | MIN_ROWS [=] value
  | PACK_KEYS [=] {0 | 1 | DEFAULT}
  | PASSWORD [=] &apos;string&apos;
  | ROW_FORMAT [=] {DEFAULT|DYNAMIC|FIXED|COMPRESSED|REDUNDANT|COMPACT}
  | TABLESPACE tablespace_name [STORAGE {DISK|MEMORY|DEFAULT}]
  | UNION [=] (tbl_name[,tbl_name]...)

partition_options:
    PARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY(column_list)
        | RANGE{(expr) | COLUMNS(column_list)}
        | LIST{(expr) | COLUMNS(column_list)} }
    [PARTITIONS num]
    [SUBPARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY(column_list) }
      [SUBPARTITIONS num]
    ]
    [(partition_definition [, partition_definition] ...)]

partition_definition:
    PARTITION partition_name
        [VALUES 
            {LESS THAN {(expr | value_list) | MAXVALUE} 
            | 
            IN (value_list)}]
        [[STORAGE] ENGINE [=] engine_name]
        [COMMENT [=] &apos;comment_text&apos; ]
        [DATA DIRECTORY [=] &apos;data_dir&apos;]
        [INDEX DIRECTORY [=] &apos;index_dir&apos;]
        [MAX_ROWS [=] max_number_of_rows]
        [MIN_ROWS [=] min_number_of_rows]
        [TABLESPACE [=] tablespace_name]
        [NODEGROUP [=] node_group_id]
        [(subpartition_definition [, subpartition_definition] ...)]

subpartition_definition:
    SUBPARTITION logical_name
        [[STORAGE] ENGINE [=] engine_name]
        [COMMENT [=] &apos;comment_text&apos; ]
        [DATA DIRECTORY [=] &apos;data_dir&apos;]
        [INDEX DIRECTORY [=] &apos;index_dir&apos;]
        [MAX_ROWS [=] max_number_of_rows]
        [MIN_ROWS [=] min_number_of_rows]
        [TABLESPACE [=] tablespace_name]
        [NODEGROUP [=] node_group_id]
...

```

A pesar de todo lo que podemos definir una tabla en _mysql_, no cuenta con la posibilidad de crear _tablespaces_ en circustancias normales. Para poder disponer de esta funcionalidad, deberíamos de tener instalada la versión de **alta disponibilidad** en **cluster** de _mysql_, es decir la conocida [MySQL NDB](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster.html). En este caso la sintaxis a la hora de crear los _tablespaces_ es la siguiente:

```
CREATE TABLESPACE tablespace_name

  InnoDB and NDB:
    ADD DATAFILE 'file_name'

  InnoDB only:
    [FILE_BLOCK_SIZE = value]

  NDB only:
    USE LOGFILE GROUP logfile_group
    [EXTENT_SIZE [=] extent_size]
    [INITIAL_SIZE [=] initial_size]
    [AUTOEXTEND_SIZE [=] autoextend_size]
    [MAX_SIZE [=] max_size]
    [NODEGROUP [=] nodegroup_id]
    [WAIT]
    [COMMENT [=] 'string']

  InnoDB and NDB:
    [ENGINE [=] engine_name]
```

* Referencia: [Documentacion oficial de MySQL](https://dev.mysql.com/doc/refman/5.7/en/create-tablespace.html)

## MongoDB:

**9. Averigua si existe el concepto de índice en MongoDB y las diferencias con los índices de ORACLE. Explica los distintos tipos de índice que ofrece MongoDB.**

En _MongoDB_ existen distintos tipos de índices en función de como queremos que funcione (aplicado sobre un campo o varios, en orden ascendente, descendente, etc), mientras que en Oracle solo disponemos de un tipo que contiene la mayoría de dichas funcionalidades.

* **Simples**: Se aplican a un único campo, y con el valor **1** o **-1** indicamos si se aplica de forma ascendente o descendente respectivamente.

```
db.alumnos.createIndex( { "nombre" : 1 } )
```

* **Compound**: Se utilizan sobre dos o más valores y se comporta de la misma forma que los _simples_, por lo tanto podemos especificar en cada campo que indiquemos el sentido de la búsqueda.

```
db.alumnos.createIndex( { "nombre" : 1, "nota": -1 } ) 

# Búsqueda del campo libro en orden ascendente y el
# autor en orden descendente
```

* **Multikey**: Sirven para indexar contenido almacenado en _arrays_. Normalmente cuando _MongoDB_ recorre un documento con _arrays_, recorre cada uno de sus valores, mientras que si utilizamos este tipo de índices, podemos aseguranos de que solo recorra aquellos _arrays_ que contengan uno de los valores que hemos especificado. Para crear este tipo de índices no tenemos que hacer nada, que ya al utilizar el método _createIndex_, el propio sistema crea de forma automática un _index simple_ o un _index multikey_.


* **Unique**: Se utilizan para eliminar los campos duplicados, algo parecido a cuando relizamos una _select distinct_ en _Oracle_

```
db.alumnos.createIndex( { "nota" : 1 }, {"unique":true} )
```

* **TTL Indexes**: Personalmente no lo consideraría un tipo de índice como tal, ya que con esta propiedad especificamos el tiempo de espiración de un documento en base a un campo del mismo. En el ejemplo que expone la documentación de _Mongo_, se borrarían los documentos de la colección **eventlog** que no fuesen modificados después de una hora.

```
db.eventlog.createIndex( { "lastModifiedDate": 1 }, { expireAfterSeconds: 3600 } )
```


* Referencia: [Documentación oficial MongoDB](https://docs.mongodb.com/manual/indexes/)
