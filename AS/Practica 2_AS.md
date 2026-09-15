### Scope (alcance)
Delimita los sistemas bajo examen. Puede ser muy amplio (todos los servicios bajo un dominio principal) o muy reducido (una aplicación específica). El scope es la zona de juego autorizada. Cualquier actividad fuera del scope no solo carece de valor en el programa, sino que puede considerarse intrusión ilegal.

### Rules of Engagement (ROE):
son las reglas de compromiso que definen cómo debe interactuar el investigador con el sistema.

> Respetar scope y ROE no es opcional. Saltarse estos límites implica perder la protección legal del programa y puede conllevar responsabilidades penales. En este contexto, ¿qué diferencia a un hacker ético de un ciberdelincuente? ¿Sus conocimientos técnicos o sus decisiones éticas?

Sus decisiones éticas son la única diferencia.

Revisar estas paginas:
https://www.hackerone.com/
https://www.bugcrowd.com/
https://www.yeswehack.com/
https://www.openbugbounty.org/

## Elección
Hemos elegido: `Starbucks` http://www.starbucks.com/

Hemos encontrado la pagina: 
https://alumni.starbucks.com/login.php?forgetPassword=1

haciendo esta busqueda en google:
```
site:starbucks.com/ filetype:php
```

Haciendo esta busqueda:
```
site:starbucks.com/ "Brian Niccol" filetype:pdf
```

Hemos encontrado unos pdfs de la corporacion que podríamos explotar cambiando el directorio.
https://content-prod-live.cert.starbucks.com/binary/v2/asset/143-97402.pdf

## Herramientas OSINT en Kali Linux
```
whois starbucks.com
```

```
dig starbucks.com
```

```
nslookup starbucks.com
```

```
dnsenum starbucks.com
```
Con esta herramienta se saca información con fuerza bruta de registros DNS.

Hemos utilizado la herramienta `subfinder`:
```
subfinder -d starbucks.com > subfinder.txt
```

luego hemos buscado:
```
cat subfinder.txt | grep content-prod.live
```
Encontramos que tenemos un match con la web que habíamos encontrado anteriormente


## Siguiendo con...

https://content-prod-live.cert.starbucks.com/binary/v2/asset/143-97402.pdf
Las dos pruebas se han realizado en paralelo:
### Prueba 1
[[Script de prueba de directorios]]
Se han encontrado diferentes matches en los diferentes directorios recorridos. No en formato pdf pero si en json aunque fueran archivos `.pdf`

Se ven matches que descargan un archivo pdf que no es legible y si probamos su nombre en la url vemos un archivo json en plain text.


### Prueba 2
Para la prueba para descubrir que servidor es y qué sistemas emplea haremos:
```
dig +short content-prod-live.cert.starbucks.com
```
a lo que nos responde dos ips:
`23.211.15.153` y `23.211.15.150` probamos entonces:
```
sudo nmap -sS -p- -Pn -n -min-rate 3000 23.211.15.153
```

nos responde con servicios en la ip `23.211.15.153`:
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-15 19:36 +0200
Nmap scan report for 23.211.15.153
Host is up (0.014s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT     STATE  SERVICE
80/tcp   open   http
113/tcp  closed ident
443/tcp  open   https
2000/tcp open   cisco-sccp
5060/tcp open   sip
8883/tcp closed secure-mqtt

Nmap done: 1 IP address (1 host up) scanned in 43.88 seconds
```

Probamos:
```
sudo nmap -sCV -p80 23.211.15.153
```
Sabemos que tienen levantado un servidor akamaiGhost como mirror service.

Como no sabemos mucho con lo que hemos hecho vamos a probar algo más genérico:
```
nc 23.211.15.153 80
```


```
HTTP/1.0 408 Request Time-out
Server: AkamaiGHost
Mime-Version: 1.0
Date: Tue, 15 Sep 2026 18:02:21 GMT
Content-Type: text/html
Content-Length: 314
Expires: Tue, 15 Sep 2026 18:02:21 GMT

<HTML><HEAD>
<TITLE>Request Timeout</TITLE>
</HEAD><BODY>
<H1>Request Timeout</H1>
The server timed out while waiting for the browser's request.<P>

Reference&#32;&#35;2&#46;99d73b17&#46;1789495341&#46;0
<P>https&#58;&#47;&#47;errors&#46;edgesuite&#46;net&#47;2&#46;99d73b17&#46;1789495341&#46;0</P>
</BODY></HTML>
```

