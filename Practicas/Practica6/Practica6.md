# Práctica 6. Discos en RAID

En esta práctica se confiugran dos discos en RAID 1(RAID 1 hace una copia exacta con la misma información en ambos discos).

Se crearán dos discos nuevos y se añadiran a una máquina servidor(Ubuntu 16). Tendremos 3 discos, uno con el SO y los otros
dos discos nuevos en RAID 1.
Luego realizaremos pruebas añadiendo y retirando discos "en caliente", para comrpbar la fiabilidad de RAID 1.

## 1. Configuración del RAID por software

Antes de hacer la configuración RAID tenemos que añadir los discos con la máquina apagada.
Yo he utilizado VMware Workstation para las prácticas, para añadir los discos debemos seleccionar nuestra máquina virtual,
y en vez de arrancarla pulsamos en `Edit virtual machine settings`. Una vez en el menú de ajustes pulsamos Hard Disk y 
Add.

![Captura1](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura1.PNG)

Una vez aquí el menú para añadir el disco es bastante intuitivo, yo he creado dos discos SATA de un 1GB.

![Captura2](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura2.PNG)

Ahora que ya tenemos creados los dos nuevos discos ya podemos arrancar la máquina, como lo haríamos normalmente.
Una vez hemos entrado en nuestra máquina lo primero que vamos a hacer es instalar el software necesario
para la configuración del RAID:

`sudo apt-get install mdadm`

Ya instalado mdadm lo primero que debemos conocer es la información de identificación que Linux le ha dado a los discos:

`sudo fdisk -l`

![Captura discos](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura3.PNG)

Como vemos en la imagen yo tengo 3 discos: 

	-/dev/sda(20GB): Donde está instalado el sistema operativo.
	-/dev/sdb(1GB):  Uno de los discos nuevos que he añadido.
	-/dev/sdc(1GB):  El otro de los nuevos discos añadidos.

Ahora que sabemos los identificadores de los discos utilizamos mdadm para crear el RAID1, usaremos el dispositivo 
`/dev/md0`, la orden para hacerlo es la siguiente:

`sudo mdadm -C /dev/md0 --level=raid1 --raid-devices=2 /dev/sdb /dev/sdc`

Explicamos un poco la orden utilizada:

	Con -C indicamos el dispositivo donde haremos el raid.
	--level=raid1 es el tipo de RAID que creamos en nuestro caso RAID1.
	--raid-devices=2 aquí indicamos los discos que utilizamos para crear el RAID, en este caso 2.
	Por último indicamos el identificador de los discos, sdb y sdc.

![Captura mdam](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura4.PNG)

!!Importante:
En este punto, el dispositivo se habrá creado con el nombre /dev/md0, sin embargo, en cuanto reiniciemos 
la máquina, Linux lo renombrará y pasará a llamarlo /dev/md127.

Ya hemos creado el RAID, como aún no hemos reiniciado la máquina seguiremos usando /dev/md0 como identificador.
Lo primero que tenemos que hacer es darle formato:

`sudo mkfs /dev/md0`

Si no le indicamos más opciones, por defecto, mkfs utiliza formato ext2.

Una vez hecho esto ya podemos crear el directorio en el que montaremos el RAID:

`sudo mkdir /dat`

Con mount montamos el RAID en el directorio:

`sudo mount /dev/md0 /dat`

Podemos utilizar `sudo mount` para ver que el proceso se ha relizado correctamente.
Si no hemos tenido ningun problema usamos `sudo mdadm --detail /dev/md0` para comprobar el 
estado del RAID.

![captura detalles RAID](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura5.PNG)

## 2. Configurar el montaje del RAID al inicio del sistema

Una vez configurado el RAID conviene configurar el sistema para que monte el dispositivo RAID
cuando arranque el sistema. Para hacer esto deberemos editar el archivo `/etc/fstab`.
Lo primero que debemos saber es obtener los UUID de todos los dispositivos de almacenamiento
que tenemos, para ello utilizamos:

`ls -l /dev/disk/by-uuid/`

![CapturaUUID](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura6.PNG)

Ahora que ya sabemos el UUID ya podemos editar el archivo `/etc/fstab`:

`sudo nano /etc/fstab`

Añadiremos al final del mismo la siguiente línea:

`UUID=ccbbbbcc-dddd-eeee-ffff-aaabbbcccddd /dat ext2 defaults 0 0`

![Captura7](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura7.PNG)

Con esto ya tendremos montado el RAID cada vez que arranquemos la máquina.

## 3. Simular fallos en los discos

Por último para probar la fiabilidad de nuestro RAID 1 podemos simular un fallo en uno de los discos:
Para ello utilizamos mdadm:

`sudo mdadm --manage --set-faulty /dev/md0 /dev/sdb`

Con esto simulamos un daño en el disco `/dev/sdb`.
Ahora realizamos una "retirada en caliente":

`sudo mdadm --manage --remove /dev/md0 /dev/sdb`

Y por último reemplazamos "en caliente" el disco averiado:

`sudo mdadm --manage --add /dev/md0 /dev/sdb`

![Captura8](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura8.PNG)

## 4. Configurar un servidor NFS[Parte Opcional]

Para configurar un servidor NFS lo primero que debemos instalar son las utilidades de Linux para
crear un servidor NFS, normalmente viene ya instalado, pero por si acaso:

`sudo apt-get install nfs-common nfs-kernel-server`

Ahora, antes de arrancar el servicio NFS, es necesario indicar qué carpetas queremos compartir, 
con qué permisos y con quién. Para esto editamos el archivo `/etc/exports`:

`sudo nano /etc/exports`

Tendremos que añadir una lína similar a la siguiente:

`/dat 192.168.153.129(ro)`

La sintáxis es la siguiente:

	Primero indicamos la carpeta que queremos compartir
	Luego indicamos la IP/rango de IPS con quien queremos compartir
	Por último los permisos, en este caso ro, read only, también podría ser rw.

![Captura9](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura9.PNG)

Ahora damos permiso para compartir las carpetas con la siguiente orden:

`sudo exportfs -a`

Una vez hecho esto arrancamos el servidor NFS con la siguiente orden:

`sudo service nfs-kernel-server start`

Con esto ya tenemos el servidor preparado para compartir las carpetas por NFS.
Ahora configuramos el cliente para poder ver las carpetas, primero instalamos el cliente:

`sudo apt-get install nfs-common`

Una vez instalado creamos la carpeta donde estará la carpeta compartida:

`sudo mkdir -p /mnt/nfs/dat` 

Ahora montamos la carpeta:

`sudo mount IPMS:/dat /mnt/nfs/dat`

Donde IPMS es la IP del servidor.
Por último creamos un archivo en el servidor para comprobar que lo vemos en el cliente:

![captura10](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica6/img/Captura10.PNG)

Como lo he hecho con permisos de solo lectura solo el servidor podrá crear y modificar los archivos,
el cliente solo podrá leerlos.

-------------------------------------------------------------------------------------------------------
!!Importante:
Cuando reiniciemos la máquina el dispositivo creado con el nombre /dev/md0 será renombrado por Linux
y pasará a llamarlo /dev/md127.