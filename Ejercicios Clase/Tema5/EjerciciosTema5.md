Ejercicio T5.1:
Buscar información sobre cómo calcular el número de conexiones por segundo.
Para empezar, podéis revisar las siguientes webs:
http://bit.ly/1ye4yHz
http://bit.ly/1PkZbLJ

Para calcular en nº de conexiones por segundo necesitamos obtener los Handled connections(conexiones manejadas) y los
Hanled request(solicitudes manejadas) de nuestro servidor, obtener estos datos puede ser distinto, entre otras cosas
depende de si solo estamos usando un apache, o varios servidores manejados mediante un balanceador.
Para obtener el número de conexiones por segundo hacemos la división: 
	handles requests/handled connections


Ejercicio T5.2:
Revisar los análisis de tráfico que se ofrecen en:
http://bit.ly/1g0dkKj
Instalar wireshark y observar cómo fluye el tráfico de red en uno de los servidores web mientras se le hacen peticiones
HTTP.

Aqui vemos una captura del tráfico producido de una petición de mi máquina anfitriona a uno de mis servidores de prácticas.
He aplicado filtros para obtener solo el tráfico generado de una petición HTTP:

![captura Wireshark](https://github.com/campoy94/SWAP/tree/master/Ejercicios%20Clase/Tema5/ejercicio2T5.PNG)

Ejercicio T5.3:
Buscar información sobre características, disponibilidad para diversos SO, etc de herramientas para monitorizar las
prestaciones de un servidor.

	-top(Linux): es una herramienta para monitorizar los recursos que estamos utilizando en el sistema. 
	Podemos destacar la línea %Cpu(s) en la que vemos el porcentaje de CPU que se está utilizando.
	Kib Mem: la memoría RAM que estamos utilizando, y la que tenemos libre.
	Kib Swap: El total de memoria disponible para hacer swaping, el utilizado  el libre.
	Luego teneos una lista de los programas activos en el momento con diferente información como su PID,
	el usuario que lo lanzó, sus consumos CPU, RAM...
	Una herramienta similar mas moderna es el htop, que tiene colores y queda muy chulo a la hora de enseñarlo.

	-vmstat(Linux): es una herramienta para monitorizar recursos del sistema, podemos ver el uso de memoria RAM, SWAP,
	Input/Output, paginación, interrupciones...
	Hay diferentes modos de lanzarlo, si simplemente ponemos vmstat nos muestra los datos en el momento de lanzarlo.
	Otra forma de lanzarlo es vmstat -n donde n es el número de segundos en los que nos lanzará una muestra.

	-netstat(Linux y Windows): es una herramienta que muestra un listado de las conexiones activas en la máquina,
	tanto entrantes como salientes.

	-Administrador de tareas(Windows): es una aplicación clasica en windows, que cuenta con interfaz gráfica 
	para monitorizar el rendimiento de la máquina, nos muestra datos del consumo de CPU, RAM, Disco, Red...
	Podemos controlar el consumo de esto por Procesos, en general, por usuarios...