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
```

Vamos a probar a hacer `curl` a los servicios `http` y `https`:

```
$ curl 192.168.10.23:8080
AS_{http8080_COMPROBARHTTPS}
```
Nos responde con una #flag 



