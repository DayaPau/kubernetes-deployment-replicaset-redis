# Despliegue de Redis con Kubernetes: Deployment y ReplicaSet

**Práctica individual — Procesos de Software**

## 1. Descripción

Esta práctica despliega **Redis** en un clúster local de Kubernetes gestionado con **Minikube** y Docker Desktop. Se utiliza un recurso **Deployment**, que administra un **ReplicaSet** encargado de mantener la cantidad deseada de **Pods**. Se comprueban el despliegue inicial, el escalamiento horizontal, la respuesta de Redis y la autorrecuperación ante la eliminación de un Pod.

> **Alcance:** demostración académica de la gestión de réplicas de Kubernetes. Las instancias Redis son independientes: **tres Pods no implican replicación de datos ni alta disponibilidad de Redis**.

## 2. Objetivos

- Desplegar una plataforma distinta de Nginx mediante un Deployment.
- Comprender la relación entre **Deployment → ReplicaSet → Pods → contenedores**.
- Escalar de una a tres réplicas y verificar su estado.
- Eliminar un Pod administrado para observar su reemplazo automático.
- Verificar que Redis responde al comando `PING`.

## 3. Conceptos clave

| Recurso | Función |
|---|---|
| **Pod** | Unidad básica de ejecución de Kubernetes; en esta práctica, contiene un contenedor Redis. |
| **ReplicaSet** | Procura mantener en ejecución la cantidad deseada de Pods definidos por una plantilla. |
| **Deployment** | Declara el estado deseado de la aplicación y administra sus ReplicaSets, escalamiento y actualizaciones. |
| **kubectl** | Cliente de línea de comandos que se comunica con la API de Kubernetes. |
| **Minikube** | Ejecuta un clúster Kubernetes local para aprendizaje y desarrollo. |

```text
redis-deployment (Deployment: 3 réplicas)
└── redis-deployment-569d4bd869 (ReplicaSet)
    ├── Pod Redis A
    ├── Pod Redis B
    └── Pod Redis C
```

*Los nombres concretos de los ReplicaSets y Pods se generan automáticamente y pueden variar entre ejecuciones.*

## 4. ¿Qué es Redis y para qué sirve?

**Redis** es un almacén de datos en memoria que puede utilizarse, entre otros casos, para:

- **Caché:** guardar resultados consultados frecuentemente y reducir consultas repetidas a una base de datos principal.
- **Sesiones:** almacenar información temporal asociada a sesiones de aplicaciones.
- **Contadores y límites de solicitudes:** mantener contadores rápidos y apoyar mecanismos de *rate limiting*.
- **Mensajería y colas:** coordinar procesamiento de trabajos mediante funcionalidades adecuadas de Redis.

**Ejemplo:** una aplicación de gestión agrícola puede guardar temporalmente en Redis indicadores consultados frecuentemente, en lugar de recalcularlos desde la base de datos ante cada solicitud.

## 5. Entorno utilizado

- Windows 11 y PowerShell.
- Docker Desktop con contenedores Linux y WSL2.
- Minikube con driver Docker y perfil `minikube-docker`.
- `kubectl` para administrar el clúster.
- Imagen de contenedor `redis:7-alpine`.

> Las versiones concretas instaladas en otra computadora pueden ser diferentes. Debe existir espacio de almacenamiento libre suficiente para Docker y Minikube.

## 6. Estructura del repositorio

```text
kubernetes-redis-deployment-replicaset/
├── redis-deployment.yml
├── README.md
├── evidencias/
│   ├── 01-deployment-inicial.png
│   ├── 02-replicaset-inicial.png
│   ├── 03-redis-pong.png
│   ├── 04-escalamiento.png
│   ├── 05-autorrecuperacion.png
│   └── 06-estado-final.png

```


## 7. Configuración: `redis-deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  labels:
    app: redis
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis-container
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "256Mi"
              cpu: "250m"
```

**Explicación:** `replicas` establece el número deseado de Pods; `selector.matchLabels` debe coincidir con las etiquetas en `template.metadata.labels`; `template` describe el contenedor; `requests` reserva recursos para planificar los Pods y `limits` impone límites de consumo. El puerto 6379 es el puerto habitual de Redis; `containerPort` **no expone** Redis fuera del clúster.

## 8. Pasos para reproducir

Los comandos se ejecutan en **PowerShell**. Se recomienda trabajar con el perfil de Minikube utilizado en la práctica.

### 8.1 Iniciar y verificar Kubernetes

Con Docker Desktop funcionando:

```powershell
minikube start -p minikube-docker --driver=docker
minikube -p minikube-docker status
kubectl --context=minikube-docker get nodes
```

El nodo debe aparecer en estado `Ready` y el servidor API de Minikube en `Running`.

### 8.2 Crear el Deployment

Desde la carpeta que contiene `redis-deployment.yml`:

```powershell
kubectl --context=minikube-docker apply -f redis-deployment.yml
kubectl --context=minikube-docker rollout status deployment/redis-deployment --timeout=120s
kubectl --context=minikube-docker get deployment,replicaset,pods -l app=redis
```

Con la configuración final, el resultado esperado es un Deployment con `3/3` réplicas listas, un ReplicaSet con `DESIRED=3`, `CURRENT=3`, `READY=3` y tres Pods `Running` con `READY=1/1`.

### 8.3 Verificar el funcionamiento de Redis

```powershell
kubectl --context=minikube-docker exec deployment/redis-deployment -- redis-cli ping
```

Resultado esperado y comprobado durante la práctica:

```text
PONG
```

Esta prueba comprueba la respuesta de una de las instancias, no de todas simultáneamente.

### 8.4 Demostrar escalamiento horizontal

La prueba original empezó con `replicas: 1` y luego ejecutó:

```powershell
kubectl --context=minikube-docker scale deployment redis-deployment --replicas=3
kubectl --context=minikube-docker rollout status deployment/redis-deployment --timeout=120s
kubectl --context=minikube-docker get replicasets
kubectl --context=minikube-docker get pods -l app=redis
```

**Resultado observado:** tres Pods Redis `1/1 Running`. Para repetir toda la prueba de escalamiento desde el estado final, puedes escalar temporalmente a una réplica y regresar a tres:

```powershell
kubectl --context=minikube-docker scale deployment redis-deployment --replicas=1
kubectl --context=minikube-docker rollout status deployment/redis-deployment --timeout=120s
kubectl --context=minikube-docker scale deployment redis-deployment --replicas=3
kubectl --context=minikube-docker rollout status deployment/redis-deployment --timeout=120s
```

> `kubectl scale` modifica el estado del clúster, **no el archivo YAML**. Este repositorio conserva `replicas: 3` como configuración final declarada.

### 8.5 Demostrar autorrecuperación

Consultar primero los Pods:

```powershell
kubectl --context=minikube-docker get pods -l app=redis
```

Elegir uno de los nombres mostrados y eliminar **solo ese Pod**:

```powershell
kubectl --context=minikube-docker delete pod <NOMBRE_DEL_POD>
```

`<NOMBRE_DEL_POD>` es un marcador que debe sustituirse por el nombre real antes de ejecutar el comando. Después observar los cambios:

```powershell
kubectl --context=minikube-docker get pods -l app=redis -w
```

Presionar `Ctrl + C` para detener la observación y verificar:

```powershell
kubectl --context=minikube-docker get replicasets
kubectl --context=minikube-docker get pods -l app=redis
```

**Resultado observado en la práctica:** al eliminar `redis-deployment-569d4bd869-8ts8m`, Kubernetes creó `redis-deployment-569d4bd869-mhh2h`, recuperando tres Pods `1/1 Running`.

### 8.6 Mantener consistente el archivo YAML

Después de las pruebas, confirmar que `redis-deployment.yml` define `replicas: 3` y aplicarlo:

```powershell
kubectl --context=minikube-docker apply -f redis-deployment.yml
kubectl --context=minikube-docker get deployment,replicaset,pods -l app=redis
```

**Resultado observado:** Deployment `3/3`, ReplicaSet con tres réplicas listas y tres Pods `Running` sin reinicios.

## 9. Evidencias para el informe

| Evidencia | Qué demuestra |
|---|---|
| Deployment inicial (`1/1`) | Creación y disponibilidad inicial. |
| ReplicaSet inicial | Creación automática del controlador de réplicas. |
| Respuesta `PONG` | Funcionamiento básico de Redis. |
| Tres Pods `Running` | Escalamiento horizontal. |
| Nuevo Pod tras eliminar otro | Autorrecuperación por ReplicaSet. |
| Estado final `3/3` | Concordancia entre Deployment, ReplicaSet y Pods. |

Las capturas se incorporan en `evidencias/` antes de la entrega.

## 10. Limitaciones y consideraciones

- Los tres Pods ejecutan **instancias independientes de Redis**. Kubernetes ReplicaSet no sincroniza sus datos.
- No se configuraron Redis Sentinel, Redis Cluster, almacenamiento persistente ni mecanismos de respaldo. Esta implementación no está diseñada para producción ni para garantizar durabilidad o alta disponibilidad de datos.
- No se creó un `Service` para conectar clientes externos: la verificación se realizó con `kubectl exec` dentro del contenedor.
- El nombre de imagen `redis:7-alpine` permite actualizaciones dentro de esa etiqueta. Para un despliegue reproducible de producción conviene fijar una versión exacta o digest y aplicar controles de seguridad adicionales.
- No se requiere exponer Redis a Internet para esta práctica.

## 11. Conclusión

La práctica permitió desplegar Redis mediante un Deployment, observar la creación automática de un ReplicaSet y demostrar su capacidad para mantener tres Pods en ejecución. Se verificó la respuesta básica de Redis con `PING`/`PONG`, el escalamiento horizontal y el reemplazo de un Pod eliminado. La experiencia evidencia la diferencia entre ejecutar un Pod individual y administrar el estado deseado mediante controladores de Kubernetes.

## 12. Referencias

- [Documentación oficial de Kubernetes: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Documentación oficial de Kubernetes: ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Documentación oficial de Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [Documentación de Redis](https://redis.io/docs/latest/)
