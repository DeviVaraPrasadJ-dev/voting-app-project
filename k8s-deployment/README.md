# Voting App – Kubernetes Deployment

This project deploys a complete microservices-based **Cats vs Dogs Voting Application** on Kubernetes using Docker images, StatefulSets, Deployments, Services, ConfigMaps, and Secrets.

It includes five core components:
- **vote service** (Python, frontend for voting)
- **result service** (Node.js, results UI)
- **worker service** (Spring Boot, Kafka consumer + DB writer)
- **postgres** (StatefulSet database)
- **kafka** (message broker)

---

# 📌 Architecture Overview

### User → Vote UI → Kafka → Worker → PostgreSQL → Result UI → User


### ✔ vote service  
- Shows voting UI  
- Sends vote events to Kafka topic  

### ✔ worker service  
- Consumes Kafka messages  
- Writes results to PostgreSQL  

### ✔ result service  
- Reads from PostgreSQL  
- Displays real-time results  

### ✔ PostgreSQL  
- Stores all processed votes  
- Deployed via StatefulSet  
- Persistent storage via PVC + StorageClass  

### ✔ Kafka  
- Receives and distributes vote messages  
- Works with a single broker (no zookeeper needed for this deployment)

---

# 📁 Kubernetes Resources Created

## 1. Namespace
```bash
kubectl apply -f namespace.yaml
```


## 2. Secret (Database Credentials)
```bash
kubectl apply -f postgresql-secret.yaml
```


## 3. ConfigMap (DB Name, Kafka Topic, Kafka Broker)
```bash
kubectl apply -f postgre-configmap.yaml
```
---

# 🗄 StorageClass (Dynamic Provisioning for PostgreSQL)

PostgreSQL runs as a **StatefulSet**, and each replica needs its own PersistentVolumeClaim (PVC).  
Your PVC was stuck in `Pending` because Kubernetes requires a StorageClass to dynamically create a PersistentVolume.

To fix this, we used a pre-built StorageClass from a public GitHub repository.

### ⭐ StorageClass Used (Local Path Provisioner)

To enable persistent storage for PostgreSQL, we installed the Local Path Provisioner from Rancher and set it as the default StorageClass.

#### Install StorageClass
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

#### Set it as the Default StorageClass
```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### Verify
```bash
kubectl get storageclass
```

### ✔ Apply StorageClass
```bash
kubectl apply -f storageclass.yaml
```

## 4. PostgreSQL StatefulSet + Headless Service
```bash
kubectl apply -f postgre-sql-statefull-deployment.yaml
kubectl apply -f postgre-sql-service.yaml
```

## 5. Kafka Deployment + Service
```bash
kubectl apply -f kafka-deployment.yaml
kubectl apply -f kafka-service.yaml
```

## 6. Worker Deployment + Service
```bash
kubectl apply -f worker-deployment.yaml
kubectl apply -f worker-service.yaml
```

## 7. Vote Deployment + Service
```bash
kubectl apply -f vote-deployment.yaml
kubectl apply -f vote-service.yaml
```

## 8. Result Deployment + Service
```bash
kubectl apply -f result-deployment.yaml
kubectl apply -f result-service.yaml
```

## Apply Everything in One Command
```bash
kubectl apply -f .
```
# 🐳 Docker Image Build & Push

## 1. Login to Docker Hub
```bash
docker login
```
## 2. Build Images
```bash
docker build -t <dockerhub-username>/vote:latest   ./vote
docker build -t <dockerhub-username>/result:latest ./result
docker build -t <dockerhub-username>/worker:latest ./worker
```
## 3. Push Images
```bash
docker push <dockerhub-username>/vote:latest
docker push <dockerhub-username>/result:latest
docker push <dockerhub-username>/worker:latest
```

# 🌐 Access the Application
## Get Node IP
```bash
kubectl get nodes -o wide
```

## URLs (NodePort)

### Vote UI:
```bash
http://<node-ip>:30080
```
### Result UI:
```bash
http://<node-ip>:30081
```

# 🔄 How Services Communicate
## vote → kafka
#### Publishes vote messages:

```bash
KAFKA_BOOTSTRAP_SERVERS=kafka:9092
KAFKA_TOPIC=votes
```
## worker → kafka
#### Consumes the same topic:

```bash
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

## worker → postgres
#### Writes vote counts:

```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/voting_db
```
## result → postgres
#### Reads aggregated results:

```bash
POSTGRES_HOST=postgres
POSTGRES_DB=voting_db
```
| Component    | Type                   | Purpose                               |
|--------------|------------------------|----------------------------------------|
| **vote**     | Deployment + NodePort  | UI for voting → sends events to Kafka |
| **worker**   | Deployment             | Consumes Kafka → saves to PostgreSQL  |
| **result**   | Deployment + NodePort  | Shows real-time results               |
| **kafka**    | Deployment + ClusterIP | Message broker                        |
| **postgres** | StatefulSet + PVC      | Persistent DB                         |

