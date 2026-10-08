## Punto 3 — Directorio remoto para copias de seguridad

### LinuxBackup

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

### LinuxServer

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

### Comprobación

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