# Deployment Plan for Python RESTful API on Kubernetes

## Overview

Completed task of plan the deployment for a Python RESTful API web application to a **Kubernetes cluster**.  
---

## 1. Infrastructure

The application will be deployed as a set of Kubernetes objects:

| Object | Purpose | Description |
|--------|----------|-------------|
| **Deployment** | Application pods management | Describe number of replicas, rolling updates, and self-healing. |
| **Service** | Internal/External connection | Exposes the app pods as a ClusterIP service and externally via Ingress. |
| **Ingress** | HTTP access | Routes external HTTP traffic to the Service, providing domain-based routing and TLS termination. |
| **ConfigMap** | Non-sensitive configuration | Stores environment-specific parameters such as API settings or environment names. |
| **Secret** | Sensitive data storage | Manages credentials such as Postgres username, password, API keys. |
| **PersistentVolumeClaim (PVC)** | Data persistence | Ensures Postgres data remains available after pod restarts or rescheduling. |

### Example Structure

```
k8s/
 ├─ app/
 │   ├─ deployment.yaml
 │   ├─ service.yaml
 │   ├─ ingress.yaml
 │   ├─ configmap.yaml
 │   └─ secret.yaml
 └─ db/
     ├─ statefulset.yaml
     ├─ service.yaml
     └─ pvc.yaml
```

---

## 2. Containerization

The application will be containerized via **Docker**.

### Dockerfile Example

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PORT=8000
EXPOSE 8000

CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:8000"]
```

### Notes
- The base image `python:3.12-slim` is chosen because it have small size and security updates.
- `gunicorn` is used as a production-grade WSGI server.
- Environment variables such as database URLs are injected via Kubernetes Secrets and ConfigMaps.

---

## 3. Database Deployment (PostgreSQL)

The Postgres database will be deployed **inside the cluster** using **StatefulSet** for data persistence.  

## Using StatefulSet

- Provides stable network identities and persistent storage per replica.
- Data is stored in a **PersistentVolume** bound to the StatefulSet.

---

## 4. Environment Configuration

Environment variables and credentials will be handled through:

| Type | Example | Kubernetes Object |
|------|----------|------------------|
| Database credentials | POSTGRES_USER, POSTGRES_PASSWORD | Secret |
| Application settings | APP_ENV, DEBUG_MODE | ConfigMap |

### Example Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
data:
  POSTGRES_USER: bXl1c2Vy
  POSTGRES_PASSWORD: bXlwYXNzd29yZA==
```

### Example ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
```

---

## 5. Networking

### Service

A **ClusterIP** service is used to expose the API internally to other services, while **Ingress** exposes it externally.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: python-api
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

TLS certificates can be automated via **cert-manager**.

---

## 6. Scaling Strategy

For scalability and reliability, i will use the **Horizontal Pod Autoscaler (HPA)**.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: python-api
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Additionally:
- Resource limits are defined in the Deployment to control pod scheduling and prevent resource exhaustion.
- Liveness and readiness probes ensure smooth rolling updates and fault tolerance.

---

## 7. Monitoring and Logging

### Monitoring
- **Prometheus** for metrics collection.
- **Grafana** for visualization.
- Application exposes metrics via `/metrics` endpoint (using `prometheus_client` for Python).

### Logging
- Application logs to **stdout/stderr**, collected automatically by Kubernetes.
- Centralized logging can be achieved via:
  - **EFK Stack** (Elasticsearch, Fluentd, Kibana), or
  - **Grafana Loki** for lightweight log aggregation.

---

