# Samba como servicio de usuarios centralizado

**Equipo principal:** LinuxBackup (`192.168.57.10`)  

**Estado:** servicios anteriores desactivados y aprovisionamiento del dominio iniciado. Su finalización todavía no se ha confirmado.
## 1. Objetivo y punto de partida
Esta práctica cambia la autenticación centralizada de OpenLDAP a un dominio Active Directory gestionado por Samba. El objetivo es que los equipos Linux y Windows puedan utilizar las mismas cuentas de dominio.

LinuxBackup actuará como controlador de dominio y proporcionará el directorio, el DNS del dominio y la autenticación mediante Kerberos. Hasta este momento también prestaba servicios con OpenLDAP, dnsmasq y Samba como servidor de archivos.

Los nombres que utilizamos son:

| Parámetro                | Valor                    |
| ------------------------ | ------------------------ |
| Nombre del servidor      | `linuxbackup`            |
| Nombre completo          | `linuxbackup.blue.local` |
| Dirección IP             | `192.168.57.10/24`       |
| Dominio DNS              | `blue.local`             |
| Realm de Kerberos        | `BLUE.LOCAL`             |
| Nombre corto del dominio | `BLUE`                   |

## 2. Punto 1.1 — Desactivación de los servicios anteriores
### 2.1. Revisión inicial
En LinuxBackup consultamos el estado de los servicios y si estaban configurados para arrancar automáticamente:
```bash
systemctl is-active smbd nmbd samba-ad-dc dnsmasq slapd
systemctl is-enabled smbd nmbd samba-ad-dc dnsmasq slapd
```
Detectamos que `smbd`, `nmbd` y `slapd` estaban activos y habilitados. Los primeros resultados sobre dnsmasq fueron confusos; posteriormente comprobamos directamente que estaba funcionando y atendiendo consultas DNS.
### 2.2. Desactivación de Samba anterior y OpenLDAP
Ejecutamos:
```bash
sudo systemctl disable --now smbd nmbd slapd
```
`--now` detiene los servicios en ese momento y `disable` evita que se inicien automáticamente al reiniciar. Esto no elimina sus datos ni sus archivos de configuración.
Comprobamos el resultado con:
```bash
systemctl is-active smbd nmbd slapd
systemctl is-enabled smbd nmbd slapd
```
Se confirmó que estaban inactivos y deshabilitados. Un resultado `not-found` para `slapd` se debía a haber escrito mal el nombre, y se corrigió la consulta.
### 2.3. Revisión del montaje remoto de backups
En LinuxServer revisamos el recurso utilizado anteriormente para las copias de seguridad:
```bash
findmnt /opt/backups_remote
grep -nE 'backups_remote|cifs' /etc/fstab
```
Ninguno devolvió salida. No encontramos un montaje en esa ruta ni líneas que coincidieran con esos términos en `/etc/fstab`, por lo que no modificamos este archivo. Esta comprobación no equivale a revisar todos los posibles recursos remotos o tareas de backup.
### 2.4. Comprobación y desactivación de dnsmasq
En `LinuxBackup` comprobamos la resolución externa y el DNS utilizado:
```bash
getent hosts deb.debian.org
cat /etc/resolv.conf
```
La resolución funcionaba y el archivo contenía:
```text
search blue.local
nameserver 192.168.57.10
```
Como el servidor utilizaba su propia IP para resolver nombres, revisamos qué proceso ocupaba el puerto DNS:
```bash
sudo ss -lntup 'sport = :53'
```
La salida mostró `dnsmasq` escuchando por TCP y UDP en `127.0.0.1:53` y `192.168.57.10:53`. Por tanto, seguía activo y había que detenerlo para liberar el puerto que utilizará el DNS de Samba.
Ejecutamos:
```bash
sudo systemctl disable --now dnsmasq
```
Y realizamos las comprobaciones:
```bash
systemctl is-active dnsmasq
systemctl is-enabled dnsmasq
sudo ss -lntup 'sport = :53'
```
Se confirmó la realización de este paso, aunque no se adjuntó la salida final de estos comandos.
### 2.5. DNS temporal para instalar los paquetes
Tras detener dnsmasq, la IP del propio servidor ya no debía utilizarse como DNS hasta que estuviera funcionando Samba. Para mantener la resolución externa editamos:
```bash
sudo nano /etc/resolv.conf
```
El contenido quedó así:
```text
search blue.local
nameserver 8.8.8.8
```
Comprobamos de nuevo:
```bash
getent hosts deb.debian.org
```
La resolución volvió a devolver direcciones, confirmando que podíamos continuar con la instalación. Esta configuración es temporal: cuando el DNS de Samba funcione, LinuxBackup volverá a utilizar `192.168.57.10`.
## 3. Punto 1.2 — Instalación y preparación del controlador de dominio
### 3.1. Instalación de paquetes
En LinuxBackup actualizamos la información de los repositorios e instalamos los paquetes indicados:

  

```bash

sudo apt update

sudo apt install samba-ad-dc samba-ad-provision samba-common-bin krb5-user dnsutils chrony

```

  

| Paquetes | Finalidad |

|---|---|

| `samba-ad-dc`, `samba-ad-provision` | Controlador de dominio e inicialización de su estructura |

| `samba-common-bin` | Herramientas de administración de Samba |

| `krb5-user` | Herramientas cliente de Kerberos, como `kinit` y `klist` |

| `dnsutils` | Comprobaciones DNS |

| `chrony` | Sincronización de la hora, necesaria para Kerberos |

  

Instalar chrony no implica que ya hayamos verificado su sincronización; esa comprobación sigue pendiente.

  

### 3.2. Configuración inicial de Kerberos y corrección de una errata

  

Durante la instalación se introdujeron estos valores:

  

| Pregunta | Valor |

|---|---|

| Realm predeterminado | `BLUE.LOCAL` |

| Servidor Kerberos | `linuxbackup.blue.local` |

| Servidor administrativo | `linuxbackup.blue.local` |

  

Revisamos el archivo generado:

  

```bash

cat /etc/krb5.conf

```

  

Encontramos una errata en `admin_server`: se había escrito `linuxbakcup.blue.local`. La corregimos con:

  

```bash

sudo nano /etc/krb5.conf

```

  

El bloque corregido es:

  

```ini

[realms]

    BLUE.LOCAL = {

        kdc = linuxbackup.blue.local

        admin_server = linuxbackup.blue.local

    }

```

  

Posteriormente sustituiremos esta configuración por la que genere Samba durante el aprovisionamiento. Esa sustitución todavía no se ha realizado.

  

### 3.3. Comprobación del nombre y la dirección del servidor

  

Revisamos:

  

```bash

hostname -f

ip -br -4 addr

cat /etc/hosts

```

  

El nombre completo era `linuxbackup.blue.local` y la interfaz `enp0s3` estaba activa con `192.168.57.10/24`. Ambos valores coinciden con el servidor que queremos aprovisionar.

  

### 3.4. Preparación de `/etc/hosts`

  

El archivo contenía las direcciones de todos los equipos de la maqueta. Para que los nombres de los demás equipos pasen a gestionarse mediante el DNS de Samba, guardamos una copia y retiramos sus entradas:

  

```bash

sudo cp -a /etc/hosts /etc/hosts.pre-samba-ad

sudo nano /etc/hosts

```

  

Conservamos las entradas de localhost, del propio servidor y las líneas de IPv6:

  

```text

127.0.0.1 localhost

192.168.57.10 linuxbackup.blue.local linuxbackup

  

# The following lines are desirable for IPv6 capable hosts

::1 localhost ip6-localhost ip6-loopback

ff02::1 ip6-allnodes

ff02::2 ip6-allrouters

```

  

Se eliminaron las entradas de WindowsServer, LinuxServer, LinuxClient, WindowsClient1, LinuxRouter1 y LinuxRouter2. Sus registros deberán existir después en DNS; eliminarlos de este archivo no los crea automáticamente en Samba.

  

Se indicó comprobar de nuevo `hostname -f` y se confirmó la realización del paso.

  

### 3.5. Conservación de la configuración anterior de Samba

  

Apartamos el archivo de configuración anterior:

  

```bash

sudo mv -i /etc/samba/smb.conf /etc/samba/smb.conf.pre-ad

```

  

Así conservamos la configuración del servidor de archivos y dejamos libre la ruta para que el aprovisionamiento genere la configuración del controlador de dominio. Esto no borra los directorios compartidos, pero sus recursos no quedan automáticamente trasladados a la nueva configuración.

  

### 3.6. Inicio del aprovisionamiento

  

Ejecutamos:

  

```bash

sudo samba-tool domain provision --use-rfc2307 --interactive

```

  

El aprovisionamiento crea las bases de datos, la estructura inicial y la configuración del dominio. `--use-rfc2307` incorpora soporte para atributos Unix; no asigna por sí solo todos los UID y GID que pudieran necesitarse posteriormente.

  

Se confirmó que las preguntas se rellenaron según el enunciado:

  

| Parámetro | Valor indicado en la práctica |

|---|---|

| Realm | `BLUE.LOCAL` |

| Domain | `BLUE` |

| Server Role | `dc` |

| DNS backend | `SAMBA_INTERNAL` |

| DNS forwarder | `8.8.8.8` |

| Contraseña de Administrator | La indicada en el enunciado, introducida igual en ambas preguntas |

  

El DNS integrado resolverá los nombres del dominio y utilizará el reenviador para consultar dominios externos. Kerberos proporcionará la autenticación mediante tickets.

  

**Al cerrar esta memoria, el proceso se había iniciado, pero todavía no se había recibido su salida final. No damos por confirmado que el dominio haya terminado de crearse correctamente.**

  

## 4. Estado actual y comprobaciones pendientes

  

| Elemento | Estado documentado |

|---|---|

| Samba anterior y OpenLDAP | Detenidos y deshabilitados |

| dnsmasq | Desactivación confirmada por el usuario |

| DNS temporal externo | Resolución comprobada con `deb.debian.org` |

| Paquetes del controlador de dominio | Instalación realizada; se continuó al aprovisionamiento |

| Nombre completo e IP | Comprobados y correctos |

| `/etc/hosts` | Preparado, con copia del original |

| Configuración Samba anterior | Apartada antes del aprovisionamiento |

| Aprovisionamiento | Iniciado; resultado final pendiente |

| Servicio `samba-ad-dc` | Arranque y funcionamiento todavía no comprobados |

| Autenticación Kerberos | Todavía no comprobada |

| Clientes Linux y Windows | Todavía no incorporados al dominio |

  

Los siguientes pasos, que aún no se han ejecutado, serán confirmar el resultado del aprovisionamiento, utilizar el archivo de Kerberos generado por Samba, habilitar el controlador de dominio y volver a configurar el DNS del servidor con su propia IP. Después comprobaremos los registros DNS de LDAP y Kerberos, la obtención de un ticket y la creación del usuario de dominio `dummyadmin3`.

  

## 5. Archivos modificados o conservados

  

| Archivo | Cambio |

|---|---|

| `/etc/resolv.conf` | DNS temporal `8.8.8.8` |

| `/etc/krb5.conf` | Corrección del nombre en `admin_server` |

| `/etc/hosts` | Retirada de entradas de otros equipos |

| `/etc/hosts.pre-samba-ad` | Copia del archivo hosts anterior |

| `/etc/samba/smb.conf.pre-ad` | Configuración anterior de Samba conservada |

  

Esta memoria recoge lo realizado y confirmado durante la sesión. No incluye todavía las pruebas finales del dominio ni la configuración de clientes.