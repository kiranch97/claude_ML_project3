# claude_ML_project3: MLflow Model Registry on Kubernetes

> P1 MLOps Platform — MLflow Tracking Server + PostgreSQL + MinIO  
> Deployed on Kubernetes (Docker Desktop) — Part of the kiranch97 Solutions Architecture

---

## Overview

This project deploys a **production-grade MLflow stack** on Kubernetes, enabling:

- **Experiment Tracking** — log metrics, parameters, and tags from any ML notebook
- **Model Registry** — version, stage, and manage trained models (Staging / Production)
- **Artifact Storage** — store model files, plots, and datasets via MinIO (S3-compatible)
- **Metadata Backend** — PostgreSQL for reliable experiment/run metadata storage

It integrates directly with the Jupyter environment from `claude_ML_project2` so notebooks can log experiments and register models without any extra configuration.

---

## Architecture

```
                    Browser
                       |
              localhost:30085 (NodePort)
              mlflow.local:30080 (Ingress)
                       |
            +----------v-----------+
            |   Namespace: mlflow  |
            |                      |
            |  MLflow Tracking     |
            |  Server (port 5000)  |
            +----+----------+------+
                 |          |
        +--------v--+   +---v----------+
        | PostgreSQL|   |    MinIO     |
        | metadata  |   | artifact     |
        | store     |   | store (S3)   |
        | port:5432 |   | port:9000    |
        | PVC: 5Gi  |   | PVC: 10Gi   |
        +-----------+   | UI: 9001     |
                        +--------------+
```

---

## Components

### MLflow Tracking Server
- Image: `ghcr.io/mlflow/mlflow:v2.9.2`
- Backend store: PostgreSQL (`postgresql://mlflow:mlflow@postgres:5432/mlflow`)
- Artifact store: MinIO S3 (`s3://mlflow-artifacts/`)
- Port: 5000 (NodePort: 30085)

### PostgreSQL
- Image: `postgres:15-alpine`
- Database: `mlflow`
- User/Password: `mlflow/mlflow`
- PVC: 5Gi (default StorageClass)

### MinIO
- Image: `minio/minio:latest`
- Bucket: `mlflow-artifacts`
- Access Key: `minioadmin` / Secret Key: `minioadmin`
- API Port: 9000 | UI Port: 9001 (NodePort: 30090)
- PVC: 10Gi (default StorageClass)

---

## Directory Structure

```
claude_ML_project3/
├── README.md
└── k8s/
    ├── namespace.yaml
    ├── postgres.yaml          # PostgreSQL deployment + service + PVC
    ├── minio.yaml             # MinIO deployment + service + PVC + NodePort
    ├── minio-init-job.yaml    # Job to create the mlflow-artifacts bucket
    ├── mlflow.yaml            # MLflow server deployment + service
    ├── service-nodeport.yaml  # NodePort service for MLflow UI (30085)
    └── ingress.yaml           # Ingress: mlflow.local -> mlflow:5000
```

---

## Quick Start

### Prerequisites
- Docker Desktop with Kubernetes enabled
- nginx-ingress running (from `claude-project1`)
- `kubectl` on `docker-desktop` context

### Deploy

```bash
git clone https://github.com/kiranch97/claude_ML_project3.git
cd claude_ML_project3

# Deploy in order
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/minio.yaml

# Wait for postgres and minio to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n mlflow --timeout=120s
kubectl wait --for=condition=ready pod -l app=minio -n mlflow --timeout=120s

# Create the MinIO bucket
kubectl apply -f k8s/minio-init-job.yaml

# Deploy MLflow
kubectl apply -f k8s/mlflow.yaml
kubectl apply -f k8s/service-nodeport.yaml
kubectl apply -f k8s/ingress.yaml
```

### Add to /etc/hosts

```
127.0.0.1  mlflow.local
```

---

## Access Details

| Service | URL | Credentials |
|---------|-----|-------------|
| MLflow UI | http://localhost:30085 | None |
| MLflow via Ingress | http://mlflow.local:30080 | None |
| MinIO Console | http://localhost:30090 | minioadmin / minioadmin |

---

## Using MLflow from Jupyter (claude_ML_project2)

In any Jupyter notebook, add this to start logging:

```python
import mlflow
import mlflow.sklearn
import mlflow.tensorflow

# Point to your MLflow server
mlflow.set_tracking_uri("http://mlflow.jupyter-ml.svc.cluster.local:5000")
# OR use NodePort from outside cluster:
# mlflow.set_tracking_uri("http://localhost:30085")

mlflow.set_experiment("spam-classifier")

with mlflow.start_run():
    # Log parameters
    mlflow.log_param("epochs", 20)
    mlflow.log_param("batch_size", 32)
    mlflow.log_param("lstm_units", 16)

    # Train your model...
    history = model.fit(train_sequences, train_Y, ...)

    # Log metrics
    mlflow.log_metric("accuracy", test_accuracy)
    mlflow.log_metric("loss", test_loss)

    # Log the model
    mlflow.tensorflow.log_model(model, "spam_lstm_model")

print("Run logged! Check MLflow UI at http://localhost:30085")
```

### Register a model for production

```python
# After logging, register the model
result = mlflow.register_model(
    f"runs:/{run.info.run_id}/spam_lstm_model",
    "SpamClassifier"
)

# Promote to Production
client = mlflow.tracking.MlflowClient()
client.transition_model_version_stage(
    name="SpamClassifier",
    version=result.version,
    stage="Production"
)
```

---

## Verification

```bash
kubectl get all -n mlflow
kubectl get pvc -n mlflow
kubectl get ingress -n mlflow

# Check MLflow logs
kubectl logs -n mlflow deployment/mlflow -f

# Check postgres
kubectl logs -n mlflow deployment/postgres

# Check minio
kubectl logs -n mlflow deployment/minio
```

---

## Integration with Solutions Architecture

This is **P1** of the kiranch97 Solutions Architecture roadmap:

| Project | Status |
|---------|--------|
| claude-project1: nginx-ingress + ArgoCD + WebApp | Done |
| claude_ML_project2: Jupyter on K8s | Done |
| **claude_ML_project3: MLflow Model Registry** | **This project** |
| claude_ML_project4: Logging Stack (Loki) | Next |
| claude_ML_project5: Model Serving | Planned |

---

## Cleanup

```bash
kubectl delete namespace mlflow
```

---

## Author

**kiranch97** — Built collaboratively with Claude AI
Date: March 2026
