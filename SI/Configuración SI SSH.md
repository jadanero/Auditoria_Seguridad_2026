```
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/rsa_ssh
ssh-copy-id -i ~/.ssh/rsa_linux.pub dummyadmin@192.168.57.10
```

```
palangana2026.ABC
```

`/.ssh/config`
```
Host router1
    HostName 192.168.56.254
    User dummyadmin
    IdentityFile ~/.ssh/rsa_ssh
```
Y finalmente le daremos permisos para que funcione bien.
```
sudo chown dummyadmin:dummyadmin ~/.ssh/config
sudo chmod 600 ~/.ssh/config
```
