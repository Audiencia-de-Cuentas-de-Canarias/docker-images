# Docker Images from Oracle (Audiencia de Cuentas de Canarias)

## Construir la imagen

Descargar el fichero `LINUX.X64_193000_db_home.zip` (2.9 GB) de https://www.oracle.com/database/technologies/oracle-database-software-downloads.html#license-lightbox (fichero zip llamado "Oracle Database 19c for Linux x86-64") y colocarlo en `./OracleDatabase/SingleInstance/dockerfiles/19.3.0`.

A continuación construir la imagen *Oracla Database 19.3.0 Standard Edition 2*:

```sh
cd ./OracleDatabase/SingleInstance/dockerfiles

./buildContainerImage.sh -v 19.3.0 -e
```

## Ejecutar el contenedor Oracle

Ejecutar el siguiente comando. Se ejecuta el contenedor y se queda en modo interactivo para ver el log en consola y poder cerrarlo ordenadamente si se desea con CTRL+C.

```sh
docker run -d --name oracle19c \
	-p 1521:1521 -p 5500:5500 -p 2484:2484 \
	--ulimit nofile=1024:65536 --ulimit nproc=2047:16384 --ulimit stack=10485760:33554432 --ulimit memlock=3221225472 \
	-e ORACLE_EDITION=enterprise \
	-e ORACLE_CHARACTERSET=WE8ISO8859P15 \
	-e ORACLE_PWD=Octubre20 \
	--restart always \
	oracle/database:19.3.0-ee
```

Si se desea externalizar la carpeta con los ficheros físicos de la base de datos, crear una carpeta por ejemplo en `/opt/oracle/oradata` , darle permisos totales de escritura, y montar la carpeta como volumen adicional al contenedor:

```sh
sudo mkdir -p /opt/oracle/oradata
sudo chmod -R 777 /opt/oracle/oradata

docker run -d --name oracle19c \
	-p 1521:1521 -p 5500:5500 -p 2484:2484 \
	--ulimit nofile=1024:65536 --ulimit nproc=2047:16384 --ulimit stack=10485760:33554432 --ulimit memlock=3221225472 \
	-e ORACLE_EDITION=enterprise \
	-e ORACLE_CHARACTERSET=WE8ISO8859P15 \
	-e ORACLE_PWD=Octubre20 \
    -v /opt/oracle/oradata:/opt/oracle/oradata \
	--restart always \
	oracle/database:19.3.0-ee
```

> Nota: Si es la primera vez que se ejecuta el contenedor, se tiene que montar y levantar la base de datos, y el proceso puede tardar unos 20 minutos. En la consola del contenedor se puede ver el progreso. Las siguientes veces solo es levantar la instancia y solo dura unos segundos.

Una vez ejecutado el contenedor, se puede acceder a los logs con el comando:

```sh
docker logs -f oracle19c
```

## Importación de datos de la Plataforma de Rendición de Cuentas Locales

1. Copiar los ficheros de importación a la carpeta `/opt/oracle/admin/ORCLCDB/dpdump/` del contenedor.

```sh
docker exec -it oracle19c mkdir -p /opt/oracle/admin/ORCLCDB/dpdump/

docker cp /mnt/c/Users/Benito/Downloads/TBC2_EXP_TCUV2_11032024.dpexp oracle19c:/opt/oracle/admin/ORCLCDB/dpdump/TBC2_EXP_TCUV2.dpexp
docker cp /mnt/c/Users/Benito/Downloads/TBC2_EXP_SCICA_11032024.dpexp oracle19c:/opt/oracle/admin/ORCLCDB/dpdump/TBC2_EXP_SCICA.dpexp
```

> Cambiar las rutas locales y nombre de los ficheros por los que procedan.

2. Establecemos los objetos iniciales para la importación de los datos (tablespace y esquemas)

Entramos al contenedor:

```sh
docker exec -it oracle19c bash
```

Ejecutamos sqlplus como SYS:

```sh
sqlplus SYS/Octubre20 AS SYSDBA
```

Y creamos los objetos:

```sql
alter session set "_ORACLE_SCRIPT"=true;

CREATE TABLESPACE TS_EXP_TCUV2
DATAFILE 'ts_exp_tcuv2.dat' 
SIZE 1000M
ONLINE;

alter database datafile '/opt/oracle/product/19c/dbhome_1/dbs/ts_exp_tcuv2.dat' autoextend on next 1000m maxsize unlimited;

CREATE USER REXP_SCICA IDENTIFIED BY REXP_SCICA;
ALTER USER "REXP_SCICA" DEFAULT ROLE ALL;
grant all privileges to REXP_SCICA;
ALTER USER REXP_SCICA DEFAULT TABLESPACE TS_EXP_TCUV2;

CREATE USER EXP_TCUV2 IDENTIFIED BY EXP_TCUV2;
ALTER USER "EXP_TCUV2" DEFAULT ROLE ALL;
grant all privileges to EXP_TCUV2;
ALTER USER EXP_TCUV2 DEFAULT TABLESPACE TS_EXP_TCUV2;

CREATE USER REXP_TCU IDENTIFIED BY REXP_TCU;
ALTER USER "REXP_TCU" DEFAULT ROLE ALL;
grant all privileges to REXP_TCU;
ALTER USER REXP_TCU DEFAULT TABLESPACE TS_EXP_TCUV2;

quit
```

3. Importamos los datos

Entramos al contenedor (si no estamos dentro ya):

```sh
docker exec -it oracle19c bash
```

Importamos los datos:

```sh
export NLS_LANG=SPANISH_SPAIN.WE8ISO8859P15

impdp \"SYS/Octubre20 AS SYSDBA\" directory=DATA_PUMP_DIR dumpfile=TBC2_EXP_TCUV2.dpexp remap_tablespace=TCUV2_MV:TS_EXP_TCUV2 CONTENT=all TABLE_EXISTS_ACTION=REPLACE

impdp \"SYS/Octubre20 AS SYSDBA\" directory=DATA_PUMP_DIR dumpfile=TBC2_EXP_SCICA.dpexp remap_tablespace=TS_EXP:TS_EXP_TCUV2 remap_schema=EXP_SCICA:EXP_TCUV2 CONTENT=all TABLE_EXISTS_ACTION=REPLACE

exit
```

> Nota: Es importante establecer el encoding correcto con la variable NLS_LANG para que no haya errores y pérdida de datos durante la importación.

## Parar el contenedor oracle

Se puede parar ordenadamente la base de datos pulsando simplemente *CTRL+C* en la consola del contenedor, o bien haciendo un `docker stop oracle19c`. Cuando se vuelva a arrancar con un `docker start oracle19c` se mantendrán todos los datos.

## Eliminar el contenedor oracle

Para eliminar definitivamente el contenedor oracle, ejecutar:

```sh
docker rm -f oracle19c
```

> ¡OJO!: ESTO SUPONE LA PÉRDIDA DE TODOS LOS DATOS, Y HABRÁ QUE REPETIR TODO EL PROCESO DESDE EL PRINCIPIO

> Nota: Si el contenedor se ejecutó externalizando el directorio de datos de oracle (ej: `/opt/oracle/oradata`), el hecho de eliminar el contenedor no implica la pérdida de datos. Para borrar los datos definitivamente, ejecutar:

```sh
sudo rm -rf /opt/oracle/oradata/*
sudo rm -rf /opt/oracle/oradata/.ORCLCDB.created
```