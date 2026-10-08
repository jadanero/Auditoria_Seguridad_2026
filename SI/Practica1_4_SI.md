## Configuración del servidor DNS
#### 1. LinuxBackup: instalación y configuración de `dnsmasq`

Se instalaron el servidor DNS **`dnsmasq`** y las herramientas de consulta DNS:

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

Esta configuración permite atender consultas locales y de la red, resolver los nombres de `blue.local` desde `/etc/hosts` y reenviar las consultas externas a `10.0.2.3`. La opción `no-resolv` evita que `dnsmasq` utilice su propio servicio como servidor de reenvío cuando se cambie `/etc/resolv.conf`.

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

#### 2. DNS utilizado por LinuxBackup, LinuxClient y LinuxServer

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

#### 3. Comprobaciones por máquina

| Máquina | Comprobación | Resultado |
|---|---|---|
| LinuxBackup | `dig debian.org` | La consulta utilizó `192.168.57.10` como servidor DNS. |
| LinuxClient | `dig linuxserver.blue.local` y `dig debian.org` | Resolución interna y externa correcta mediante `192.168.57.10`, desde otra subred. |
| LinuxClient | `getent hosts linuxserver.blue.local` y `getent hosts linuxserver` | Ambos nombres devolvieron `192.168.56.10`. |
| LinuxServer | `getent hosts linuxbackup.blue.local` y `getent hosts linuxbackup` | Ambos nombres devolvieron `192.168.57.10`. |
| LinuxServer | `getent hosts debian.org` | Se obtuvieron direcciones IPv6 del dominio externo. |

#### 4. Alcance realizado

El servidor DNS está operativo y se ha configurado su uso en **LinuxBackup, LinuxClient y LinuxServer**. Queda pendiente cambiar el DNS de **LinuxRouter1, LinuxRouter2 y los equipos Windows**, además de verificar que la configuración se mantiene tras reiniciar.

## Directorios personales mediante Samba

En **LinuxBackup** se crearon las cuentas Samba de los usuarios Linux existentes:
```bash
sudo smbpasswd -a alice 
contraseña = alice
sudo smbpasswd -a bob 
contraseña = bob
sudo smbpasswd -a dummyadmin 
contraseña = palangana2026.ABC
```

Se modificó la sección `[homes]` de `/etc/samba/smb.conf` para permitir acceso autenticado y escritura en el directorio personal de cada usuario:

```ini
[homes]
   comment = Home Directories
   browseable = no
   read only = no
   guest ok = no
   valid users = %S
   create mask = 0700
   directory mask = 0700
```

`valid users = %S` restringe cada recurso a su usuario. Las máscaras limitan los permisos de los nuevos archivos y directorios al propietario. Se verificó la configuración con `testparm` y se recargó Samba:

```bash
sudo testparm -s
sudo systemctl reload smbd
```

Desde **LinuxClient**, mediante `smbclient`, se comprobó que **alice, bob y dummyadmin** podían autenticarse y subir archivos a sus respectivos directorios.

En LinuxBackup se comprobó que el archivo subido por Alice pertenecía a `alice:alice` y tenía permisos `700`.

Finalmente, se intentó acceder al recurso de Bob utilizando las credenciales de Alice:

```bash
smbclient //192.168.57.10/bob -U alice -c 'ls'
```

El servidor devolvió `NT_STATUS_ACCESS_DENIED`, confirmando la restricción de acceso entre usuarios.

## Directorio remoto para copias de seguridad

#### LinuxBackup

Se creó una cuenta dedicada al almacenamiento de backups, sin acceso interactivo, y se estableció su contraseña Samba:

```bash
sudo adduser --system --group --no-create-home --shell /usr/sbin/nologin backupweb
sudo smbpasswd -a backupweb
```

Se creó el directorio con acceso exclusivo para dicha cuenta:

```bash
sudo install -d -o backupweb -g backupweb -m 0700 /srv/samba/backups_linuxserver
```

Se añadió a `/etc/samba/smb.conf`:

```ini
[BackupsLinuxServer]
   path = /srv/samba/backups_linuxserver
   browseable = no
   read only = no
   guest ok = no
   valid users = backupweb
   create mask = 0600
   directory mask = 0700
```

Esta configuración permite almacenar las copias mediante autenticación como `backupweb`, con permisos privados. Se validó y recargó el servicio:

```bash
sudo testparm -s
sudo systemctl reload smbd
```

#### LinuxServer

Se instaló `cifs-utils` y se creó el punto de montaje:

```bash
sudo apt install cifs-utils
sudo mkdir -p /opt/backups_remote
```

Se guardaron las credenciales Samba en `/root/.smb-backupweb`, con propietario root y permisos `600`:

```text
username=backupweb
password=CONTRASEÑA
```

Se añadió a `/etc/fstab`:

```fstab
//192.168.57.10/BackupsLinuxServer /opt/backups_remote cifs credentials=/root/.smb-backupweb,vers=3.0,_netdev,nofail,x-systemd.automount 0 0
```

La entrada utiliza SMB 3.0 y monta el recurso al acceder al directorio. `_netdev` identifica su dependencia de la red y `nofail` permite arrancar aunque el servidor remoto no esté disponible.

Tras validar `/etc/fstab`, se recargó systemd, se retiró el montaje manual de prueba y se activó el montaje automático:

```bash
sudo systemctl daemon-reload
sudo umount /opt/backups_remote
sudo systemctl start opt-backups_remote.automount
```

#### Comprobación

Se creó un archivo desde LinuxServer:

```bash
echo "Prueba de escritura desde LinuxServer" | sudo tee /opt/backups_remote/prueba-linuxserver.txt
```

En LinuxBackup se verificó que el archivo estaba en el directorio compartido, pertenecía a `backupweb:backupweb` y tenía permisos `600`.

Después de activar el montaje automático, se accedió al archivo y se comprobó el montaje:

```bash
sudo cat /opt/backups_remote/prueba-linuxserver.txt
findmnt -t cifs --target /opt/backups_remote
```

Se obtuvo el contenido esperado y el recurso apareció montado como `cifs` con acceso de lectura y escritura.

## Automatización de las copias de seguridad de LinuxServer

#### 1. Objetivo y equipos utilizados

Se configuró una copia de seguridad diaria del directorio completo `/var/www/html` de **LinuxServer**, utilizando `rdiff-backup`.

Este directorio contiene el código PHP, las plantillas, las imágenes y los archivos JSON utilizados por la aplicación. Por tanto, la copia incluye tanto la página web como sus datos.

Las copias se almacenan en:

```text
/opt/backups_remote/web-prueba-sin-cache
```

Esta ruta está dentro del recurso Samba montado en `/opt/backups_remote/`. Los archivos se guardan realmente en **LinuxBackup**, en:

```text
/srv/samba/backups_linuxserver/web-prueba-sin-cache
```

Se conservó el nombre del repositorio utilizado durante las pruebas para mantener su historial.

#### 2. Instalación de rdiff-backup

En LinuxServer se instaló la herramienta:

```bash
sudo apt install rdiff-backup
rdiff-backup --version
```

La versión instalada fue **2.2.6**.

`rdiff-backup` mantiene una copia del estado más reciente del directorio y almacena información sobre los cambios anteriores. Esto permite recuperar archivos correspondientes a una fecha concreta sin guardar una copia completa independiente en cada ejecución.

#### 3. Problema encontrado al utilizar Samba

La primera copia se intentó realizar directamente sobre el recurso CIFS:

```bash
sudo rdiff-backup --api-version 201 backup /var/www/html /opt/backups_remote/web
```

La ejecución falló durante las comprobaciones iniciales del sistema de archivos de destino, con el error:

```text
OSError: [Errno 39] Directory not empty
```

El error afectaba al directorio temporal:

```text
web/rdiff-backup-data/rdiff-backup.tmp.0
```

Antes de copiar, rdiff-backup crea y elimina elementos temporales para comprobar las capacidades del destino. La copia no terminó correctamente y devolvió código de salida `1`.

Se probó también la API moderna mediante `--api-version 201`. Esta opción eliminó el aviso de compatibilidad de la interfaz anterior, pero no solucionó el error del directorio temporal.

#### 4. Desactivación de la caché de búsqueda CIFS

Se comprobó el valor de la caché:

```bash
cat /proc/fs/cifs/LookupCacheEnabled
```

El resultado fue `1`, indicando que estaba activada.

Esta caché pertenece al cliente CIFS de LinuxServer y conserva temporalmente información sobre la búsqueda y existencia de archivos en los recursos Samba. Su finalidad es reducir las consultas por red.

Se realizó una prueba deteniendo el montaje, desactivando la caché y volviendo a activar el recurso:

```bash
cd ~
sudo systemctl stop opt-backups_remote.automount opt-backups_remote.mount

echo 0 | sudo tee /proc/fs/cifs/LookupCacheEnabled

sudo systemctl start opt-backups_remote.automount
sudo ls /opt/backups_remote >/dev/null
```

Después se utilizó un repositorio nuevo:

```bash
sudo rdiff-backup --api-version 201 backup /var/www/html /opt/backups_remote/web-prueba-sin-cache
```

La copia terminó con código `0`. También se completaron correctamente las copias posteriores y la restauración.

Estos resultados apuntan a que la caché de búsqueda CIFS influía en el fallo. No permiten atribuirle con certeza toda la causa, porque también se remontó el recurso y se utilizó un destino nuevo.

**Alcance del ajuste:**

- No borra archivos ni desactiva toda la caché del sistema.
- Desactiva la caché de búsqueda de nombres de todos los montajes CIFS de LinuxServer.
- Puede aumentar las consultas al servidor y reducir el rendimiento.
- Es un cambio temporal que se pierde al reiniciar o recargar el módulo CIFS.
- Se adoptó como solución para este laboratorio; no es un requisito general de rdiff-backup.

#### 5. Script de backup

Se creó en LinuxServer el archivo:

```text
/usr/local/sbin/backup-web.sh
```

Con el siguiente contenido:

```bash
#!/bin/bash
set -euo pipefail
PATH=/usr/sbin:/usr/bin:/sbin:/bin
umask 077

ORIGEN="/var/www/html"
MONTAJE="/opt/backups_remote"
DESTINO="$MONTAJE/web-prueba-sin-cache"

# Evitar copias simultáneas.
exec 9>/run/lock/backup-web.lock
flock -n 9 || exit 0

echo "$(date -Is) Inicio del backup"

# Activar el montaje y comprobar que el destino es remoto.
timeout 30 ls "$MONTAJE" >/dev/null
if ! findmnt -rn -t cifs --mountpoint "$MONTAJE" >/dev/null; then
    echo "ERROR: el recurso Samba no está montado" >&2
    exit 1
fi

# Ajuste utilizado en las pruebas que funcionaron.
echo 0 > /proc/fs/cifs/LookupCacheEnabled

rdiff-backup --api-version 201 backup "$ORIGEN" "$DESTINO"

echo "$(date -Is) Backup terminado correctamente"
```

Se restringió su ejecución a root:

```bash
sudo chmod 700 /usr/local/sbin/backup-web.sh
```

#### 6. Explicación del script

##### Intérprete y control de errores

```bash
#!/bin/bash
set -euo pipefail
```

La primera línea indica que el script utiliza Bash.

Las opciones siguientes permiten detectar fallos:

- `-e`: detiene el script si un comando falla, salvo en los contextos donde se comprueba expresamente su resultado.
- `-u`: considera un error utilizar una variable no definida.
- `pipefail`: permite detectar fallos en cualquiera de los comandos de una tubería.

Por ejemplo, si rdiff-backup falla, el script termina y no muestra el mensaje final de éxito.

##### Entorno de ejecución y permisos

```bash
PATH=/usr/sbin:/usr/bin:/sbin:/bin
umask 077
```

Cron utiliza un entorno más limitado que una terminal interactiva. Se define `PATH` para que encuentre las herramientas necesarias.

`umask 077` solicita permisos privados para los nuevos archivos creados por el proceso. Los permisos finales sobre Samba también dependen de la configuración del recurso y de las capacidades de CIFS.

##### Rutas utilizadas

```bash
ORIGEN="/var/www/html"
MONTAJE="/opt/backups_remote"
DESTINO="$MONTAJE/web-prueba-sin-cache"
```

Se separan las rutas en variables para facilitar su lectura y modificación:

| Variable | Función |
|---|---|
| `ORIGEN` | Directorio completo de la aplicación y sus datos. |
| `MONTAJE` | Punto de acceso al recurso Samba. |
| `DESTINO` | Repositorio donde rdiff-backup mantiene la copia y el historial. |

##### Prevención de ejecuciones simultáneas

```bash
exec 9>/run/lock/backup-web.lock
flock -n 9 || exit 0
```

Se abre un archivo de bloqueo mediante el descriptor `9`.

`flock -n 9` intenta obtener el bloqueo sin esperar. Si otra ejecución del script ya lo tiene, la nueva termina sin iniciar otra copia. Esto evita que cron y una ejecución manual escriban simultáneamente en el mismo repositorio.

El bloqueo se libera cuando termina el script. En este caso, un código `0` también puede indicar que se omitió la ejecución porque otra ya estaba activa.

##### Registro del inicio

```bash
echo "$(date -Is) Inicio del backup"
```

Se escribe la fecha y hora de inicio. Al ejecutarse desde cron, este mensaje se añade al archivo de log.

##### Activación y comprobación del montaje

```bash
timeout 30 ls "$MONTAJE" >/dev/null
```

El acceso al directorio activa el montaje bajo demanda configurado anteriormente en `/etc/fstab`.

Se limita la espera a 30 segundos y se oculta el listado, porque solo interesa comprobar el acceso. Si el comando falla, el script se detiene.

```bash
if ! findmnt -rn -t cifs --mountpoint "$MONTAJE" >/dev/null; then
    echo "ERROR: el recurso Samba no está montado" >&2
    exit 1
fi
```

Se comprueba que `/opt/backups_remote` es exactamente un punto de montaje de tipo CIFS.

Esta comprobación evita un error importante: si Samba no estuviera montado, `/opt/backups_remote` seguiría existiendo como directorio local. Sin verificarlo, el backup podría escribirse en el disco de LinuxServer en lugar de almacenarse en LinuxBackup.

##### Ajuste de caché

```bash
echo 0 > /proc/fs/cifs/LookupCacheEnabled
```

Se aplica en cada ejecución el ajuste con el que funcionaron las pruebas. La redirección requiere privilegios administrativos, por lo que el script se ejecuta como root.

El ajuste se realiza después de activar el montaje para asegurar que el módulo CIFS está cargado y que existe su entrada en `/proc`.

Las pruebas realizadas durante la sesión funcionaron. Queda pendiente comprobar el comportamiento de la primera ejecución después de un reinicio, ya que la prueba inicial también incluyó un desmontaje y montaje del recurso.

##### Ejecución de la copia

```bash
rdiff-backup --api-version 201 backup "$ORIGEN" "$DESTINO"
```

Se copia el directorio completo al repositorio remoto. En las siguientes ejecuciones, rdiff-backup actualiza la copia actual y mantiene el historial necesario para recuperar estados anteriores.

`--api-version 201` selecciona la API moderna utilizada en las pruebas.

##### Registro de finalización

```bash
echo "$(date -Is) Backup terminado correctamente"
```

Este mensaje solo se alcanza si los pasos anteriores terminan correctamente.

#### 7. Programación diaria mediante cron

Se preparó un archivo de log privado:

```bash
sudo touch /var/log/backup-web.log
sudo chmod 600 /var/log/backup-web.log
```

Se creó `/etc/cron.d/backup-web` con la programación diaria:

```cron
0 3 * * * root /usr/local/sbin/backup-web.sh >> /var/log/backup-web.log 2>&1
```

| Campo | Significado |
|---|---|
| `0 3` | A las 03:00, según la hora de LinuxServer. |
| `* * *` | Todos los días, meses y días de la semana. |
| `root` | Usuario que ejecuta el script. |
| `>>` | Añade la salida al log sin sobrescribirlo. |
| `2>&1` | Envía también los errores al mismo log. |

Se establecieron los permisos y se habilitó cron:

```bash
sudo chmod 644 /etc/cron.d/backup-web
sudo systemctl enable --now cron
```

El servicio quedó en estado `active`.

Para que la copia se ejecute, LinuxServer, LinuxBackup y la infraestructura de red necesaria deben estar disponibles. Cron no recupera automáticamente una ejecución perdida mientras LinuxServer está apagado.

#### 8. Comprobaciones realizadas

##### Copia de archivos

Se comparó el archivo original con el copiado:

```bash
sudo cmp /var/www/html/index.php /opt/backups_remote/web-prueba-sin-cache/index.php
echo $?
```

No se mostraron diferencias y se obtuvo `0`, confirmando que ambos archivos eran idénticos.

##### Recuperación de una versión anterior

Se creó un archivo de prueba con el contenido:

```text
Version 1 del archivo de prueba
```

Se realizó una copia, se cambió el contenido a la versión 2 y se ejecutó otro backup.

El historial se consultó con:

```bash
sudo rdiff-backup --api-version 201 list increments /opt/backups_remote/web-prueba-sin-cache
```

Se restauró la versión correspondiente a las 09:38:48 en una ubicación diferente:

```bash
sudo rdiff-backup --api-version 201 restore \
  --at '2026-10-07T09:38:48+02:00' \
  /opt/backups_remote/web-prueba-sin-cache/prueba-restauracion.txt \
  /tmp/prueba-restaurada-v1.txt
```

La comprobación mostró:

| Archivo | Contenido |
|---|---|
| `/tmp/prueba-restaurada-v1.txt` | Version 1 del archivo de prueba |
| `/var/www/html/prueba-restauracion.txt` | Version 2 del archivo de prueba |

Se demostró la recuperación de una versión anterior sin sobrescribir la actual.

##### Ejecución del script y prueba de cron

El script se ejecutó manualmente y terminó con código `0`.

Para comprobar cron durante la sesión, se cambió temporalmente la frecuencia a cada minuto. El log registró:

```text
2026-10-07T10:00:01+02:00 Inicio del backup
NOTE:    Starting backup operation from source path /var/www/html to
         destination path /opt/backups_remote/web-prueba-sin-cache
2026-10-07T10:00:07+02:00 Backup terminado correctamente
```

Después de la prueba, la programación definitiva debe quedar nuevamente en `0 3 * * *`.

#### 9. Límites de la configuración

La copia incluye los archivos JSON y las imágenes, pero no congela la aplicación durante el proceso. Si se modifican datos mientras se ejecuta el backup, no se garantiza que todos los archivos correspondan al mismo instante.

Además, no se ha configurado eliminación automática de versiones antiguas ni rotación del log. Será necesario vigilar el espacio disponible en LinuxBackup.

## Análisis periódico de la web con ClamAV

#### 1. Objetivo
Se configuró en **LinuxServer** un análisis diario de `/var/www/html` mediante ClamAV. Los archivos detectados se trasladan a `/opt/quarantine/`, fuera del directorio web, y los resultados se guardan en un log.

#### 2. Actualización de firmas y cuarentena
ClamAV ya estaba instalado, con versión **1.4.3**, pero sus firmas estaban desactualizadas. Se actualizaron mediante:

```bash
sudo freshclam
```

La base de firmas pasó de la versión `28130` a la `28146`. Se habilitaron las actualizaciones automáticas:

```bash
sudo systemctl enable --now clamav-freshclam
```

Se creó el directorio de cuarentena:

```bash
sudo install -d -o root -g root -m 0700 /opt/quarantine
```

Su ubicación fuera de `/var/www/html` evita que los archivos trasladados se sirvan como contenido web. Los permisos `700` permiten acceder únicamente a root.

#### 3. Script de análisis

Se creó `/usr/local/sbin/scan-web.sh` con este contenido:
```bash
#!/bin/bash
set -uo pipefail
PATH=/usr/sbin:/usr/bin:/sbin:/bin
umask 077

WEB="/var/www/html"
CUARENTENA="/opt/quarantine"

# Evitar análisis simultáneos.
exec 9>/run/lock/scan-web.lock
flock -n 9 || exit 0

echo "$(date -Is) Inicio del análisis"

clamscan --recursive --infected --move="$CUARENTENA" "$WEB"
RESULTADO=$?

case "$RESULTADO" in
    0)
        echo "$(date -Is) Análisis terminado sin detecciones"
        ;;
    1)
        echo "$(date -Is) Se detectaron archivos: revisar los mensajes de traslado a cuarentena"
        ;;
    *)
        echo "$(date -Is) ERROR durante el análisis; código $RESULTADO" >&2
        ;;
esac

exit "$RESULTADO"
```

Se restringió su acceso y ejecución a root:

```bash
sudo chmod 700 /usr/local/sbin/scan-web.sh
```

#### 4. Explicación del script

| Elemento | Función |
|---|---|
| `#!/bin/bash` | Ejecuta el script mediante Bash. |
| `set -uo pipefail` | Detecta variables no definidas y fallos en tuberías. |
| `PATH=...` | Permite encontrar los comandos cuando se ejecuta desde cron. |
| `umask 077` | Establece permisos privados para nuevos archivos creados por el proceso cuando sea aplicable. |
| `WEB` y `CUARENTENA` | Definen el directorio analizado y el destino de los archivos detectados. |
| `exec 9>...` y `flock -n 9` | Evitan que dos ejecuciones del script analicen simultáneamente la web. |
| `date -Is` | Añade fecha y hora a los mensajes. |
| `RESULTADO=$?` | Guarda el código de salida de ClamAV. |
| `case` | Distingue entre un análisis limpio, una detección y un error. |
| `exit "$RESULTADO"` | Conserva el resultado de ClamAV como resultado del script. |

El comando principal es:

```bash
clamscan --recursive --infected --move="$CUARENTENA" "$WEB"
```

- `--recursive` incluye todas las subcarpetas.
- `--infected` muestra las detecciones sin listar cada archivo limpio.
- `--move` traslada los archivos detectados a cuarentena, en lugar de borrarlos.

No se utiliza `set -e`, porque ClamAV devuelve `1` cuando detecta una amenaza. Ese resultado debe registrarse como detección, sin interrumpir el script antes de mostrar el mensaje correspondiente.

Los códigos principales son:

| Código | Significado |
|---|---|
| `0` | No se detectaron amenazas. |
| `1` | Se detectó al menos una amenaza. |
| `2` | Se produjo un error. |

Una detección no demuestra por sí sola que el traslado haya funcionado: también deben revisarse los mensajes de ClamAV. Por eso se comprobaron expresamente las ubicaciones del archivo de prueba.

#### 5. Archivos incluidos y exclusiones

Se analiza **todo `/var/www/html`**, sin excluir archivos por extensión.

| Contenido incluido | Justificación |
|---|---|
| Código PHP | Puede contener código malicioso o haber sido modificado. |
| Plantillas HTML | Forman parte del contenido publicado y pueden alterarse. |
| Archivos JSON de `data/` | Contienen datos utilizados y modificados por la aplicación. |
| Imágenes y carpetas de subidas | Reciben archivos de usuarios; una extensión de imagen no garantiza que el contenido sea seguro. |

No se incluyen en este análisis:

- **`/opt/quarantine/`**: contiene archivos ya aislados y no forma parte de la web.
- **`/opt/backups_remote/`**: contiene copias históricas y requiere un análisis independiente.
- Los directorios del sistema ajenos a la aplicación, porque el objetivo de esta tarea es revisar el contenido web.

#### 6. Programación diaria

Se creó un log privado:

```bash
sudo touch /var/log/scan-web.log
sudo chmod 600 /var/log/scan-web.log
```

En `/etc/cron.d/scan-web` se añadió:

```cron
0 2 * * * root /usr/local/sbin/scan-web.sh >> /var/log/scan-web.log 2>&1
```

La tarea ejecuta el análisis **todos los días a las 02:00**, según la hora de LinuxServer, como root. La salida y los errores se añaden a `/var/log/scan-web.log`.

Se establecieron los permisos del archivo:

```bash
sudo chmod 644 /etc/cron.d/scan-web
```

El servicio cron estaba activo. Se eligió una hora anterior al backup diario de las 03:00, aunque esta separación horaria no garantiza que ambos procesos nunca coincidan.

#### 7. Comprobaciones

##### Análisis inicial

Se ejecutó:

```bash
sudo /usr/local/sbin/scan-web.sh
echo $?
```

El análisis examinó **68 archivos**, no encontró amenazas y terminó con código `0`.

##### Detección y cuarentena con EICAR

Se creó un archivo EICAR, una cadena de prueba antivirus inofensiva, en:

```text
/var/www/html/images/fotos/eicar-test.txt
```

Se volvió a ejecutar el script. ClamAV mostró:

```text
Eicar-Test-Signature FOUND
moved to '/opt/quarantine/eicar-test.txt'
```

El resumen indicó **69 archivos analizados y una detección**.

Se comprobaron ambas ubicaciones:

```bash
sudo ls -l /var/www/html/images/fotos/eicar-test.txt
sudo ls -l /opt/quarantine/eicar-test.txt
```

El archivo ya no existía en el directorio web y estaba presente en cuarentena, confirmando su detección y traslado.

##### Ejecución desde cron

Se cambió temporalmente la frecuencia a cada minuto para comprobar la ejecución automática. Una vez iniciado el análisis, se restauró la programación diaria.

El log registró:

```text
2026-10-07T10:42:01+02:00 Inicio del analisis
Scanned files: 68
Infected files: 0
Time: 277.794 sec (4 m 37 s)
2026-10-07T10:46:39+02:00 Analisis terminado sin detecciones
```

Esto confirmó que cron ejecuta el script y guarda sus resultados.

#### 8. Alcance de la protección

El análisis es periódico, no una protección en tiempo real: un archivo malicioso podría permanecer en la web hasta el siguiente escaneo. LinuxServer debe estar encendido a la hora programada; cron no recupera automáticamente las ejecuciones perdidas mientras está apagado.

La ausencia de detecciones significa que ClamAV no encontró amenazas reconocidas durante ese análisis, pero no garantiza que la aplicación carezca de vulnerabilidades.