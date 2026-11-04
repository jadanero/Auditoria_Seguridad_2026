Usuario: SI07
Contraseña: <span class="spoiler">jadanerolabajo2627</span>

```
rdesktop -g 1920x1000 -k es 10.6.24.7
```

Usuario: si
Contraseña: <span class="spoiler">adaner0labaj0?2627</span>

Usuario: dummyadmin
Contraseña: palangana2026.ABC
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
- [x] Deberá disponer de un servidor web **Apache**.
- [x] Deberá tener instalado y operativo **PHP**.
- [x] Deberá tener instalado y operativo **MariaDB**.
- [x] El usuario `dummyadmin` deberá disponer de permisos para utilizar **sudo** y poder ejecutar cualquier comando con privilegios administrativos.
- [x] Deberá instalarse **PowerShell** en el sistema.

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

>Se deberá crear una máquina virtual con **Windows Server** utilizando los siguientes recursos:
>- **Memoria RAM:** 4 GB.
>- **Disco duro:** 30 GB.
>- **Procesadores:** 2 CPUs.
>- **Edición:** Standard con Experiencia de escritorio.
>- Usuario: dummyadmin
>- **Contraseña:** `palangana2026.ABC`.

#### Configuración del sistema
>Se deberá configurar el sistema con los siguientes parámetros:
- [ ] Nombre del equipo: windowsserver. 
- [ ] Usuario: dummyadmin. 
- [ ] El usuario dummyadmin deberá pertenecer al grupo Administradores. 
- [ ]  Deberá habilitarse la administración remota.
#### Instalación
#fixed 
[[Problemas encontrados maquina lab SI]]


**Abrir PowerShell** y ejecutarla con privilegios
```
start-process powershell -verb runas
```

Para meter al usuario dummyadmin en el grupo de administradores:
```
 net localgroup Administradores dummyadmin /add
```

```
PS C:\WINDOWS\system32> net localgroup Administradores dummyadmin /add
Error de sistema 1378.

El nombre de cuenta especificado ya pertenece al grupo.

PS C:\WINDOWS\system32> net localgroup Administradores
Nombre de alias       Administradores
Comentario            Los administradores tienen acceso completo y sin restricciones al equipo o dom

Miembros

-------------------------------------------------------------------------------
Administrador
dummyadmin
Se ha completado el comando correctamente.
```

Cambiamos el perfil de red a privada ya que es un requisito para activar PSRemoting
```
set-netconnectionprofile -interfacealias ethernet -networkcategory private
```

Permitimos conexiones entrantes:
```
PS C:\WINDOWS\system32> winrm quickconfig
El servicio WinRM ya está ejecutándose en esta máquina.
WinRM no está configurado para permitir acceso remoto al equipo para administración.
Se deben realizar estos cambios:

Configurar LocalAccountTokenFilterPolicy para conceder derechos administrativos en modo remoto a usuarios locales.

¿Desea realizar estos cambios [y/n]? y

WinRM se actualizó para administración remota.

LocalAccountTokenFilterPolicy configurado para conceder derechos administrativos en modo remoto a usuarios locales.
```

```
PS C:\WINDOWS\system32> get-service winrm

Status   Name    DisplayName
------   ----    -----------
Running  winrm   Administración remota de Windows (W...
```

Para instalar el servidor OpenSSH:
```
add-windowscapability -online -name OpenSSH.server
``` 

Para configurar el inicio automático del servicio SSH:
```
set-service -startuptype automatic -name sshd
start-service sshd
```

Después de reiniciar `WindowsServer` comprobamos que funciona con:
```
get-service sshd
```

## 4. Instalación de Windows 11

>Se deberá crear una máquina virtual con Windows Server utilizando los siguientes recursos: 
>• Memoria RAM: 4 GB. 
>• Disco duro: 30 GB. 
>• Procesadores: 2 CPUs. 
>• Edición: Standard con Experiencia de escritorio. 
>• Contraseña: palangana2026.ABC 
>
>Aunque la instalación inicial se realizará utilizando la interfaz gráfica, las siguientes configuraciones deberán realizar

Configurar el usuario y la contraseña 

Para agregar al usuario dummyadmin al grupo de administradores
```
net localgroup Administradores dummyadmin /add
``` 

Cambiamos el perfil a privado igual que en WindowsServer para habilitar la administración remota:
```
set-netconnectionprofile -interfacealias ethernet -networkcategory private
```

Para habilitar la administración remota
```
enable-psremoting -force
```

Igual que en el WindowsServer:
```
winrm quickconfig
```

Comprobamos finalmente que los servicios están bien configurados después del reinicio.