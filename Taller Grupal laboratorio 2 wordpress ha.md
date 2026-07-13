# Laboratorio: WordPress con balanceo de carga, Redis y tolerancia a fallos

**Curso:** Sistemas Operativos Avanzado  
**Plataforma:** Ubuntu 24.04 en WSL, Docker Engine y Docker Compose  
**Fecha de revisión:** 6 de julio de 2026 — versión con UID/GID dinámico  
**Duración estimada:** 2 a 3 horas  

---

## 1. Propósito del laboratorio

En este laboratorio se desplegará un sitio WordPress compuesto por varios contenedores que cooperan dentro de una red Docker:

- Un contenedor de **MariaDB** para almacenar los datos persistentes de WordPress.
- Un contenedor de **Redis** para la caché persistente de objetos y, opcionalmente, las sesiones PHP utilizadas por algunos plugins.
- Dos contenedores idénticos de **Apache y PHP**, denominados `web1` y `web2`.
- Un contenedor de **HAProxy** que distribuye las solicitudes HTTP entre ambos servidores web.
- Un directorio compartido del host, `/home/orh`, que contiene los archivos de WordPress.

El usuario Linux `orh`, con UID y GID `1003:1003`, será el propietario de todos los archivos del sitio. La imagen web permanecerá genérica: el UID y el GID se suministrarán al iniciar los contenedores mediante variables de entorno. El usuario `cmendez` administrará Docker, Compose, HAProxy, MariaDB, Redis y los archivos de infraestructura.

El laboratorio permite observar:

1. Separación entre la administración de la aplicación y la infraestructura.
2. Identidades numéricas de usuarios y grupos en Linux.
3. Bind mounts entre el host y los contenedores.
4. Balanceo de carga sin persistencia de backend.
5. Estado compartido entre réplicas web.
6. Comprobaciones de salud.
7. Conmutación ante el fallo de una réplica web.
8. Limitaciones de una arquitectura desplegada en un único host.

---

## 2. Resultados de aprendizaje

Al finalizar, el estudiante podrá:

- Construir una imagen personalizada de Apache/PHP.
- Explicar cómo Linux aplica permisos mediante UID y GID.
- Ejecutar procesos web dentro del contenedor con el mismo UID del propietario del sitio en el host.
- Definir servicios, redes, volúmenes y dependencias en Docker Compose.
- Configurar HAProxy con el algoritmo `roundrobin`.
- Distinguir entre caché, sesión PHP y autenticación de WordPress.
- Comprobar que dos servidores web pueden atender el mismo sitio.
- Probar la continuidad del servicio al detener una réplica web.
- Identificar los puntos únicos de fallo de la solución.

---

## 3. Arquitectura

```mermaid
flowchart LR
    U[Usuario<br/>Navegador] -->|HTTP localhost:803| H[HAProxy]

    H -->|roundrobin| W1[web1<br/>Apache + PHP 8.3]
    H -->|roundrobin| W2[web2<br/>Apache + PHP 8.3]

    W1 --> DB[(MariaDB)]
    W2 --> DB

    W1 --> R[(Redis)]
    W2 --> R

    W1 --> F[/home/orh<br/>archivos WordPress]
    W2 --> F

    H -. estadísticas .-> S[localhost:8404/stats]
```

Todos los contenedores se ejecutan en el mismo host WSL y se comunican mediante la red Docker `webnet`.

### 3.1 Flujo de una solicitud

1. El navegador envía una solicitud a `http://localhost:803`.
2. HAProxy selecciona `web1` o `web2`.
3. El servidor web ejecuta el mismo código WordPress desde `/home/orh`.
4. WordPress consulta la misma base de datos MariaDB.
5. WordPress puede utilizar Redis como caché persistente de objetos.
6. La respuesta regresa al navegador mediante HAProxy.

### 3.2 Estado compartido

| Estado o recurso | Ubicación |
|---|---|
| Entradas, usuarios, opciones y tokens de WordPress | MariaDB |
| Archivos, plugins, temas y medios | `/home/orh` |
| Caché persistente de objetos | Redis, base lógica 0 |
| Sesiones PHP de plugins que usan `session_start()` | Redis, base lógica 1 |
| Cookie de autenticación | Navegador del usuario |
| Configuración y salts de WordPress | `/home/orh/wp-config.php` |

> **Importante:** WordPress Core no depende normalmente de `$_SESSION` para mantener el inicio de sesión. La continuidad de la autenticación se obtiene porque ambos servidores usan la misma base de datos, el mismo `wp-config.php`, las mismas claves criptográficas y la misma URL pública. Redis Object Cache mejora el acceso a objetos; no sustituye por sí solo el mecanismo de autenticación de WordPress.

---

## 4. Alcance de la tolerancia a fallos

La práctica agrega redundancia solamente en la **capa web**:

```text
HAProxy
├── web1
└── web2
```

Si `web1` falla, HAProxy puede continuar utilizando `web2`. Sin embargo, la solución completa todavía tiene varios puntos únicos de fallo:

- El host Windows/WSL.
- Docker Engine.
- HAProxy.
- MariaDB.
- Redis.
- El sistema de archivos que contiene `/home/orh`.

Por tanto, esta práctica demuestra **balanceo de carga y tolerancia al fallo de una réplica web**, pero no constituye una plataforma de alta disponibilidad integral.

En términos de sistemas operativos, la tolerancia a fallos consiste en mantener la operación normal a pesar de fallos de hardware o software y suele requerir redundancia. La disponibilidad representa la fracción de tiempo durante la cual el servicio puede atender solicitudes.

---

## 5. Prerrequisitos

### 5.1 Plataforma

- Windows con WSL habilitado.
- Ubuntu 24.04 dentro de WSL.
- Docker Engine o Docker Desktop con integración WSL.
- Docker Compose v2.
- Acceso `sudo`.
- Usuario Linux `cmendez`.
- Usuario Linux `orh` con UID y GID `1003:1003`.

### 5.2 Verificar Docker

```bash
docker --version
docker compose version
docker info
```

El último comando debe ejecutarse sin errores.

### 5.3 Verificar el usuario del sitio

```bash
getent passwd orh
id orh
```

Resultado esperado:

```text
orh:x:1003:1003:...:/home/orh:/bin/bash
uid=1003(orh) gid=1003(orh) groups=1003(orh)
```

Verifique el directorio:

```bash
sudo stat -c '%U:%G %u:%g %a %n' /home/orh
```

El propietario debe ser `orh:orh`, con UID y GID `1003:1003`.

---

## 6. Separación de responsabilidades

| Elemento | Administrador |
|---|---|
| Contenido de `/home/orh` | `orh` |
| Plugins, temas y archivos WordPress | `orh` |
| Proyecto Docker Compose | `cmendez` |
| Dockerfiles | `cmendez` |
| Configuración de HAProxy | `cmendez` |
| Contenedores y redes Docker | `cmendez` |
| MariaDB y Redis | `cmendez` |
| Actualización de imágenes | `cmendez` |

Aunque `cmendez` no sea propietario ordinario de los archivos, un usuario con `sudo` o acceso al socket de Docker tiene capacidad administrativa equivalente a `root`. La separación planteada organiza las responsabilidades operativas, pero no constituye una frontera de seguridad frente al administrador del host.

---

## 7. Estructura del proyecto

El proyecto será administrado por `cmendez` en:

```text
/home/cmendez/grafana/grafana/redis/web/haproxy
```

Cree los directorios:

```bash
mkdir -p /home/cmendez/grafana/grafana/redis/web/haproxy/web
mkdir -p /home/cmendez/grafana/grafana/redis/web/haproxy/haproxy

cd /home/cmendez/grafana/grafana/redis/web/haproxy
```

La estructura final será:

```text
haproxy/
├── .env
├── compose.yaml
├── haproxy/
│   └── haproxy.cfg
└── web/
    ├── Dockerfile
    ├── docker-entrypoint.sh
    └── redis-session.ini
```

Los archivos de WordPress no se almacenan dentro del proyecto. Se ubican en:

```text
/home/orh
```

---

## 8. Crear una imagen Apache/PHP genérica

La imagen no contendrá un UID o GID específico. La identidad de `www-data` se ajustará cada vez que el contenedor arranque, antes de iniciar Apache.

### 8.1 Crear `web/Dockerfile`

Cree el archivo `web/Dockerfile`:

```dockerfile
FROM php:8.3-apache-bookworm

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        curl \
        libfreetype6-dev \
        libicu-dev \
        libjpeg62-turbo-dev \
        libpng-dev \
        libzip-dev \
        passwd \
        unzip \
    && docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
    && docker-php-ext-install -j"$(nproc)" \
        gd \
        intl \
        mysqli \
        opcache \
        pdo \
        pdo_mysql \
        zip \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && a2enmod rewrite headers \
    && sed -ri \
        's/AllowOverride None/AllowOverride All/g' \
        /etc/apache2/apache2.conf \
    && echo "ServerName localhost" \
        > /etc/apache2/conf-available/servername.conf \
    && a2enconf servername \
    && rm -rf /var/lib/apt/lists/*

RUN printf '\n# Permisos predeterminados de la aplicación\numask 0027\n' \
    >> /etc/apache2/envvars

COPY redis-session.ini \
    /usr/local/etc/php/conf.d/zz-redis-session.ini

COPY docker-entrypoint.sh \
    /usr/local/bin/wordpress-entrypoint

RUN chmod 0755 /usr/local/bin/wordpress-entrypoint

ENTRYPOINT ["wordpress-entrypoint"]
CMD ["apache2-foreground"]
```

La imagen no contiene instrucciones como:

```dockerfile
ARG APP_UID=1003
ARG APP_GID=1003
RUN usermod ...
```

Por tanto, la imagen construida no queda asociada al usuario `orh` ni al número `1003`.

### 8.2 Crear `web/docker-entrypoint.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

APP_UID="${APP_UID:-33}"
APP_GID="${APP_GID:-33}"

error() {
    printf 'ERROR: %s\n' "$*" >&2
    exit 1
}

case "${APP_UID}" in
    ''|*[!0-9]*)
        error "APP_UID debe ser un entero positivo."
        ;;
esac

case "${APP_GID}" in
    ''|*[!0-9]*)
        error "APP_GID debe ser un entero positivo."
        ;;
esac

if [ "$(id -u)" -ne 0 ]; then
    error "El contenedor debe iniciar como root para ajustar www-data."
fi

if ! id www-data >/dev/null 2>&1; then
    error "No existe el usuario interno www-data."
fi

current_uid="$(id -u www-data)"
current_gid="$(id -g www-data)"

uid_owner="$(
    getent passwd "${APP_UID}" |
    cut -d: -f1 ||
    true
)"

gid_owner="$(
    getent group "${APP_GID}" |
    cut -d: -f1 ||
    true
)"

if [ -n "${uid_owner}" ] && [ "${uid_owner}" != "www-data" ]; then
    error "El UID ${APP_UID} ya pertenece a ${uid_owner} dentro del contenedor."
fi

if [ -n "${gid_owner}" ] && [ "${gid_owner}" != "www-data" ]; then
    error "El GID ${APP_GID} ya pertenece al grupo ${gid_owner} dentro del contenedor."
fi

if [ "${current_gid}" != "${APP_GID}" ]; then
    groupmod --gid "${APP_GID}" www-data
fi

if [ "${current_uid}" != "${APP_UID}" ] || \
   [ "${current_gid}" != "${APP_GID}" ]; then
    usermod \
        --uid "${APP_UID}" \
        --gid "${APP_GID}" \
        www-data
fi

# Solo se preparan directorios internos de ejecución.
# No se cambia recursivamente la propiedad de /var/www/html.
install -d \
    -o www-data \
    -g www-data \
    -m 0755 \
    /var/run/apache2 \
    /var/lock/apache2

printf \
    'Iniciando Apache: www-data usa UID=%s GID=%s\n' \
    "$(id -u www-data)" \
    "$(id -g www-data)"

exec docker-php-entrypoint "$@"
```

Asigne permiso de ejecución:

```bash
chmod 0755 web/docker-entrypoint.sh
```

### 8.3 Funcionamiento del remapeo

Los permisos de Linux se evalúan mediante números, no mediante nombres.

Para este laboratorio, `.env` suministrará:

```text
APP_UID solicitado = 1003
APP_GID solicitado = 1003
```

En el host:

```text
UID 1003 = orh
GID 1003 = orh
```

Después de ejecutar el script dentro del contenedor:

```text
UID 1003 = www-data
GID 1003 = www-data
```

Cuando PHP crea un archivo en el bind mount, el host registra al propietario numérico `1003:1003` y lo muestra como `orh:orh`.

No se configura:

```yaml
user: "1003:1003"
```

El contenedor debe iniciar como `root` para modificar `/etc/passwd` y `/etc/group`. Luego Apache crea procesos trabajadores con la identidad `www-data`, ya ajustada.

El script es idempotente: si `www-data` ya tiene el UID y GID solicitados, no los modifica nuevamente.

### 8.4 Ventaja operativa

La misma imagen puede utilizarse para sitios con propietarios diferentes:

```text
Sitio A: UID/GID 1003
Sitio B: UID/GID 1050
Sitio C: UID/GID 2100
```

Solo se cambian variables de entorno y se recrean los contenedores. No es necesario reconstruir la imagen.

---

## 9. Configurar sesiones PHP en Redis

Cree `web/redis-session.ini`:

```ini
; WordPress Core no utiliza normalmente sesiones PHP para autenticar.
; Esta configuración cubre plugins que sí llaman session_start().

session.save_handler = redis
session.save_path = "tcp://redis:6379?database=1&prefix=PHPREDIS_SESSION:"

session.use_strict_mode = 1
session.cookie_httponly = 1
session.cookie_samesite = "Lax"
```

La distribución lógica será:

```text
Redis DB 0: caché de objetos de WordPress
Redis DB 1: sesiones PHP de plugins
```

Redis no se expone directamente al host; solamente los contenedores conectados a `webnet` pueden acceder al puerto 6379.

---

## 10. Configurar HAProxy

Cree `haproxy/haproxy.cfg`:

```haproxy
global
    log stdout format raw local0
    maxconn 2000

defaults
    log global
    mode http

    option httplog
    option dontlognull
    option forwardfor
    option http-server-close
    option redispatch

    retries 2

    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend wordpress_front
    bind :80

    http-request set-header X-Forwarded-Proto http
    http-request set-header X-Forwarded-Port 803

    default_backend wordpress_pool

backend wordpress_pool
    balance roundrobin

    # Comprobación HTTP de la aplicación.
    option httpchk
    http-check send meth GET uri /wp-login.php ver HTTP/1.1 hdr Host localhost
    http-check expect status 200

    # Permite identificar el servidor que produjo la respuesta.
    http-response set-header X-Backend %[srv_name]

    server web1 web1:80 check inter 1s fall 1 rise 2
    server web2 web2:80 check inter 1s fall 1 rise 2

listen stats
    bind :8404
    mode http

    stats enable
    stats uri /stats
    stats refresh 2s
```

### 10.1 Balanceo sin persistencia

La configuración no incluye:

```haproxy
cookie SERVERID insert
balance source
stick-table
stick on
```

Por tanto, HAProxy no fuerza al cliente a permanecer en un backend determinado.

El algoritmo `roundrobin` selecciona sucesivamente servidores disponibles del pool. Las comprobaciones de salud retiran de la rotación a un backend que deja de responder.

> La página de estadísticas no tiene autenticación porque es un laboratorio local. No debe exponerse de esta forma en una red de producción.

---

## 11. Configurar la identidad y crear `compose.yaml`

### 11.1 Crear `.env`

Cree `.env` en la raíz del proyecto:

```dotenv
WORDPRESS_UID=1003
WORDPRESS_GID=1003
```

Compruebe que coincidan con el propietario del sitio:

```bash
id orh
```

Proteja el archivo:

```bash
chmod 600 .env
```

La imagen no contiene estos números. Compose los entrega al contenedor al iniciarlo.

### 11.2 Crear `compose.yaml`

```yaml
name: wordpress-ha-lab

x-wordpress-identity: &wordpress_identity
  APP_UID: "${WORDPRESS_UID:?Defina WORDPRESS_UID en el archivo .env}"
  APP_GID: "${WORDPRESS_GID:?Defina WORDPRESS_GID en el archivo .env}"

services:
  mariadb:
    image: mariadb:11.4
    restart: unless-stopped

    environment:
      MARIADB_ROOT_PASSWORD: rootpassword
      MARIADB_DATABASE: mi_base_de_datos
      MARIADB_USER: usuario
      MARIADB_PASSWORD: password

    volumes:
      - mariadb_data:/var/lib/mysql

    networks:
      - webnet

    healthcheck:
      test:
        - CMD
        - healthcheck.sh
        - --connect
        - --innodb_initialized
      interval: 5s
      timeout: 5s
      retries: 20
      start_period: 20s

  redis:
    image: redis:7.4-alpine
    restart: unless-stopped

    command:
      - redis-server
      - --appendonly
      - "yes"
      - --save
      - "60"
      - "1"

    volumes:
      - redis_data:/data

    networks:
      - webnet

    healthcheck:
      test:
        - CMD
        - redis-cli
        - ping
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 5s

  web1: &wordpress_web
    image: wordpress-ha-php:8.3

    build:
      context: ./web
      dockerfile: Dockerfile

    restart: unless-stopped

    volumes:
      - /home/orh:/var/www/html

    depends_on:
      mariadb:
        condition: service_healthy
      redis:
        condition: service_healthy

    networks:
      - webnet

    healthcheck:
      test:
        - CMD-SHELL
        - curl -fsS http://127.0.0.1/wp-login.php >/dev/null || exit 1
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 30s

    environment:
      <<: *wordpress_identity
      BACKEND_NAME: web1

    hostname: web1

  web2:
    <<: *wordpress_web

    environment:
      <<: *wordpress_identity
      BACKEND_NAME: web2

    hostname: web2

  haproxy:
    image: haproxy:3.2-alpine
    restart: unless-stopped

    ports:
      - "803:80"
      - "8404:8404"

    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro

    depends_on:
      - web1
      - web2

    networks:
      - webnet

networks:
  webnet:
    driver: bridge

volumes:
  mariadb_data:
  redis_data:
```

### 11.3 Notas

- El UID y GID no son argumentos de construcción.
- `APP_UID` y `APP_GID` se entregan en cada inicio del contenedor.
- Ambos backends utilizan la misma imagen genérica.
- La redefinición de `environment` en `web2` conserva explícitamente `APP_UID` y `APP_GID`.
- MariaDB y Redis tienen volúmenes persistentes.
- Los puertos de MariaDB y Redis no se publican en el host.
- HAProxy determina mediante health checks cuáles backends permanecen en rotación.

### 11.4 Verificar la configuración resuelta

```bash
docker compose config |
grep -E 'APP_UID|APP_GID|BACKEND_NAME'
```

Resultado esperado:

```text
APP_UID: "1003"
APP_GID: "1003"
BACKEND_NAME: web1
...
APP_UID: "1003"
APP_GID: "1003"
BACKEND_NAME: web2
```

---

## 12. Descargar WordPress como usuario `orh`

### 12.1 Preparar el directorio

Asegure la propiedad:

```bash
sudo chown -R orh:orh /home/orh
sudo chmod 750 /home/orh
```

Compruebe el contenido existente:

```bash
sudo -u orh ls -la /home/orh
```

Si el directorio contiene información que debe conservarse, realice una copia antes de continuar.

### 12.2 Limpiar un laboratorio anterior

Este comando elimina todo el contenido de `/home/orh`:

```bash
sudo find /home/orh \
  -mindepth 1 \
  -maxdepth 1 \
  -exec rm -rf {} +
```

> Ejecútelo únicamente cuando se confirme que el directorio se utilizará para una instalación nueva.

### 12.3 Descargar WordPress

```bash
sudo -u orh bash -c '
  curl -fsSL https://wordpress.org/latest.tar.gz |
  tar -xz --strip-components=1 -C /home/orh
'
```

### 12.4 Aplicar permisos

```bash
sudo chown -R orh:orh /home/orh

sudo find /home/orh \
  -type d \
  -exec chmod 750 {} \;

sudo find /home/orh \
  -type f \
  -exec chmod 640 {} \;
```

Verifique:

```bash
sudo stat -c '%U:%G %u:%g %a %n' /home/orh
sudo stat -c '%U:%G %u:%g %a %n' /home/orh/wp-login.php
```

Resultado esperado:

```text
orh:orh 1003:1003 750 /home/orh
orh:orh 1003:1003 640 /home/orh/wp-login.php
```

---

## 13. Validar la configuración antes del despliegue

Desde el directorio del proyecto:

```bash
cd /home/cmendez/grafana/grafana/redis/web/haproxy
```

Validar Compose:

```bash
docker compose config
```

La salida debe mostrar la configuración resuelta sin errores de sintaxis.

Validar la configuración de HAProxy mediante un contenedor temporal:

```bash
docker run --rm \
  -v "$PWD/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro" \
  haproxy:3.2-alpine \
  haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Resultado esperado:

```text
Configuration file is valid
```

---

## 14. Construir y levantar la plataforma

Construir la imagen web genérica:

```bash
docker compose build --no-cache web1 web2
```

La construcción no utiliza `WORDPRESS_UID` ni `WORDPRESS_GID`.

Levantar todos los servicios:

```bash
docker compose up -d
```

Durante cada arranque, el `entrypoint` ajusta `www-data` y luego inicia Apache.

Revisar el estado:

```bash
docker compose ps
```

Resultado esperado después del periodo de inicialización:

```text
mariadb    running (healthy)
redis      running (healthy)
web1       running (healthy)
web2       running (healthy)
haproxy    running
```

Consultar los logs:

```bash
docker compose logs --tail=50 mariadb
docker compose logs --tail=50 redis
docker compose logs --tail=50 web1
docker compose logs --tail=50 web2
docker compose logs --tail=50 haproxy
```

---

## 15. Verificar las identidades de los procesos

Consultar el usuario `www-data` dentro de ambos contenedores:

```bash
docker compose exec web1 id www-data
docker compose exec web2 id www-data
```

Resultado esperado para los valores actuales de `.env`:

```text
uid=1003(www-data) gid=1003(www-data) groups=1003(www-data)
```

Revise el mensaje del `entrypoint`:

```bash
docker compose logs web1 web2 |
grep 'Iniciando Apache'
```

Resultado esperado:

```text
Iniciando Apache: www-data usa UID=1003 GID=1003
```

Revisar los procesos Apache:

```bash
docker compose exec web1 \
  ps -eo user,uid,gid,pid,cmd
```

Se debe observar:

```text
root         0     0     1 apache2 -DFOREGROUND
www-data  1003  1003    ... apache2 -DFOREGROUND
```

El proceso maestro se ejecuta como `root`, pero las solicitudes se atienden mediante procesos con UID y GID `1003:1003`.

---

## 16. Verificar escritura en el bind mount

Crear un archivo desde `web1`:

```bash
docker compose exec web1 sh -c '
  touch /var/www/html/prueba-web1.txt
  stat -c "%U:%G %u:%g %a %n" /var/www/html/prueba-web1.txt
'
```

Crear otro desde `web2`:

```bash
docker compose exec web2 sh -c '
  touch /var/www/html/prueba-web2.txt
  stat -c "%U:%G %u:%g %a %n" /var/www/html/prueba-web2.txt
'
```

Revisarlos en el host:

```bash
ls -ln /home/orh/prueba-web1.txt
ls -ln /home/orh/prueba-web2.txt

ls -l /home/orh/prueba-web1.txt
ls -l /home/orh/prueba-web2.txt
```

Ambos deben mostrarse como propiedad de `orh:orh`, UID y GID `1003:1003`.

Eliminar las pruebas:

```bash
sudo -u orh rm \
  /home/orh/prueba-web1.txt \
  /home/orh/prueba-web2.txt
```

---

## 17. Instalar WordPress

Abra en el navegador:

```text
http://localhost:803
```

Configure la conexión:

| Campo | Valor |
|---|---|
| Base de datos | `mi_base_de_datos` |
| Usuario | `usuario` |
| Contraseña | `password` |
| Servidor de la base de datos | `mariadb` |
| Prefijo de tablas | `wp_` |

El servidor de base de datos debe ser `mariadb`, no `localhost`. Dentro de un contenedor, `localhost` identifica al propio contenedor.

Complete:

- Título del sitio.
- Usuario administrador de WordPress.
- Contraseña.
- Correo electrónico.

La URL pública del laboratorio es:

```text
http://localhost:803
```

Después de la instalación, proteja `wp-config.php`:

```bash
sudo chown orh:orh /home/orh/wp-config.php
sudo chmod 600 /home/orh/wp-config.php
```

Verifique:

```bash
sudo stat -c '%U:%G %u:%g %a %n' /home/orh/wp-config.php
```

Resultado esperado:

```text
orh:orh 1003:1003 600 /home/orh/wp-config.php
```

---

## 18. Configurar WordPress para Redis

Edite el archivo como `orh`:

```bash
sudo -u orh nano /home/orh/wp-config.php
```

Antes de la línea:

```php
/* That's all, stop editing! Happy publishing. */
```

agregue:

```php
/*
 * URL pública servida por HAProxy.
 */
define( 'WP_HOME', 'http://localhost:803' );
define( 'WP_SITEURL', 'http://localhost:803' );

/*
 * Redis Object Cache.
 */
define( 'WP_REDIS_CLIENT', 'phpredis' );
define( 'WP_REDIS_HOST', 'redis' );
define( 'WP_REDIS_PORT', 6379 );
define( 'WP_REDIS_DATABASE', 0 );
define( 'WP_REDIS_PREFIX', 'wordpress-ha:' );

define( 'WP_REDIS_TIMEOUT', 1 );
define( 'WP_REDIS_READ_TIMEOUT', 1 );

/*
 * Permite modificaciones directas realizadas por WordPress.
 * Los archivos continuarán perteneciendo al UID 1003.
 */
define( 'FS_METHOD', 'direct' );
```

Asegure nuevamente los permisos:

```bash
sudo chown orh:orh /home/orh/wp-config.php
sudo chmod 600 /home/orh/wp-config.php
```

Los dos contenedores leen exactamente el mismo archivo y, por tanto, comparten:

- Configuración de base de datos.
- URL del sitio.
- Claves `AUTH_KEY`, `LOGGED_IN_KEY` y demás salts.
- Configuración de Redis.

---

## 19. Instalar Redis Object Cache

Ingrese a:

```text
http://localhost:803/wp-admin
```

Luego:

1. Abra **Plugins**.
2. Seleccione **Añadir plugin**.
3. Busque **Redis Object Cache**.
4. Instale y active el plugin.
5. Abra **Ajustes → Redis**.
6. Seleccione **Enable Object Cache**.

El estado esperado es:

```text
Status: Connected
Client: PhpRedis
Host: redis
Port: 6379
Database: 0
```

### 19.1 Verificar la extensión de PHP

```bash
docker compose exec web1 php -m \
  | grep -E 'mysqli|pdo_mysql|redis'

docker compose exec web2 php -m \
  | grep -E 'mysqli|pdo_mysql|redis'
```

Resultado esperado:

```text
mysqli
pdo_mysql
redis
```

### 19.2 Probar Redis desde PHP

```bash
docker compose exec web1 php -r '
$r = new Redis();
$r->connect("redis", 6379);
var_dump($r->ping());
'
```

Repita desde `web2`:

```bash
docker compose exec web2 php -r '
$r = new Redis();
$r->connect("redis", 6379);
var_dump($r->ping());
'
```

### 19.3 Revisar las claves

Base lógica usada por WordPress Object Cache:

```bash
docker compose exec redis redis-cli -n 0 DBSIZE
docker compose exec redis redis-cli -n 0 --scan | head
```

Base lógica reservada para sesiones PHP:

```bash
docker compose exec redis redis-cli -n 1 DBSIZE
docker compose exec redis redis-cli -n 1 --scan | head
```

La base 1 puede permanecer vacía si ningún plugin utiliza sesiones PHP.

---

## 20. Agregar un indicador visible del backend

Para que los estudiantes puedan observar el cambio entre `web1` y `web2`, se puede crear un **must-use plugin**. Los plugins de este directorio se cargan automáticamente.

Crear el directorio:

```bash
sudo install \
  -d \
  -o orh \
  -g orh \
  -m 0750 \
  /home/orh/wp-content/mu-plugins
```

Crear `/home/orh/wp-content/mu-plugins/backend-indicator.php`:

```bash
sudo tee \
  /home/orh/wp-content/mu-plugins/backend-indicator.php \
  >/dev/null <<'PHP'
<?php
/**
 * Plugin Name: Indicador del backend
 * Description: Muestra qué réplica web atendió la solicitud.
 */

function laboratorio_backend_name(): string {
    $backend = getenv('BACKEND_NAME');

    if ($backend !== false && $backend !== '') {
        return $backend;
    }

    return gethostname() ?: 'desconocido';
}

add_action(
    'admin_bar_menu',
    static function (WP_Admin_Bar $admin_bar): void {
        $admin_bar->add_node(
            [
                'id'    => 'laboratorio-backend',
                'title' => 'Backend: ' . esc_html(laboratorio_backend_name()),
                'href'  => false,
            ]
        );
    },
    100
);

add_action(
    'send_headers',
    static function (): void {
        header('X-WordPress-Backend: ' . laboratorio_backend_name());
    }
);
PHP
```

Corregir propiedad y permisos:

```bash
sudo chown orh:orh \
  /home/orh/wp-content/mu-plugins/backend-indicator.php

sudo chmod 640 \
  /home/orh/wp-content/mu-plugins/backend-indicator.php
```

Al iniciar sesión, la barra de administración mostrará:

```text
Backend: web1
```

o:

```text
Backend: web2
```

---

## 21. Probar el balanceo de carga

### 21.1 Comprobación mediante encabezados HTTP

Ejecute:

```bash
for i in $(seq 1 10); do
    printf "Solicitud %02d: " "$i"

    curl -sI \
      -H 'Connection: close' \
      "http://localhost:803/?probe=$i" \
    | awk -F': ' '
        tolower($1) == "x-backend" {
            gsub("\r", "", $2);
            print $2
        }
      '
done
```

Resultado esperado:

```text
Solicitud 01: web1
Solicitud 02: web2
Solicitud 03: web1
Solicitud 04: web2
Solicitud 05: web1
Solicitud 06: web2
```

También puede consultar el encabezado generado por WordPress:

```bash
curl -sI http://localhost:803/ \
  | grep -Ei 'x-backend|x-wordpress-backend'
```

### 21.2 Comprobación en el navegador

1. Inicie sesión en WordPress.
2. Observe el indicador de backend en la barra administrativa.
3. Recargue varias veces.
4. Verifique que las solicitudes son atendidas por `web1` y `web2`.
5. Confirme que el usuario permanece autenticado.

### 21.3 Por qué una recarga no siempre alterna visualmente

Una página genera varias solicitudes:

- Documento HTML.
- CSS.
- JavaScript.
- Imágenes.
- Fuentes.
- API REST.

El navegador también puede reutilizar conexiones HTTP. `roundrobin` distribuye solicitudes entre backends disponibles, pero una pulsación de recarga no equivale necesariamente a una única solicitud. El ciclo con `curl`, una sola solicitud y `Connection: close` es una prueba más controlada.

---

## 22. Revisar HAProxy

Abra:

```text
http://localhost:8404/stats
```

Los servidores `web1` y `web2` deben mostrarse como `UP`.

Consultar logs:

```bash
docker compose logs -f haproxy
```

Verificar la configuración dentro del contenedor:

```bash
docker compose exec haproxy \
  haproxy -c \
  -f /usr/local/etc/haproxy/haproxy.cfg
```

---

## 23. Probar la continuidad del inicio de sesión

### 23.1 Condiciones necesarias

La sesión autenticada se conserva porque:

1. El navegador usa la misma URL: `localhost:803`.
2. Envía la misma cookie a HAProxy.
3. Los dos backends usan el mismo código.
4. Los dos backends leen el mismo `wp-config.php`.
5. Ambos poseen las mismas claves y salts.
6. Ambos consultan la misma base de datos.
7. El token de sesión de WordPress puede validarse desde cualquiera de las réplicas.

### 23.2 Procedimiento

1. Inicie sesión en:

   ```text
   http://localhost:803/wp-admin
   ```

2. Confirme que aparece la barra de administración.
3. Recargue hasta observar respuestas de ambos backends.
4. Confirme que el usuario continúa autenticado.
5. Mantenga abierta la página para la prueba de fallo.

---

## 24. Prueba de tolerancia al fallo de `web1`

Detener la primera réplica:

```bash
docker compose stop web1
```

Revisar servicios:

```bash
docker compose ps
```

Revisar HAProxy:

```text
http://localhost:8404/stats
```

Después de la comprobación de salud:

```text
web1: DOWN
web2: UP
```

Recargue WordPress.

Resultado esperado:

- El sitio continúa disponible.
- La respuesta proviene de `web2`.
- El usuario continúa autenticado.
- La cookie de inicio de sesión no se pierde.

Comprobación por consola:

```bash
curl -sI http://localhost:803/ \
  | grep -i x-backend
```

Resultado esperado:

```text
X-Backend: web2
```

Restaurar `web1`:

```bash
docker compose start web1
```

Esperar hasta que esté saludable:

```bash
docker compose ps
```

HAProxy lo reincorporará después de dos comprobaciones exitosas, debido a `rise 2`.

---

## 25. Prueba de tolerancia al fallo de `web2`

Detener la segunda réplica:

```bash
docker compose stop web2
```

Verifique que el sitio responde mediante `web1`:

```bash
curl -sI http://localhost:803/ \
  | grep -i x-backend
```

Restaurar:

```bash
docker compose start web2
```

### 25.1 Interpretación correcta

La sesión del usuario no debería perderse. Sin embargo, una solicitud que ya estuviera en ejecución exactamente durante la detención abrupta podría fallar o requerir una recarga. La prueba demuestra que:

- HAProxy detecta el backend fallido.
- Retira el backend de la rotación.
- Las solicitudes posteriores se dirigen al backend saludable.
- La autenticación sigue siendo válida en la otra réplica.

No debe interpretarse como una garantía absoluta de que ninguna solicitud en vuelo pueda verse afectada.

---

## 26. Verificar la propiedad después de usar WordPress

Instale un plugin, cargue una imagen o actualice un tema desde WordPress. Después revise:

```bash
sudo find /home/orh \
  -maxdepth 4 \
  -printf '%u:%g %m %p\n' \
  | head -50
```

Los archivos creados por PHP deben mostrarse como:

```text
orh:orh
```

Revisar archivos que no tengan UID o GID 1003:

```bash
sudo find /home/orh \
  \( ! -uid 1003 -o ! -gid 1003 \) \
  -printf '%u:%g %m %p\n'
```

Si no hay salida, toda la propiedad es consistente.

Para corregir una instalación de laboratorio:

```bash
sudo chown -R orh:orh /home/orh
```

---

## 27. Operaciones habituales

### Iniciar

```bash
docker compose up -d
```

### Detener sin eliminar

```bash
docker compose stop
```

### Detener y eliminar contenedores y red

```bash
docker compose down
```

### Reconstruir la imagen web

Solo es necesario cuando cambian el `Dockerfile`, el `entrypoint` o las extensiones PHP:

```bash
docker compose up -d \
  --build \
  --force-recreate \
  web1 web2
```

### Aplicar otro UID/GID sin reconstruir la imagen

Edite `.env`:

```dotenv
WORDPRESS_UID=1050
WORDPRESS_GID=1050
```

Recree solamente los backends:

```bash
docker compose up -d \
  --force-recreate \
  --no-deps \
  web1 web2
```

Compruebe:

```bash
docker compose exec web1 id www-data
docker compose exec web2 id www-data
```

El propietario numérico del bind mount también debe coincidir. Para este laboratorio se mantienen `1003:1003` y `/home/orh`.

### Ver logs

```bash
docker compose logs -f
```

### Mostrar procesos y salud

```bash
docker compose ps
docker stats
```

### Ingresar a un backend

```bash
docker compose exec web1 bash
docker compose exec web2 bash
```

### Consultar MariaDB

```bash
docker compose exec mariadb \
  mariadb \
  -uusuario \
  -ppassword \
  mi_base_de_datos
```

### Consultar Redis

```bash
docker compose exec redis redis-cli
```

---

## 28. Reiniciar completamente el laboratorio

Para eliminar contenedores y red, conservando los datos:

```bash
docker compose down
```

Para eliminar también los volúmenes de MariaDB y Redis:

```bash
docker compose down -v
```

> `down -v` elimina la base de datos y los datos persistentes de Redis. Debe utilizarse solamente cuando se desea reiniciar el laboratorio desde cero.

Para limpiar WordPress:

```bash
sudo find /home/orh \
  -mindepth 1 \
  -maxdepth 1 \
  -exec rm -rf {} +
```

Después repita la descarga de WordPress.

---

## 29. Solución de problemas

### 29.1 `web1` y `web2` aparecen como `unhealthy`

Verifique que WordPress fue descargado:

```bash
ls -l /home/orh/wp-login.php
```

Revise:

```bash
docker compose logs web1
docker compose logs web2
```

Pruebe desde el contenedor:

```bash
docker compose exec web1 \
  curl -v http://127.0.0.1/wp-login.php
```

### 29.2 Error `Permission denied`

Verifique las identidades:

```bash
id orh
docker compose exec web1 id www-data
docker compose exec web2 id www-data
```

Todas deben indicar UID y GID `1003:1003`.

Compruebe las variables entregadas:

```bash
docker compose exec web1 env |
grep -E '^APP_(UID|GID)='
```

Revise el arranque dinámico:

```bash
docker compose logs web1 web2 |
grep -E 'Iniciando Apache|ERROR'
```

Revise la ruta:

```bash
namei -l /home/orh
```

Corrija para el laboratorio:

```bash
sudo chown -R orh:orh /home/orh
sudo find /home/orh -type d -exec chmod 750 {} \;
sudo find /home/orh -type f -exec chmod 640 {} \;
sudo chmod 600 /home/orh/wp-config.php
```

### 29.3 El contenedor termina por una colisión de UID o GID

El `entrypoint` se detiene si la identidad solicitada ya pertenece a otra cuenta interna:

```text
ERROR: El UID 1003 ya pertenece a otro_usuario dentro del contenedor.
```

Revise la imagen:

```bash
docker run --rm wordpress-ha-php:8.3 \
  getent passwd 1003

docker run --rm wordpress-ha-php:8.3 \
  getent group 1003
```

Seleccione una identidad sin colisión. No utilice identidades duplicadas, porque dificultan el control de acceso.

### 29.4 WordPress no conecta con MariaDB

Use:

```text
Servidor: mariadb
Puerto: 3306
```

No use `localhost`.

Verifique:

```bash
docker compose exec web1 \
  php -r '
  $db = new mysqli(
      "mariadb",
      "usuario",
      "password",
      "mi_base_de_datos"
  );
  echo $db->connect_error ?: "Conexion correcta\n";
  '
```

### 29.5 `Access denied` después de cambiar credenciales

Las variables de inicialización de MariaDB solamente se aplican cuando el volumen está vacío.

En un laboratorio que pueda reiniciarse:

```bash
docker compose down -v
docker compose up -d
```

Esto elimina la base de datos existente.

### 29.6 Redis Object Cache indica `Not connected`

Verifique:

```bash
docker compose exec redis redis-cli ping
```

Resultado:

```text
PONG
```

Pruebe desde PHP:

```bash
docker compose exec web1 php -r '
$r = new Redis();
$r->connect("redis", 6379);
echo $r->ping(), PHP_EOL;
'
```

Confirme en `wp-config.php`:

```php
define( 'WP_REDIS_HOST', 'redis' );
define( 'WP_REDIS_PORT', 6379 );
```

### 29.7 HAProxy devuelve `503 Service Unavailable`

Revise:

```bash
docker compose ps
docker compose logs haproxy
```

Compruebe la conectividad:

```bash
docker compose exec haproxy \
  wget -qO- http://web1/wp-login.php >/dev/null \
  && echo "web1 responde"

docker compose exec haproxy \
  wget -qO- http://web2/wp-login.php >/dev/null \
  && echo "web2 responde"
```

### 29.8 Redirección incorrecta o bucle

Compruebe que ambos valores son exactamente:

```php
define( 'WP_HOME', 'http://localhost:803' );
define( 'WP_SITEURL', 'http://localhost:803' );
```

No acceda directamente a `web1` o `web2`; sus puertos no se publican y la URL oficial es la de HAProxy.

### 29.9 El puerto ya está ocupado

```bash
sudo ss -lntp | grep -E ':803|:8404'
```

Cambie el puerto del host si fuera necesario:

```yaml
ports:
  - "8080:80"
```

En ese caso también debe ajustar `WP_HOME`, `WP_SITEURL` y `X-Forwarded-Port`.

---

## 30. Limitaciones y evolución hacia una arquitectura distribuida

En esta práctica, ambos contenedores montan un directorio local del mismo host:

```text
/home/orh
```

Esto funciona porque `web1` y `web2` están en la misma máquina. En un clúster con varios nodos, un bind mount local no representa automáticamente el mismo contenido en todos los hosts.

Una implementación distribuida requeriría:

- Almacenamiento compartido o replicado.
- Dos balanceadores y una IP virtual.
- MariaDB Galera o un sistema equivalente.
- Redis Sentinel, Redis Cluster o un servicio administrado.
- Docker Swarm, Kubernetes u otro orquestador.
- Programación y reprogramación automática de réplicas.
- Gestión de secretos.
- TLS.
- Copias de seguridad y recuperación.
- Métricas, logs y alertas centralizadas.

Una evolución posible sería:

```text
Internet
   |
VIP / balanceadores redundantes
   |
réplicas WordPress en varios nodos
   |
+----------------+----------------+
|                                 |
MariaDB redundante         Redis redundante
|
almacenamiento compartido o externo
```

---

## 31. Evidencias que debe entregar el estudiante

1. Salida de:

   ```bash
   docker compose ps
   ```

2. Salida de:

   ```bash
   docker compose exec web1 id www-data
   docker compose exec web2 id www-data
   ```

3. Comprobación de propiedad de archivos en `/home/orh`.
4. Captura o salida de diez solicitudes alternadas entre `web1` y `web2`.
5. Captura de HAProxy con ambos backends `UP`.
6. Captura de Redis Object Cache conectado.
7. Captura con `web1` detenido y `web2` atendiendo.
8. Evidencia de que el usuario continúa autenticado.
9. Captura con `web2` detenido y `web1` atendiendo.
10. Respuestas a las preguntas de análisis.

---

## 32. Preguntas de análisis

1. ¿Por qué es relevante que `www-data` tenga UID 1003 dentro de ambos contenedores?
2. ¿Qué ventaja ofrece ajustar el UID/GID en el `entrypoint` en lugar de incorporarlo como `ARG` de construcción?
3. ¿Qué ocurriría si `web1` utilizara UID 33 y `web2` UID 1003?
4. ¿Por qué Redis Object Cache no es, por sí solo, el responsable de mantener el inicio de sesión nativo de WordPress?
5. ¿Qué datos deben ser idénticos o compartidos para que cualquier réplica valide la cookie de autenticación?
6. ¿Qué diferencia existe entre balanceo de carga y alta disponibilidad?
7. ¿Por qué esta solución no es completamente tolerante a fallos?
8. ¿Qué ocurre si falla MariaDB?
9. ¿Qué ocurre si falla Redis? Distinga entre caché de objetos y sesiones PHP de plugins.
10. ¿Qué limitación presenta `/home/orh` si las réplicas se trasladan a hosts distintos?
11. ¿Qué función cumplen `fall 1` y `rise 2` en HAProxy?
12. ¿Por qué no se expone MariaDB ni Redis mediante `ports`?
13. ¿Qué riesgo representa que un usuario pertenezca al grupo `docker`?
14. ¿Por qué una solicitud que estaba en ejecución podría fallar aunque exista otra réplica?
15. ¿Qué componentes deberían duplicarse para eliminar los puntos únicos de fallo?
16. ¿Cómo cambiaría este laboratorio al implementarlo en Docker Swarm o Kubernetes?

---

## 33. Criterios de evaluación sugeridos

| Criterio | Porcentaje |
|---|---:|
| Estructura y construcción correcta de la imagen | 15 % |
| Gestión dinámica correcta del UID/GID | 15 % |
| Despliegue completo con Compose | 15 % |
| Instalación funcional de WordPress | 10 % |
| Integración de Redis | 10 % |
| Balanceo sin persistencia | 10 % |
| Prueba de fallo y continuidad de autenticación | 15 % |
| Análisis técnico y conclusiones | 10 % |

---

## 34. Conclusiones esperadas

Al finalizar la práctica, debe comprobarse que:

- `web1` y `web2` ejecutan la misma imagen genérica.
- El UID/GID se aplica al iniciar cada contenedor y no queda incorporado en la imagen.
- Los procesos trabajadores de Apache usan el UID y GID suministrados en tiempo de ejecución; en este laboratorio, `1003:1003`.
- Los archivos creados desde WordPress pertenecen a `orh:orh`.
- MariaDB mantiene el estado persistente principal.
- Redis proporciona una caché de objetos común.
- HAProxy distribuye solicitudes sin afinidad de backend.
- Un usuario autenticado puede ser atendido por cualquiera de las réplicas.
- Al detener una réplica, HAProxy utiliza la otra.
- La autenticación no se pierde por cambiar de backend.
- La arquitectura continúa dependiendo de varios componentes únicos.

---

## 35. Referencias

- Docker. *Compose file reference: Services*.  
  <https://docs.docker.com/reference/compose-file/services/>

- Docker. *Control startup and shutdown order in Compose*.  
  <https://docs.docker.com/compose/how-tos/startup-order/>

- Docker. *Docker Compose Quickstart*.  
  <https://docs.docker.com/compose/gettingstarted/>

- WordPress.org. *Requirements*.  
  <https://wordpress.org/about/requirements/>

- WordPress.org. *Redis Object Cache*.  
  <https://wordpress.org/plugins/redis-cache/>

- HAProxy Technologies. *Backends and roundrobin load balancing*.  
  <https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/backends/>

- HAProxy Technologies. *Health checks*.  
  <https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/health-checks/>

- Stallings, William. *Operating Systems: Internals and Design Principles*, 9.ª edición. Secciones sobre tolerancia a fallos, disponibilidad, redes y clústeres.
