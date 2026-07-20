# Propuestas de proyectos de plataforma de Inteligencia Artificial

**Curso:** Sistemas Operativos Avanzados  
**Enfoque:** infraestructura, contenedores, procesos, red, almacenamiento, observabilidad, seguridad y planificación de recursos  
**Nivel:** posterior a Sistemas Operativos I  

---

## 1. Propósito general

Estas propuestas buscan que los estudiantes comprendan que una plataforma de Inteligencia Artificial no consiste únicamente en ejecutar un modelo. Detrás de una solución de IA existen procesos, contenedores, servicios de red, almacenamiento, control de recursos, seguridad, observabilidad y mecanismos de tolerancia a fallos.

Desde la perspectiva de Sistemas Operativos Avanzados, la IA se convierte en un caso moderno para estudiar cómo el sistema operativo y las plataformas distribuidas permiten ejecutar cargas intensivas de cómputo de forma controlada.

---

## 2. Relación con Sistemas Operativos Avanzados

Los proyectos permiten trabajar conceptos como:

- procesos e hilos;
- llamadas al sistema;
- planificación de CPU;
- uso de memoria RAM y, si aplica, GPU/VRAM;
- contenedores Linux;
- namespaces y cgroups;
- redes virtuales;
- almacenamiento persistente;
- balanceo de carga;
- observabilidad;
- seguridad de servicios;
- automatización de despliegues;
- tolerancia a fallos.

La idea principal para los estudiantes es:

```text
IA visible:
    chat, predicción, clasificación, generación de texto o inferencia

Infraestructura invisible:
    procesos
    memoria
    red
    disco
    contenedores
    scheduler
    seguridad
    observabilidad
    persistencia
```

---

# Proyecto 1: Plataforma local de modelos LLM con Open WebUI + Ollama

## Descripción

Diseñar e implementar una plataforma local donde los usuarios puedan interactuar con un modelo de lenguaje liviano desde una interfaz web. El objetivo es observar cómo una carga de IA consume recursos del sistema operativo y cómo se comporta cuando se ejecuta dentro de contenedores.

Este proyecto es adecuado para laboratorios sin GPU, usando modelos pequeños que puedan ejecutarse en CPU.

## Arquitectura propuesta

```text
Usuario
  │
  ▼
Open WebUI :3000
  │
  ▼
Ollama :11434
  │
  ▼
Modelo local
  │
  ▼
CPU / RAM / disco
```

## Componentes mínimos

- Docker o Docker Compose.
- Contenedor de Open WebUI.
- Contenedor de Ollama.
- Volumen persistente para modelos.
- Volumen persistente para datos de Open WebUI.
- Modelo liviano descargado localmente.
- Herramientas de observabilidad básica: `docker stats`, `ps`, `top`, `htop`, `free`, `du`, `curl`.

## Posibles extensiones

- Agregar HAProxy, Nginx o Traefik como reverse proxy.
- Agregar autenticación básica.
- Agregar Prometheus y Grafana.
- Agregar límites de CPU y memoria en Docker Compose.
- Comparar el comportamiento con diferentes modelos.

## Relación con Sistemas Operativos

Este proyecto permite estudiar:

- ejecución de procesos dentro de contenedores;
- consumo de CPU durante inferencia;
- consumo de memoria al cargar un modelo;
- persistencia de modelos en volúmenes;
- publicación de puertos;
- comunicación entre contenedores;
- límites de recursos con cgroups;
- diferencia entre aplicación, contenedor y proceso real del sistema.

## Experimentos sugeridos

### 1. Medir consumo de recursos durante inferencia

Ejecutar una consulta desde Open WebUI y observar:

```bash
docker stats
```

También se puede revisar desde el host:

```bash
ps -eo pid,ppid,comm,%cpu,%mem,args | grep -Ei 'ollama|open-webui'
free -h
```

### 2. Limitar memoria del contenedor

Modificar el servicio de Ollama para limitar memoria y observar el comportamiento.

Ejemplo conceptual:

```yaml
deploy:
  resources:
    limits:
      memory: 4g
```

En Docker Compose sin Swarm también se puede usar:

```yaml
mem_limit: 4g
```

### 3. Verificar persistencia del modelo

Detener y levantar nuevamente los contenedores:

```bash
docker compose down
docker compose up -d
```

Comprobar si el modelo sigue disponible sin descargarlo nuevamente.

## Entregables

Cada grupo debe entregar:

1. `compose.yaml` funcional.
2. Evidencia de Open WebUI funcionando.
3. Modelo utilizado y justificación de selección.
4. Medición de CPU y memoria durante una consulta.
5. Explicación de dónde se guardan los modelos.
6. Análisis de qué ocurre si se limita CPU o memoria.
7. Diagrama de arquitectura.

## Pregunta central

> ¿Qué recursos del sistema operativo consume un modelo local cuando responde una consulta?

---

# Proyecto 2: Plataforma MLOps mínima con MLflow + PostgreSQL + MinIO

## Descripción

Construir una plataforma mínima para registrar experimentos de aprendizaje automático, guardar métricas, almacenar artefactos, versionar modelos y consultar resultados desde una interfaz web.

El objetivo no es entrenar un modelo complejo, sino comprender la infraestructura necesaria para administrar el ciclo de vida de un modelo.

## Arquitectura propuesta

```text
Script / Notebook de entrenamiento
        │
        ▼
MLflow Tracking Server
        │
        ├── PostgreSQL
        │     metadatos, experimentos, parámetros, métricas
        │
        └── MinIO / S3
              artefactos, modelos, archivos generados
```

## Componentes mínimos

- MLflow Tracking Server.
- PostgreSQL como backend de metadatos.
- MinIO como almacenamiento de objetos compatible con S3.
- Script Python de entrenamiento.
- Modelo simple, por ejemplo:
  - clasificación Iris;
  - regresión básica;
  - clasificación de texto;
  - clasificación de tiquetes sintéticos;
  - dataset pequeño de prueba.

## Posibles extensiones

- Agregar Jupyter Notebook.
- Agregar autenticación básica frente a MLflow.
- Agregar versionamiento de modelos.
- Agregar una API de inferencia que cargue el modelo desde MLflow o MinIO.
- Agregar Prometheus/Grafana para monitoreo.
- Agregar respaldo de PostgreSQL y MinIO.

## Relación con Sistemas Operativos

Este proyecto permite estudiar:

- servicios persistentes;
- almacenamiento de metadatos;
- almacenamiento de objetos;
- volúmenes Docker;
- red entre contenedores;
- variables de entorno;
- credenciales;
- procesos batch;
- permisos sobre archivos;
- reproducibilidad;
- separación entre datos, metadatos y artefactos;
- recuperación ante fallos.

## Experimentos sugeridos

### 1. Registrar un experimento

Ejecutar un script que registre:

- parámetros;
- métricas;
- artefactos;
- modelo entrenado.

Ejemplo conceptual:

```python
import mlflow

with mlflow.start_run():
    mlflow.log_param("modelo", "baseline")
    mlflow.log_metric("accuracy", 0.91)
    mlflow.log_artifact("reporte.txt")
```

### 2. Verificar almacenamiento en PostgreSQL y MinIO

Analizar qué información queda en la base de datos y qué información queda como objeto en MinIO.

Preguntas:

- ¿Dónde se guarda el nombre del experimento?
- ¿Dónde se guarda el archivo del modelo?
- ¿Dónde se guardan las métricas?
- ¿Qué ocurre si se borra el volumen de MinIO?
- ¿Qué ocurre si se borra la base PostgreSQL?

### 3. Simular fallo de un componente

Detener PostgreSQL:

```bash
docker compose stop postgres
```

Observar qué ocurre con MLflow.

Luego levantarlo de nuevo:

```bash
docker compose start postgres
```

Repetir el experimento con MinIO.

## Entregables

Cada grupo debe entregar:

1. `compose.yaml` funcional.
2. Script de entrenamiento.
3. Captura de la interfaz de MLflow con al menos un experimento.
4. Evidencia del modelo o artefactos en MinIO.
5. Explicación de la función de PostgreSQL.
6. Explicación de la función de MinIO.
7. Análisis de fallo de PostgreSQL o MinIO.
8. Diagrama de arquitectura.

## Pregunta central

> ¿Por qué una plataforma de IA necesita base de datos, almacenamiento de objetos y control de versiones, además del modelo?

---

# Proyecto 3: Plataforma de inferencia escalable con Kubernetes

## Descripción

Implementar un servicio de inferencia desplegado en Kubernetes, con réplicas, balanceo de carga, health checks, límites de recursos y pruebas de tolerancia a fallos.

El modelo puede ser simple. Lo importante es estudiar cómo Kubernetes administra la ejecución, disponibilidad y escalamiento del servicio.

## Arquitectura propuesta

```text
Cliente
  │
  ▼
Ingress / NodePort
  │
  ▼
Service Kubernetes
  │
  ▼
Deployment con varias réplicas
  │
  ▼
FastAPI + modelo de inferencia
```

## Componentes mínimos

- Cluster local con `kind`, `k3d`, MicroK8s o Minikube.
- Imagen Docker de la API de inferencia.
- `Deployment`.
- `Service`.
- `ConfigMap`.
- `Secret`, si se usan credenciales.
- Límites de CPU y memoria.
- Readiness probe.
- Liveness probe.

## Variante simple

Crear una API con FastAPI que cargue un modelo pequeño o simule una inferencia.

Ejemplo de endpoints:

```text
GET /health
POST /predict
```

## Variante avanzada

Usar KServe o una plataforma similar para desplegar el modelo como recurso de inferencia. Esta variante es más cercana a una arquitectura de producción, pero requiere más componentes y mayor tiempo de configuración.

## Relación con Sistemas Operativos

Este proyecto permite estudiar:

- planificación de Pods;
- reinicio automático;
- balanceo de carga;
- límites con cgroups;
- namespaces de Kubernetes;
- red virtual del cluster;
- servicios;
- health checks;
- escalamiento horizontal;
- tolerancia a fallos;
- observabilidad;
- diferencia entre Docker Compose y Kubernetes.

## Experimentos sugeridos

### 1. Crear varias réplicas

Ejemplo:

```bash
kubectl scale deployment ia-inferencia --replicas=3
```

Validar:

```bash
kubectl get pods -o wide
kubectl get svc
```

### 2. Generar tráfico

```bash
for i in {1..30}; do
  curl -s http://localhost:8080/predict
  echo
done
```

Analizar si las respuestas provienen de distintas réplicas.

### 3. Simular fallo de una réplica

Eliminar un Pod:

```bash
kubectl delete pod <nombre-del-pod>
```

Observar cómo Kubernetes crea uno nuevo:

```bash
kubectl get pods -w
```

### 4. Aplicar límites de recursos

Ejemplo conceptual:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Analizar qué ocurre si la aplicación supera memoria o CPU.

## Entregables

Cada grupo debe entregar:

1. Código de la API de inferencia.
2. Dockerfile.
3. Manifiestos YAML de Kubernetes.
4. Evidencia del servicio funcionando.
5. Evidencia de varias réplicas.
6. Prueba de eliminación de un Pod y recuperación automática.
7. Prueba de límites de recursos.
8. Diagrama de arquitectura.
9. Comparación breve entre Docker Compose y Kubernetes.

## Pregunta central

> ¿Qué aporta Kubernetes sobre Docker Compose cuando una API de IA debe ejecutarse de forma confiable y escalable?

---

# 4. Comparación de los proyectos

| Proyecto | Complejidad | Requiere GPU | Tema principal |
|---|---:|---:|---|
| Open WebUI + Ollama | Baja-media | No | Inferencia local y consumo de recursos |
| MLflow + PostgreSQL + MinIO | Media | No | Ciclo de vida y almacenamiento de modelos |
| Kubernetes + API de inferencia | Media-alta | No | Escalabilidad, scheduling y tolerancia a fallos |

---

# 5. Recomendación de asignación

## Ruta inicial

Para estudiantes que apenas inician con plataformas de IA:

```text
Proyecto 1: Open WebUI + Ollama
```

Motivo: permite ver rápidamente un sistema funcionando y facilita discutir CPU, memoria, procesos y persistencia.

## Ruta de ingeniería de plataforma

Para estudiantes con mayor interés en infraestructura:

```text
Proyecto 2: MLflow + PostgreSQL + MinIO
```

Motivo: muestra que una plataforma de IA requiere metadatos, artefactos, versionamiento y almacenamiento persistente.

## Ruta avanzada

Para proyecto final o estudiantes con experiencia previa en contenedores:

```text
Proyecto 3: Kubernetes + API de inferencia
```

Motivo: conecta IA con scheduling, réplicas, tolerancia a fallos, red y administración de recursos.

---

# 6. Criterios generales de evaluación

Se sugiere evaluar cada proyecto con una rúbrica común.

| Criterio | Peso sugerido |
|---|---:|
| Arquitectura y diseño | 20% |
| Implementación funcional | 25% |
| Relación con conceptos de Sistemas Operativos | 25% |
| Observabilidad y medición de recursos | 15% |
| Documentación y claridad del informe | 15% |

---

# 7. Preguntas finales para discusión

1. ¿Qué procesos principales ejecuta la plataforma?
2. ¿Qué recursos del sistema operativo consume?
3. ¿Dónde se almacenan los datos persistentes?
4. ¿Qué ocurre si se reinicia un contenedor?
5. ¿Qué ocurre si se limita memoria o CPU?
6. ¿Qué componente representa mayor riesgo de seguridad?
7. ¿Qué logs o métricas permiten diagnosticar fallos?
8. ¿Qué diferencia hay entre ejecutar una IA como script local y ejecutarla como plataforma?
9. ¿Qué parte del proyecto se relaciona con planificación de CPU?
10. ¿Qué parte se relaciona con almacenamiento?
11. ¿Qué parte se relaciona con redes?
12. ¿Qué parte se relaciona con seguridad?

---

# 8. Referencias de consulta

- Open WebUI: https://docs.openwebui.com/
- Ollama: https://ollama.com/
- MLflow: https://mlflow.org/docs/latest/
- MinIO: https://min.io/docs/minio/container/index.html
- PostgreSQL: https://www.postgresql.org/docs/
- Docker Compose: https://docs.docker.com/compose/
- Kubernetes: https://kubernetes.io/docs/
- FastAPI: https://fastapi.tiangolo.com/
- KServe: https://kserve.github.io/website/
