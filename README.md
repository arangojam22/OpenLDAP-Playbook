<details>
<summary><h2><b> 👩‍💻Playbook/OpenLDAP</b></h2></summary>

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
<img src="capturas/Captura%20de%20pantalla%202026-10-14%20184408.png" alt="aplicación de sudo nano /etc/hosts" /> 

Añadir, conservando las entradas existentes de localhost:
Esta entrada permite resolver el FQDN y el alias en la propia VM sin depender de un servidor DNS para ese nombre. No publica el nombre para otras máquinas.

<img src="capturas/Captura%20de%20pantalla%202026-10-13%20184541.png" 

Resultado esperado: resuelve a 20.0.0.5

### Paso 3. Instalación del software

Con la conectividad de la Red NAT operativa, actualizar los índices de paquetes e instalar el servidor y sus utilidades:

```bash
sudo apt update
sudo apt install slapd ldap-utils
```
<img src="capturas/Captura%20de%20pantalla%202026-10-14%20185947.png" alt="comprobación de instalación de slapd-utils" />

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

Esta validación demuestra autenticación LDAP; no configura por sí sola el inicio de sesión de usuarios, que requiere integración adicional del cliente.

