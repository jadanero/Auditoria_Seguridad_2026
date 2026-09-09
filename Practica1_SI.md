

Usuario: SI07
Contraseña: <span class="spoiler">jadanerolabajo2627</span>

```
rdesktop -g 1920x1000 -k es 10.6.24.07
```

Usuario: si
Contraseña: <span class="spoiler">adaner0labaj0?2627</span>

## 1. Introducción

>Dentro de la misma red se encuentra disponible, en la dirección `10.6.24.100`, un servidor Samba que contiene tres imágenes ISO. Estas imágenes deberán descargarse, ya que serán necesarias para crear las máquinas virtuales que compondrán la infraestructura.

Para hacer esto deberemos:
```
sudo apt install smbclient
```

Después para poder instalar cada uno de los sistemas(Ej debian) :
```
mkdir practica1/ISOs
cd practica1/ISOs

smbclient -L //10.6.24.100
smbclient //10.6.24.100/ISOs 
l
get debian-trixie.iso
```

## 2. Instalación del servidor Debian

>Se deberá crear e instalar una máquina virtual con **Debian Server**, utilizando los siguientes parámetros:
>- **Instalación:** servidor sin entorno gráfico.
>- **Memoria RAM:** 1 GB.
>- **Disco duro:** 4 GB.
>- **Acceso remoto:** SSH.
>- **Nombre del equipo:** `linuxserver`.
>- **Dominio:** `blue.local`.
>- **Usuario:** `dummyadmin`.
>- **Contraseña:** `palangana2026.ABC`.

**El servidor deberá cumplir los siguientes requisitos:**
- [ ] Deberá disponer de un servidor web **Apache**.
- [ ] Deberá tener instalado y operativo **PHP**.
- [ ] Deberá tener instalado y operativo **MariaDB**.
- [ ] El usuario `dummyadmin` deberá disponer de permisos para utilizar **sudo** y poder ejecutar cualquier comando con privilegios administrativos.
- [ ] Deberá instalarse **PowerShell** en el sistema.

[[Configurar maquina nueva desde 0 Linux]]

**Configuración de HTTPS**
Comprobar que en la maquina linux tiene instalado `Apache` y `SSL`
```
apache2 --version
```

```
openssl --version
```

En caso contrario:
```
sudo apt install apache2 openssl php mariadb-server libapache-mod-php phhp-mysql
```

```
sudo systemctl status apache2

sudo a2enmod ssl ##para habilitar mod_ssl
sudo a2ensite default-ssl
```





Vamos a guardar la clave en `/etc/ssl/private/`:
```
sudo openssl genrsa -out /etc/ssl/private/linuxserver.key 2048
```

Comprobamos que existe:
```
sudo ls -l /etc/ssl/private/linuxserver.key
```

