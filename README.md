# Taller Kubernetes: Pod, Deployment y ReplicaSet con Grafana

Taller individual de Procesos de Software (Configuración de cluster). Se despliega primero un Pod de nginx y luego **Grafana** (plataforma no vista en clase) con un **Deployment** que administra un **ReplicaSet**, y se expone con un Service de tipo NodePort.

## Entorno

| Elemento | Valor |
|---|---|
| Sistema | Windows 11 |
| Clúster | Minikube 1.39.0 con el driver Docker (Docker Desktop 29.8.0) |
| Kubernetes | v1.37.0, un nodo (`minikube`) |
| kubectl | v1.36.1 |

```powershell
winget install Kubernetes.minikube
minikube start --driver=docker --cpus=2 --memory=3072
kubectl get nodes
```

## Estructura del repositorio

```
01-pod/first_pod_nginx.yml            Pod de nginx (primer Pod)
02-deployment/grafana-deployment.yml  Deployment de Grafana (3 réplicas)
02-deployment/grafana-service.yml     Service NodePort de Grafana
evidencias/                           Salidas de los comandos y capturas
```

## Parte 1: el primer Pod (nginx)

Un **Pod** es la unidad mínima de despliegue de Kubernetes: uno o más contenedores que comparten red y almacenamiento.

```powershell
kubectl apply -f 01-pod/first_pod_nginx.yml
kubectl get pods -o wide
kubectl expose pod myfirstpod --type=NodePort --port=80
kubectl get svc myfirstpod
minikube service myfirstpod --url     # túnel necesario con el driver Docker en Windows
```

Resultado: `myfirstpod` en `1/1 Running` y servicio `80:32197/TCP`. Evidencia: [`evidencias/01-pod.txt`](evidencias/01-pod.txt) y [captura del navegador](evidencias/captura-01-pod-nginx-navegador.jpg).

Un Pod suelto tiene una limitación: si se borra o su nodo falla, nadie lo vuelve a crear. Por eso se usan ReplicaSet y Deployment.

## Parte 2: Deployment y ReplicaSet con Grafana

### Los conceptos

| Objeto | Qué hace |
|---|---|
| **ReplicaSet** | Garantiza que siempre haya N Pods idénticos corriendo. Si uno muere, crea otro. Usa una etiqueta (`selector`) para saber qué Pods le pertenecen. |
| **Deployment** | Administra ReplicaSets. Añade actualizaciones graduales (*rolling update*), historial de versiones y *rollback*. Es lo que se usa en la práctica; el ReplicaSet se crea solo. |

Relación: `Deployment` → crea y gestiona → `ReplicaSet` → crea y vigila → `Pods`.

### La configuración

[`02-deployment/grafana-deployment.yml`](02-deployment/grafana-deployment.yml) define `replicas: 3`, la estrategia `RollingUpdate` (máximo 1 Pod no disponible y 1 extra durante el cambio), una *readiness probe* sobre `/api/health` y límites de CPU y memoria. [`grafana-service.yml`](02-deployment/grafana-service.yml) expone el puerto 3000 mediante NodePort y reparte el tráfico entre los Pods con la etiqueta `app=grafana`.

```powershell
kubectl apply -f 02-deployment/grafana-deployment.yml
kubectl apply -f 02-deployment/grafana-service.yml
kubectl rollout status deployment/grafana
kubectl get deployments,replicasets,pods
```

Resultado: `deployment grafana 3/3`, ReplicaSet `grafana-84bd96d55c` creado automáticamente con 3 réplicas. Evidencia: [`evidencias/02-deployment.txt`](evidencias/02-deployment.txt).

### Exposición de la plataforma

```powershell
kubectl get svc grafana               # 3000:32391/TCP
minikube service grafana --url        # abre el túnel; usa la URL que imprime
```

Grafana responde en el navegador con la pantalla de inicio de sesión (versión 11.2.0): [captura](evidencias/captura-02-grafana-navegador.jpg). Las credenciales por defecto de Grafana (`admin` / `admin`) se usan solo en este laboratorio local y no se guardan en el repositorio.

### Pruebas de las funcionalidades

Salida completa en [`evidencias/03-replicaset-demo.txt`](evidencias/03-replicaset-demo.txt) y [`evidencias/04-rolling-update.txt`](evidencias/04-rolling-update.txt).

1. **Autorrecuperación (ReplicaSet).** Se borró el Pod `grafana-84bd96d55c-2m4bv`. El ReplicaSet creó de inmediato un reemplazo (`grafana-84bd96d55c-n2gf9`) y volvió a haber 3 Pods.
   ```powershell
   kubectl delete pod <nombre-del-pod>
   kubectl get pods -l app=grafana
   ```
2. **Escalado.** De 3 a 5 réplicas y de vuelta a 3, sin tocar el YAML.
   ```powershell
   kubectl scale deployment grafana --replicas=5
   kubectl scale deployment grafana --replicas=3
   ```
3. **Actualización gradual (Deployment).** Se cambió la imagen de `11.2.0` a `11.3.0`. El Deployment creó un segundo ReplicaSet (`grafana-79b856fd96`), subió sus Pods de a uno mientras bajaba los del anterior, y siempre hubo Pods disponibles. El ReplicaSet viejo quedó con 0 réplicas, pero se conserva para poder volver atrás.
   ```powershell
   kubectl set image deployment/grafana grafana=grafana/grafana-oss:11.3.0
   kubectl rollout status deployment/grafana
   kubectl get replicaset
   ```
4. **Rollback.** Con un comando se volvió a la versión 11.2.0, reactivando el ReplicaSet anterior.
   ```powershell
   kubectl rollout history deployment/grafana
   kubectl rollout undo deployment/grafana
   ```

Estado final: [`evidencias/05-estado-final.txt`](evidencias/05-estado-final.txt).

## Usos

**Del Deployment y el ReplicaSet**

- **Alta disponibilidad:** varias réplicas atienden al mismo tiempo; si un Pod o un nodo falla, se reemplaza solo.
- **Escalado:** subir o bajar réplicas según la carga, a mano o con un autoescalador.
- **Actualizaciones sin caída:** cambiar de versión de una aplicación sin cortar el servicio.
- **Rollback rápido:** volver a la versión anterior si algo sale mal.
- **Declarativo:** el estado deseado queda escrito en un YAML versionable en Git.

**De Grafana**

- Visualizar métricas de servidores, contenedores y aplicaciones en paneles (*dashboards*).
- Conectar con fuentes como Prometheus, Loki, Elasticsearch, MySQL o PostgreSQL.
- Configurar alertas (correo, Slack, Telegram) cuando una métrica pasa un umbral.
- Monitorear el propio clúster de Kubernetes, junto con Prometheus.

**Limitación de este laboratorio.** Con 3 réplicas, cada Pod de Grafana guarda sus datos en su propio disco temporal (SQLite). Para producción, las réplicas deben compartir una base de datos externa (por ejemplo PostgreSQL) o usar almacenamiento persistente.

## Limpieza

```powershell
kubectl delete -f 02-deployment/
kubectl delete service myfirstpod
kubectl delete pod myfirstpod
minikube stop
```
