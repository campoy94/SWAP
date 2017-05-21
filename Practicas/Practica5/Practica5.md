# Práctica 5. Replicación de bases de datos MySQL

## 1. Crear BD en M1

Entramos en MySQL tecleando `mysql -uroot -p`, nos pedirá nuestra contraseña.
Una vez dentro de MySQL podremos crear nuestra base de datos, en la consola de MySQL:

`	mysql> create database contactos;

	mysql> use contactos;

	mysql> show tables;

	mysql> create table datos(nombre varchar(100),tlf int);

	mysql> show tables;

	mysql> insert into datos(nombre,tlf) values ("juan",123456789);

	[insertamos varias tuplas]

	mysql> select * from datos;

	[veremos las tuplas insertadas]

	mysql> exit[para salir]`

![Captura1](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura1.PNG)


## 2. Replicar una BD MySQL con mysqldump

mysqldump es una herramienta que nos ofrece MySQL para crear copias de nuestras Bases de Datos. Lo usaremos para crear una
copia de seguridad de nuestra base, y transferirla a M2.
Para asegurarnos de que nadie acceda y modifique la base de datos entraremos a nuestro MySQL y pondremos lo siguiente:

`mysql> FLUSH TABLRS WITH READ LOCK;

 mysql> exit`

Una vez hecho esto procedemos a utilizar mysqldump para hacer la copia de los datos.
Fuera de la base de datos en nuestra consola escribimos:

`mysqldump "nombreBD" -u root -p > /tmp/nombreBD.sql

	password:*****`

Ya tenemos nuestra copia, como habíamos bloqueado las tablas debemos desbloquearlas, entramos en mysql:

`mysql> UNLOCK TABLES;

 mysql> exit`

Una vez tenemos nuestra copia ya podemos pasarla a M2 para replicar nuestra base de datos.
En M2 hacemos:

`scp M1:/tmp/nombreBD.sql /tmp/`

Ahora ya tenemos la copia de seguridad de la BD en nuestra M2, sin embargo este archivo .sql es un archivo de texto
plano que contiene las sentencias necesarias para replicar la base de datos, pero estas sentencias no incluyen la creaci´pon
de la base de datos en sí. Por tanto debemos crear la base de datos en M2 antes de replicarla.
Para hacerlo entramos en MySQL en M2:

`mysql -u root -p

	mysql> CREATE DATABASE nombreBD;

	mysql> exit`

Una vez creada la base de datos en M2 podemos volcar la copia de seguridad:

`mysql -u root -p nombreBD < /tmp/nombreBD.sql`

![Captura2](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura2.PNG)

Ahora ya tenemos replicada la BD, podemos comprobarlo entrando en ella y haciendo:

`mysql> select * from datos;`

Veremos que son los mismos que teniamos en M1.

## 3.Replicación de BD mediante una configuración maestro-esclavo

Acabamos de ver como replicar la base de datos de una forma manual, sin embargo esto en un ambiente real esto no es
algo viable. Por ello MySQL incorpora la opción de configurar un demonio para hacer replicación de las Bases de Datos
sobre un esclavo a partir de los datos que almacena el maestro. En esta parte vamos a configurar el maestro-esclavo.

Para empezar este proceso debemos tener los mismos datos en ambas máquinas.

Primero configuramos el servidor maestro, lo haremos en M1.

Lo primero que hay que hacer es editar(como root) el archivo de configuración de MySQL:

	`nano /etc/mysql/my.conf.d/mysqld.cnf`

Comentamos el parámetro bind address, que sirve para que escuche de un servidor:

	`#bind-address 127.0.0.1`

Descomentamos la linea del id de servidor:

	`server-id = 1`

También descomentamos la línea del registro binario, este contiene la información del registro
de actualizaciones en un formato eficiente y seguro para realizar las transacciones:

	`log_bin = /var/log/mysql/bin.log`

Ya tenemos todo lo necesario, guardamos el archivo y reiniciamos MySQL:

	`/etc/init.d/mysql restart`

El servidor maestro ya está configurado.

![captura3](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura3.PNG)

Vemos que no nos ha dado ningun error, si lo hubiera hecho debemos dirigirnos al archivo
`/var/log/mysql/error.log` para comprobar cual ha sido el error.

Una vez configurado el maestro vamos a configurar el esclavo, como antes editamos como root el archivo
de configuración en el servidor esclavo:

`nano /etc/mysql/my.conf.d/mysqld.cnf`

Buscamos server-id y lo descomentamos como hemos hecho antes, pero esta vez el id será 2:

`server-id = 2`

Nos salimos del editor guardando los cambios y volvemos a iniciar el MySQL:

`/etc/init.d/mysql restart`


![Captura4](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura4.PNG)

Comprobamos que iniciamos sin problema el servicio, si nos hubiera dado error deberiamos mirar el log
de errores.

Ahora volvemos al servidor maestro y entramos en mysql, crearemos un nuevo usuario 'esclavo' y le daremos permisos
de replicación:

`	mysql> CREATE USER esclavo IDENTIFIED BY 'esclavo';

	mysql> GRANT REPLICATION SLAVE ON *.* TO 'esclavo'@'%' IDENTIFIED BY 'esclavo';

	mysql> FLUSH PRIVILEGES;

	mysql> FLUSH TABLES;

	mysql> FLUSH TABLES WITH READ LOCK;`

Por último obtenemos los datos de la BD que vamos a replicar para usarlos en la configuración del esclavo:

	`mysql> SHOW MASTER STATUS;`

![Captura5](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura5.PNG)

Ahora volvemos a máquina esclavo y entramos en MySQL para darle los datos del maestro:

`	mysql> CHANGE MASTER TO MASTER_HOST='IPM1',
	MASTER_USER='esclavo', MASTER_PASSWORD='esclavo',
 	MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=501,
	MASTER_PORT=3306;`

Si todo está correcto solo nos queda arrancar el esclavo:

`mysql> START SLAVE;`

![Captura6](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura6.PNG)

Ya está en marcha la configuración maestro-esclavo, los demonios de MySQL debería replicar automáticamente
lo que hagamos en la máquina maestro.
Para comprobar que funciona volvemos al maestro, lo primero que debemos hacer es desbloquear las tablas:

`mysql> UNLOCK TABLES;`

Antes de empezar a hacer cambios conviene comprobar que el esclavo no tiene ningun problema, volvemos al 
esclavo y hacemos lo siguiente:

`mysql> SHOW SLAVE STATUS\G`

Comprobamos que el valor de la variable "Seconds_Behind_Master" no es "null". Si es así todo esta correcto.

![Captura7](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura7.PNG)

Ahora solo queda introducir algo en el maestro para comprobar que realmente está funcionando:

![Captura8](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica5/img/Captura8.PNG)

La configuración maestro-esclavo está funcionando perfectamente.


-----------------------------------------------------------------------------------------------------------------
Cosas a tener en cuenta:

Quitar la configuración de IPTABLES de la práctica anterior, ya que teniamos cerrados los puertos de MySQL.