## Configuración de un servidor `OpenLDAP` en `LinuxBackup`
#### Instalación del servicio `OpenLDAP`
Para realizar la instalación hacemos:
```bash
sudo apt install slapd ldap-utils
```
y pondremos la contraseña: `patata2026.ABC`
El resultado muestra esta salida:
```
dn: dc=blue,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: blue.local
dc: blue
structuralObjectClass: organization
entryUUID: fbfd155a-56b3-1041-8b8d-b11c788f3952
creatorsName: cn=admin,dc=blue,dc=local
createTimestamp: 20261007160038Z
entryCSN: 20261007160038.799417Z#000000#000#000000
modifiersName: cn=admin,dc=blue,dc=local
modifyTimestamp: 20261007160038Z
```

#### Creación de la estructura organizativa del directorio
>Una vez creada la raíz del directorio, se procederá a organizar la información mediante diferentes unidades organizativas (OU). Esta organización permite separar los diferentes tipos de objetos almacenados en LDAP y facilita posteriormente las búsquedas y la administración del directorio.

