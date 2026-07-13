# Laboratorio: observabilidad del kernel con eBPF en una plataforma WordPress de alta disponibilidad

**Curso:** Sistemas Operativos II  
**Tema:** procesos, llamadas al sistema, red, I/O, planificación, contenedores y observabilidad del kernel  
**Ambiente:** Ubuntu 24.04 sobre WSL2 + Docker Compose  

---

## 1. Propósito del laboratorio

En este laboratorio se continúa la implementación de una plataforma WordPress con alta disponibilidad local y se incorpora **eBPF** como mecanismo de observabilidad del sistema operativo. El objetivo no es únicamente verificar que WordPress responda, sino observar qué ocurre en el kernel Linux cuando una solicitud HTTP atraviesa HAProxy, llega a un contenedor Apache/PHP, consulta Redis o MariaDB y realiza operaciones de red, archivos, CPU y disco.

Al finalizar, se espera que el estudiante pueda explicar que un contenedor no es una máquina virtual completa: es un conjunto de procesos Linux aislados mediante namespaces, controlados mediante cgroups y ejecutados sobre el mismo kernel del host. eBPF permite observar eventos de ese kernel común sin modificar el código de WordPress, HAProxy, Redis o MariaDB.

---

## 2. Topología del laboratorio

El laboratorio usa un stack definido con Docker Compose llamado `wordpress-ha-lab`.

```text
Cliente curl / navegador
        │
        ▼
localhost:803
        │
        ▼
HAProxy :80
        │ balance roundrobin
        ├──────────────► web1 :80  Apache + PHP 8.3 + WordPress
        │
        └──────────────► web2 :80  Apache + PHP 8.3 + WordPress
                          │
                          ├──────► Redis :6379
                          │
                          └──────► MariaDB :3306
```

Servicios principales:

| Servicio | Función | Imagen / componente |
|---|---|---|
| `haproxy` | Balanceador HTTP | `haproxy:3.2-alpine` |
| `web1` | Backend WordPress 1 | `wordpress-ha-php:8.3` |
| `web2` | Backend WordPress 2 | `wordpress-ha-php:8.3` |
| `redis` | Almacenamiento de sesiones PHP | `redis:7.4-alpine` |
| `mariadb` | Base de datos | `mariadb:11.4` |

Puertos publicados en el host:

| Puerto host | Puerto contenedor | Uso |
|---:|---:|---|
| `803` | `80` | Acceso HTTP a WordPress por HAProxy |
| `8404` | `8404` | Estadísticas de HAProxy |

---

## 3. Conceptos mínimos

### 3.1 Contenedor

Un contenedor ejecuta procesos aislados del resto del sistema mediante mecanismos del kernel. Los principales son:

- **Namespaces:** aíslan vistas del sistema, por ejemplo red, procesos, hostname, montajes y usuarios.
- **cgroups:** limitan y contabilizan recursos como CPU, memoria e I/O.
- **Filesystem por capas y volúmenes:** permiten separar la imagen base de los datos persistentes.
- **Capabilities y seccomp:** reducen privilegios disponibles dentro del contenedor.

Idea clave:

```text
Docker no crea un kernel nuevo para cada contenedor.
Docker ejecuta procesos Linux aislados sobre el kernel del host.
```

### 3.2 Llamada al sistema

Una **llamada al sistema** o *system call* es una entrada controlada desde una aplicación hacia el kernel. Por ejemplo:

| Llamada | Uso típico |
|---|---|
| `openat()` | Abrir archivos |
| `read()` | Leer datos |
| `write()` | Escribir datos |
| `connect()` | Abrir conexión TCP |
| `accept4()` | Aceptar conexión entrante |
| `futex()` | Sincronización entre hilos |
| `epoll_wait()` | Esperar eventos de red o archivos |

WordPress, Apache, HAProxy, Redis y MariaDB terminan usando llamadas al sistema aunque se ejecuten dentro de contenedores.

### 3.3 eBPF

**eBPF** permite ejecutar programas verificados dentro del kernel para observar eventos del sistema. Puede engancharse a tracepoints, kprobes, uprobes, eventos de red, eventos de planificación y otros puntos de instrumentación.

En este laboratorio se usará principalmente:

- `bpftrace`: herramienta de alto nivel para escribir trazas cortas.
- BCC tools: herramientas listas como `execsnoop`, `tcpconnect`, `tcplife`, `biolatency` y `runqlat`.

### 3.4 Tracepoint, kprobe y comm

| Término | Significado |
|---|---|
| `tracepoint` | Punto estático de instrumentación definido por el kernel |
| `kprobe` | Punto dinámico sobre una función del kernel |
| `comm` | Nombre corto del proceso que generó el evento |
| `pid` | Identificador del proceso en el namespace observado |
| `uid` | Identificador de usuario asociado al proceso |

---

## 4. Advertencias de seguridad

Este laboratorio debe ejecutarse en un ambiente controlado. eBPF requiere privilegios elevados porque observa eventos del kernel y puede exponer información sensible, por ejemplo nombres de archivos, conexiones, rutas internas o parámetros de procesos.

No ejecutar estas prácticas en servidores de producción sin autorización.

---

## 5. Preparación del ambiente

Ejecutar los comandos en Ubuntu/WSL2, no dentro de los contenedores.

### 5.1 Validar sistema operativo y kernel

```bash
hostnamectl
uname -a
```

Resultado esperado aproximado:

```text
Virtualization: wsl
Operating System: Ubuntu 24.04.x LTS
Kernel: Linux 5.15.x-microsoft-standard-WSL2
Architecture: x86-64
```

### 5.2 Instalar herramientas

```bash
sudo apt update

sudo apt install -y \
  bpftrace \
  bpfcc-tools \
  bpftool \
  linux-tools-common \
  linux-perf \
  curl \
  jq \
  iproute2 \
  procps \
  sysstat \
  strace
```

### 5.3 Validar soporte básico de eBPF

```bash
sudo bpftrace --info
```

Validar si existe BTF:

```bash
ls -lh /sys/kernel/btf/vmlinux
```

Listar algunos eventos disponibles:

```bash
sudo bpftrace -l 'tracepoint:syscalls:sys_enter_*' | head
sudo bpftrace -l 'tracepoint:sched:*' | head
sudo bpftrace -l 'tracepoint:block:*' | head
```

Si aparece un error relacionado con `tracefs` o `debugfs`, intentar:

```bash
sudo mount -t tracefs nodev /sys/kernel/tracing 2>/dev/null || true
sudo mount -t debugfs nodev /sys/kernel/debug 2>/dev/null || true
```

---

## 6. Levantar y verificar el laboratorio WordPress HA

Ubicarse en el directorio del proyecto:

```bash
cd ~/grafana/grafana/redis
```

Verificar servicios:

```bash
docker compose ps
```

Levantar servicios si no están activos:

```bash
docker compose up -d
```

Verificar respuesta HTTP:

```bash
curl -I http://localhost:803/wp-login.php
```

Verificar balanceo por HAProxy:

```bash
for i in {1..10}; do
  curl -sI http://localhost:803/wp-login.php | grep -Ei 'HTTP|X-Backend'
  echo
 done
```

Resultado esperado: el encabezado `X-Backend` debería alternar entre `web1` y `web2`.

Consultar estadísticas de HAProxy:

```bash
curl -s http://localhost:8404/stats | head
```

También se puede abrir en el navegador:

```text
http://localhost:8404/stats
```

---

## 7. Práctica 1: relacionar contenedores con procesos reales

### 7.1 Obtener PID real de cada contenedor

```bash
for c in $(docker ps -q); do
  name=$(docker inspect -f '{{.Name}}' "$c" | sed 's#^/##')
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  echo "$name PID_HOST=$pid"
 done
```

### 7.2 Observar procesos en el host WSL

```bash
ps -eo pid,ppid,user,comm,args | egrep 'haproxy|apache2|mariadbd|redis-server|PID'
```

### 7.3 Observar namespaces de HAProxy

```bash
HAPROXY_PID=$(docker inspect -f '{{.State.Pid}}' wordpress-ha-lab-haproxy-1)

sudo lsns -p "$HAPROXY_PID"
```

### 7.4 Observar cgroups de HAProxy

```bash
sudo cat /proc/$HAPROXY_PID/cgroup
```

### Análisis esperado

Responder:

1. ¿El contenedor tiene un kernel propio?
2. ¿Por qué HAProxy aparece como un proceso normal del sistema?
3. ¿Qué namespaces se observan?
4. ¿Qué relación hay entre el proceso del contenedor y los cgroups?

---

## 8. Práctica 2: observar creación de procesos

### 8.1 Usando bpftrace

En una terminal:

```bash
sudo bpftrace -e '
tracepoint:sched:sched_process_fork
{
  printf("FORK parent=%s(%d) child=%s(%d)\n",
    str(args->parent_comm), args->parent_pid,
    str(args->child_comm), args->child_pid);
}

tracepoint:sched:sched_process_exec
{
  printf("EXEC pid=%d comm=%s file=%s\n",
    pid, comm, str(args->filename));
}
'
```

En otra terminal generar procesos:

```bash
docker compose exec web1 bash -lc 'php -v'
docker compose exec web1 bash -lc 'ls -la /var/www/html | head'
docker compose exec redis redis-cli ping
docker compose exec mariadb mariadb -uusuario -ppassword mi_base_de_datos -e 'SELECT NOW();'
```

Detener la traza con `Ctrl+C`.

### 8.2 Usando BCC

En una terminal:

```bash
sudo execsnoop-bpfcc
```

Si el comando no existe, buscar el nombre instalado:

```bash
command -v execsnoop-bpfcc || ls /usr/sbin/*execsnoop* /usr/share/bcc/tools/execsnoop 2>/dev/null
```

En otra terminal:

```bash
docker compose exec web2 bash -lc 'id && whoami && php -m | head'
```

### Análisis esperado

Explicar por qué `docker compose exec` genera eventos de creación y ejecución de procesos en el host.

---

## 9. Práctica 3: contar llamadas al sistema durante tráfico HTTP

En una terminal:

```bash
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_*
{
  @[comm] = count();
}

interval:s:5
{
  print(@, 20);
  clear(@);
}
'
```

En otra terminal generar tráfico:

```bash
for i in {1..100}; do
  curl -s -o /dev/null http://localhost:803/wp-login.php
 done
```

Detener con `Ctrl+C`.

### Filtrar solo Apache

```bash
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_*
/comm == "apache2"/
{
  @[probe] = count();
}

interval:s:5
{
  print(@, 20);
  clear(@);
}
'
```

Generar nuevamente tráfico:

```bash
for i in {1..50}; do
  curl -s -o /dev/null http://localhost:803/wp-login.php
 done
```

### Análisis esperado

Responder:

1. ¿Qué procesos generan más llamadas al sistema?
2. ¿Por qué aparece `curl` si la aplicación está en contenedores?
3. ¿Por qué puede aparecer `apache2`, `haproxy`, `redis-server` o `mariadbd`?
4. ¿Qué indica que un proceso tenga muchas llamadas `futex`, `epoll_wait`, `read` o `write`?

---

## 10. Práctica 4: observar apertura de archivos por Apache

En una terminal:

```bash
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_openat
/comm == "apache2"/
{
  printf("apache2 pid=%d uid=%d openat file=%s\n", pid, uid, str(args->filename));
}
'
```

En otra terminal:

```bash
curl -s -o /dev/null http://localhost:803/wp-login.php
```

Detener con `Ctrl+C`.

### Análisis esperado

Explicar qué archivos intenta abrir Apache/PHP al procesar una solicitud. Relacionar esto con:

- sistema de archivos;
- permisos;
- UID/GID;
- archivos PHP;
- configuración de WordPress;
- archivos temporales o bibliotecas compartidas.

---

## 11. Práctica 5: comparar strace y eBPF

### 11.1 strace sobre HAProxy

Obtener PID de HAProxy:

```bash
HAPROXY_PID=$(docker inspect -f '{{.State.Pid}}' wordpress-ha-lab-haproxy-1)
echo "$HAPROXY_PID"
```

Adjuntar `strace`:

```bash
sudo strace -p "$HAPROXY_PID" -f -e trace=network
```

En otra terminal:

```bash
curl -s -o /dev/null http://localhost:803/wp-login.php
```

Detener `strace` con `Ctrl+C`.

### 11.2 eBPF para observar eventos de red globales

```bash
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_accept4,
tracepoint:syscalls:sys_enter_connect,
tracepoint:syscalls:sys_enter_sendto,
tracepoint:syscalls:sys_enter_recvfrom
{
  @[comm, probe] = count();
}

interval:s:5
{
  print(@);
  clear(@);
}
'
```

Generar tráfico:

```bash
for i in {1..50}; do
  curl -s -o /dev/null http://localhost:803/wp-login.php
 done
```

### Análisis esperado

Comparar:

| Herramienta | Alcance | Ventaja | Limitación |
|---|---|---|---|
| `strace` | Un proceso o árbol de procesos | Fácil de interpretar | Puede generar mucho overhead |
| `bpftrace` | Eventos globales del kernel con filtros | Observabilidad amplia | Requiere privilegios y conocimiento del kernel |

---

## 12. Práctica 6: observar conexiones TCP

### 12.1 Usando BCC tcpconnect

En una terminal:

```bash
sudo tcpconnect-bpfcc
```

En otra terminal:

```bash
for i in {1..20}; do
  curl -s -o /dev/null http://localhost:803/wp-login.php
 done
```

Detener con `Ctrl+C`.

### 12.2 Usando BCC tcplife

En una terminal:

```bash
sudo tcplife-bpfcc
```

En otra terminal:

```bash
for i in {1..20}; do
  curl -s -o /dev/null http://localhost:803/wp-login.php
 done
```

### 12.3 Revisar red Docker

```bash
docker network ls

docker network inspect wordpress-ha-lab_webnet | jq '.[0].Containers'
```

### Análisis esperado

Explicar el recorrido de una solicitud:

```text
curl -> localhost:803 -> Docker port publishing -> HAProxy -> web1/web2 -> Redis/MariaDB si aplica
```

Responder:

1. ¿Qué conexiones TCP se observan?
2. ¿Por qué algunas conexiones parecen locales?
3. ¿Qué diferencia hay entre una conexión cliente-HAProxy y una conexión HAProxy-backend?

---

## 13. Práctica 7: probar sesiones PHP con Redis

El laboratorio tiene PHP configurado para guardar sesiones en Redis. Para verificarlo de forma explícita se creará un archivo PHP temporal.

### 13.1 Crear archivo PHP de prueba

```bash
docker compose exec -u www-data web1 sh -lc 'cat > /var/www/html/ebpf-session-test.php << "PHP"
<?php
session_start();
$_SESSION["contador"] = ($_SESSION["contador"] ?? 0) + 1;
header("Content-Type: text/plain");
echo "session_id=" . session_id() . PHP_EOL;
echo "contador=" . $_SESSION["contador"] . PHP_EOL;
echo "backend=" . gethostname() . PHP_EOL;
PHP'
```

### 13.2 Consumir la página por HAProxy

```bash
curl -i -c /tmp/ebpf-cookies.txt -b /tmp/ebpf-cookies.txt http://localhost:803/ebpf-session-test.php
curl -i -c /tmp/ebpf-cookies.txt -b /tmp/ebpf-cookies.txt http://localhost:803/ebpf-session-test.php
curl -i -c /tmp/ebpf-cookies.txt -b /tmp/ebpf-cookies.txt http://localhost:803/ebpf-session-test.php
```

### 13.3 Ver claves en Redis

```bash
docker compose exec redis redis-cli -n 1 keys 'PHPREDIS_SESSION:*'
```

Ver tamaño de la base 1:

```bash
docker compose exec redis redis-cli -n 1 DBSIZE
```

### 13.4 Observar actividad TCP mientras se usan sesiones

En una terminal:

```bash
sudo tcpconnect-bpfcc
```

En otra terminal:

```bash
for i in {1..20}; do
  curl -s -c /tmp/ebpf-cookies.txt -b /tmp/ebpf-cookies.txt http://localhost:803/ebpf-session-test.php >/dev/null
 done
```

### 13.5 Limpiar archivo de prueba

```bash
docker compose exec -u www-data web1 rm -f /var/www/html/ebpf-session-test.php
rm -f /tmp/ebpf-cookies.txt
```

### Análisis esperado

Responder:

1. ¿Por qué Redis permite que la sesión sobreviva aunque HAProxy envíe solicitudes a `web1` o `web2`?
2. ¿Qué problema habría si las sesiones se guardaran solo en archivos locales dentro de cada contenedor web?
3. ¿Qué relación tiene esto con alta disponibilidad?

---

## 14. Práctica 8: observar latencia de disco

En este laboratorio hay I/O persistente en:

- `mariadb_data`, usado por MariaDB.
- `redis_data`, usado por Redis con AOF y snapshots.
- `/home/orh`, montado como `/var/www/html` en los contenedores WordPress.

### 14.1 Observar latencia de bloque con BCC

En una terminal:

```bash
sudo biolatency-bpfcc 1
```

En otra terminal provocar escritura en WordPress:

```bash
docker compose exec -u www-data web1 sh -lc '
for i in $(seq 1 1000); do
  echo "linea $i $(date +%s%N)" >> /var/www/html/ebpf-lab-write.txt
 done
sync
'
```

Provocar escritura en Redis:

```bash
docker compose exec redis sh -lc '
redis-cli SET ebpf:lab "$(date)"
redis-cli BGSAVE
'
```

Provocar escritura en MariaDB:

```bash
docker compose exec mariadb mariadb -uroot -prootpassword -e "
CREATE DATABASE IF NOT EXISTS ebpf_lab;
USE ebpf_lab;
CREATE TABLE IF NOT EXISTS t (
  id INT PRIMARY KEY AUTO_INCREMENT,
  valor VARCHAR(200),
  creado TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO t(valor)
SELECT CONCAT('fila-', RAND())
FROM information_schema.columns
LIMIT 500;
"
```

### 14.2 Limpiar datos de prueba

```bash
docker compose exec -u www-data web1 rm -f /var/www/html/ebpf-lab-write.txt

docker compose exec mariadb mariadb -uroot -prootpassword -e "DROP DATABASE IF EXISTS ebpf_lab;"

docker compose exec redis redis-cli DEL ebpf:lab
```

### Análisis esperado

Responder:

1. ¿Cuál operación produjo más actividad de bloque?
2. ¿La latencia de disco observada corresponde únicamente al contenedor?
3. En WSL2, ¿qué capas adicionales podrían influir en la latencia?

---

## 15. Práctica 9: observar planificación de CPU

### 15.1 Observar latencia de cola de ejecución

En una terminal:

```bash
sudo runqlat-bpfcc 1
```

En otra terminal generar carga CPU dentro de `web1`:

```bash
docker compose exec web1 bash -lc '
php -r "
\$t = time() + 10;
while (time() < \$t) {
  hash(\"sha256\", random_bytes(1024));
}
"
'
```

### 15.2 Observar cambios de contexto

```bash
sudo bpftrace -e '
tracepoint:sched:sched_switch
{
  @[comm] = count();
}

interval:s:5
{
  print(@, 20);
  clear(@);
}
'
```

Generar nuevamente carga CPU:

```bash
docker compose exec web2 bash -lc '
php -r "
\$t = time() + 10;
while (time() < \$t) {
  hash(\"sha256\", random_bytes(1024));
}
"
'
```

### Análisis esperado

Responder:

1. ¿El planificador agenda contenedores o procesos/hilos?
2. ¿Qué significa una mayor latencia en la cola de ejecución?
3. ¿Cómo se relaciona esto con CPU-bound, I/O-bound y tiempo de respuesta?

---

## 16. Práctica 10: validar UID/GID y permisos de WordPress

El contenedor ajusta el usuario interno `www-data` para que use el UID/GID definidos por el ambiente. Esto permite que los archivos montados en `/var/www/html` correspondan al usuario esperado del host.

### 16.1 Revisar UID/GID dentro del contenedor

```bash
docker compose exec web1 id www-data
docker compose exec web2 id www-data
```

### 16.2 Ver procesos Apache

```bash
docker compose exec web1 bash -lc 'ps -eo pid,user,group,comm | grep apache2'
```

### 16.3 Ver permisos del directorio WordPress

```bash
docker compose exec web1 bash -lc 'stat -c "%U %G %a %n" /var/www/html'
```

### Análisis esperado

Responder:

1. ¿Por qué conviene evitar que los archivos queden propiedad de `root`?
2. ¿Qué problema se busca resolver al hacer coincidir UID/GID del contenedor con el host?
3. ¿Por qué no conviene hacer `chown -R` automático sobre todo `/var/www/html` cada vez que inicia el contenedor?

---

## 17. Comandos de cierre del laboratorio

Verificar que los servicios sigan activos:

```bash
docker compose ps
```

Verificar WordPress:

```bash
curl -I http://localhost:803/wp-login.php
```

Verificar HAProxy stats:

```bash
curl -s http://localhost:8404/stats | head
```

Detener el laboratorio al finalizar:

```bash
docker compose down
```

Si se desea eliminar volúmenes de datos de prueba:

```bash
docker compose down -v
```

> Cuidado: `docker compose down -v` elimina volúmenes como `mariadb_data` y `redis_data`.

---

## 18. Problemas frecuentes

### 18.1 `bpftrace: command not found`

Instalar herramientas:

```bash
sudo apt update
sudo apt install -y bpftrace bpfcc-tools bpftool
```

### 18.2 `tracepoint not found`

Listar eventos disponibles:

```bash
sudo bpftrace -l 'tracepoint:*' | head
```

No todos los kernels exponen exactamente los mismos eventos.

### 18.3 Error de permisos

Usar `sudo`:

```bash
sudo bpftrace --info
```

### 18.4 Herramientas BCC con nombres diferentes

Buscar herramientas instaladas:

```bash
ls /usr/sbin/*bpfcc | head
ls /usr/share/bcc/tools 2>/dev/null | head
```

### 18.5 Redis no muestra claves de sesión

WordPress por sí solo no siempre usa sesiones PHP. Por eso este laboratorio crea `ebpf-session-test.php` para forzar `session_start()` y validar Redis explícitamente.

### 18.6 MariaDB rechaza credenciales

Para este laboratorio se usan estas credenciales de prueba:

```text
Usuario root: root / rootpassword
Base: mi_base_de_datos
Usuario aplicación: usuario / password
```

---

## 19. Entregable sugerido

Cada estudiante o grupo debe entregar un breve informe con:

1. Captura o salida de `docker compose ps`.
2. PID real de cada contenedor.
3. Salida parcial de `execsnoop` o `bpftrace` mostrando creación de procesos.
4. Conteo de llamadas al sistema durante tráfico HTTP.
5. Evidencia de conexiones TCP con `tcpconnect` o `tcplife`.
6. Evidencia de sesiones PHP almacenadas en Redis.
7. Evidencia de latencia de disco con `biolatency`.
8. Evidencia de planificación con `runqlat` o `sched_switch`.
9. Respuestas a las preguntas de análisis de cada práctica.

---

## 20. Preguntas finales de reflexión

1. ¿Qué diferencia hay entre observar una aplicación desde sus logs y observarla desde el kernel?
2. ¿Por qué eBPF es útil en ambientes con contenedores?
3. ¿Qué eventos del laboratorio corresponden a procesos?
4. ¿Qué eventos corresponden a red TCP/IP?
5. ¿Qué eventos corresponden a almacenamiento?
6. ¿Qué eventos corresponden al planificador de CPU?
7. ¿Qué información sensible podría exponerse mediante trazas eBPF?
8. ¿Cómo se relaciona este laboratorio con los temas de procesos, memoria, I/O, red y seguridad del sistema operativo?

---

## 21. Referencias de consulta

- Documentación de bpftrace: https://bpftrace.org/docs/
- Documentación del verificador eBPF del kernel Linux: https://docs.kernel.org/bpf/verifier.html
- Proyecto BCC: https://github.com/iovisor/bcc
- Documentación de Docker sobre contenedores: https://docs.docker.com/
- Documentación de HAProxy: https://www.haproxy.org/#docs
- Documentación de Redis: https://redis.io/docs/latest/
- Documentación de MariaDB: https://mariadb.com/kb/en/documentation/
