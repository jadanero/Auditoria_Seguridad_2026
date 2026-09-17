## Objetivos
Se configurarán dos nuevas máquinas virtuales Linux: 
- [x] LinuxBackup: equipo destinado a actuar como servidor de backups y recepción de logs. 
- [x] LinuxClient: equipo cliente desde el que se realizarán las diferentes pruebas. 

Además, se utilizará la máquina LinuxServer configurada en la primera parte de la práctica. Durante la práctica se trabajará con los siguientes servicios y tecnologías: 
- [x] Configuración de red mediante NAT y Host-Only. 
- [x]  Gestión de usuarios. 
- [x] rsyslog para la centralización de logs. 
- [x] NFS para compartir directorios en red. 
- [x] Samba para compartir archivos. 
- [x] SSH mediante autenticación con claves y claves firmadas por una CA. 
- [ ] EncFS para crear directorios cifrados. 
- [ ] rdiff-backup para realizar copias de seguridad. 
- [ ] ClamAV para analizar archivos y generar registros

## Configuración inicial de las máquinas linux

Para diferenciarlas al hacer un clon cambiaremos el nombre de las maquinas. Para conseguir eso haremos:
```
sudo nano /etc/hostname
```
Donde deberán de ser `LinuxBackup` y `LinuxClient`. Después `reboot` a las máquinas y se habrán actualizado los hostnames

Para ambas máquinas cambiaremos una pequeña linea de configuración de los hostnames
```
sudo hostinamectl set-hostname linuxbackup
sudo nano /etc/hosts    # Cambiamos el nombre dentro del archivo
```

`/etc/hosts`
```
...
linuxbackup.blue.local linuxbackup
...
```

```
hostnamectl             # Comprobamos el cambio del nombre en el sistema
```
#### Creación de usuarios
Para la máquina `Linuxbackups` deberemos crear usuarios, que por terminal:
```
sudo adduser <usuario>
```
En el que añadiremos `alice`, `bob` y `trudy`. En todos ellos la contraseña será: `<usuario>`

Comprobamos que existen los `users`:
```
id alice bob trudy
```
Con este comando podemos ver los diferentes grupos a los que pertenece cada usuario.
```
cat /etc/passwd
```
Con este veremos todos los usuarios. 

## Configuración de rsyslog
Instalamos en las tres máquinas. `LinuxServer`, `LinuxClient` y `LinuxBackup`
```
sudo apt install rsyslog
```
Para comprobar que esté activo en cada uno:
```
sudo systemctl status rsyslog
```

#### LinuxBackup
En la máquina que va a funcionar como server, que va a ser la `LinuxBackup` deberemos cambiar el archivo de configuración para convertirlo en servidor y que pueda recibir los logs por TCP. Tendremos un fichero con el nombre `01-receptor.conf`.

`/etc/rsyslog.d/01-receptor.conf`
```
module(load="imtcp")
input(type="imtcp" port="513")
```

También tendremos un archivo con el nombre `02-almacenamiento.conf`. El cual se encargará de guardar de forma ordenada los logs que nos vayan llegando.

`/etc/rsyslog.d/02-almacenamiento.conf`
```
$template RemoteLogs,"/var/log/remote/%FROMHOST-IP%/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
```
Dividiremos el nombre de los logs en cada una de las `IPs` `Servicio` y `Hostname` encargado.

Hacemos `restart` del servidor `rsyslog`:
```
sudo systemctl restart rsyslog
```

#### LinuxServer y LinuxClient
Para que se puedan comunicar con el servidor de logs crearemos un archivo en ambos ordenadores en el directorio `/etc/rsyslog.d/01-remoto.conf` y escribiremos:
```
*.* @@192.168.56.103:513
```

Al terminar la configuración haremos un `restart` del servidor `rsyslog`:
```
sudo systemctl restart rsyslog
```

## Configuración de NFS
Para las máquinas `LinuxServer` y `LinuxClient`montaremos un servicio de NFS. Instalaremos:
```
apt install nfs-kernel-server
```

Configuramos en `LinuxServer`:
```
sudo mkdir - p srv/nfs/share
sudo chown nobody:nogroup /srv/nfs/share
sudo chmod 755 /srv/nfs/share
```

También en `/etc/exports`:
```
/srv/nfs/share 192.168.56.0/24(rw,sync,no_subtree_check)
```

Seguiremos la configuración aplicando los cambios y reiniciando la máquina:
```
sudo exportfs -a
systemctl restart nfs-kernel-server
```


La configuración para la parte del cliente será con automontaje en la máquina `LinuxClient`. Primero prepararemos el entorno.
```
sudo mkdir /mnt/nfsshare
```

 modificando el archivo y añadiendo la línea `/etc/fstab`:
 ```
 192.168.56.102:/srv/nfs/share /mnt/nfsshare nfs defaults 0 0
 ```

Después de la modificación deberemos hacer un restart del servidor:
```
sudo systemctl daemon-reload
sudo mount -a
```

Para comprobar que se ha montado bien el sistema:
```
df -h | grep nfs
```

Creamos un archivo de prueba llamado `prueba.txt` en el directorio `/mnt/nfsshare/`:
```
echo "Prueba NFS exitosa" > /mnt/nfsshare/prueba.txt
```
**Funciona!**

## Configuración de Samba
En la máquina `LinuxServer`:
```
sudo apt install samba
```

Comprobamos que está instalado:
```
smbd --version
```

Creamos el directorio:
```
sudo mkdir -p /srv/samba/smbshare
sudo chmod 755 /srv/samba/smbshare
sudo chown -R nobody:nogroup /srv/samba/smbshare
```

Para la configuración del servicio `Samba` modificaremos su configuración en el archivo `/etc/samba/smb.conf`:
```
[SMBShare]
    path = /srv/samba/smbshare
    browseable = yes
    read only = yes
    guest ok = yes
    writeable = no
    public = yes
```


Vamos a crear un archivo que se va a compartir:
```
echo "Prueba SAMBA exitosa" > /smbshare/archivo_prueba.txt
```

Para la máquina `LinuxClient` habrá que hacer la instalación de `sambaclient`:
```
sudo apt install smbclient
```

Para conectarnos con el servicio que ofrece `LinuxServer`:
```
smbclient -L //192.168.56.102/SMBShare -N
```
Hacemos una busqueda y vemos el archivo compartido con `ls`
**Funciona!**

## Configuración SSH
Explicaremos como se ha hecho la configuración para una de las máquinas.

Dentro de la máquina `LinuxClient` tendremos que configurar un par de claves para conectarnos con los usuarios `dummyadmin` de las otras dos máquinas:
```
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_backup
ssh-copy-id -i ~/.ssh/rsa_backup.pub dummyadmin@192.168.56.103

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_server
ssh-copy-id -i ~/.ssh/rsa_server.pub dummyadmin@192.168.56.102
```
De esta forma comprobamos que nos permite acceder por `ssh` sin necesidad de poner contraseña desde `LinuxClient`. Para facilitarnos aun mas la vida crearemos el archivo de configuración ssh.
```
sudo nano .ssh/config
```

`/.ssh/config`
```
Host backup
    HostName 192.168.56.103
    User dummyadmin
    IdentityFile ~/.ssh/rsa_backup

Host server
    HostName 192.168.56.102
    User dummyadmin
    IdentityFile ~/.ssh/rsa_server
```
Y finalmente le daremos permisos para que funcione bien.
```
sudo chown dummyadmin:dummyadmin ~/.ssh/config
sudo chmod 600 ~/.ssh/config
```

Ahora simplemente con poner en la terminal:
```
ssh server
```
o
```
ssh backup
```
Nos conectaremos respectivamente a las diferentes maquinas
Haremos la misma configuración desde los otros dos ordenadores `LinuxBackup` y `LinuxServer`.


**NO HE HECHO LA CA**

Además de la autenticación mediante claves SSH convencionales, se configurará el
acceso mediante claves SSH firmadas por una Autoridad de Certificación
(CA) común

```
ssh-keygen -f 
```

## Configuración de EncFS
Para las máquinas `LinuxBackup` y `LinuxClient`:
```
sudo apt update
sudo apt install encfs
```
Y comprobamos que está instalado
```
encfs --version
```

En la maquina `LinuxClient` crearemos el usuario `alice`

Escribiremos esto:
```
dummyadmin@linuxclient:~$ df -h | grep nfs
192.168.56.102:/srv/nfs/share  3.4G  1.9G  1.3G  59% /mnt/nfsshare
```
Para saber donde tenemos montado el directorio de nfs

Ahora que lo sabemos podremos fabricar los directorios donde irán los datos encriptados y los datos en plano:
```
sudo mkdir -p /mnt/nfsshare/Alice
sudo mkdir -p /home/alice/Alice
sudo chown alice:alice /home/alice/Alice
```











