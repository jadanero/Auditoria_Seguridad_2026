```
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_linux
ssh-copy-id -i ~/.ssh/rsa_linux.pub dummyadmin@192.168.57.10
ssh-copy-id -i ~/.ssh/rsa_linux.pub dummyadmin@192.168.56.10
ssh-copy-id -i ~/.ssh/rsa_linux.pub dummyadmin@192.168.56.101

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_windows
ssh-copy-id -i ~/.ssh/rsa_windows.pub dummyadmin@192.168.57.11
ssh-copy-id -i ~/.ssh/rsa_windows.pub Administrador@192.168.57.11
ssh-copy-id -i ~/.ssh/rsa_windows.pub dummyadmin@192.168.56.102
ssh-copy-id -i ~/.ssh/rsa_windows.pub Administrador@192.168.56.102

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_router
ssh-copy-id -i ~/.ssh/rsa_router.pub dummyadmin@192.168.56.254
ssh-copy-id -i ~/.ssh/rsa_router.pub dummyadmin@192.168.56.253
```

```
palangana2026.ABC
```

`/.ssh/config`
```
Host linuxbackup
    HostName 192.168.57.10
    User dummyadmin
    IdentityFile ~/.ssh/rsa_linux

Host linuxserver
    HostName 192.168.56.10
    User dummyadmin
    IdentityFile ~/.ssh/rsa_linux

Host linuxclient
    HostName 192.168.56.101
    User dummyadmin
    IdentityFile ~/.ssh/rsa_linux
    
Host windowsserver
    HostName 192.168.57.11
    User dummyadmin
    IdentityFile ~/.ssh/rsa_windows
    
Host windowsserver-admin
    HostName 192.168.57.11
    User administrador
    IdentityFile ~/.ssh/rsa_windows
    
Host windowsclient
    HostName 192.168.56.102
    User dummyadmin
    IdentityFile ~/.ssh/rsa_windows
    
Host windowsclient-admin
    HostName 192.168.56.102
    User administrador
    IdentityFile ~/.ssh/rsa_windows
    
Host router1
    HostName 192.168.56.254
    User dummyadmin
    IdentityFile ~/.ssh/rsa_router
    
Host router2
    HostName 192.168.56.253
    User dummyadmin
    IdentityFile ~/.ssh/rsa_router
```
Y finalmente le daremos permisos para que funcione bien.
```
sudo chown dummyadmin:dummyadmin ~/.ssh/config
sudo chmod 600 ~/.ssh/config
```
