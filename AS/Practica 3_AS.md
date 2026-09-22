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
Hay que comparar los modos de descubrimiento UDP o TCP, con y sin elevacion de privilegios.
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
Se desactiva el firewall con: `./firewall_down` y para activarlo `./firewall_up`
