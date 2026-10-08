## Punto 2 — Directorios personales mediante Samba

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