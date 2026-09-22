## Instalación
Para instalar docker haremos:
```
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker --now
```

Luego arrancamos el docker con `./start.sh`

Cuando la arrancamos deberemos de cumplir lo que se nos pide en el startup:
- [ ] Queremos descubrir la `ip_target`
- [ ] Escanear los puertos TCP/UDP abiertos y enumerar los servicios corriendo
- [ ] Modificar las reglas del firewall para ver como influye en el escaneo

Ahora vamos a familiarizarnos con las herramientas típicas de descubrimiento activo.
`gobuster`y `seclist`
## C2
Hemos hecho en el bridge que ha creado el docker un `arp-scan`
```
sudo arp-scan -I <Interfaz_docker> --localhost
```
con esto hemos obtenido la `ip_target`

## C3
Utilizamos la herramienta `nmap` y la herramienta `ping` a la misma `ip` para comprobar las diferencias:
```
sudo nmap -PE -sn <IP_TARGET>
```

Cuando hacemos `ping` vemos los ICMP Request y los ICMP Reply. Sin embargo cuando hacemos `nmap` ,de este modo, analiza como si fuera un `arp-scan`, para que trabaje en capa 3 y compararlo con un ping hay que añadir:
```
sudo nmap -PE -sn <IP_TARGET> -disable-arp-ping --send-ip
```

## C4
Hay que comparar los modos de descubrimiento UDP o TCP, con y sin elevación de privilegios.
`TCP`:
```
sudo nmap -PS -sn <IP_TARGET> -disable-arp-ping --send-ip
```
`TCP+Reset`:
```
sudo nmap -PA -sn <IP_TARGET> -disable-arp-ping --send-ip
```
`UDP`:
```
sudo nmap -PU -sn <IP_TARGET> -disable-arp-ping --send-ip
```

## C5
Realizar los mismos escaneos TCP y UDP pero con el firewall desactivado
Se desactiva o activar el firewall: 
```
./change_iptables_docker.sh firewall_down.sh | firewall_up.sh
```

TCP connect scan:
```
sudo nmap -sT <IP_TARGET> -disable-arp-ping --send-ip
```
TCP SYN scan:
```
sudo nmap -sS <IP_TARGET> -disable-arp-ping --send-ip
```
UDP:
```
sudo nmap -sU <IP_TARGET> -p22,80 -disable-arp-ping --send-ip
```

Con el firewall bajado podemos comprobar que hay dos servicios activados en el puerto 80 y en el 22.

Ahora activamos le firewall y repetimos los mismos comandos.

Podemos comprobar que en TCP ambos nos tiran las conexiones y están en ignore state. En UDP nos resuelve y nos da los servicios en el puerto 80 y en el 22.

## C6
Con el firewall activado funcionan `WindowScan` y `AckScan`, `W`y `A` respectivamente

#### Spoofing
Y podemos hacer `mac spoofing` con este comando:
```
sudo nmap -spoofmac 00:11:22:33:44:55 -Pn -n <IP_TARGET>
```
Para poder hacer `ip spoofing` hacemos:
```
sudo nmap -e <interfaz> -Pn -S <IP_DUMMY> <IP_TARGET> -disable-arp-ping --send-ip
```
NO NOS FUNCIONA

#### Decoy
Un decoy es meter distintas direcciones IP para esconder la tuya propia
```
sudo nmap -D <{DECOY_IPs,ATTACKER_IP}> <IP_TARGET>
```
En el `wireshark` tenemos peticiones con `IP_origen` las decoy y las de nuestra maquina

## C7
Queremos descubrir el sistema operativo. Por el `ping` que sabemos que nos ha salido un `TTL` de 64 por lo que intuimos que es un sistema `Linux`. Tambien lo hemos comprobado de forma mas fina así:
```
sudo nmap -A <IP_TARGET>
```

## C8




porque nos sale como `ls -l /usr/bin/nmap`
sacar el kernel de la maquina atacada ubuntu13.5
script