## Práctica 1.4 — Configuración del servidor DNS

### 1. LinuxBackup: instalación y configuración de dnsmasq

Se instalaron el servidor DNS **dnsmasq** y las herramientas de consulta DNS:

```bash
sudo apt update
sudo apt install dnsmasq dnsutils
```

Se modificó `/etc/hosts` para asociar las direcciones de los equipos con sus nombres completos y cortos:

```text
192.168.57.10  linuxbackup.blue.local    linuxbackup
192.168.57.11  windowsserver.blue.local  windowsserver
192.168.56.10  linuxserver.blue.local    linuxserver
192.168.56.101 linuxclient.blue.local    linuxclient
192.168.56.102 windowsclient1.blue.local windowsclient1
192.168.56.254 linuxrouter1.blue.local   linuxrouter1
192.168.57.254 linuxrouter2.blue.local   linuxrouter2
```

La entrada original de LinuxBackup en `127.0.1.1` se sustituyó por su dirección de red, para que los clientes obtengan una IP accesible.

Se creó `/etc/dnsmasq.d/blue-local.conf`:

```ini
listen-address=127.0.0.1,192.168.57.10
bind-interfaces

domain=blue.local
local=/blue.local/

no-resolv
server=10.0.2.3
```

Esta configuración permite atender consultas locales y de la red, resolver los nombres de `blue.local` desde `/etc/hosts` y reenviar las consultas externas a `10.0.2.3`. La opción `no-resolv` evita que dnsmasq utilice su propio servicio como servidor de reenvío cuando se cambie `/etc/resolv.conf`.

Se validó la sintaxis y se reinició el servicio:

```bash
sudo dnsmasq --test --conf-file=/etc/dnsmasq.d/blue-local.conf
sudo systemctl restart dnsmasq
```

**Comprobación:** la validación devolvió `syntax check OK`; el servicio quedó activo y escuchando en el puerto 53 TCP y UDP. Se verificaron las consultas:

```bash
dig @192.168.57.10 linuxserver.blue.local
dig @192.168.57.10 debian.org
```

Ambas devolvieron `NOERROR`. La primera obtuvo `192.168.56.10` y la segunda las direcciones del dominio externo.

### 2. DNS utilizado por LinuxBackup, LinuxClient y LinuxServer

En las tres máquinas se configuró `/etc/resolv.conf` con:

```text
search blue.local
nameserver 192.168.57.10
```

Así utilizan exclusivamente LinuxBackup como DNS. El dominio de búsqueda permite resolver nombres cortos dentro de `blue.local`.

También se sustituyeron los DNS externos de `/etc/network/interfaces` por:

```text
dns-nameservers 192.168.57.10
dns-search blue.local
```

Se conservaron las direcciones IP y las puertas de enlace. No fue necesario reiniciar la red para aplicar el cambio de `/etc/resolv.conf`.

### 3. Comprobaciones por máquina

| Máquina | Comprobación | Resultado |
|---|---|---|
| LinuxBackup | `dig debian.org` | La consulta utilizó `192.168.57.10` como servidor DNS. |
| LinuxClient | `dig linuxserver.blue.local` y `dig debian.org` | Resolución interna y externa correcta mediante `192.168.57.10`, desde otra subred. |
| LinuxClient | `getent hosts linuxserver.blue.local` y `getent hosts linuxserver` | Ambos nombres devolvieron `192.168.56.10`. |
| LinuxServer | `getent hosts linuxbackup.blue.local` y `getent hosts linuxbackup` | Ambos nombres devolvieron `192.168.57.10`. |
| LinuxServer | `getent hosts debian.org` | Se obtuvieron direcciones IPv6 del dominio externo. |

### 4. Alcance realizado

El servidor DNS está operativo y se ha configurado su uso en **LinuxBackup, LinuxClient y LinuxServer**. Queda pendiente cambiar el DNS de **LinuxRouter1, LinuxRouter2 y los equipos Windows**, además de verificar que la configuración se mantiene tras reiniciar.