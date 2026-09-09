Para configurar una maquina nueva desde 0 tenemos que instalar todas las herramientas desde 0
```
su -
```

```
apt install sudo
```

Para poder ver las listas donde hemos instalado algo en el pc deberemos editar el buscador.
```
cd etc/apt/
nano sources.list
```

Aquí deberemos de tener un archivo asi:
```
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware

deb http://deb.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Para permitirle a un usuario utilizar sudo hay que darle permisos:
```
usr/sbin/usermod -aG sudo dummyadmin
usr/bin/adduser dummyadmin sudo
groups dummyadmin
```
Con esto podemos comprobar que efectivamente pertenece `dummyadmin` al grupo `sudo`

Cambiamos al usuario `dummyadmin` comprobando que funciona:
```
su - dummyadmin
sudo whoami
```