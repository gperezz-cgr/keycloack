# Laboratorio 00 — Preparación de AlmaLinux 9.8 para Keycloak

> **Guía ampliada y bitácora de ejecución real** · Curso Administración Keycloak · Guía base: **Patricio Canelo** · 21 de octubre de 2026  
> **Alcance:** preparar el VPS; **no instala Keycloak**. Integra la secuencia del profesor, las verificaciones realizadas y los imprevistos encontrados.

## Índice

1. [Objetivo y arquitectura](#1-objetivo-y-arquitectura)
2. [Datos del laboratorio y convenciones](#2-datos-del-laboratorio-y-convenciones)
3. [Paso 1 — Acceso SSH, Webmin y Virtualmin](#3-paso-1--acceso-ssh-webmin-y-virtualmin)
4. [Paso 2 — Actualizar AlmaLinux](#4-paso-2--actualizar-almalinux)
5. [Paso 3 — FQDN, DNS y cloud-init](#5-paso-3--fqdn-dns-y-cloud-init)
6. [Paso 4 — Certbot y Let's Encrypt](#6-paso-4--certbot-y-lets-encrypt)
7. [Paso 5 — HTTPS en Webmin](#7-paso-5--https-en-webmin)
8. [Paso 6 — Apache y listado de directorios](#8-paso-6--apache-y-listado-de-directorios)
9. [Paso 7 — Servidor virtual en Virtualmin](#9-paso-7--servidor-virtual-en-virtualmin)
10. [Paso 8 — Usuario de trabajo y sudo](#10-paso-8--usuario-de-trabajo-y-sudo)
11. [Paso 9 — Docker Engine y Compose](#11-paso-9--docker-engine-y-compose)
12. [Paso 10 — Docker sin sudo](#12-paso-10--docker-sin-sudo)
13. [Paso 11 — Reinicio y prueba integral](#13-paso-11--reinicio-y-prueba-integral)
14. [Imprevistos y soluciones](#14-imprevistos-y-soluciones)
15. [Diccionario de comandos y paquetes](#15-diccionario-de-comandos-y-paquetes)
16. [Seguridad, mantenimiento y mejoras](#16-seguridad-mantenimiento-y-mejoras)
17. [Checklist final y próximos pasos](#17-checklist-final-y-próximos-pasos)
18. [Incidente SSL de Webmin y Virtualmin — diagnóstico y solución paso a paso](#18-incidente-posterior-al-laboratorio-webmin-entrega-un-certificado-autofirmado-en-lugar-de-lets-encrypt-resuelto)

---

## 1. Objetivo y arquitectura

Preparar un servidor Linux accesible por SSH, con nombre de dominio consistente, certificado TLS válido para Webmin, Apache/Virtualmin revisados y Docker Engine + Docker Compose operativos desde un usuario de trabajo. Esta es la infraestructura base para **futuros laboratorios de Keycloak**, no un despliegue de Keycloak.

```text
PC Windows / PowerShell
        |
        | SSH (22) / HTTPS (10000)
        v
VPS Contabo — AlmaLinux 9.8
  FQDN: gperezz.planeta0.com
  ├─ SSH / usuario gperezz / sudo
  ├─ Webmin (administración del sistema, HTTPS 10000)
  ├─ Virtualmin (administración de dominios y VirtualHosts)
  ├─ Apache httpd (HTTP 80 / HTTPS 443)
  ├─ Let's Encrypt / Certbot (certificados TLS)
  └─ Docker Engine + containerd + Buildx + Compose
       └─ Futuro despliegue de Keycloak (aún NO instalado)
```

**Conceptos:** un **VPS** es una máquina virtual; **FQDN** es el nombre de dominio completo; **DNS** traduce nombres a direcciones IP; **TLS** cifra la comunicación; **VirtualHost** permite a Apache servir sitios diferenciados; **Docker Engine** ejecuta contenedores; **Compose** describe y coordina varios servicios mediante YAML.

## 2. Datos del laboratorio y convenciones

| Parámetro | Valor observado |
|---|---|
| Proveedor | Contabo |
| Distribución | AlmaLinux 9.8 (Olive Jaguar), familia RHEL 9 |
| FQDN | `gperezz.planeta0.com` |
| IP pública IPv4 | `169.58.144.183` |
| IP pública IPv6 | `2a02:c207:2367:7994::1` |
| Usuario inicial | `root` |
| Usuario operativo | `gperezz` (UID 1002; creado por Virtualmin) |
| Grupos del usuario | `gperezz`, `wheel`, `docker` |
| Apache | Servicio `httpd` |
| Kernel actualizado observado | `5.14.0-687.58.1.el9_8` |
| Docker Engine | `29.9.2` |
| containerd | `2.4.2` |
| Buildx | `0.38.1` |
| Docker Compose | `v5.6.1` |
| Certificado Let's Encrypt | `gperezz.planeta0.com`, expiración observada 2027-01-19 |

**Convenciones:** los comandos con `#` en el prompt se ejecutan como `root`; los de `$` como usuario normal. **No copies los caracteres del prompt**. Si adaptas esta guía a otro VPS, reemplaza FQDN, usuario e IP. Nunca publiques contraseñas, claves privadas ni tokens.

## 3. Paso 1 — Acceso SSH, Webmin y Virtualmin

Desde **PowerShell**:

```powershell
ssh root@169.58.144.183
# O si el DNS está operativo:
ssh root@gperezz.planeta0.com
```

- `ssh`: abre una sesión remota cifrada mediante Secure Shell.
- `root@...`: indica usuario y servidor de destino.
- Si es el primer acceso, verifica la huella digital del servidor antes de aceptar la clave del host.

Abre el panel en el navegador:

```text
https://gperezz.planeta0.com:10000
```

- `https`: HTTP protegido por TLS.
- `:10000`: puerto habitual de Webmin; no es el puerto estándar HTTPS `443`.
- **Webmin** administra el sistema; **Virtualmin** se integra para gestionar dominios, usuarios y sitios web.

Si la contraseña inicial de `root` es temporal, la guía contempla cambiarla desde la sesión de root:

```bash
passwd
```

`passwd` modifica la contraseña **del usuario actual**; `passwd OTRO_USUARIO` cambia la de ese usuario si tienes privilegios. **No es obligatorio volver a cambiar contraseñas que ya sean seguras**.

**Verificación:** acceso SSH funcional y panel Webmin accesible.

## 4. Paso 2 — Actualizar AlmaLinux

```bash
cat /etc/almalinux-release
uname -a
dnf update -y
```

- `cat`: **muestra el contenido de un archivo**; no instala nada y normalmente ya viene incluido en el paquete `coreutils`.
- `/etc/almalinux-release`: indica versión de la distribución.
- `uname -a`: muestra información del kernel y arquitectura.
- `dnf`: administrador de paquetes de AlmaLinux/RHEL; `update` instala actualizaciones; `-y` acepta confirmaciones automáticamente.

En este VPS se actualizó el kernel a `5.14.0-687.58.1.el9_8`. Tras actualizar el kernel, reiniciar permite cargarlo:

```bash
reboot
# Reconectar por SSH después del arranque
uname -r
cat /etc/almalinux-release
```

- `reboot`: reinicia el servidor y corta temporalmente las conexiones.
- `uname -r`: muestra **solo la versión del kernel en ejecución**.

**Imprevisto registrado:** el hostname inicial era `fqdn.example.com`; se corrigió en el paso siguiente. No se debe confundir el nombre de host con la versión del sistema operativo.

## 5. Paso 3 — FQDN, DNS y cloud-init

### 5.1 Verificar y fijar el nombre del servidor

```bash
hostnamectl
hostname -f
# Solo si el nombre actual es incorrecto:
sudo hostnamectl set-hostname gperezz.planeta0.com
hostnamectl --static
hostname -f
```

- `hostnamectl`: consulta o modifica el nombre del sistema bajo systemd.
- `hostname -f`: muestra el nombre completo que puede resolver el sistema.
- `hostnamectl --static`: muestra el hostname persistente configurado.

**Resultado observado:** `gperezz.planeta0.com`.

### 5.2 DNS público y resolución local

```bash
sudo dnf install -y bind-utils
dig +short gperezz.planeta0.com A
dig +short gperezz.planeta0.com AAAA
getent hosts gperezz.planeta0.com
ip -4 addr
```

- `bind-utils` es el paquete que proporciona `dig`.
- `dig ... A` consulta registros IPv4; `AAAA` consulta IPv6; `+short` simplifica la salida.
- `getent hosts` consulta la resolución de nombres del **sistema**, que puede combinar `/etc/hosts` y DNS; no es una prueba exclusivamente de DNS público.
- `ip -4 addr` muestra direcciones IPv4 de las interfaces.

**Resultado registrado:** el registro A correspondía a `169.58.144.183`. La resolución IPv6 incluyó `2a02:c207:2367:7994::1` y direcciones `fe80::` (estas últimas son IPv6 de enlace local, no direcciones públicas globales).

### 5.3 Qué indicaba la guía del profesor sobre `/etc/hosts`

La guía base propone, **si falta la entrada**, asociar el FQDN con la IP pública en `/etc/hosts`, conservando `localhost`, y **evitar duplicar el mismo FQDN en loopback y en la IP pública**. Ese criterio es útil para evitar ambigüedad en la resolución local.

```bash
cat /etc/hosts
```

En **este VPS**, sin embargo, `/etc/hosts` es administrado automáticamente por **cloud-init**. Por eso editar directamente el archivo podía ser temporal.

### 5.4 Incidente real: cloud-init restauraba el nombre antiguo

La cabecera de `/etc/hosts` indicaba `manage_etc_hosts: True` y conservaba:

```text
127.0.0.1 vmi3677994.contaboserver.net vmi3677994
::1       vmi3677994.contaboserver.net vmi3677994
```

Investigamos:

```bash
sudo cat /etc/cloud/templates/hosts.redhat.tmpl
sudo grep -RInE 'fqdn:|hostname:|preserve_hostname:|manage_etc_hosts:' /etc/cloud/cloud.cfg /etc/cloud/cloud.cfg.d/
sudo cloud-init query v1.local_hostname
sudo cloud-init query v1.fqdn
sudo cloud-init query merged_cfg.manage_etc_hosts
```

**Hallazgos:** `preserve_hostname: true`; `v1.local_hostname` devolvió `gperezz`; `v1.fqdn` produjo `has no 'fqdn'` (**no es un fallo de DNS**); `merged_cfg.manage_etc_hosts` devolvió `true`.

La plantilla original contenía variables Jinja:

```text
127.0.0.1 {{fqdn}} {{hostname}}
::1 {{fqdn}} {{hostname}}
```

### 5.5 Corrección aplicada y verificada

Se respaldó la plantilla:

```bash
sudo cp /etc/cloud/templates/hosts.redhat.tmpl /etc/cloud/templates/hosts.redhat.tmpl.bak
sudo nano /etc/cloud/templates/hosts.redhat.tmpl
```

Se sustituyeron **solo esas dos líneas** por:

```text
127.0.0.1 gperezz.planeta0.com gperezz
::1 gperezz.planeta0.com gperezz
```

Se regeneró `/etc/hosts` sin reiniciar:

```bash
sudo cloud-init single --name update_etc_hosts --frequency always
cat /etc/hosts
```

**Resultado comprobado:** el archivo quedó con `gperezz.planeta0.com gperezz` en las entradas `127.0.0.1` y `::1`; se conservaron las líneas de `localhost`.

> **Diferencia importante con la guía original:** el profesor recomienda asociar el FQDN a la IP pública y no duplicarlo en loopback; **la solución realmente aplicada en este VPS fue fijarlo en loopback dentro de la plantilla de cloud-init**. Ambas opciones no deben mezclarse sin revisar las necesidades del servidor. El cambio realizado fue comprobado en `/etc/hosts`, pero **no se hizo un segundo reinicio después de esa corrección**, por lo que su persistencia tras ese nuevo reinicio queda pendiente de prueba. Para entornos de producción, evaluar si el FQDN debe resolver localmente a loopback o a la IP pública, especialmente con Apache/Virtualmin y servicios que usan la identidad de red.

**Explicación de utilidades:** `cp` copia archivos; `nano` es un editor de texto en terminal; `grep` busca patrones; `-R` recursivo, `-I` ignora binarios, `-n` muestra número de línea, `-E` usa expresiones regulares extendidas; `cloud-init` configura instancias cloud durante el arranque; `single` ejecuta un módulo concreto; `--frequency always` fuerza esa ejecución.

## 6. Paso 4 — Certbot y Let's Encrypt

```bash
sudo dnf install -y epel-release
sudo dnf install -y certbot
sudo certbot certificates
```

- `epel-release`: habilita el repositorio Extra Packages for Enterprise Linux.
- `certbot`: cliente ACME que solicita y gestiona certificados TLS, normalmente de Let's Encrypt.
- `certbot certificates`: enumera certificados que Certbot conoce y sus fechas de caducidad.

**Caso real:** no había certificado para el FQDN. La guía indica usar el modo `standalone`, que necesita responder temporalmente en el puerto `80`:

```bash
sudo systemctl stop httpd
sudo certbot certonly --standalone -d "$(hostname -f)"
sudo systemctl start httpd
sudo certbot certificates
sudo systemctl is-active httpd
```

- `systemctl stop/start`: detiene/inicia un servicio gestionado por systemd.
- `certonly`: obtiene el certificado **sin instalarlo automáticamente en Apache**.
- `--standalone`: levanta temporalmente un servidor HTTP de validación.
- `-d`: dominio a certificar.
- `$(hostname -f)`: sustitución de comando: ejecuta `hostname -f` y coloca su salida como argumento.

**Precaución:** **inicia Apache aunque falle Certbot**. Detenerlo provoca una interrupción temporal del sitio. El puerto 80 debe ser alcanzable desde Internet durante la validación HTTP-01.

**Resultado real:** certificado emitido para `gperezz.planeta0.com`, caducidad observada **2027-01-19**. Rutas:

```text
/etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem
/etc/letsencrypt/live/gperezz.planeta0.com/privkey.pem
```

- `fullchain.pem`: certificado y cadena de intermediarios.
- `privkey.pem`: **clave privada confidencial; jamás compartir ni publicar**.

**Mejora posterior:** comprobar el mecanismo de renovación automática y que Webmin recargue el certificado renovado. El hecho de que Certbot emita un certificado **no demuestra por sí solo** que todos los servicios lo renueven o recarguen automáticamente.

## 7. Paso 5 — HTTPS en Webmin

En el navegador: `https://gperezz.planeta0.com:10000`.

Ruta de menú: **Webmin → Webmin Configuration → SSL Encryption**.

Configurar:

```text
Private key file:
/etc/letsencrypt/live/gperezz.planeta0.com/privkey.pem

Certificate file (Separate file):
/etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem
```

Guardar y volver a abrir el panel mediante el **FQDN**, no la IP. Los certificados se emiten para nombres concretos: abrir `https://IP:10000` puede producir advertencia de nombre no coincidente.

**Imprevisto real:** Chrome inicialmente mostraba «No es seguro» a pesar de que el certificado era válido y la conexión negociaba TLS 1.3. La inspección apuntaba a recursos/contenido activo con errores de certificado y a una excepción anterior del navegador. Una ventana de incógnito limpia no mostró la advertencia. **Lección:** separar problemas del certificado del servidor de problemas de recursos cargados, excepciones y estado del navegador; no desactivar las advertencias por defecto.

## 8. Paso 6 — Apache y listado de directorios

```bash
systemctl is-active httpd
sudo httpd -S
sudo grep -RIn 'Options.*Indexes' /etc/httpd/conf /etc/httpd/conf.d 2>/dev/null
```

- `httpd`: nombre del servidor Apache en AlmaLinux.
- `httpd -S`: muestra VirtualHosts y su asociación con puertos y archivos.
- `Options Indexes`: puede permitir que Apache muestre un listado de archivos cuando no existe una página índice; en un sitio público puede exponer información.
- `2>/dev/null`: oculta errores enviados a stderr; no altera los resultados normales.

**Caso real:** se encontró en `/etc/httpd/conf/httpd.conf` (línea 149 en ese momento):

```apache
Options Indexes FollowSymLinks
```

Se editó:

```bash
sudo nano /etc/httpd/conf/httpd.conf
```

Y quedó:

```apache
Options FollowSymLinks
```

- `FollowSymLinks` permite seguir enlaces simbólicos cuando Apache tiene permiso; no es equivalente a habilitar el listado.
- **No eliminar `Indexes` indiscriminadamente** en otros bloques: algunas directivas pueden tener otro propósito.

Antes de aplicar:

```bash
sudo httpd -t
sudo systemctl reload httpd
systemctl is-active httpd
```

`httpd -t` valida sintaxis (`Syntax OK`); `reload` recarga configuración sin realizar una parada completa del servicio.

**Resultado:** Apache `active`. La configuración puede cambiar posteriormente si Virtualmin regenera VirtualHosts: revisar siempre las opciones efectivas del sitio.

## 9. Paso 7 — Servidor virtual en Virtualmin

La guía dice **reutilizar el servidor virtual si ya existe**; si no existe, crearlo para el FQDN.

**Situación real:** Virtualmin mostraba el asistente inicial (*Post-Installation Wizard*), que se omitió; inicialmente no había VirtualHosts del dominio.

1. Abrir Virtualmin y ejecutar **Re-check and refresh configuration**.
2. **Imprevisto:** advertencia sobre `clamdscan`/servicio ClamAV detenido. Se verificó con `systemctl status clamd@scan` y se inició con `sudo systemctl start clamd@scan`. La comprobación posterior dejó de mostrar el aviso. **No se comprobó habilitación persistente al arranque**, por lo que no asumirla.
3. Crear **Virtual Server** con dominio `gperezz.planeta0.com` y contraseña administrativa segura.
4. **Imprevisto:** Virtualmin rechazó la creación con `The feature mail is not allowed for virtual server matching the hostname`. Se desmarcó **Mail for domain** y se creó el servidor virtual sin esa función.
5. Virtualmin creó el usuario Unix `gperezz` (UID 1002), que posteriormente se reutilizó como usuario de trabajo.

Verificar:

```bash
sudo httpd -S
systemctl is-active httpd
```

**Salida observada:**

```text
*:80  gperezz.planeta0.com (/etc/httpd/conf/httpd.conf:367)
*:443 gperezz.planeta0.com (/etc/httpd/conf/httpd.conf:409)
active
```

**Interpretación:** Apache tiene VirtualHosts HTTP y HTTPS para el FQDN. Virtualmin puede gestionar su configuración; no crear manualmente un VirtualHost duplicado.

## 10. Paso 8 — Usuario de trabajo y sudo

### Ruta general del profesor (si el usuario aún no existe)

```bash
sudo useradd -m -s /bin/bash alumno
sudo passwd alumno
sudo usermod -aG wheel alumno
id alumno
```

- `useradd`: crea un usuario Unix.
- `-m`: crea su directorio personal.
- `-s /bin/bash`: asigna Bash como shell.
- `passwd alumno`: configura la contraseña del usuario.
- `usermod -aG wheel`: añade el usuario al grupo `wheel` **sin quitarle otros grupos** (`-a` = append, `-G` = grupos suplementarios).
- `id`: muestra UID, GID y pertenencias a grupos.

### Ruta realmente ejecutada (usuario ya creado por Virtualmin)

**No ejecutamos `useradd` ni cambiamos la contraseña**, porque `gperezz` ya existía y podía iniciar sesión con la contraseña definida en Virtualmin.

```bash
id gperezz
sudo usermod -aG wheel gperezz
id gperezz
grep -E '^%wheel[[:space:]]+ALL=' /etc/sudoers
```

Resultados:

```text
uid=1002(gperezz) gid=1002(gperezz) groups=1002(gperezz),10(wheel)
%wheel  ALL=(ALL)       ALL
```

`wheel` es el grupo administrativo; la línea de sudoers confirma su permiso. Si no existiera, el profesor indica revisarla mediante `visudo`, **nunca editar `/etc/sudoers` sin validar su sintaxis**.

### Acceso por clave SSH (alternativa de la guía)

La guía contempla copiar una clave pública autorizada si root tiene una y se usa autenticación por clave:

```bash
sudo ls -l /root/.ssh/authorized_keys
# SOLO si el archivo contiene claves válidas y se desea reutilizarlas:
sudo mkdir -p /home/alumno/.ssh
sudo cp /root/.ssh/authorized_keys /home/alumno/.ssh/authorized_keys
sudo chown -R alumno:alumno /home/alumno/.ssh
sudo chmod 700 /home/alumno/.ssh
sudo chmod 600 /home/alumno/.ssh/authorized_keys
```

- `mkdir -p`: crea directorios, incluidos padres si faltan.
- `chown -R`: cambia propietario y grupo recursivamente.
- `chmod 700`: solo el propietario puede acceder al directorio `.ssh`.
- `chmod 600`: solo el propietario puede leer/escribir `authorized_keys`.

**Caso real:** `/root/.ssh/authorized_keys` existía pero tenía **0 bytes**. Se omitió la copia porque no había claves autorizadas que transferir. Se utilizó SSH con contraseña.

### Prueba de usuario y sudo

Abrir **otra** ventana de PowerShell, sin cerrar la sesión de root:

```powershell
ssh gperezz@gperezz.planeta0.com
```

Luego:

```bash
whoami
sudo whoami
```

**Resultado:** `gperezz` y luego `root`.

**Imprevisto:** la primera vez `sudo` mostró el mensaje introductorio y pidió `[sudo] password for gperezz:`. Eso **no significa cambiar contraseña**, solo autenticar la elevación. Si se cancela o no se escribe, puede aparecer `sudo: a password is required`. Repetimos el comando, se ingresó la contraseña y devolvió `root`.

## 11. Paso 9 — Docker Engine y Compose

**Trabajar desde la sesión del usuario operativo `gperezz`.**

### 11.1 Actualizar y revisar paquetes previos

```bash
sudo dnf update -y
rpm -qa | grep -Ei 'docker|podman|containerd|runc' || true
```

- `rpm -qa`: lista paquetes RPM instalados.
- `grep -Ei`: filtra nombres sin distinguir mayúsculas (`-i`) y con expresiones extendidas (`-E`).
- `|| true`: evita que una búsqueda sin coincidencias termine como error para un script.
- **No desinstalar Podman u otros paquetes automáticamente**; primero evaluar conflictos reales.

### 11.2 Administrar repositorios

```bash
sudo dnf install -y dnf-plugins-core
```

Este paquete proporciona extensiones como `dnf config-manager`. **Resultado real:** ya estaba instalado (`Nothing to do. Complete!`).

**Diferencia entre guía y ejecución:** la guía del profesor indica el repositorio **RHEL**:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

En nuestra sesión utilizamos el repositorio oficial **CentOS** compatible con el entorno AlmaLinux 9:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

**No ejecutar ambos indiscriminadamente:** son rutas alternativas. En este VPS se usó la segunda, y la instalación terminó correctamente. `config-manager --add-repo` registra una fuente de paquetes; **no instala Docker todavía**.

Se ejecutó además:

```bash
sudo dnf update -y
```

Resultado: sin actualizaciones pendientes.

### 11.3 Instalar componentes

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

| Paquete | Función |
|---|---|
| `docker-ce` | Docker Engine, demonio que ejecuta contenedores |
| `docker-ce-cli` | Cliente `docker` para enviar órdenes al Engine |
| `containerd.io` | Runtime que administra contenedores e imágenes a bajo nivel |
| `docker-buildx-plugin` | Construcción avanzada de imágenes, integración con BuildKit |
| `docker-compose-plugin` | Comando `docker compose` para stacks de varios servicios |

Se instalaron también dependencias como `container-selinux` y `iptables-legacy`. El instalador importó la clave GPG del repositorio Docker y finalizó con `Complete!`.

**Versiones observadas:** Engine/CLI 29.9.2; containerd 2.4.2; Buildx 0.38.1; Compose 5.6.1. Estas versiones corresponden al día de ejecución; pueden variar en una instalación posterior.

### 11.4 Iniciar y habilitar Docker

```bash
sudo systemctl enable --now docker
sudo systemctl is-active docker
```

- `enable`: activa el inicio automático del servicio en próximos arranques.
- `--now`: también lo inicia inmediatamente.
- `is-active`: debe devolver `active`.

Verificar clientes:

```bash
sudo docker --version
docker compose version
```

**Resultados:** `Docker version 29.9.2` y `Docker Compose version v5.6.1`.

### 11.5 Prueba funcional

La guía propone inicialmente:

```bash
sudo docker run --rm hello-world
```

En nuestra ejecución real la prueba `hello-world` se hizo **sin sudo**, una vez configurado el grupo `docker` y después del reinicio. El resultado fue `Hello from Docker!`.

- `docker run`: crea y ejecuta un contenedor.
- `--rm`: elimina el contenedor una vez finalizado; **no elimina necesariamente la imagen descargada**.
- `hello-world`: imagen pequeña de prueba descargada desde Docker Hub si no existe localmente.

## 12. Paso 10 — Docker sin sudo

```bash
getent group docker || sudo groupadd docker
sudo usermod -aG docker gperezz
id gperezz
```

- `getent group docker`: comprueba si existe el grupo.
- `|| sudo groupadd docker`: solo lo crea si no existe.
- `usermod -aG docker`: añade al usuario al grupo sin reemplazar otros grupos.

**Resultado real:**

```text
uid=1002(gperezz) gid=1002(gperezz) groups=1002(gperezz),10(wheel),987(docker)
```

La sesión SSH abierta antes del cambio no siempre recoge inmediatamente la nueva pertenencia a grupos. Se usó:

```bash
newgrp docker
docker ps
```

`newgrp docker` abre un entorno de shell con el grupo actualizado. `docker ps` lista contenedores **en ejecución**. Se obtuvo una tabla vacía sin errores: **éxito**; todavía no había contenedores ejecutándose.

> **Seguridad:** el grupo `docker` concede capacidades equivalentes a root. Esta práctica se acepta para el VPS de laboratorio, pero no debe asumirse apropiada para usuarios no confiables en producción.

## 13. Paso 11 — Reinicio y prueba integral

```bash
sudo reboot
```

Desde PowerShell, una vez que el VPS vuelva a responder:

```powershell
ssh gperezz@gperezz.planeta0.com
```

Verificación completa realizada:

```bash
whoami
hostname -f
cat /etc/almalinux-release
systemctl is-active httpd
systemctl is-active docker
id -nG
docker --version
docker compose version
docker ps
docker run --rm hello-world
```

**Resultados registrados:**

```text
gperezz
gperezz.planeta0.com
AlmaLinux release 9.8 (Olive Jaguar)
active
active
gperezz wheel docker
Docker version 29.9.2, build a82e528
Docker Compose version v5.6.1
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**Lectura técnica:** SSH y usuario correctos; FQDN correcto; Apache y Docker sobreviven al reinicio; grupos conservados; cliente Docker y Compose disponibles; demonio capaz de descargar y ejecutar un contenedor.

**Detalle detectado:** el aviso de reinicio identificó al emisor como `root@fqdn.example.com`, mientras que `hostname -f` ya mostraba el nombre nuevo. Esto puede deberse a un componente que conservó un nombre antiguo; no demuestra que el hostname actual sea incorrecto. Se revisó después `/etc/hosts` y se corrigió la plantilla de cloud-init. **No se volvió a reiniciar tras esa última corrección.**

## 14. Imprevistos y soluciones

| Incidente | Causa o explicación | Acción aplicada | Verificación / pendiente |
|---|---|---|---|
| Hostname inicial `fqdn.example.com` | Valor de fábrica o configuración inicial | `hostnamectl set-hostname` | `hostname -f` correcto |
| `/etc/hosts` con nombre antiguo Contabo | Archivo generado por `cloud-init` (`manage_etc_hosts: true`) | Respaldo y edición de plantilla; ejecución de `update_etc_hosts` | Contenido corregido; **reinicio posterior pendiente** |
| `cloud-init query v1.fqdn` no existe | Campo no expuesto en los datos de instancia | No asumir fallo DNS; consultar hostname y configuración efectiva | DNS y hostname verificados por separado |
| Certbot sin certificado previo | No había certificado para FQDN | Detener Apache, `certbot certonly --standalone`, reiniciar Apache | Certificado emitido y Apache `active` |
| Chrome «No es seguro» | Contenido/estado del navegador con errores de certificado | Revisar certificado y abrir sesión limpia de incógnito | Sin advertencia en incógnito; vigilar recursos mixtos |
| Apache permitía `Indexes` | Directiva de listado en configuración principal | Retirar `Indexes`, validar y recargar | `Syntax OK` y servicio `active` |
| Virtualmin advertía por ClamAV | `clamd@scan` detenido | Iniciar servicio | Advertencia desapareció; arranque automático no confirmado |
| Virtualmin rechazó dominio igual al hostname con Mail | Restricción de función de correo para ese dominio | Desmarcar `Mail for domain` | Virtual Server creado |
| Usuario `gperezz` ya existía | Virtualmin lo había creado | Reutilizarlo; agregar a `wheel` | `sudo whoami` → `root` |
| `authorized_keys` vacío | No había claves públicas autorizadas en root | No copiar archivo vacío; usar contraseña | SSH correcto |
| `sudo: a password is required` | Se canceló/omitió la contraseña, no un cambio de clave | Repetir `sudo whoami` e ingresar contraseña actual | `root` |
| `dnf-plugins-core` ya instalado | Paquete presente | No reinstalar innecesariamente | `Complete!` |
| Guía usa repo RHEL; sesión usó CentOS | Diferente URL de repositorio Docker | Se usó solo repo CentOS | Paquetes instalados y Docker operativo |
| `docker ps` vacío | Ningún contenedor ejecutándose | Ninguna corrección necesaria | Sin errores de permisos |

## 15. Diccionario de comandos y paquetes

### 15.1 Comandos de uso cotidiano

| Comando | Qué hace | ¿Instalación adicional? |
|---|---|---|
| `cat archivo` | Imprime el contenido de un archivo | Habitualmente incluido en `coreutils` |
| `ls -l ruta` | Lista archivos con permisos, propietario y tamaño | Habitualmente `coreutils` |
| `pwd` | Muestra directorio actual | Comando del shell o `coreutils` |
| `cd ruta` | Cambia directorio actual | Integrado en Bash |
| `whoami` | Muestra el usuario efectivo | Habitualmente `coreutils` |
| `id` / `id -nG` | Muestra identidad y grupos | Habitualmente `coreutils` |
| `cp origen destino` | Copia un archivo | Habitualmente `coreutils` |
| `mkdir -p ruta` | Crea directorios | Habitualmente `coreutils` |
| `chmod` | Modifica permisos | Habitualmente `coreutils` |
| `chown` | Cambia propietario/grupo | Habitualmente `coreutils` |
| `grep` | Busca texto/patrones | Habitualmente preinstalado |
| `nano` | Editor de terminal | Si falta: `sudo dnf install -y nano` |
| `vi` / `vim` | Editores de terminal | `vi` suele estar disponible; `vim` si hace falta: `sudo dnf install -y vim-enhanced` |
| `dig` | Consulta registros DNS | `sudo dnf install -y bind-utils` |
| `getent` | Consulta bases de datos del sistema (usuarios, grupos, hosts) | Normalmente disponible mediante glibc |
| `ip` | Muestra/configura interfaces y rutas | Normalmente `iproute` |
| `ssh` | Cliente SSH | Normalmente `openssh-clients` |
| `sudo` | Ejecuta un comando con privilegios autorizados | Paquete `sudo` |
| `dnf` | Instala y actualiza RPM | Incluido en AlmaLinux |
| `rpm` | Consulta/gestiona paquetes RPM | Incluido en AlmaLinux |
| `systemctl` | Controla servicios systemd | Incluido en AlmaLinux |
| `journalctl` | Consulta registros de systemd | Incluido en AlmaLinux |
| `hostnamectl` | Administra hostname | Incluido con systemd |
| `uname` | Consulta datos del kernel | Habitualmente `coreutils` |
| `passwd` | Cambia contraseñas de usuario | Normalmente instalado |
| `useradd` / `usermod` | Crea/modifica usuarios | Normalmente `shadow-utils` |
| `newgrp` | Cambia grupo efectivo de la sesión | Normalmente disponible |
| `visudo` | Edita sudoers con comprobación | Paquete `sudo` |
| `httpd` | Servidor Apache y utilidades | Paquete `httpd` |
| `certbot` | Gestiona certificados ACME | `sudo dnf install -y certbot` |
| `cloud-init` | Inicialización de sistemas cloud | Instalado en este VPS |
| `docker` | Cliente de Docker Engine | `docker-ce-cli` |
| `docker compose` | Gestiona stacks de Compose | `docker-compose-plugin` |

**No necesitas instalar `cat`, `cd`, `ls`, `whoami` y demás utilidades básicas una por una** en un AlmaLinux normal. Antes de instalar algo, verifica si existe:

```bash
command -v cat
command -v nano
command -v dig
rpm -q coreutils nano bind-utils
```

- `command -v`: indica si un comando está disponible en la sesión.
- `rpm -q`: consulta si un paquete está instalado; si no lo está, informa `is not installed`.

### 15.2 Sintaxis que conviene dominar

| Sintaxis | Significado | Ejemplo |
|---|---|---|
| `sudo COMANDO` | Elevar privilegios solo para ese comando | `sudo dnf update -y` |
| `-y` | Aceptar preguntas de confirmación | `dnf install -y nano` |
| `-R` | Operación recursiva, según comando | `grep -R`, `chown -R` |
| `-n` | Número de línea en `grep` | `grep -n 'Indexes' archivo` |
| `-E` | Expresiones regulares extendidas | `grep -E 'foo|bar'` |
| `$(COMANDO)` | Sustituir por la salida de un comando | `"$(hostname -f)"` |
| `|` | Enviar salida de un comando al siguiente | `rpm -qa | grep docker` |
| `||` | Ejecutar derecha si izquierda falla | `getent group docker || sudo groupadd docker` |
| `2>/dev/null` | Descartar salida de error | `grep ... 2>/dev/null` |
| `--rm` | Borrar contenedor al terminar | `docker run --rm hello-world` |
| `--now` | Ejecutar acción ahora además de habilitar | `systemctl enable --now docker` |

### 15.3 Lectura de servicios y registros

```bash
systemctl status docker
systemctl status httpd
journalctl -u docker -n 50 --no-pager
journalctl -u httpd -n 50 --no-pager
```

- `status` informa estado y mensajes recientes.
- `journalctl -u SERVICIO` filtra registros por unidad.
- `-n 50` muestra las últimas 50 entradas; `--no-pager` evita abrir un paginador interactivo.

Estos comandos son **diagnóstico adicional**, no requisitos ejecutados y verificados durante el laboratorio.

## 16. Seguridad, mantenimiento y mejoras

Estas son **recomendaciones complementarias**, no pasos que el profesor exigió ni cambios que se hayan aplicado automáticamente:

1. **SSH:** preferir autenticación por clave, guardar claves privadas de forma segura y limitar el acceso de `root` cuando ya se haya probado un usuario alternativo. No deshabilitar el acceso actual antes de verificar el nuevo.
2. **Webmin (10000):** restringir acceso mediante firewall/VPN o IP de administración cuando sea posible; usar TLS válido y contraseñas robustas.
3. **Docker:** recordar que el grupo `docker` equivale en la práctica a privilegios root; no usarlo para cuentas no confiables.
4. **Certificados:** verificar renovación y recarga de Webmin/Apache; Certbot puede renovar el archivo en disco sin que un servicio recargue el certificado por sí solo.
5. **Apache:** comprobar la configuración efectiva de cada VirtualHost y evitar listados de directorios públicos innecesarios.
6. **Virtualmin:** revisar las funciones realmente necesarias, especialmente correo y ClamAV, para no habilitar servicios que no se usarán.
7. **cloud-init:** documentar las personalizaciones de plantilla y comprobar su persistencia después de un nuevo reinicio; evitar editar simultáneamente plantilla y archivo generado con políticas contradictorias.
8. **Copias de seguridad:** guardar configuración antes de cambios, pero **nunca publicar** respaldos que contengan secretos o claves privadas.
9. **Actualizaciones:** ejecutar actualizaciones de seguridad y planificar reinicios cuando cambie el kernel.
10. **Próximo laboratorio:** no desplegar Keycloak hasta contar con las instrucciones del Laboratorio 01 y sus requisitos de puertos, DNS, base de datos y TLS.

### Validaciones opcionales pendientes

```bash
# Estado de servicios después de un futuro reinicio
systemctl is-active httpd docker

# Persistencia del nombre y resolución local
hostnamectl --static
cat /etc/hosts
getent hosts gperezz.planeta0.com

# Certificados conocidos por Certbot
sudo certbot certificates

# Renovación simulada: ejecutar en una ventana de mantenimiento;
# podría requerir liberar temporalmente el puerto 80 según el método configurado.
# sudo certbot renew --dry-run

# Revisar si ClamAV arranca automáticamente
systemctl is-enabled clamd@scan
```

**Advertencia sobre `certbot renew --dry-run`:** si el certificado se emitió mediante `standalone`, la simulación puede fallar porque Apache ocupa el puerto 80. Configurar primero un método de renovación compatible o hooks adecuados; no asumir que la renovación automática está resuelta.

## 17. Checklist final y próximos pasos

### Checklist del profesor (estado observado)

- [x] Accedí por SSH al servidor.
- [x] Actualicé AlmaLinux y comprobé su versión.
- [x] Verifiqué y corregí el FQDN y el contenido de `/etc/hosts`.
- [x] Verifiqué que el FQDN resuelve a la IP pública asignada.
- [x] Instalé Certbot y comprobé el certificado de mi FQDN.
- [x] Accedí a Webmin por HTTPS con el FQDN.
- [x] Confirmé Apache activo y revisé `Options Indexes`.
- [x] Identifiqué y creé el servidor virtual en Virtualmin.
- [x] Dispongo de usuario personal con `sudo`.
- [x] Instalé Docker Engine y Docker Compose.
- [x] Ejecuté `docker ps` y `docker compose version` sin `sudo`.
- [x] Reinicié el VPS y comprobé los servicios y `hello-world`.

**Matiz de verificación:** el reinicio satisfactorio ocurrió **antes** de la corrección final de cloud-init. El nuevo contenido de `/etc/hosts` se verificó inmediatamente, pero su persistencia tras **otro** reinicio sigue pendiente de comprobación. También quedan como mejoras opcionales verificar la renovación automática del certificado y el arranque automático de ClamAV.

### Resultado

El servidor **está preparado para avanzar al Laboratorio 01 de Keycloak**: AlmaLinux 9.8, SSH, Webmin/Virtualmin, Apache, certificado TLS y Docker/Compose funcionan según las comprobaciones realizadas. **Keycloak aún no se ha instalado.**

---

### Créditos y procedencia

- **Guía base del profesor:** *Laboratorio 00 — Preparación del servidor AlmaLinux 9.8*, curso Administración Keycloak, **Patricio Canelo**.
- **Ampliación:** bitácora de comandos, salidas y resolución de incidentes registrados durante la práctica del 21-10-2026.
- **Distinción metodológica:** se señalan explícitamente las diferencias entre la guía base, lo que se ejecutó realmente y las recomendaciones posteriores. Esto evita presentar mejoras propuestas como si ya estuvieran implementadas.

---

## 18. Incidente posterior al laboratorio: Webmin entrega un certificado autofirmado en lugar de Let's Encrypt (resuelto)

> **Caso real — 21 de octubre de 2026.** Esta sección se añadió al final del README para documentar el diagnóstico y la solución **paso a paso**. No forma parte de los pasos originales del profesor. El objetivo fue recuperar el HTTPS confiable de `https://gperezz.planeta0.com:10000` sin eliminar los certificados de Virtualmin ni alterar Apache o Docker.

### 18.1. Síntomas

- En Webmin apareció **«Connection lost — Cannot establish a connection to the server at gperezz.planeta0.com:10000»**.
- El navegador marcaba la conexión como **«No es seguro»**, incluso en modo incógnito.
- El certificado mostrado tenía **CN e issuer `*.gperezz.planeta0.com`** y vencimiento **20-10-2031**: era un certificado **autofirmado**, distinto del certificado válido de Let's Encrypt.
- Un reinicio inicial de Webmin **no** corrigió el problema.

**Importante:** la evidencia confirmó el problema de certificado, pero no demostró que este fuese la única causa del mensaje «Connection lost». La recuperación de la navegación web debe verificarse por separado.

### 18.2. Comprobar servicio, puerto y certificado de Certbot

Desde la sesión SSH del usuario `gperezz`:

```bash
sudo systemctl is-active webmin
sudo ss -lntp | grep ':10000'
sudo certbot certificates
```

**Para qué sirve cada comando:**

- `systemctl is-active webmin`: verifica que el servicio Webmin esté ejecutándose; `active` significa que está activo.
- `ss -lntp`: lista puertos TCP en escucha y los procesos asociados; `grep ':10000'` filtra el puerto HTTPS de Webmin.
- `certbot certificates`: enumera los certificados administrados por Certbot, su vigencia y las rutas de sus archivos.

**Resultado observado:** Webmin estaba `active`, `miniserv.pl` escuchaba en `0.0.0.0:10000` y `[::]:10000`, y Certbot conservaba el certificado válido para `gperezz.planeta0.com`, con vencimiento el **19-01-2027**.

### 18.3. Comprobar el certificado que Webmin entrega realmente

```bash
echo | openssl s_client -connect 127.0.0.1:10000 -servername gperezz.planeta0.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

**Explicación:** `openssl s_client` realiza una conexión TLS al puerto local de Webmin; `-servername` envía el nombre de dominio mediante **SNI (Server Name Indication)**. La salida se canaliza (`|`) a `openssl x509`, que muestra sujeto (`-subject`), emisor (`-issuer`) y fechas (`-dates`). `2>/dev/null` oculta mensajes diagnósticos de stderr, sin modificar archivos.

**Resultado defectuoso observado:**

```text
subject=CN=*.gperezz.planeta0.com
issuer=CN=*.gperezz.planeta0.com
notAfter=Oct 20 13:50:36 2031 GMT
```

Que sujeto y emisor coincidan era consistente con el certificado autofirmado visto en el navegador. **Reiniciar Webmin en ese momento no cambió el resultado**, así que no era simplemente un certificado antiguo retenido en memoria.

### 18.4. Revisar el archivo de configuración completo, no solo sus campos principales

Primero se consultaron los parámetros básicos:

```bash
sudo grep -E '^(keyfile|certfile|extracas|ssl|port)=' /etc/webmin/miniserv.conf
```

Estos apuntaban correctamente a Let's Encrypt:

```ini
port=10000
ssl=1
keyfile=/etc/letsencrypt/live/gperezz.planeta0.com/privkey.pem
certfile=/etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem
```

Sin embargo, **eso no era suficiente**: existían directivas adicionales que la búsqueda limitada no mostraba. Para descubrirlas se ejecutó:

```bash
sudo grep -nEi 'cert|key|pem|sni|virtual|ssl' /etc/webmin/miniserv.conf
```

- `grep` busca coincidencias en el archivo.
- `-n` añade el número de línea; `-E` habilita expresiones regulares extendidas; `-i` ignora diferencias de mayúsculas y minúsculas.
- Este comando **solo lee** la configuración.

**Hallazgo decisivo:** Virtualmin había registrado tres directivas adicionales para el mismo dominio:

```ini
ipextracas_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.combined
ipcert_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.cert
ipkey_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.key
```

Estas entradas asociaban un certificado específico al nombre solicitado mediante SNI. Por eso Webmin entregaba el certificado autofirmado de Virtualmin **en lugar del certificado principal de Let's Encrypt**.

### 18.5. Descartar que el certificado presentado fuese el `miniserv.pem` antiguo

Se comprobaron los archivos de certificado **sin mostrar claves privadas**:

```bash
sudo ls -la /etc/webmin/miniserv*
sudo openssl x509 -in /etc/webmin/miniserv.pem -noout -subject -issuer -dates -fingerprint -sha256
sudo openssl x509 -in /etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem -noout -subject -issuer -dates -fingerprint -sha256
```

- `ls -la` muestra archivos, permisos y fechas.
- `openssl x509 -in ...` lee un certificado X.509.
- `-fingerprint -sha256` calcula una huella para distinguir certificados sin revelar secretos.

Se observó que `/etc/webmin/miniserv.pem` contenía otro certificado autofirmado, **`fqdn.example.com`**, mientras que el de Let's Encrypt tenía **CN `gperezz.planeta0.com`** y emisor **Let's Encrypt YE1**. Por lo tanto, **no se eliminó `miniserv.pem`**: no era la causa demostrada.

### 18.6. Solución aplicada: respaldar y comentar únicamente las tres entradas SNI

**Paso A — Crear copia de seguridad:**

```bash
sudo cp -a /etc/webmin/miniserv.conf /etc/webmin/miniserv.conf.bak
```

`cp -a` copia el archivo preservando sus atributos en la medida de lo posible. Este respaldo permite revertir el cambio si fuese necesario.

**Paso B — Editar la configuración:**

```bash
sudo nano /etc/webmin/miniserv.conf
```

`nano` es un editor de texto de terminal. Si no estuviera instalado, puede instalarse con `sudo dnf install -y nano`. Dentro del editor, buscar las **tres** directivas `ipextracas_`, `ipcert_` e `ipkey_` del dominio y agregar `#` al comienzo de cada línea:

```ini
#ipextracas_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.combined
#ipcert_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.cert
#ipkey_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.key
```

**Conservar sin cambios** las líneas principales:

```ini
keyfile=/etc/letsencrypt/live/gperezz.planeta0.com/privkey.pem
certfile=/etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem
```

Guardar con **Ctrl+O**, **Enter** y salir con **Ctrl+X**. No borrar los certificados de `/etc/ssl/virtualmin/`: otros servicios podrían necesitarlos.

> **Error que ocurrió:** al pegar las líneas `ipcert_...=...` directamente en Bash, apareció `-bash: ... No such file or directory`. **No eran comandos de terminal**, sino líneas de configuración que debían editarse *dentro* de `miniserv.conf`. Ese intento no solucionó ni modificó la configuración.

**Paso C — Verificar el archivo antes de reiniciar:**

```bash
sudo grep -nE '^(#?ip(cert|key|extracas)_|keyfile=|certfile=)' /etc/webmin/miniserv.conf
```

**Resultado real obtenido:**

```text
31:keyfile=/etc/letsencrypt/live/gperezz.planeta0.com/privkey.pem
58:certfile=/etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem
61:#ipextracas_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.combined
62:#ipcert_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.cert
63:#ipkey_gperezz.planeta0.com,*.gperezz.planeta0.com=/etc/ssl/virtualmin/179259062417422/ssl.key
```

### 18.7. Reiniciar únicamente Webmin y validar la solución

```bash
sudo systemctl restart webmin
sudo systemctl is-active webmin
echo | openssl s_client -connect 127.0.0.1:10000 -servername gperezz.planeta0.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

- `restart webmin` hace que Webmin relea su configuración y puede interrumpir brevemente la sesión web.
- `is-active webmin` debe responder `active`.
- La prueba de OpenSSL confirma el certificado **efectivamente entregado**, no solo el que figura en el archivo.

**Resultado final real:**

```text
active
subject=CN=gperezz.planeta0.com
issuer=C=US, O=Let's Encrypt, CN=YE1
notBefore=Oct 21 12:25:46 2026 GMT
notAfter=Jan 19 12:25:45 2027 GMT
```

**Conclusión verificada:** el conflicto SNI quedó resuelto y Webmin pasó a entregar el certificado válido de Let's Encrypt. **Quedó pendiente de confirmación en esta bitácora** una nueva prueba de navegación en `https://gperezz.planeta0.com:10000` sin advertencia y sin el mensaje «Connection lost».

### 18.8. Qué hacer si vuelve a ocurrir

1. Revisar primero el certificado **realmente servido** con `openssl s_client`, incluyendo `-servername`.
2. Inspeccionar todas las directivas `ipcert_`, `ipkey_` e `ipextracas_` de `/etc/webmin/miniserv.conf`, además de `keyfile` y `certfile`.
3. Confirmar si Virtualmin volvió a crear asociaciones SNI tras cambiar el certificado o la configuración del dominio.
4. Hacer respaldo antes de editar; no borrar certificados ajenos ni desactivar TLS.
5. Tras corregir la configuración, reiniciar **solo** Webmin y repetir la prueba TLS.
6. Para un entorno más duradero, evaluar gestionar el certificado del dominio desde Virtualmin y sincronizarlo correctamente con Webmin, en vez de depender permanentemente de comentar entradas que Virtualmin podría regenerar.
7. Recordar que **la renovación automática de Let's Encrypt y la recarga posterior de Webmin** requieren una comprobación independiente; este incidente no demuestra que estén automatizadas.

**Lección principal:** un servicio puede tener `certfile` y `keyfile` correctos y aun así entregar otro certificado por una regla SNI específica. Para diagnosticar HTTPS hay que verificar **qué certificado sirve realmente el puerto y el nombre de dominio**, no limitarse a leer la configuración principal.
