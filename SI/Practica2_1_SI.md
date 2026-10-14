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

Creamos tambien el grupo al que va a pertenecer el usuario `dummyadmin` en el archivo `dummyadmin_group.ldif`:
```
dn: cn=dummyadmin,ou=Groups,dc=blue,dc=local
objectClass: posixGroup
cn: dummyadmin
gidNumber: 10001
```
nos responde:
```
adding new entry "cn=dummyadmin,ou=Groups,dc=blue,dc=local"
```

## Configuración de un cliente `OpenLDAP` en `LinuxClient`
En la máquina `linuxclient` deberemos descargar el `ldap`:
```bash
sudo apt update
sudo apt install sssd-ldap ldap-utils
```
Se utiliza **SSSD (System Security Services Daemon)** como intermediario entre el sistema Linux y el servidor LDAP. De esta forma, las consultas de usuarios y grupos y la autenticación pueden realizarse contra el directorio centralizado.

#### Configuración de SSSD
Se configuró el archivo `/etc/sssd/sssd.conf` con la siguiente configuración:
```
[sssd]
services = nss, pam
domains = blue

[domain/blue]
id_provider = ldap
auth_provider = ldap

ldap_uri = ldap://192.168.57.10
ldap_search_base = dc=blue,dc=local

ldap_user_search_base = ou=People,dc=blue,dc=local
ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_user_uid_number = uidNumber
ldap_user_gid_number = gidNumber
ldap_user_home_directory = homeDirectory
ldap_user_shell = loginShell

ldap_group_search_base = ou=Groups,dc=blue,dc=local
ldap_group_object_class = posixGroup
ldap_group_name = cn
ldap_group_gid_number = gidNumber
ldap_id_use_start_tls = false
```

La configuración define `LDAP` como proveedor tanto de identidad como de autenticación:
```ini
id_provider = ldap
auth_provider = ldap
```
El servidor utilizado es:
```ini
ldap_uri = ldap://192.168.57.10
```
y la búsqueda se realiza dentro de:
```text
dc=blue,dc=local
```
Los usuarios se buscan en:
```text
ou=People,dc=blue,dc=local
```
mientras que los grupos se buscan en:
```text
ou=Groups,dc=blue,dc=local
```
Además, se estableció explícitamente:
```ini
ldap_id_use_start_tls = false
```
para evitar que SSSD intente establecer STARTTLS durante esta fase de la práctica. Esto es coherente con que la configuración de LDAPS/TLS se aborda posteriormente

Posteriormente se reinició el servicio:
```bash
sudo systemctl restart sssd
```
y se comprobó su estado:
```bash
sudo systemctl status sssd --no-pager
```
El servicio quedó en estado:
```text
Active: active (running)
```

#### Eliminación del usuario local y autenticación centralizada
El objetivo del apartado 2.5 es comprobar que `dummyadmin` ya no necesita existir como usuario local en `LinuxClient`, sino que puede ser identificado y autenticado mediante el servidor LDAP utilizando SSSD.
Antes de realizar la prueba se comprobó la existencia del usuario local:
```bash
grep '^dummyadmin:' /etc/passwd
```
Una vez eliminado el usuario local:
```bash
sudo userdel dummyadmin
```
SSSD debe obtener la información de `dummyadmin` directamente desde LDAP.
Para comprobarlo:
```bash
getent passwd dummyadmin
```
#### Resultado de la configuración
Al finalizar estos apartados se dispone de una arquitectura en la que:
```text
                    ┌──────────────────────┐
                    │     LinuxBackup      │
                    │                      │
                    │      OpenLDAP        │
                    │   192.168.57.10      │
                    │                      │
                    │ dc=blue,dc=local     │
                    │ ├── People           │
                    │ │   └── dummyadmin   │
                    │ └── Groups 
                    │     └── dummyadmin
                    └──────────┬───────────┘
                               │
                               │ LDAP
                               │
                    ┌──────────▼───────────┐
                    │     LinuxClient      │
                    │                      │
                    │        SSSD          │
                    │     NSS + PAM        │
                    └──────────────────────┘
```

De esta forma, los datos de los usuarios se mantienen centralizados en `LinuxBackup`, mientras que `LinuxClient` utiliza SSSD para consultar y autenticar dichos usuarios contra `OpenLDAP`.

No se porqué el usuario existe pero me pide comunicación por TLS cosa que no me permite seguir. El profesor me dice que continue configurando los certificados CA y después que vuelva a probar.

Terminamos esta parte con el servicio correcto pero mal la configuración.

## Configuración de LDAPS
#### `linuxbackup`
###### Crear la configuración del certificado
Crearemos en el directorio para los certificados en `linuxbackup`:
```bash
sudo mkdir -p /etc/ldap/certs
sudo nano ../../etc/ldap/certs/schema.conf 
```
donde la configuración del `schema.conf` será:
```conf
[req]
distinguished_name = dn
req_extensions = ext
prompt = no

[dn]
CN = linuxbackup.blue.local

[ext]
subjectAltName = DNS:linuxbackup.blue.local
```

###### Crear la Autoridad de Certificación
Para eso iremos a la carpeta de los certificados y generaremos la clave privada de la CA:
```bash
cd ../../etc/ldap/certs
sudo openssl genrsa -out archivoCA.key 2048
```
Generamos el certificado de la CA:
```bash
sudo openssl req -x509 -new -key archivoCA.key -sha256 -days 36500 -out archivoCA.crt -subj "/CN=blue.local CA"
```

###### Generar la clave privada del servidor LDAP
Ahora generamos la clave privada que utilizará **LinuxBackup** para LDAPS:
```bash
sudo openssl genrsa -out archivo.key 2048
```
Comprobamos que existe:
```bash
sudo ls -l archivo.key
```
La clave debe permanecer protegida. Podemos dejarla accesible únicamente para root:
```bash
sudo chmod 600 archivo.key
```

###### Generar la CSR
Generamos la solicitud de certificado:
```bash
sudo openssl req -new -key archivo.key -out archivo.csr -config schema.conf
```
Podemos comprobar la CSR:
```bash
sudo openssl req -in archivo.csr -noout -subject
```
Debería aparecer:
```
subject=CN = linuxbackup.blue.local
```
Y es importante comprobar el SAN:
```bash
sudo openssl req -in archivo.csr -noout -text | grep -A1 "Subject Alternative Name"
```
Debería aparecer:
```
X509v3 Subject Alternative Name:
    DNS:linuxbackup.blue.local
```

###### Firmar el certificado
Ahora utilizamos la CA que acabamos de crear para firmar la CSR:

```bash
sudo openssl x509 -req -in archivo.csr -CA archivoCA.crt -CAkey archivoCA.key -CAcreateserial -out archivo.crt -days 36500 -sha256 -extensions ext -extfile schema.conf
```
Comprobamos el certificado:
```bash
sudo openssl x509 -in archivo.crt -noout -subject -issuer
```
Deberíamos obtener:
```
subject=CN = linuxbackup.blue.local
issuer=CN = blue.local CA
```
Y comprobamos el SAN:
```bash
sudo openssl x509 -in archivo.crt -noout -text | grep -A1 "Subject Alternative Name"
```
Debe aparecer:
```
DNS:linuxbackup.blue.local
```
Finalmente podemos verificar que la CA ha firmado correctamente el certificado:
```bash
sudo openssl verify -CAfile archivoCA.crt archivo.crt
```
Resultado esperado:
```
archivo.crt: OK
```

###### Resultado hasta aquí
En `LinuxBackup` deberíamos tener:
```ls
/etc/ldap/certs/
├── archivoCA.key       ← clave privada de la CA
├── archivoCA.crt       ← certificado de la CA
├── archivo.key         ← clave privada de LinuxBackup
├── archivo.csr         ← solicitud de certificado
├── archivo.crt         ← certificado de LinuxBackup
├── archivoCA.srl       ← generado al firmar
└── schema.conf
```

###### Ajustamos los permisos
En `LinuxBackup`, ejecuta:
```bash
sudo chown root:openldap /etc/ldap/certs/archivo.key
sudo chmod 640 /etc/ldap/certs/archivo.key

sudo chown root:root /etc/ldap/certs/archivo.crt
sudo chmod 644 /etc/ldap/certs/archivo.crt

sudo chown root:root /etc/ldap/certs/archivoCA.crt
sudo chmod 644 /etc/ldap/certs/archivoCA.crt
```
La clave privada debe poder leerla el proceso de `OpenLDAP`, pero no los usuarios normales. El certificado público puede ser legible.

###### Reconfigurar `OpenLDAP`
Vamos a modificar la configuración dinámica de `OpenLDAP` mediante `cn=config`.
Crea el archivo:
```bash
sudo nano /etc/ldap/certs/tls.ldif
```
Introduce:
```
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/certs/archivoCA.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/certs/archivo.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/certs/archivo.key
```
Aplica los cambios localmente usando la autenticación EXTERNAL por el socket LDAP:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/certs/tls.ldif
```
Deberías obtener varias respuestas `modifying entry "cn=config"` o una confirmación equivalente.
###### Activar el listener LDAPS
Comprueba cómo está arrancando actualmente `slapd`:
```bash
sudo systemctl cat slapd
```
Los listeners suelen configurarse en:
```
sudo nano /etc/default/slapd
```
Busca la variable `SLAPD_SERVICES`. Debe incluir tanto LDAP local como LDAPS. Por ejemplo:
```
SLAPD_SERVICES="ldap:/// ldapi:/// ldaps:///"
```
Después reiniciamos:
```
sudo systemctl restart slapd
sudo systemctl status slapd --no-pager
```
###### Verificar LDAPS en `LinuxBackup`
Ejecuta:
```
openssl s_client \
  -connect 127.0.0.1:636 \
  -servername linuxbackup.blue.local \
  -CAfile /etc/ldap/certs/archivoCA.crt \
  -verify_hostname linuxbackup.blue.local
```
Al final de la salida, buscamos:
```
Verify return code: 0 (ok)
```

#### `linuxclient`
###### Configurar `LinuxClient` para confiar en la CA
Una vez que LDAPS funcione en `LinuxBackup`, copia solo el certificado público de la CA, nunca `archivoCA.key`, a `LinuxClient`.
Desde `LinuxClient`, puedes copiarlo por SSH si tienes acceso:
```bash
sudo mkdir -p /etc/ldap/certs
sudo scp dummyadmin@192.168.57.10:/etc/ldap/certs/archivoCA.crt /tmp/archivoCA.crt
sudo cp /tmp/archivoCA.crt /etc/ldap/certs/archivoCA.crt
sudo chmod 644 /etc/ldap/certs/archivoCA.crt
```
Prueba la conexión desde `LinuxClient`:
```bash
LDAPTLS_CACERT=/etc/ldap/certs/archivoCA.crt \
ldapsearch -x \
  -H ldaps://linuxbackup.blue.local \
  -D "cn=admin,dc=blue,dc=local" \
  -W \
  -b "dc=blue,dc=local" \
  -s base
```
Para que el nombre `linuxbackup.blue.local` resuelva correctamente, comprueba en `LinuxClient`:
```bash
getent hosts linuxbackup.blue.local
```
###### Cambiar SSSD a LDAPS
Solo cuando la consulta anterior funcione, cambia en `/etc/sssd/sssd.conf` de `LinuxClient`:
```
[sssd]
services = nss, pam
domains = blue

[domain/blue]
id_provider = ldap
auth_provider = ldap

ldap_uri = ldaps://linuxbackup.blue.local
ldap_search_base = dc=blue,dc=local

ldap_user_search_base = ou=People,dc=blue,dc=local
ldap_user_object_class = posixAccount
ldap_user_name = uid
ldap_user_uid_number = uidNumber
ldap_user_gid_number = gidNumber
ldap_user_home_directory = homeDirectory
ldap_user_shell = loginShell

ldap_group_search_base = ou=Groups,dc=blue,dc=local
ldap_group_object_class = posixGroup
ldap_group_name = cn
ldap_group_gid_number = gidNumber

ldap_tls_cacert = /etc/ldap/certs/archivoCA.crt
ldap_tls_reqcert = demand
```
Mantén el resto de tu configuración de usuarios y grupos. Después:
```
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
sudo journalctl -u sssd --no-pager -n 30
```
Para hacer la comprobación del usuario usando `LDAPS`accedemos por `ssh` que es lo que nos estuvo fallando el día de la práctica. FUNCIONA

## Creación de un schema propio para almacenar claves SSH
El objetivo de este apartado es ampliar el esquema de OpenLDAP para permitir almacenar las claves públicas SSH de los usuarios directamente en el directorio LDAP. De esta manera, SSSD podrá consultar las claves públicas y utilizarlas posteriormente para la autenticación SSH.

En este apartado se utilizará exclusivamente el usuario `dummyadmin` para realizar las pruebas, sin modificar las entradas de otros usuarios.
#### Creación del esquema personalizado
El esquema se crea en el servidor `LinuxBackup`, dentro de la configuración dinámica de `OpenLDAP`:
```
cn=schema,cn=config
```
El DN previsto para el nuevo esquema es:
```
cn=blueSSH,cn=schema,cn=config
```
Antes de crearlo, se comprueba qué esquemas están cargados:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// \
  -b "cn=schema,cn=config" \
  -LLL dn cn
```
La consulta devuelve los esquemas `core`, `cosine`, `nis` e `inetorgperson`. No existe previamente un esquema denominado `blueSSH`.
#### Definición del esquema
Se crea el archivo `blueSSH.ldif`:
```bash
sudo nano blueSSH.ldif
```
El archivo contiene la definición del atributo para almacenar la clave pública y una clase auxiliar que permite utilizar dicho atributo en las entradas de usuario:
```
dn: cn=blueSSH,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: blueSSH
olcAttributeTypes: ( 1.3.6.1.4.1.42.1.1 NAME 'ssh-authorized-key' DESC 'SSH public key' EQUALITY octetStringMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
olcObjectClasses: ( 1.3.6.1.4.1.42.1.2 NAME 'blueSSHAccount' DESC 'Auxiliary class for SSH public keys' SUP top AUXILIARY MAY ssh-authorized-key )
```
Los elementos principales son:
- `ssh-authorized-key`: atributo destinado a almacenar la clave pública SSH.
- `blueSSHAccount`: clase auxiliar que permite incorporar el atributo a una entrada LDAP de usuario.
- `1.3.6.1.4.1.42.1`: OID base utilizado para las definiciones del esquema.
- `1.3.6.1.4.1.1466.115.121.1.40`: sintaxis LDAP utilizada para almacenar una secuencia de octetos.

**Nota:** el enunciado original utiliza el nombre `ssh_authorized_key`, pero `OpenLDAP` rechaza esa definición porque el guion bajo no es válido en un nombre de atributo LDAP. Por ello, se ha utilizado `ssh-authorized-key`.
### Carga del esquema
El esquema se carga mediante la interfaz local de administración de `OpenLDAP`, utilizando SASL/EXTERNAL:
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f blueSSH.ldif
```
La carga debe finalizar correctamente antes de continuar con la modificación de las entradas de usuario.
Para comprobar que el esquema está disponible:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=schema,cn=config" -LLL '(cn=blueSSH)' dn cn
```
#### Usuario al que se asociará la clave
La configuración se realizará únicamente para el usuario `dummyadmin`, cuya entrada se encuentra en:
```
uid=dummyadmin,ou=People,dc=blue,dc=local
```
La clave pública se almacenará como atributo de esa entrada LDAP. No se guardará la clave privada en el directorio.

La clave privada permanecerá en `LinuxClient`, mientras que la clave pública se incorporará a LDAP para que SSSD pueda recuperarla cuando se configure la autenticación SSH.

#### En `LinuxClient`, generamos el par de claves y mostramos la pública:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/dummyadmin_ed25519
cat ~/.ssh/dummyadmin_ed25519.pub
```

#### En `LinuxBackup`, crea el archivo `dummyadmin_ssh.ldif`:

```bash
nano dummyadmin_ssh.ldif
```

Introducimos lo siguiente:
```
dn: uid=dummyadmin,ou=People,dc=blue,dc=local
changetype: modify
add: objectClass
objectClass: blueSSHAccount
-
add: ssh-authorized-key
ssh-authorized-key: CLAVE SSH + COMENTARIO
```
#### Aplica la modificación en `LinuxBackup`:
```bash
sudo ldapmodify -x \
  -D "cn=admin,dc=blue,dc=local" -W \
  -f dummyadmin_ssh.ldif
```

#### Comprueba que se ha guardado en LDAP:
```bash
sudo ldapsearch -x \
  -D "cn=admin,dc=blue,dc=local" -W \
  -b "uid=dummyadmin,ou=People,dc=blue,dc=local" \
  -LLL ssh-authorized-key
```
Si todo funciona, veremos la clave pública almacenada como atributo LDAP. FUNCIONA.

Modificamos en `linuxclient` de donde se obtiene la configuración de `ssh`
Elimino esta línea:

```
Include /usr/....
```

En su lugar, añade al final del archivo estas directivas:

```
PubkeyAuthentication yes
AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys
AuthorizedKeysCommandUser nobody
```

## Configuración en el resto de dispositivos: `linuxrouter1` `linuxrouter2` `linuxserver`
En `linuxbackup` crearemos los usuarios y grupos: `alice` `trudy` `bob` con sus respectivos grupos. Al final `LDAP` quedará asi:
```bash
ldapsearch -x -D "cn=admin,dc=blue,dc=local" -W \
  -b "dc=blue,dc=local" \
  "(|(objectClass=posixAccount)(objectClass=posixGroup))"
```
La respuesta:
```
# extended LDIF
#
# LDAPv3
# base <dc=blue,dc=local> with scope subtree
# filter: (|(objectClass=posixAccount)(objectClass=posixGroup))
# requesting: ALL
#

# dummyadmin, People, blue.local
dn: uid=dummyadmin,ou=People,dc=blue,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
objectClass: blueSSHAccount
uid: dummyadmin
cn: dummyadmin
sn: dummyadmin
givenName: dummyadmin
displayName: dummyadmin
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/dummyadmin
loginShell: /bin/bash
userPassword:: e1NTSEF9cmMxcjlEUnMvQzN5andGb2s0U2YvYVAzYktaOWd3b2s=
ssh-authorized-key: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICErU2w4wkvRo7fHBsHGgG
 anvwc3pj2PEsaGeVtMXHl7 dummyadmin@linuxclient

# dummyadmin, Groups, blue.local
dn: cn=dummyadmin,ou=Groups,dc=blue,dc=local
objectClass: posixGroup
cn: dummyadmin
gidNumber: 10001

# alice, Groups, blue.local
dn: cn=alice,ou=Groups,dc=blue,dc=local
objectClass: posixGroup
cn: alice
gidNumber: 10002

# alice, People, blue.local
dn: uid=alice,ou=People,dc=blue,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: alice
cn: alice
sn: alice
givenName: alice
displayName: alice
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/alice
loginShell: /bin/bash
userPassword:: e1NTSEF9NDM3eGtRZC9kNzY4MTF3aHJlNDdEb2QzeXZDS1ZDb2o=

# bob, People, blue.local
dn: uid=bob,ou=People,dc=blue,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: bob
cn: bob
sn: bob
givenName: bob
displayName: bob
uidNumber: 10003
gidNumber: 10003
homeDirectory: /home/bob
loginShell: /bin/bash
userPassword:: e1NTSEF9aHVBSlY2Q0p4bVZZdk1pZkFibkZibzlWRXBxVG5BaVk=

# trudy, People, blue.local
dn: uid=trudy,ou=People,dc=blue,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: trudy
cn: trudy
sn: trudy
givenName: trudy
displayName: trudy
uidNumber: 10004
gidNumber: 10004
homeDirectory: /home/trudy
loginShell: /bin/bash
userPassword:: e1NTSEF9N09PcERrQkg3dDdNaWZVdzNtS1VlMEl4WHovLzlYK2E=

# bob, Groups, blue.local
dn: cn=bob,ou=Groups,dc=blue,dc=local
objectClass: posixGroup
cn: bob
gidNumber: 10003

# trudy, Groups, blue.local
dn: cn=trudy,ou=Groups,dc=blue,dc=local
objectClass: posixGroup
cn: trudy
gidNumber: 10004
```
Esto nos indica que todos los usuarios están creados. Que solo tiene una contraseña ssh el usuario `dummyadmin` en la máquina `linuxclient`

Instalaremos en las maquinas `linux` restantes de la maqueta:
```bash
sudo apt install sssd-ldap ldap-utils
```

