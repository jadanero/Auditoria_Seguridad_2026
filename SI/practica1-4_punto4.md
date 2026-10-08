## Práctica 1.4 — Punto 4: automatización de las copias de seguridad de LinuxServer

### 1. Objetivo y equipos utilizados

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

### 2. Instalación de rdiff-backup

En LinuxServer se instaló la herramienta:

```bash
sudo apt install rdiff-backup
rdiff-backup --version
```

La versión instalada fue **2.2.6**.

`rdiff-backup` mantiene una copia del estado más reciente del directorio y almacena información sobre los cambios anteriores. Esto permite recuperar archivos correspondientes a una fecha concreta sin guardar una copia completa independiente en cada ejecución.

### 3. Problema encontrado al utilizar Samba

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

### 4. Desactivación de la caché de búsqueda CIFS

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

### 5. Script de backup

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

### 6. Explicación del script

#### Intérprete y control de errores

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

#### Entorno de ejecución y permisos

```bash
PATH=/usr/sbin:/usr/bin:/sbin:/bin
umask 077
```

Cron utiliza un entorno más limitado que una terminal interactiva. Se define `PATH` para que encuentre las herramientas necesarias.

`umask 077` solicita permisos privados para los nuevos archivos creados por el proceso. Los permisos finales sobre Samba también dependen de la configuración del recurso y de las capacidades de CIFS.

#### Rutas utilizadas

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

#### Prevención de ejecuciones simultáneas

```bash
exec 9>/run/lock/backup-web.lock
flock -n 9 || exit 0
```

Se abre un archivo de bloqueo mediante el descriptor `9`.

`flock -n 9` intenta obtener el bloqueo sin esperar. Si otra ejecución del script ya lo tiene, la nueva termina sin iniciar otra copia. Esto evita que cron y una ejecución manual escriban simultáneamente en el mismo repositorio.

El bloqueo se libera cuando termina el script. En este caso, un código `0` también puede indicar que se omitió la ejecución porque otra ya estaba activa.

#### Registro del inicio

```bash
echo "$(date -Is) Inicio del backup"
```

Se escribe la fecha y hora de inicio. Al ejecutarse desde cron, este mensaje se añade al archivo de log.

#### Activación y comprobación del montaje

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

#### Ajuste de caché

```bash
echo 0 > /proc/fs/cifs/LookupCacheEnabled
```

Se aplica en cada ejecución el ajuste con el que funcionaron las pruebas. La redirección requiere privilegios administrativos, por lo que el script se ejecuta como root.

El ajuste se realiza después de activar el montaje para asegurar que el módulo CIFS está cargado y que existe su entrada en `/proc`.

Las pruebas realizadas durante la sesión funcionaron. Queda pendiente comprobar el comportamiento de la primera ejecución después de un reinicio, ya que la prueba inicial también incluyó un desmontaje y montaje del recurso.

#### Ejecución de la copia

```bash
rdiff-backup --api-version 201 backup "$ORIGEN" "$DESTINO"
```

Se copia el directorio completo al repositorio remoto. En las siguientes ejecuciones, rdiff-backup actualiza la copia actual y mantiene el historial necesario para recuperar estados anteriores.

`--api-version 201` selecciona la API moderna utilizada en las pruebas.

#### Registro de finalización

```bash
echo "$(date -Is) Backup terminado correctamente"
```

Este mensaje solo se alcanza si los pasos anteriores terminan correctamente.

### 7. Programación diaria mediante cron

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

### 8. Comprobaciones realizadas

#### Copia de archivos

Se comparó el archivo original con el copiado:

```bash
sudo cmp /var/www/html/index.php /opt/backups_remote/web-prueba-sin-cache/index.php
echo $?
```

No se mostraron diferencias y se obtuvo `0`, confirmando que ambos archivos eran idénticos.

#### Recuperación de una versión anterior

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

#### Ejecución del script y prueba de cron

El script se ejecutó manualmente y terminó con código `0`.

Para comprobar cron durante la sesión, se cambió temporalmente la frecuencia a cada minuto. El log registró:

```text
2026-10-07T10:00:01+02:00 Inicio del backup
NOTE:    Starting backup operation from source path /var/www/html to
         destination path /opt/backups_remote/web-prueba-sin-cache
2026-10-07T10:00:07+02:00 Backup terminado correctamente
```

Después de la prueba, la programación definitiva debe quedar nuevamente en `0 3 * * *`.

### 9. Límites de la configuración

La copia incluye los archivos JSON y las imágenes, pero no congela la aplicación durante el proceso. Si se modifican datos mientras se ejecuta el backup, no se garantiza que todos los archivos correspondan al mismo instante.

Además, no se ha configurado eliminación automática de versiones antiguas ni rotación del log. Será necesario vigilar el espacio disponible en LinuxBackup.