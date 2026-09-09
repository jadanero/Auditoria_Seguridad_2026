# Checkpoint - C1
**ICMP**
[[ICMP]] significa Internet Control Message Protocol. Es un protocolo de red que se utiliza principalmente para enviar mensajes de control y diagnóstico entre dispositivos.

**Por qué y a quién contesta el destino 8.8.8.8**
Nos contesta(ICMP Reply) a nosotros porque le hemos hecho un ICMP Request. Es google.com

**Sabría decir cuál es el siguiente salto para su máquina virtual**
`traceroute 8.8.8.8` nos da el descubrimiento del siguiente salto que es `10.6.25.254`

**Sabría decir qué es esta dirección IP a la que ha hecho ping**
`dns.google` lo vemos en el `traceroute`

**Existe alguna dirección alternativa que ofrezca algo similar**
si pongo `1.1.1.1`

# Checkpoint - C2
```
curl -I http://www.google.es/
```

**Qué versión del protocolo HTTP se está usando**
`HTTP 1.1`

**Cómo podría cambiarlo en las opciones de curl**
```
curl --http1.0 -I <url>
```

**La comunicación está siendo cifrada**
En principio no está cifrada porque tiene TLS

**Qué protocolo adicional se encarga del cifrado**
TLS

**A qué puerto se realiza la conexión cuando es cifrada**
443

# Checkpoint - C3
```
ip a
ip route
curl ifconfig.me
```
**Puede identificar las direcciones IP y MAC de su gateway por defecto**
con `arp -a` la MAC y la IP

**Puede saber qué puerto tiene asignado en la tabla NAT de su gateway por defecto**


**Tendría sentido que hubiera varias capas de NAT hasta salir a internet**
Si para tener una red privada

**Qué problemas podría tener esto**
Se pierde la identidad

# Checkpoint - C4
**Qué ocurre si por algún motivo en su tabla ARP aparece una dirección MAC distinta a la real del equipo al que quiere conectarse**
Le llega el paquete al que está suplantando la mac o se pierde

**Podría un atacante aprovechar ARP para engañar a su máquina y redirigir el tráfico hacia él**
Si, se llama ARP spoofing

**Si varios hosts responden a la misma petición ARP con diferentes MAC, ¿cómo cree que se comportaría su sistema?**
El último que llegue se queda guardado. Pero el primero que llegue se le manda el paquete original.

# Checkpoint - C5
**Servidor**
```
#!/ usr/bin/env python3
import socket
# Direcci ón IP y puerto de escucha
HOST = " 127.0.0.1 "
PORT = 12345
# Crear socket TCP
with socket.socket(socket.AF_INET , socket.SOCK_STREAM ) as s : # IPv4 , TCP
	s.bind((HOST,PORT))
	s.listen(1)
	print(f"\n[*] Servidor escuchando en { HOST }:{ PORT }")
	# Aceptar conexión del cliente
	conn , addr = s.accept()
	with conn:
		print(f"\n[+] Conexión de {addr}")
		# Recibir mensaje del cliente y responder
		data = conn.recv(1024)
		print(f"\n[+] Mensaje recibido del cliente: {data.decode ()}")
		conn.sendall(f"Adiós\n".encode())
```
**Cliente**
```
nc localhost 12345
```

**En qué orden se inicia y se cierra la conexión**
Empieza el cliente y responde el servidor

**Qué pasaría si el cliente enviara más de 1024 bytes de datos** 
Se limita por la función a 1024 por mensaje

**Qué pasaría si varios clientes quisieran conectarse al servidor** 
Como se está ocupando el unico puerto que tiene abierto no funcionaría

# Checkpoint - C6
**Averigüe la opción que debe utilizar en netcat para enviar datos mediante UDP.**
```
nc -u <IP> <Port>
```

**Qué ocurre si el servidor no está levantado y el cliente intenta enviar un mensaje**
Te llega un mensaje de port unreacheable

**Se produce algún error inmediato como en TCP o el mensaje se pierde**
No porque es UDP

**Pruebe a lanzar varias veces el cliente seguidas: ¿recibe siempre respuesta?**
OKAY

**En el mismo orden en que las envió**
No tiene porqué llegar en orden porque es UDP.
# Checkpoint - C7
```
netstat -ntupla
```

**Qué hace cada opción del comando**
`-n` <IP:PUERTOS>
`-t` TCP
`-u` UDP
`-p` PID
`-l` los que estan en escucha
`-a` los sockets conectados

**Qué servicios aparecen escuchando**
`puerto 22` ssh
`puerto 3389` RDP

# Checkpoint - C8
Para forzar una resolución de nombres de dominio:
```
dig www.upna.es
```

**A qué dirección IP se ha enviado su consulta DNS**
`10.1.1.193`

**Analizando la naturaleza de la petición, ¿por qué cree que este servicio utiliza tradicionalmente UDP en lugar de establecer una conexión TCP completa?**
UDP por la velocidad. Es mas eficiente que TCP

**Observe el contenido del paquete capturado: ¿es capaz de leer el dominio www.upna.es en texto claro?**
Si. `www.upna.es`

**Qué implicaciones de privacidad y seguridad tiene que el historial de navegación DNS viaje sin cifrar**
Perder la privacidad o suplantación de identidad

**Conoce algún protocolo moderno que solucione este problema**
Si. DNS sobre TLS. DNS sobre HTTPS. DNSSEC.


