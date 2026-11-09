# Kurbernetes Course Tasks

**In this repo you will find all tasks regards Kurbernetes course provided by MLOps MENA community**

## Task Session 1: Multi-Container Pods (Init Container & Sidecar)

Implemented a multi-container Pod that serves an Nginx “Hello” page with a continuously updating timestamp.

### Features

- Three-container Pod architecture
- Init container that prints `done` and exits successfully
- Nginx main container serving `index.html` on port 80
- Sidecar container updating the timestamp every 5 seconds
- Shared process namespace using `shareProcessNamespace: true`
- No Kubernetes volumes; the sidecar updates Nginx files through `/proc/<pid>/root`
- Three-node Minikube cluster
- Init-container failure test using exit code 1
- Recovery after restoring the successful init container

### Architecture

```text
Browser / curl
      |
      v
NodePort Service
      |
      v
Pod: nginx-timestamp
      |
      |-- init-done
      |     prints "done"
      |     exits successfully
      |
      |-- nginx
      |     creates index.html
      |     serves page on port 80
      |
      |-- time-updater
            finds Nginx master PID
            updates index.html every 5 seconds
            through /proc/<nginx-pid>/root
```

### How to Deploy

```bash
minikube start -p task1 --driver=docker --nodes=3 --cpus=2 --memory=4096
kubectl config use-context task1
kubectl apply -f Task-1/nginx-timestamp-pod.yaml
kubectl apply -f Task-1/nginx-timestamp-service.yaml
```

### How to Test

Get the Service URL:

```bash
minikube service nginx-timestamp-svc --url -p task1
```

Test from WSL:

```bash
for i in {1..3}; do
  curl http://$(minikube ip -p task1):30080
  echo
  sleep 6
done
```

The timestamp should change approximately every five seconds.

### Init Container Failure Test

Change the init container command to:

```yaml
args:
  - echo "init failed"; exit 1
```

For a bare Pod, delete and reapply it:

```bash
kubectl delete pod nginx-timestamp
kubectl apply -f nginx-timestamp-pod.yaml
kubectl get pod nginx-timestamp -w
```

Expected status:

```text
Init:Error
Init:BackOff
Init:CrashLoopBackOff
```

Then restore:

```yaml
args:
  - echo "done"
```

Apply again:

```bash
kubectl delete pod nginx-timestamp
kubectl apply -f nginx-timestamp-pod.yaml
kubectl get pod nginx-timestamp -w
```

### Screenshots

- Pod running with `2/2` containers ready
- Timestamp changing after browser refresh or repeated `curl`
- `Init:CrashLoopBackOff` after intentionally failing the init container
- Pod recovery after restoring the successful init command
- Three-node Minikube cluster

### Files

- [Pod manifest](./Task-1/nginx-timestamp-pod.yaml)
- [Service manifest](./Task-1/nginx-timestamp-service.yaml)
- [Task 1 README](./Task-1/README.md)



## Task Session 2: FastAPI + MySQL with ConfigMap, Secret & Ingress

Implemented a multi-tier guestbook application using FastAPI and MySQL on Kubernetes.

### Features

- FastAPI application deployed with 2 replicas
- MySQL database with persistent storage
- ConfigMap for non-sensitive application configuration
- Secret for database credentials
- PersistentVolume and PersistentVolumeClaim for MySQL data
- Node affinity to pin MySQL storage to `minikube-m02`
- NGINX Ingress Controller and Ingress resource for external access
- Persistence verification by deleting and recreating the MySQL Pod

### Architecture

```text
Browser / curl
      |
      v
NGINX Ingress Controller
      |
      v
webapp-api Service
      |
      v
FastAPI Deployment (2 replicas)
      |
      v
MySQL Service
      |
      v
MySQL Pod
      |
      v
PVC -> PV -> hostPath on minikube-m02
```

### How to Deploy

```bash
kubectl apply -f Task-2/k8s/00-namespace.yaml
kubectl apply -f Task-2/k8s/01-configmap.yaml
kubectl apply -f Task-2/k8s/02-secret.yaml
kubectl apply -f Task-2/k8s/03-mysql-pv.yaml
kubectl apply -f Task-2/k8s/04-mysql-pvc.yaml
kubectl apply -f Task-2/k8s/05-mysql-deployment.yaml
kubectl apply -f Task-2/k8s/06-mysql-service.yaml
kubectl apply -f Task-2/k8s/07-webapp-deployment.yaml
kubectl apply -f Task-2/k8s/08-webapp-service.yaml
kubectl apply -f Task-2/k8s/09-ingress.yaml
```

### How to Test

```bash
curl http://127.0.0.1/api/info
curl http://127.0.0.1/api/messages
```

Create a message:

```bash
curl -X POST http://127.0.0.1/api/messages \
  -H "Content-Type: application/json" \
  -d '{"name":"Abeer","message":"Hello Kubernetes"}'
```

### Persistence Test

Delete the MySQL Pod:

```bash
kubectl delete pod -l app=mysql -n task2
```

Then verify that previously submitted messages remain available:

```bash
curl http://127.0.0.1/api/messages
```

### Screenshots

- Ingress access to `/api/info`
- Ingress access to `/api/messages`
- Bound PV and PVC
- MySQL Pod deletion and replacement
- Messages preserved after MySQL Pod restart
