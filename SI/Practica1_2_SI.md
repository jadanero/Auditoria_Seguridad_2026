## Objetivos
Se configurarán dos nuevas máquinas virtuales Linux: 
- [ ] LinuxBackup: equipo destinado a actuar como servidor de backups y recepción de logs. 
- [ ] LinuxClient: equipo cliente desde el que se realizarán las diferentes pruebas. 

Además, se utilizará la máquina LinuxServer configurada en la primera parte de la práctica. Durante la práctica se trabajará con los siguientes servicios y tecnologías: 
- [ ] Configuración de red mediante NAT y Host-Only. 
- [ ]  Gestión de usuarios. 
- [ ] rsyslog para la centralización de logs. 
- [ ] NFS para compartir directorios en red. 
- [ ] Samba para compartir archivos. 
- [ ] SSH mediante autenticación con claves y claves firmadas por una CA. 
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







