```
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_linuxbackup
ssh-copy-id -i ~/.ssh/rsa_linuxbackup.pub dummyadmin@192.168.57.10

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_linuxserver
ssh-copy-id -i ~/.ssh/rsa_linuxserver.pub dummyadmin@192.168.56.10

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_linuxclient
ssh-copy-id -i ~/.ssh/rsa_linuxserver.pub dummyadmin@192.168.56.101

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_windowsserver
ssh-copy-id -i ~/.ssh/rsa_windowsserver.pub dummyadmin@192.168.57.11

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_windowsclient
ssh-copy-id -i ~/.ssh/rsa_windowsclient.pub dummyadmin@192.168.56.102

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_linuxclient
ssh-copy-id -i ~/.ssh/rsa_linuxserver.pub dummyadmin@192.168.56.101

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_router1
ssh-copy-id -i ~/.ssh/rsa_router1.pub dummyadmin@192.168.56.254

ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_router2
ssh-copy-id -i ~/.ssh/rsa_router2.pub dummyadmin@192.168.56.253
```

`/.ssh/config`
```
Host linuxbackup
    HostName 192.168.57.10
    User dummyadmin
    IdentityFile ~/.ssh/rsa_linuxbackup

Host linuxserver
    HostName 192.168.56.10
    User dummyadmin
    IdentityFile ~/.ssh/rsa_linuxserver

Host linuxclient
    HostName 192.168.56.101
    User dummyadmin
    IdentityFile ~/.ssh/rsa_linuxclient
    
Host windowsserver
    HostName 192.168.57.11
    User dummyadmin
    IdentityFile ~/.ssh/rsa_windowsserver
    
Host windowsclient
    HostName 192.168.56.102
    User dummyadmin
    IdentityFile ~/.ssh/rsa_windowsclient
    
Host router1
    HostName 192.168.56.254
    User dummyadmin
    IdentityFile ~/.ssh/rsa_router1
    
Host router2
    HostName 192.168.56.253
    User dummyadmin
    IdentityFile ~/.ssh/rsa_router2  

```
Y finalmente le daremos permisos para que funcione bien.
```
sudo chown dummyadmin:dummyadmin ~/.ssh/config
sudo chmod 600 ~/.ssh/config
```
