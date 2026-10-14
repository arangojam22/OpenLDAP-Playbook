# Playbook de despliegue técnico de OpenLDAP

Autor: Jam Luis Arango Jiménez

## Objetivo general

Instalar, configurar y poner en marcha un servicio de directorio OpenLDAP en una máquina virtual de VirtualBox. Este playbook reúne los pasos operativos, la población inicial del directorio y la resolución de incidencias realizadas durante la práctica.

Las capturas están organizadas por su función en el procedimiento, no por su orden de subida. La correspondencia entre imágenes y pasos se basa en las descripciones aportadas por el autor.

## A. Ficha técnica y prerrequisitos del entorno

| Parámetro | Valor |
|---|---|
| Virtualización | VirtualBox |
| Sistema operativo | Ubuntu Server 24.04 LTS, según la descripción del repositorio |
| Topología requerida | Red NAT de VirtualBox sin DHCP |
| Dirección IPv4 | 20.0.0.5/24 |
| Configuración de red | Netplan; DHCP IPv4 desactivado |
| FQDN | ldapserver.planetafp.local |
| Alias local | ldapserver |
| Dominio del directorio | planetafp.local |
| Base del DIT | dc=planetafp,dc=local |
| Organización | planetafp |
| Administrador LDAP | cn=admin,dc=planetafp,dc=local |
| Paquetes | slapd y ldap-utils |
| Acceso necesario | Usuario con permisos sudo en la VM |

Antes del despliegue, comprobar que la Red NAT de VirtualBox utiliza la subred 20.0.0.0/24 y tiene el servicio DHCP desactivado. La desactivación de DHCP en Netplan afecta al cliente de la VM; no sustituye la configuración del servicio DHCP de VirtualBox.

El nombre de la interfaz, la puerta de enlace y los servidores DNS deben verificarse en la VM. No se han confirmado sus valores en esta documentación. Se pueden consultar con:

```bash
ip -br address
ip route
cat /etc/os-release
```

### Componentes principales

- OpenLDAP: implementación del servicio de directorio LDAP. Almacena entradas organizadas jerárquicamente y permite consultarlas, modificarlas y autenticar identidades.
- slapd: demonio que atiende las solicitudes LDAP y gestiona las bases del directorio.
- ldap-utils: utilidades de administración y consulta, como ldapadd, ldapsearch, ldapmodify, ldappasswd y ldapwhoami.
- libldap: biblioteca cliente utilizada por aplicaciones y herramientas para comunicarse con servidores LDAP; las dependencias necesarias se resuelven durante la instalación.

## B. Procedimiento de despliegue paso a paso

### Paso 1. Configuración de red estática

Se desactivó DHCP IPv4 y se asignó la dirección 20.0.0.5/24 mediante Netplan. La captura siguiente corresponde a la edición del YAML con dhcp4: false y addresses.

El bloque siguiente representa la configuración mínima requerida. Sustituir INTERFAZ_REAL por el nombre obtenido con ip -br address; no es una transcripción exacta del archivo de la captura. Para disponer de salida a Internet, añadir la ruta por defecto y los servidores DNS correspondientes a la Red NAT configurada, cuyos valores están pendientes de confirmar.

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    INTERFAZ_REAL:
      dhcp4: false
      addresses:
        - 20.0.0.5/24
```

Guardar la configuración en el archivo YAML activo de /etc/netplan/, evitando definiciones incompatibles de la misma interfaz. Usar espacios, nunca tabulaciones.

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20163247.png" alt="Edición de Netplan: DHCP desactivado y dirección estática" />

Validar y aplicar la configuración:

```bash
sudo netplan generate
sudo netplan apply
ip -br address
ip route
```

netplan generate genera la configuración del backend; netplan apply la activa. La captura siguiente muestra la ejecución de estos comandos, según la descripción del autor. Verificar que la interfaz conserva 20.0.0.5/24.

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20163354.png" alt="Generación y aplicación de la configuración Netplan" />

### Paso 2. Resolución de nombres local

Editar el archivo de resolución local:

```bash
sudo nano /etc/hosts
```

Añadir, conservando las entradas existentes de localhost:

```text
20.0.0.5 ldapserver.planetafp.local ldapserver
```

Esta entrada permite resolver el FQDN y el alias en la propia VM sin depender de un servidor DNS para ese nombre. No publica el nombre para otras máquinas.

Comprobar:

```bash
getent hosts ldapserver.planetafp.local
getent hosts ldapserver
```

Resultado esperado: ambos nombres resuelven a 20.0.0.5. No se ha identificado una captura específica de este paso entre las evidencias descritas.

### Paso 3. Instalación del software

Con la conectividad de la Red NAT operativa, actualizar los índices de paquetes e instalar el servidor y sus utilidades:

```bash
sudo apt update
sudo apt install slapd ldap-utils
```

Durante la instalación, el asistente puede solicitar la contraseña del administrador LDAP. La captura documenta esa pantalla. La contraseña no se reproduce en el playbook.

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20163516.png" alt="Solicitud de contraseña del administrador durante la configuración de slapd" />

### Paso 4. Reconfiguración del servicio

Ejecutar:

```bash
sudo dpkg-reconfigure slapd
```

Registrar las respuestas del asistente. Las preguntas pueden variar según la versión del paquete:

| Parámetro | Respuesta o estado de documentación |
|---|---|
| Omitir la configuración de OpenLDAP | No, para configurar el servicio; respuesta exacta pendiente de evidencia |
| Nombre de dominio DNS | planetafp.local |
| Nombre de organización | planetafp |
| Contraseña del administrador y confirmación | Contraseña elegida durante la práctica; omitida por seguridad |
| Eliminar la base de datos al purgar slapd | Respuesta real pendiente de confirmar. Para conservar los datos al purgar, seleccionar No |
| Mover la base de datos anterior, si se pregunta | Respuesta real pendiente de confirmar; decidir tras revisar y respaldar los datos previos |

El dominio planetafp.local determina la base dc=planetafp,dc=local. La captura siguiente corresponde a la introducción del dominio:

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20163814.png" alt="Configuración del dominio DNS planetafp.local" />

Se estableció planetafp como nombre de organización:

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20163829.png" alt="Configuración de planetafp como organización" />

No volver a ejecutar el asistente sobre una instalación con datos sin revisar previamente sus efectos y disponer de una copia de seguridad.

## C. Verificación y comprobación

### Estado del demonio slapd

```bash
sudo systemctl status slapd --no-pager
sudo systemctl is-active slapd
```

Resultado esperado: active (running) en el estado detallado y active en la comprobación breve. La captura siguiente corresponde al servicio activo:

<img src="capturas/Captura%20de%20pantalla%202026-10-14%20171655.png" alt="Servicio slapd en estado active running" />

### Presencia de esquemas del sistema

Consultar los esquemas cargados mediante la configuración local:

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=schema,cn=config -s one '(objectClass=olcSchemaConfig)' cn
```

Para la estructura de esta práctica, comprobar en particular core, cosine e inetorgperson. Los nombres pueden incluir prefijos numéricos como {0}. También pueden existir otros esquemas, como nis. Esta es una comprobación operativa propuesta; no se ha aportado una captura específica de su ejecución.

### Comprobación de la base del directorio

```bash
ldapsearch -x -LLL -H ldap://localhost -b 'dc=planetafp,dc=local' -s base '(objectClass=*)' dn o
```

Durante la práctica se obtuvo:

```text
dn: dc=planetafp,dc=local
o: planetafp
```

## D. Población del directorio y operaciones básicas

Se utilizaron dos archivos LDIF: ou.ldif para crear el contenedor y jam.ldif para añadir al usuario dentro de él. Crear un archivo LDIF no modifica el servidor: es necesario importarlo.

### 1. Archivo ou.ldif: creación de la unidad organizativa

La OU Usuarios organiza las entradas de usuarios dentro de la base. Su DN identifica su ubicación en el árbol; organizationalUnit define el tipo de entrada y ou establece su nombre.

```ldif
dn: ou=Usuarios,dc=planetafp,dc=local
objectClass: top
objectClass: organizationalUnit
ou: Usuarios
```

La captura siguiente muestra la edición de ou.ldif en GNU Nano:

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20191552.png" alt="Archivo ou.ldif con la unidad organizativa Usuarios" />

Importar primero la OU, ya que debe existir antes de añadir al usuario:

```bash
ldapadd -x -H ldap://localhost -D 'cn=admin,dc=planetafp,dc=local' -W -f ou.ldif
```

Resultado esperado, sin errores posteriores:

```text
adding new entry "ou=Usuarios,dc=planetafp,dc=local"
```

### 2. Archivo jam.ldif: creación del usuario

La entrada uid=jam se sitúa dentro de ou=Usuarios. El archivo empleado durante la práctica fue:

```ldif
dn: uid=jam,ou=Usuarios,dc=planetafp,dc=local
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
uid: jam
cn: jam arango
sn: arango
givenName: jam
mail: jam@planetafp.local
```

Los valores de nombre y apellido son los registrados en el ejemplo de la práctica. uid identifica al usuario, cn representa el nombre común, sn el apellido, givenName el nombre de pila y mail la dirección de correo registrada. Registrar mail no crea un buzón de correo.

La captura siguiente documenta la edición de jam.ldif en GNU Nano:

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20191508.png" alt="Archivo jam.ldif con los atributos del usuario" />

### 3. Inserción y consulta del usuario

Importar el usuario:

```bash
ldapadd -x -H ldap://localhost -D 'cn=admin,dc=planetafp,dc=local' -W -f jam.ldif
```

- -x: autenticación simple.
- -H: URI del servidor LDAP; se utiliza H mayúscula.
- -D: DN con el que se autentica el administrador.
- -W: solicita su contraseña de forma interactiva.
- -f: archivo LDIF que se importa.

Resultado esperado, sin errores posteriores:

```text
adding new entry "uid=jam,ou=Usuarios,dc=planetafp,dc=local"
```

Consultar el usuario delimitando la base, el ámbito y el filtro:

```bash
ldapsearch -x -LLL -H ldap://localhost -b 'dc=planetafp,dc=local' -s sub '(uid=jam)' dn uid cn sn givenName mail
```

-b fija la base de búsqueda; -s sub recorre su subárbol; (uid=jam) selecciona el usuario. -LLL simplifica la salida LDIF. La captura siguiente reúne la importación y la consulta con los atributos del usuario, según la descripción del autor:

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20175432.png" alt="Importación de jam.ldif y consulta del usuario con ldapsearch" />

### 4. Contraseña y autenticación

El archivo jam.ldif mostrado no contiene userPassword. La contraseña se asignó después con ldappasswd. Para repetir la operación sin incluir contraseñas en el historial:

```bash
ldappasswd -x -H ldap://localhost -D 'cn=admin,dc=planetafp,dc=local' -W -S 'uid=jam,ou=Usuarios,dc=planetafp,dc=local'
```

New password y Re-enter new password corresponden a la nueva contraseña del usuario; Enter LDAP Password corresponde al administrador. La entrada de contraseñas no muestra caracteres. Una operación correcta puede terminar sin salida; revisar si existen errores y comprobar la autenticación, en lugar de exigir un texto específico de éxito.

Verificar al usuario:

```bash
ldapwhoami -x -H ldap://localhost -D 'uid=jam,ou=Usuarios,dc=planetafp,dc=local' -W
```

Resultado obtenido:

```text
dn:uid=jam,ou=Usuarios,dc=planetafp,dc=local
```

Verificar al administrador:

```bash
ldapwhoami -x -H ldap://localhost -D 'cn=admin,dc=planetafp,dc=local' -W
```

Resultado obtenido:

```text
dn:cn=admin,dc=planetafp,dc=local
```

La siguiente captura reúne ambas comprobaciones correctas. Según el autor, contiene contraseñas de laboratorio escritas en los comandos. Deben considerarse expuestas: cambiarlas y sustituir la captura por una versión con las contraseñas ocultas si se publica el repositorio. El README no las transcribe.

<img src="capturas/Captura%20de%20pantalla%202026-10-14%20173855.png" alt="Verificación de autenticación del usuario y del administrador LDAP" />

Estructura final:

```text
dc=planetafp,dc=local
└── ou=Usuarios
    └── uid=jam
```

Esta validación demuestra autenticación LDAP; no configura por sí sola el inicio de sesión de usuarios Linux, que requiere integración adicional del cliente.

## E. Control de errores y troubleshooting

### 1. No such object (32)

Se observó durante las búsquedas. Puede indicar una base o un padre inexistente, un DN incorrecto o una búsqueda que utiliza una base por defecto inadecuada. En la práctica se corrigió el comando incluyendo explícitamente la base:

```bash
ldapsearch -x -LLL -H ldap://localhost -b 'dc=planetafp,dc=local' -s base '(objectClass=*)' dn
```

Para una inserción, comprobar que la base existe y que ou.ldif se importa antes que jam.ldif. Revisar siempre el comando y el mensaje completo; el código por sí solo no identifica una causa única.

### 2. Already exists (68)

Apareció al intentar añadir una entrada ya existente. No es necesario duplicarla. Comprobar con una consulta autenticada:

```bash
ldapsearch -x -LLL -H ldap://localhost -D 'cn=admin,dc=planetafp,dc=local' -W -b 'ou=Usuarios,dc=planetafp,dc=local' -s base '(objectClass=*)' dn ou
```

Si hay que cambiar atributos de una entrada existente, usar ldapmodify en lugar de volver a crearla con ldapadd.

### 3. Unauthenticated bind (DN with no password) disallowed (53)

Este mensaje adicional indica que el cliente presentó un DN sin contraseña de autenticación. Distinguir la contraseña nueva del usuario de la contraseña con la que se autentica el administrador. Comprobar el bind por separado con ldapwhoami y -W; no desactivar las restricciones del servidor para solventarlo.

### 4. Invalid credentials (49) y restablecimiento del administrador

Se observó cuando el bind no validaba las credenciales. Verificar el DN y la contraseña; el error no demuestra por sí solo que la entrada exista ni que el único problema sea una contraseña incorrecta.

Para restablecer al administrador se utilizó acceso local a cn=config. Identificar primero el DN de configuración y el administrador reales:

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b cn=config '(olcSuffix=dc=planetafp,dc=local)' dn olcRootDN
```

En la práctica se utilizó olcDatabase={1}mdb,cn=config. Generar un hash:

```bash
slappasswd
```

slappasswd solo genera el hash; no cambia la contraseña del servidor. Preparar admin_password.ldif con el hash completo generado, sustituyendo el marcador siguiente:

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: HASH_COMPLETO_GENERADO_POR_SLAPPASSWD
```

Aplicar y comprobar:

```bash
sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f admin_password.ldif
ldapwhoami -x -H ldap://localhost -D 'cn=admin,dc=planetafp,dc=local' -W
```

La línea modifying entry anuncia el intento de modificación; confirmar que no hay errores posteriores y que ldapwhoami autentica correctamente. No guardar hashes ni contraseñas en el repositorio.

### 5. Prompt > y lista de opciones

El prompt > de Bash indica una entrada incompleta, por ejemplo una comilla sin cerrar. Pulsar Ctrl+C y volver a ejecutar el comando desde el prompt normal.

Si una herramienta muestra su ayuda, revisar la primera línea del error y las opciones. ldappasswd cambia contraseñas; ldapwhoami verifica la identidad. La opción -s de ldappasswd establece una contraseña y -S la solicita de forma interactiva. Que aparezca la ayuda no demuestra que -s sea incompatible con la versión instalada.

### 6. Prevención de errores LDIF y de resolución local

En LDIF, separar entradas mediante una línea vacía, no introducir espacios iniciales accidentales y conservar los DN exactos. Antes de importar, revisar el archivo con cat -n. Si el FQDN no resuelve, comprobar la línea de /etc/hosts con getent hosts.

### Seguridad y alcance

Los ejemplos utilizan el servidor local de la VM. Para conexiones de red con autenticación simple, configurar TLS antes de transmitir credenciales. Usar contraseñas únicas y robustas, -W en lugar de escribirlas en los comandos y revisar las capturas antes de publicar.

### Datos pendientes de completar

- Confirmar el nombre real de interfaz, la puerta de enlace y los servidores DNS de Netplan.
- Registrar las respuestas reales del asistente sobre eliminación al purgar y movimiento de la base anterior.
- Añadir evidencia específica de /etc/hosts y de los esquemas cargados.
- Sustituir la captura final por una versión sin contraseñas visibles y rotar las credenciales de laboratorio expuestas.
