# Práctica 4 - Asegurar la Granga Web

##1. Instalar un certificado SSL autofirmado para configurar el acceso por HTTPS

Primero activamos el módulo SSL de Apache y reiniciamos el servidor:

	`sudo a2enmod ssl`

	`service apache2 restart`

	![Captura e2enmod](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura1.PNG)

Creamos la carpeta para guardar los certificados:

	`mkdir /ect/apache2/ssl`

Creamos el certificado con openssl:

	`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout
	/etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt`

Rellenamos una serie de datos:

	![Captura capturaOpenssl](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura2.PNG)

Editamos del archivo de configuración del sitio:

	`nano /etc/apache2/sites-available/default-ssl.conf`

	agregamos las siguientes lineas debajo de SSLEngine on:

	`SSLCertificateFile /etc/apache2/ssl/apache.crt
	 SSLCertificateKeyFile /etc/apache2/ssl/apache.key`

Activamos el sitio default--ssl y reiniciamos apache:
	
	`a2ensite default-ssl`
	`service apache2 reload`

Para probar que ha funcionado accedemos desde un navegador de una máquina conectada a:
	https://ipM

	![CapturaHTTPSM1](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura3.PNG)
	![CapturaHTTPSM1](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura3.1.PNG)

Ahora pasamos el certificado a la máquina2:

	Desde la máquina2:
	`mkdir /etc/apache2/ssl`
	`scp usuario@IPM1:/etc/apache2/ssl/* /etc/apache2/ssl`

Activamos ssl en M2:

	`sudo a2enmod ssl`
	`service apache2 restart`
	`a2ensite default-ssl`
	`service apache2 reload`

Comprobamos que funciona:

	![CaptURAHTTPSM2](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura4.PNG)

Pasamos el certificado a la máquina balanceadora:

	Desde la máquina balanceadora:

	`mkdir ssl`
	`scp usuario@IPM1:/etc/apache2/ssl/* /ssl`

	Configuramos nginx para tráfico https:

	`nano /etc/nginx/conf.d/default.conf`

	![Captura Configuración NGINX](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura5.PNG)



##2-Cortafuegos[iptables]

configuración típica de servidor web(se puede hacer en el balanceador o en una nueva máquina(más nota))

En esta parte vamos a configurar un cortafuegos básico con IPTABLES, bloqueando todo el tráfico salvo los puertos:
 80[http], 443[https], 22[ssh].
Primero lo haremos para M1.

Para configurar iptables creamos un script con el siguiente contenido:

`	# (1) Eliminar todas las reglas (configuración limpia)
	iptables -F
	iptables -X
	iptables -Z
	iptables -t nat -F

	# (2) Política por defecto: denegar todo el tráfico
	iptables -P INPUT DROP
	iptables -P OUTPUT DROP
	iptables -P FORWARD DROP

	# (3) Permitir cualquier acceso desde localhost (interface lo)
	iptables -A INPUT  -i lo -j ACCEPT
	iptables -A OUTPUT -o lo -j ACCEPT

	# (4) Abrir el puerto 22 para permitir el acceso por SSH
	iptables -A INPUT  -p tcp --dport 22 -j ACCEPT
	iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

	# (5) Abrir los puertos HTTP (80) de servidor web
	iptables -A INPUT  -p tcp --dport 80 -j ACCEPT
	iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT

	# (6) Abrir los puertos HTTPS (443) de servidor web
	iptables -A INPUT  -p tcp --dport 443 -j ACCEPT
	iptables -A OUTPUT -p tcp --sport 443 -j ACCEPT	`

Una vez tenemos el script creado por ejemplo: script_firewall.sh le damos permisos de ejecución y lo copiamos 
a `/etc/rc.local` para que se ejecute cada vez que levantemos el sistema. 
Para comprobar que funciona lanzamos un ping a M1, este debería estar bloqueado(no está abierto el tárico ICMP). 
Ahora visitamos M1 desde un navegador y vemos que funciona, por tanto el cortafuegos está activo.

	![CapturaFirewallM1](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura6.PNG)


3-Máquina Firewall[Parte opcional]

Para esta parte he creado una nueva máquina virtual con un Ubuntu Server vacío. En esta hay que configurar iptables
a modo de firewall, pero con el añadido de redireccionar el tráfico a nuestra máquina balanceadora.
Para ello creamos un script como el siguiente:

`	#!/bin/bash
	#script Firewall

	#Habilitar en el kernel el redireccionamiento
	echo 1 > /proc/sys/net/ipv4/ip_forward

	# Configurar IPTABLES
	# (1) Eliminar todas las reglas (configuración limpia)
	iptables -F
	iptables -t nat -F
	iptables -X
	iptables -t nat -X

	# (2) Política por defecto: bloquear todo
	iptables -P INPUT DROP
	iptables -P OUTPUT DROP

	# (3) Permitir cualquier acceso desde localhost (interface lo)
	iptables -A INPUT  -i lo -j ACCEPT
	iptables -A OUTPUT -o lo -j ACCEPT

	# (4) Abrir el puerto 22 para permitir el acceso por SSH
	iptables -A INPUT  -p tcp --dport 22 -j ACCEPT
	iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

	# (5) Abrir los puertos HTTP (80) de servidor web
	iptables -A INPUT  -p tcp --dport 80 -j ACCEPT
	iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT

	# (6) Abrir los puertos HTTPS (443) de servidor web
	iptables -A INPUT  -p tcp --dport 443 -j ACCEPT
	iptables -A OUTPUT -p tcp --sport 443 -j ACCEPT	

	# (7) Redireccionar el tráfico al balanceador de carga
	iptables -t nat -A PREROUTING  -p tcp --dport 80 -j DNAT --to-destination IPMB:80
	iptables -t nat -A POSTROUTING -p tcp -d IPMB --dport 80 -j SNAT --to-source IPMF
	iptables -t nat -A PREROUTING  -p tcp --dport 443 -j DNAT --to-destination IPMB:443
	iptables -t nat -A POSTROUTING -p tcp -d IPMB --dport 443 -j SNAT --to-source IPMF
`

Donde IPMB: Dirección IP de la máquina balanceadora e IPMF: Dirección IP de la máquina firewall.
Una vez heco el script le damos permisos de ejecución y ejecutamos, comprobado que funciona lo 
movemos a `/etc/rc.local` para que se ejecute cuando la máquina arranque.
Comprobamos que funciona:

	![CapturaMaquinaFirewall](https://github.com/campoy94/SWAP/blob/master/Practicas/Practica4/Captura8.PNG)

Como vemos nos lleva a los servidores y hace el balanceo correctamente, pero no podemos hacer ping
por tanto el firewall está funcionando correctamente.

