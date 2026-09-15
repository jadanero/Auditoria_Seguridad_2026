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

