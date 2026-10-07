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

Crearemos el archivo `estructura.ldif`
```ldif
dn: ou=People,dc=blue,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=blue,dc=local
objectClass: organizationalUnit
ou: Groups
```
Una vez creado el fichero LDIF, se introducen las entradas en el directorio mediante `ldapadd`:
```bash
ldapadd -x -D "cn=admin,dc=blue,dc=local" -W -f estructura.ldif
```
Los parámetros utilizados son:
- `-x`: utiliza autenticación simple LDAP en lugar de SASL.
- `-D`: especifica el DN del usuario utilizado para realizar la operación.
- `-W`: solicita la contraseña del administrador LDAP.
- `-f estructura.ldif`: indica el fichero LDIF que contiene las entradas que se quieren añadir.

Una vez creadas las unidades organizativas, se comprueba que ambas entradas se encuentran almacenadas en el directorio:
```bash
ldapsearch -x \
  -D "cn=admin,dc=blue,dc=local" \
  -W \
  -b "dc=blue,dc=local" \
  "(objectClass=organizationalUnit)"
```
El resultado debe contener las dos unidades:
```
# extended LDIF
#
# LDAPv3
# base <dc=blue,dc=local> with scope subtree
# filter: (objectClass=organizationalUnit)
# requesting: ALL
#

# People, blue.local
dn: ou=People,dc=blue,dc=local
objectClass: organizationalUnit
ou: People

# Groups, blue.local
dn: ou=Groups,dc=blue,dc=local
objectClass: organizationalUnit
ou: Groups

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```

Tras completar el apartado, el árbol LDAP queda organizado de la siguiente forma:
```text
dc=blue,dc=local
│
├── ou=People,dc=blue,dc=local
│   └── Usuarios LDAP
│
└── ou=Groups,dc=blue,dc=local
    └── Grupos LDAP
```

#### Unidad organizativa `People`
La unidad organizativa `People` fue creada previamente en el **apartado 1.3**, junto con `Groups`. Esta unidad será utilizada para almacenar las cuentas de usuario del directorio LDAP.

Su estructura es:
```
dn: ou=People,dc=blue,dc=local
objectClass: organizationalUnit
ou: People
```

Para comprobar que la unidad se ha creado correctamente, se realiza una consulta sobre su DN:
```bash
ldapsearch -x \
  -D "cn=admin,dc=blue,dc=local" \
  -W \
  -b "ou=People,dc=blue,dc=local" \
  -s base
```

El resultado muestra:
```
dn: ou=People,dc=blue,dc=local
objectClass: organizationalUnit
ou: People
```

También se puede comprobar que `People` se encuentra directamente bajo la raíz del directorio:
```bash
ldapsearch -x \
  -D "cn=admin,dc=blue,dc=local" \
  -W \
  -b "dc=blue,dc=local" \
  -s one
```
De esta forma se verifica que `ou=People,dc=blue,dc=local` existe y está correctamente situada dentro del árbol LDAP. Los usuarios que se creen posteriormente se almacenarán bajo esta unidad.

#### Unidad organizativa `Groups`
La unidad organizativa `Groups` fue creada previamente en el **apartado 1.3**, junto con `People`. Esta unidad será utilizada para almacenar los grupos del directorio LDAP.

Su estructura es:
```
dn: ou=Groups,dc=blue,dc=local
objectClass: organizationalUnit
ou: Groups
```
Para comprobar que la unidad se ha creado correctamente:
```bash
ldapsearch -x \
  -D "cn=admin,dc=blue,dc=local" \
  -W \
  -b "ou=Groups,dc=blue,dc=local" \
  -s base
```
El resultado debe mostrar:
```
dn: ou=Groups,dc=blue,dc=local
objectClass: organizationalUnit
ou: Groups
```
También podemos comprobar que se encuentra directamente bajo la raíz:
```bash
ldapsearch -x \
  -D "cn=admin,dc=blue,dc=local" \
  -W \
  -b "dc=blue,dc=local" \
  -s one
```

Con estas comprobaciones verificamos que `ou=Groups,dc=blue,dc=local` existe y está correctamente situada dentro del árbol LDAP.

#### Creación del usuario `dummyadmin`
Para generar el hash se utilizará la herramienta `slappasswd`:
```bash
slappasswd -s "palangana2026.ABC"
```
y el hash resultante lo meteremos en el archivo que hemos creado `dummyadmin.ldif`:
```
dn: uid=dummyadmin,ou=People,dc=blue,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: dummyadmin
cn: dummyadmin
sn: dummyadmin
givenName: dummyadmin
displayName: dummyadmin
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/dummyadmin
loginShell: /bin/bash
userPassword: {SSHA}HASH_COMPLETO
```

Teniendo esto podemos añadir el usuario a la estructura:
```bash
ldapadd -x -D "cn=admin,dc=blue,dc=local" -W -f dummyadmin.ldif
```
al hacerlo nos pide la contraseña del `admin` y responde con:
```text
adding new entry "uid=dummyadmin,ou=People,dc=blue,dc=local"
```
El mensaje `adding new entry` confirma que la entrada del usuario `dummyadmin` ha sido creada correctamente dentro de la unidad organizativa `People`.
El DN completo del usuario es:
```text
uid=dummyadmin,ou=People,dc=blue,dc=local
```
Por tanto, el usuario ha quedado almacenado correctamente en el directorio LDAP.

## Configuración de un cliente `OpenLDAP` en `LinuxClient`
En la máquina `linuxclient` deberemos descargar el `ldap`:
```bash
sudo apt update
sudo apt install sssd-ldap ldap-utils
```


