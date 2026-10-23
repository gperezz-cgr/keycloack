# Laboratorio 01 — Keycloak 26.7.4 con Docker y Apache Reverse Proxy HTTPS

> **Estado:** completado funcionalmente el 21 de octubre de 2026.  
> **Entorno:** VPS AlmaLinux 9.8, Docker, Apache HTTP Server, Virtualmin/Webmin.  
> **Dominio del laboratorio:** `gperezz.planeta0.com`.  
> **Alcance:** Laboratorio 01a (arranque de Keycloak) + Laboratorio 01b (publicación HTTPS por Apache).

## 1. Objetivo y resultado final

Desplegar **Keycloak 26.7.4** en un contenedor Docker, accesible **solo desde el VPS** en `127.0.0.1:8084`, y publicar su consola mediante el **Reverse Proxy HTTPS de Apache** en:

- URL base: `https://gperezz.planeta0.com/auth0/`
- Consola administrativa: `https://gperezz.planeta0.com/auth0/admin/`
- Consola del realm `master`: `https://gperezz.planeta0.com/auth0/admin/master/console/`

**Resultado observado:** acceso exitoso a la consola `master` desde el navegador. La respuesta pública a `curl -I` fue `HTTP/1.1 302 Found` con redirección a `/auth0/admin/`, sin error de certificado.

### Flujo de red

```text
Navegador
  |
  | HTTPS :443 (certificado Let's Encrypt)
  v
Apache / VirtualHost gperezz.planeta0.com:443
  |
  | ProxyPass /auth0/
  v
127.0.0.1:8084 (host AlmaLinux)
  |
  | Docker: 8084 -> 8080
  v
Contenedor keycloak-dev — Keycloak 26.7.4
  └── /auth0/ -> /auth0/admin/
```

El puerto 8084 **no se publica en la interfaz pública**: `127.0.0.1:8084:8080` lo limita al loopback del servidor.

## 2. Prerrequisitos (Laboratorio 00)

- VPS AlmaLinux 9.8 con acceso SSH (`ssh gperezz@gperezz.planeta0.com`).
- Cuenta `gperezz` con `sudo` y acceso a Docker sin `sudo`.
- Docker Engine y Compose instalados; demonio Docker operativo.
- Apache (`httpd`) instalado y funcionando.
- Virtualmin/Webmin disponible en `https://gperezz.planeta0.com:10000/`.
- VirtualHost SSL de `gperezz.planeta0.com` en el puerto **443**.
- Certificado Let's Encrypt emitido para `gperezz.planeta0.com`.
- DNS del dominio apuntando al VPS.

> **Dónde ejecutar comandos:** salvo que se indique lo contrario, los comandos de terminal se ejecutan **dentro del VPS por SSH**, no en PowerShell de Windows. `127.0.0.1` siempre hace referencia a la máquina donde se ejecuta el comando.

## 3. Laboratorio 01a — Descargar y ejecutar Keycloak

### 3.1 Revisar el estado de Docker

```bash
docker ps -a
docker images
```

**Situación real inicial:** `docker ps -a` no mostró contenedores; `docker images` solo contenía `hello-world:latest` del Laboratorio 00. Por tanto, no existía todavía un servicio escuchando en 8084.

### 3.2 Descargar la imagen indicada en la guía

```bash
docker pull quay.io/keycloak/keycloak:26.7.4
```

**Resultado real:** descarga correcta, con mensaje `Downloaded newer image for quay.io/keycloak/keycloak:26.7.4`.

**Conceptos:** `pull` descarga una imagen; `quay.io` es el registro; `26.7.4` fija la versión. Descargar la imagen **no** inicia el servicio.

### 3.3 Crear el contenedor

Este fue el comando utilizado en el laboratorio, adaptando el hostname del profesor `key00.planeta0.com` al dominio propio `gperezz.planeta0.com`:

```bash
docker run -d \
  --name keycloak-dev \
  -p 127.0.0.1:8084:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:26.7.4 \
  start-dev \
  --http-relative-path=/auth0 \
  --hostname=https://gperezz.planeta0.com/auth0 \
  --proxy-headers=xforwarded
```

**Advertencia:** `admin/admin` se usó efectivamente durante la práctica, pero es una credencial insegura. **No se debe reutilizar** para un despliegue nuevo. Para un nuevo laboratorio, sustituir la contraseña inicial por una robusta y evitar introducir secretos en comandos que quedan en el historial o en la configuración del contenedor. En este laboratorio la contraseña se cambió posteriormente con `kcadm.sh`.

| Opción | Propósito |
|---|---|
| `docker run -d` | Crear e iniciar en segundo plano. |
| `--name keycloak-dev` | Nombre identificable del contenedor. |
| `-p 127.0.0.1:8084:8080` | Host local 8084 → contenedor 8080; no expone 8084 al exterior. |
| `KC_BOOTSTRAP_ADMIN_USERNAME` | Administrador inicial de arranque. |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | Contraseña inicial de arranque. |
| `start-dev` | Arranque en modo desarrollo, **no apto para producción**. |
| `--http-relative-path=/auth0` | Ruta base de Keycloak. |
| `--hostname=https://gperezz.planeta0.com/auth0` | URL pública canónica para redirecciones y enlaces. |
| `--proxy-headers=xforwarded` | Confía en encabezados `X-Forwarded-*` del proxy configurado. |

### 3.4 Verificar contenedor y registros

```bash
docker ps -a
docker logs --tail 80 keycloak-dev
```

**Resultado observado:** contenedor `keycloak-dev` con estado `Up` y puertos `127.0.0.1:8084->8080/tcp` (también aparecían puertos internos `8443/tcp` y `9000/tcp`, sin publicación al host).

El primer inicio demoró cerca de un minuto. Los registros mostraron sucesivamente:

```text
Updating the configuration and installing your custom providers, if any. Please wait.
Quarkus augmentation completed ...
Initializing database schema ...
Initializing master realm
Created temporary admin user with username admin
Bootstrap completed ...
Keycloak 26.7.4 ... started in 53.198s. Listening on: http://0.0.0.0:8080
```

La instalación de laboratorio utilizó la base integrada **H2** (`jdbc-h2` en los logs). **No se configuró PostgreSQL ni almacenamiento persistente externo.** No se debe considerar este despliegue como producción ni asumir que los datos sobrevivirán a eliminar/recrear el contenedor.

### 3.5 Comprobar el backend local

```bash
curl -I http://127.0.0.1:8084/auth0/
```

**Resultado final real:**

```http
HTTP/1.1 302 Found
Location: https://gperezz.planeta0.com/auth0/admin/
```

El `302` es **esperado**: Keycloak redirige la ruta base a su consola de administración. No es un error.

## 4. Laboratorio 01b — Apache Reverse Proxy HTTPS

### 4.1 Abrir el VirtualHost correcto en Virtualmin

1. Acceder a `https://gperezz.planeta0.com:10000/`.
2. Seleccionar el servidor virtual `gperezz.planeta0.com`.
3. Ir a **Web Configuration → Configure Apache SSL Website → Edit Directives**.
4. Comprobar el encabezado **`For gperezz.planeta0.com:443`**.
5. Confirmar que se está editando el bloque HTTPS, **no** el VirtualHost HTTP `:80`.

En la instalación observada, Virtualmin mostraba directivas dentro de `/etc/httpd/conf/httpd.conf`, incluida:

```apache
ProxyPass /.well-known !
```

### 4.2 Añadir las directivas de Reverse Proxy

**Inmediatamente debajo** de `ProxyPass /.well-known !`, dentro del VirtualHost HTTPS `:443`, se agregó:

```apache
# Keycloak 26.7.4 - Apache Reverse Proxy
# URL publica: https://gperezz.planeta0.com/auth0/
# Backend local: http://127.0.0.1:8084/auth0/

ProxyPreserveHost On
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"

ProxyPass        /auth0/ http://127.0.0.1:8084/auth0/
ProxyPassReverse /auth0/ http://127.0.0.1:8084/auth0/

RedirectMatch 302 ^/auth0$ /auth0/
```

| Directiva | Función |
|---|---|
| `ProxyPreserveHost On` | Preserva el encabezado `Host` del cliente. |
| `RequestHeader set X-Forwarded-Proto "https"` | Informa al backend que la conexión pública usa HTTPS. |
| `RequestHeader set X-Forwarded-Port "443"` | Informa el puerto público. |
| `ProxyPass` | Envía `/auth0/` a Keycloak en 8084. |
| `ProxyPassReverse` | Ajusta encabezados de redirección del backend. |
| `RedirectMatch` | Convierte `/auth0` en `/auth0/`. |

**No borrar** las reglas existentes de Virtualmin, SSL, PHP, Webmail, `.well-known`, ni modificar el VirtualHost de `:80`.

### 4.3 Guardar, validar y recargar Apache

En Virtualmin, pulsar **Save**. En SSH:

```bash
sudo apachectl configtest
# Resultado observado: Syntax OK

sudo systemctl reload httpd
systemctl is-active httpd
# Resultado observado: active
```

`configtest` verifica la sintaxis; `reload` aplica los cambios sin reiniciar por completo Apache. **Si no aparece `Syntax OK`, no recargar:** corregir primero el error.

### 4.4 Probar HTTPS público

```bash
curl -I https://gperezz.planeta0.com/auth0/
```

**Primer intento real:**

```text
curl: (60) SSL certificate problem: self-signed certificate
```

El error correspondía al certificado presentado por **Apache :443**, no al funcionamiento interno de Keycloak. El certificado de Webmin :10000 había sido corregido en el Laboratorio 00, pero ambos servicios administran su configuración TLS por separado.

## 5. Incidencia real — Apache entregaba un certificado autofirmado

### 5.1 Verificar el certificado Let's Encrypt disponible

```bash
sudo openssl x509 \
  -in /etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem \
  -noout -subject -issuer -dates
```

**Resultado observado:**

```text
subject=CN=gperezz.planeta0.com
issuer=C=US, O=Let's Encrypt, CN=YE1
notBefore=Oct 21 12:25:46 2026 GMT
notAfter=Jan 19 12:25:45 2027 GMT
```

### 5.2 Identificar las rutas SSL usadas por Apache

En **Configure Apache SSL Website → Edit Directives**, el VirtualHost utilizaba inicialmente:

```apache
SSLCertificateFile /etc/ssl/virtualmin/179259062417422/ssl.combined
SSLCertificateKeyFile /etc/ssl/virtualmin/179259062417422/ssl.key
```

Esas rutas apuntaban al certificado que ocasionaba la advertencia de confianza.

### 5.3 Corregir el VirtualHost HTTPS

Se reemplazaron **solo** esas dos directivas por:

```apache
SSLCertificateFile /etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/gperezz.planeta0.com/privkey.pem
```

Se conservaron `SSLEngine on`, el resto de la configuración SSL y las reglas del Reverse Proxy.

Después de guardar:

```bash
sudo apachectl configtest
# Syntax OK
sudo systemctl reload httpd
```

### 5.4 Resultado de la corrección

```bash
curl -I https://gperezz.planeta0.com/auth0/
```

**Resultado real final:**

```http
HTTP/1.1 302 Found
Server: Apache
Location: https://gperezz.planeta0.com/auth0/admin/
```

`curl` completó la validación TLS sin usar `-k`. Esto prueba que el cliente confió en el certificado presentado durante la prueba y que la solicitud llegó hasta Keycloak mediante Apache.

> **Mantenimiento pendiente, no exigido por el laboratorio:** comprobar que la renovación automática de Let's Encrypt y la gestión de certificados de Virtualmin mantengan actualizadas las rutas y recarguen Apache cuando corresponda. La validez comprobada del certificado no garantiza por sí sola que la renovación futura esté integrada.

## 6. Administración de Keycloak mediante CLI (`kcadm.sh`)

Estas acciones se realizaron como **mejora adicional de seguridad**, no como requisito indispensable del Reverse Proxy.

### 6.1 Verificar disponibilidad de la CLI

```bash
docker exec keycloak-dev /opt/keycloak/bin/kcadm.sh --help
```

Se obtuvo la ayuda de **Keycloak Admin CLI**, con comandos `config`, `get`, `create`, `update`, `delete`, `set-password`, etc.

### 6.2 Iniciar sesión administrativa

```bash
docker exec -it keycloak-dev \
  /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://127.0.0.1:8080/auth0 \
  --realm master \
  --user admin
```

Al aparecer `Enter password:`, introducir la contraseña **sin copiarla a la conversación**. El terminal no muestra los caracteres mientras se escribe.

### 6.3 Comprobar permisos sobre el realm `master`

```bash
docker exec keycloak-dev \
  /opt/keycloak/bin/kcadm.sh get realms/master
```

**Resultado observado:** JSON con `"realm": "master"` y `"enabled": true`, entre otras opciones. Esto confirmó que la sesión CLI tenía acceso al realm.

### 6.4 Error real con una opción de contraseña no soportada

Se intentó:

```bash
# NO USAR: opción no soportada en esta versión
# kcadm.sh set-password --realm master --username admin --password:env KC_NEW_PASSWORD
```

**Error observado:**

```text
Unknown options: '--password:env', 'KC_NEW_PASSWORD'
Possible solutions: --password
```

**Causa:** `--password:env` no es una opción válida para `set-password` en la CLI instalada. El comando fue rechazado; no cambió la contraseña.

Se consultó la ayuda específica:

```bash
docker exec keycloak-dev \
  /opt/keycloak/bin/kcadm.sh help set-password
```

La ayuda indicó que, si se omite `--new-password` y no hay una variable `KC_CLI_PASSWORD`, la CLI **solicita la nueva contraseña interactivamente**.

### 6.5 Cambiar la contraseña sin escribirla en el historial

```bash
docker exec -it keycloak-dev \
  /opt/keycloak/bin/kcadm.sh set-password \
  --username admin
```

**Resultado observado:** apareció `Enter password:`; se introdujo la nueva contraseña y el comando volvió al prompt sin error.

Esto constituye una señal de ejecución correcta, aunque **no se registró una prueba posterior explícita de autenticación con la contraseña nueva mediante CLI**. Sí se confirmó después el ingreso exitoso a la consola web de Keycloak; la captura no permite determinar qué contraseña se utilizó.

**Recomendación de comprobación adicional:** cerrar sesión y volver a entrar con la nueva contraseña, sin compartirla.

### 6.6 Advertencia de administrador temporal

En la consola se observó una franja amarilla:

```text
You are logged in as a temporary admin user. To harden security,
create a permanent admin account and delete the temporary one.
```

**Interpretación:** el usuario `admin` creado por el mecanismo de bootstrap sigue siendo **temporal**, incluso después de cambiar su contraseña. La práctica quedó funcionalmente terminada, pero para un entorno seguro conviene crear un administrador permanente, comprobar sus permisos e ingreso y **solo después** eliminar el temporal.

## 7. Verificación final desde el navegador

Abrir:

```text
https://gperezz.planeta0.com/auth0/admin/
```

Ingresar con las credenciales administrativas configuradas. En el caso real se observó la consola en:

```text
https://gperezz.planeta0.com/auth0/admin/master/console/
```

La interfaz mostró **`master realm`**, menús como **Clients, Client scopes, Realm roles, Users, Groups, Sessions, Events, Realm settings, Authentication, Identity providers** y la advertencia de administrador temporal.

**Conclusión:** el acceso público por HTTPS, el Reverse Proxy y la consola administrativa funcionan. El **Laboratorio 01b quedó completado funcionalmente**.

## 8. Errores encontrados y soluciones aplicadas

| Síntoma | Causa comprobada o probable | Solución / verificación |
|---|---|---|
| `curl: (7) Failed to connect to 127.0.0.1 port 8084` desde PowerShell | Se ejecutó contra el `localhost` de Windows, no el VPS. | Conectarse por SSH al VPS y repetir allí. |
| `curl: (7) Connection refused` desde AlmaLinux | No había contenedores Docker creados ni Keycloak escuchando. | `docker ps -a`; descargar imagen y ejecutar contenedor. |
| `curl: (56) Recv failure: Connection reset by peer` | Keycloak aún estaba inicializándose; los logs mostraban creación del esquema de base de datos. | Esperar arranque completo y comprobar `docker logs --tail 80 keycloak-dev`. |
| Logs terminaban en `Initializing database schema` | Inicialización inicial aún en curso. | Esperar hasta `Keycloak ... started ... Listening on ... 8080`. |
| `curl: (60) SSL certificate problem: self-signed certificate` | Apache `:443` presentaba el certificado de Virtualmin en lugar del Let's Encrypt válido. | Actualizar `SSLCertificateFile` y `SSLCertificateKeyFile`, validar y recargar Apache. |
| Navegador: `NET::ERR_CERT_AUTHORITY_INVALID` / HSTS | Certificado HTTPS no confiable en Apache. | Corregir certificado; **no** desactivar HSTS ni ignorar errores TLS. |
| `Unknown options: '--password:env'` | Sintaxis no soportada por `kcadm.sh set-password`. | Consultar `help set-password`; usar modo interactivo sin `--new-password`. |
| Advertencia `temporary admin user` | Administrador bootstrap temporal. | Crear administrador permanente y eliminar temporal tras comprobar acceso; pendiente como endurecimiento. |

## 9. Guía de diagnóstico para problemas futuros

### 9.1 Error HTTP 503 (Service Unavailable)

Apache puede estar funcionando, pero sin alcanzar Keycloak:

```bash
docker ps --filter name=keycloak-dev
docker logs --tail 80 keycloak-dev
curl -I http://127.0.0.1:8084/auth0/
```

Revisar el registro de errores del VirtualHost (ruta observada en la configuración):

```bash
sudo tail -n 80 /var/log/virtualmin/gperezz.planeta0.com_error_log
```

En AlmaLinux con SELinux, comprobar:

```bash
getenforce
getsebool httpd_can_network_connect
```

**Solo si el diagnóstico confirma un bloqueo SELinux de conexiones salientes de Apache**, la guía propone:

```bash
sudo setsebool -P httpd_can_network_connect on
```

`-P` hace persistente el ajuste: no cambiar políticas SELinux sin necesidad demostrada.

### 9.2 Error HTTP 404

- Confirmar que se editó **`gperezz.planeta0.com:443`**, no `:80` ni otro dominio.
- Confirmar `ProxyPass /auth0/ http://127.0.0.1:8084/auth0/`.
- Confirmar `--http-relative-path=/auth0` en Keycloak.
- Verificar `sudo apachectl configtest` y recarga de Apache.

### 9.3 Redirección incorrecta o bucle

Revisar la coherencia de:

```text
--http-relative-path=/auth0
--hostname=https://gperezz.planeta0.com/auth0
--proxy-headers=xforwarded
```

Y en Apache:

```apache
ProxyPreserveHost On
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"
```

### 9.4 Apache no recarga

```bash
sudo apachectl configtest
sudo systemctl status httpd --no-pager
sudo journalctl -u httpd -n 80 --no-pager
```

No recargar mientras existan errores de sintaxis.

### 9.5 Certificado vuelve a fallar

Comprobar certificado en disco:

```bash
sudo openssl x509 \
  -in /etc/letsencrypt/live/gperezz.planeta0.com/fullchain.pem \
  -noout -subject -issuer -dates
```

Comprobar **el certificado que realmente entrega Apache**:

```bash
openssl s_client -connect gperezz.planeta0.com:443 \
  -servername gperezz.planeta0.com </dev/null
```

Revisar que el VirtualHost tenga las rutas Let's Encrypt correctas, y validar que la renovación de certificados esté integrada con Apache/Virtualmin.

## 10. Comandos rápidos de operación

```bash
# Estado de Keycloak
docker ps --filter name=keycloak-dev

# Logs recientes
docker logs --tail 80 keycloak-dev

# Backend local
curl -I http://127.0.0.1:8084/auth0/

# Endpoint HTTPS público (validando certificado)
curl -I https://gperezz.planeta0.com/auth0/

# Validar Apache
sudo apachectl configtest

# Estado de Apache
systemctl is-active httpd

# Recargar Apache tras cambios válidos
sudo systemctl reload httpd
```

## 11. Lista de verificación de entrega

- [x] Imagen `quay.io/keycloak/keycloak:26.7.4` descargada.
- [x] Contenedor `keycloak-dev` iniciado.
- [x] Publicación local `127.0.0.1:8084->8080` comprobada.
- [x] Logs confirman arranque de Keycloak.
- [x] `curl -I http://127.0.0.1:8084/auth0/` devuelve `302`.
- [x] VirtualHost HTTPS `gperezz.planeta0.com:443` seleccionado.
- [x] Directivas de Reverse Proxy añadidas.
- [x] `apachectl configtest`: `Syntax OK`.
- [x] Apache recargado y activo.
- [x] Certificado de Apache corregido a Let's Encrypt.
- [x] `curl -I https://gperezz.planeta0.com/auth0/` devuelve `302` sin error TLS.
- [x] Consola administrativa accesible desde navegador.
- [x] CLI administrativa `kcadm.sh` funcional.
- [x] Se ejecutó cambio interactivo de contraseña del administrador inicial sin error.
- [ ] Confirmar explícitamente un nuevo inicio de sesión con la contraseña actualizada.
- [ ] Crear administrador permanente y retirar el administrador temporal (endurecimiento adicional).
- [ ] Comprobar renovación automática del certificado de Apache (mantenimiento adicional).
- [ ] Migrar a configuración de producción con base persistente y estrategia de respaldos (fuera de alcance).

## 12. Consideraciones importantes para no confundir laboratorio y producción

1. **`start-dev` no es modo producción.** Deshabilita o simplifica controles apropiados para un entorno de pruebas.
2. **H2 integrada no es la base recomendada para producción.** Se requiere almacenamiento persistente, respaldo y, habitualmente, PostgreSQL.
3. **No hay volumen persistente configurado en el comando observado.** Eliminar el contenedor puede provocar pérdida de datos del laboratorio.
4. **El administrador bootstrap es temporal.** Cambiar su contraseña no lo convierte en administrador permanente.
5. **La URL administrativa es pública por HTTPS.** TLS es necesario, pero no sustituye contraseñas robustas, MFA, restricciones de acceso ni endurecimiento del servicio.
6. **El secreto inicial se introdujo en `docker run`.** Puede persistir en historial y metadatos del contenedor. Para uso serio, planificar una gestión segura de secretos y rotación.
7. **La configuración de Virtualmin y Apache puede ser regenerada.** Documentar las modificaciones y verificar compatibilidad con la renovación SSL.
8. **No cambiar el mapeo de puertos sin actualizar Apache.** El proxy apunta explícitamente a `127.0.0.1:8084`.

---

## 13. Cierre

El **Laboratorio 01** logró su objetivo: **Keycloak 26.7.4 desplegado en Docker, publicado mediante Apache Reverse Proxy HTTPS con un certificado Let's Encrypt válido y acceso confirmado a la consola de administración del realm `master`**.

La secuencia real de resolución fue:

```text
Sin contenedor -> Descargar imagen -> Iniciar Keycloak
-> Esperar inicialización H2/Quarkus -> Verificar HTTP local 302
-> Configurar VirtualHost Apache :443 -> Syntax OK -> Reload
-> Detectar certificado autofirmado -> Aplicar Let's Encrypt
-> Verificar HTTPS 302 -> Autenticación CLI -> Cambio de contraseña
-> Acceso exitoso a consola web
```

**Nota de alcance:** esta documentación registra lo realizado y las incidencias observadas en la conversación. Las recomendaciones de endurecimiento y mantenimiento están identificadas como adicionales y no se presentan como pasos ya ejecutados.
