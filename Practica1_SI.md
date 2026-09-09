

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

#### Configuración de HTTPS
Comprobar que en la maquina linux tiene instalado `Apache` y `SSL`
```
apache2 --version
```

```
openssl --version
```

En caso contrario:
```
sudo apt install apache2 openssl php mariadb-server php-mysql
```

Para comprobar que están encendidos ambos sistemas`Apache` y `MariaDB`:
```
sudo systemctl status apache2 mariadb
```

Para activar el `mod_ssl`:
```
sudo a2enmod ssl ##para habilitar mod_ssl
sudo systemctl restart apache2
sudo a2ensite default-ssl
sudo reload apache2
```

Vamos a guardar la clave en `/etc/ssl/private/`:
```
sudo openssl genrsa -out /etc/ssl/private/linuxserver.key 2048
```

Comprobamos que existe:
```
sudo ls -l /etc/ssl/private/linuxserver.key
```

>El certificado deberá utilizar como Common Name (CN) el siguiente nombre: `linuxserver.blue.local`
```
sudo openssl req -new -x509 -key /etc/ssl/private/linuxserver.key -out /etc/ssl/certs/linuxserver.crt -days 365 -subj "/CN=linuxserver.blue.local"
```

Para decirle al servidor cuales son los certificados y donde están:
```
nano etc/apache2/sites-available/default-ssl.conf
```
En el que modificaremos las líneas fabricadas por default y meteremos los certificados que acabamos de crear.

Para comprobar que el servicio está funcionando con `HTTPS` deberemos buscar que tengamos una comunicación abierta en el puerto `433`:
```
ss -tlpn | grep :433
```

Para instalar `PowerShell` hemos encontrado estas instrucciones en internet: [PowerShell Debian](https://learn.microsoft.com/es-es/powershell/scripting/install/install-debian?view=powershell-7.6)
```
# Install pre-requisite packages.
sudo apt-get install -y wget

# Get the version of Debian
source /etc/os-release

# Download the Microsoft repository GPG keys
wget -q https://packages.microsoft.com/config/debian/13/packages-microsoft-prod.deb

# Register the Microsoft repository GPG keys
sudo dpkg -i packages-microsoft-prod.deb

# Delete the Microsoft repository GPG keys file
rm packages-microsoft-prod.deb

# Update the list of packages after we added packages.microsoft.com
sudo apt-get update

###################################
# Install PowerShell
sudo apt-get install -y powershell

# Start PowerShell
pwsh

```

Hacemos una última comprobación para la configuración de red `ssh`
```
sudo systemctl status ssh
ss -tlpn | grep :22
```
Con esto ya tenemos la configuración de la máquina servidor linux.

#### Configuración de la interfaz
Hemos encendido dos adaptadores de red:
- Una red `NAT Network`
- Una red tipo `Host-only`

```
sudo apt install isc-dhcp-client
sudo ip link set enp0s8 up
sudo dhclient enp0s8
sudo systemctl restart systemd-networkd
```

Para hacer persistente la ip en la máquina haremos:
```
sudo nano /etc/network/interfaces
```

En este modificaremos:
```
...
# Segunda interfaz de red (Host-Only)
auto enp0s8
iface enp0s8 inet dhcp
...
```

Para conectarnos a la máquina:
```
ssh dummyadmin@<ip_maquina>
```

---
## 3. Instalación de Windows Server
