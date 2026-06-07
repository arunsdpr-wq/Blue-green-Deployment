# Blue-Green Deployment Guide

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Project Overview](#project-overview)
3. [Local Development Setup](#local-development-setup)
4. [Dockerization](#dockerization)
5. [Kubernetes Setup](#kubernetes-setup)
6. [Deployment Steps](#deployment-steps)
7. [Blue-Green Deployment Strategy](#blue-green-deployment-strategy)
8. [Traffic Management](#traffic-management)
9. [Health Checks and Monitoring](#health-checks-and-monitoring)
10. [Rollback Procedure](#rollback-procedure)
11. [Cleanup](#cleanup)

---

## Prerequisites

Before starting the deployment process, ensure you have the following installed:

### Required Tools
- **Node.js** (v14 or higher) - [Download](https://nodejs.org/)
- **Git** - [Download](https://git-scm.com/)
- **Docker Desktop** - [Download](https://www.docker.com/products/docker-desktop)
- **Minikube** - [Download](https://minikube.sigs.k8s.io/docs/start/)
- **kubectl** - [Installation Guide](https://kubernetes.io/docs/tasks/tools/)
- **Helm** (Optional, for advanced deployments) - [Download](https://helm.sh/docs/intro/install/)

### System Requirements
- Minimum 4GB RAM available for Minikube
- At least 20GB free disk space
- Virtualization enabled in BIOS (for Minikube)

### Environment Variables Setup
Create these `.env` files in their respective directories:

**Backend (.env in `backend/`):**
```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/blue-green-db
NODE_ENV=development
```

**Frontend Blue (.env in `frontend-blue/`):**
```env
PORT=3100
BACKEND_URL=http://localhost:5000
```

**Frontend Green (.env in `frontend-green/`):**
```env
PORT=3200
BACKEND_URL=http://localhost:5000
```

---

## Project Overview

### Architecture
```
Blue-Green Deployment Project
├── Backend API (Express.js + MongoDB)
│   ├── Port: 5000
│   ├── Health Endpoint: GET /health
│   └── Routes: /api/users
│
├── Frontend Blue (Express.js Static Server)
│   ├── Port: 3100
│   ├── Health Endpoint: GET /health
│   └── Version: blue
│
└── Frontend Green (Express.js Static Server)
    ├── Port: 3200
    ├── Health Endpoint: GET /health
    └── Version: green
```

### Service Descriptions
- **Backend API**: Node.js/Express server with MongoDB integration for user management
- **Frontend Blue**: First version of the frontend UI
- **Frontend Green**: Second version of the frontend UI (for blue-green deployment)

---

## Local Development Setup

### Step 1: Clone the Repository
```bash
git clone <your-repository-url>
cd Blue-green-Deployment
```

### Step 2: Setup Backend

```bash
cd backend
npm install
```

Create/update the `.env` file:
```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/blue-green-db
NODE_ENV=development
```

Start the backend server:
```bash
npm start
# Backend will run on http://localhost:5000
```

**Verify Backend:**
```bash
curl http://localhost:5000/health
# Expected response: { "status": "ok", "message": "Backend API is running" }
```

### Step 3: Setup Frontend Blue

```bash
cd ../frontend-blue
npm install
```

Create/update the `.env` file:
```env
PORT=3100
BACKEND_URL=http://localhost:5000
```

Start the frontend blue server:
```bash
npm start
# Frontend Blue will run on http://localhost:3100
```

**Verify Frontend Blue:**
```bash
curl http://localhost:3100/health
# Expected response: { "status": "ok", "message": "Basic frontend is running", "version": "basic", "port": 3100 }
```

### Step 4: Setup Frontend Green

```bash
cd ../frontend-green
npm install
```

Create/update the `.env` file:
```env
PORT=3200
BACKEND_URL=http://localhost:5000
```

Start the frontend green server:
```bash
npm start
# Frontend Green will run on http://localhost:3200
```

**Verify Frontend Green:**
```bash
curl http://localhost:3200/health
# Expected response: { "status": "ok", "message": "Basic frontend is running", "version": "basic", "port": 3200 }
```

### Step 5: Verify All Services
```bash
# Test Backend
curl http://localhost:5000/health

# Test Frontend Blue
curl http://localhost:3100/health

# Test Frontend Green
curl http://localhost:3200/health
```

---

## Dockerization

### Step 1: Create Dockerfiles

#### Backend Dockerfile
Create `backend/Dockerfile`:
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:5000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

CMD ["npm", "start"]
```

#### Frontend Dockerfile (Blue & Green)
Create `frontend-blue/Dockerfile`:
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3100

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3100/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

CMD ["npm", "start"]
```

Create `frontend-green/Dockerfile` (identical to blue but with port 3200):
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3200

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3200/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

CMD ["npm", "start"]
```

### Step 2: Build Docker Images

Set your Docker registry username (e.g., Docker Hub):
```bash
export DOCKER_USERNAME=your-username
export IMAGE_TAG=v1.0.0
```

Build and tag the images:
```bash
# Build Backend Image
docker build -t $DOCKER_USERNAME/backend:$IMAGE_TAG ./backend
docker tag $DOCKER_USERNAME/backend:$IMAGE_TAG $DOCKER_USERNAME/backend:latest

# Build Frontend Blue Image
docker build -t $DOCKER_USERNAME/frontend-blue:$IMAGE_TAG ./frontend-blue
docker tag $DOCKER_USERNAME/frontend-blue:$IMAGE_TAG $DOCKER_USERNAME/frontend-blue:latest

# Build Frontend Green Image
docker build -t $DOCKER_USERNAME/frontend-green:$IMAGE_TAG ./frontend-green
docker tag $DOCKER_USERNAME/frontend-green:$IMAGE_TAG $DOCKER_USERNAME/frontend-green:latest
```

### Step 3: Test Docker Images Locally

```bash
# Run Backend
docker run -d --name backend -p 5000:5000 \
  -e MONGO_URI=mongodb://host.docker.internal:27017/blue-green-db \
  $DOCKER_USERNAME/backend:latest

# Run Frontend Blue
docker run -d --name frontend-blue -p 3100:3100 \
  $DOCKER_USERNAME/frontend-blue:latest

# Run Frontend Green
docker run -d --name frontend-green -p 3200:3200 \
  $DOCKER_USERNAME/frontend-green:latest

# Verify
curl http://localhost:5000/health
curl http://localhost:3100/health
curl http://localhost:3200/health

# Cleanup
docker stop backend frontend-blue frontend-green
docker rm backend frontend-blue frontend-green
```

### Step 4: Push Images to Registry (Optional)

If using Docker Hub:
```bash
docker login

docker push $DOCKER_USERNAME/backend:$IMAGE_TAG
docker push $DOCKER_USERNAME/backend:latest
docker push $DOCKER_USERNAME/frontend-blue:$IMAGE_TAG
docker push $DOCKER_USERNAME/frontend-blue:latest
docker push $DOCKER_USERNAME/frontend-green:$IMAGE_TAG
docker push $DOCKER_USERNAME/frontend-green:latest
```

---

## Kubernetes Setup

### Step 1: Start Minikube

```bash
# Start Minikube with sufficient resources
minikube start --cpus=4 --memory=8192 --disk-size=20gb

# Verify cluster is running
kubectl cluster-info
kubectl get nodes
```

### Step 2: Enable Required Addons

```bash
# Enable ingress for traffic routing
minikube addons enable ingress

# Enable metrics-server for monitoring
minikube addons enable metrics-server

# Verify addons
minikube addons list
```

### Step 3: Create Kubernetes Namespace

```bash
# Create namespace for the application
kubectl create namespace blue-green

# Set default namespace
kubectl config set-context --current --namespace=blue-green

# Verify namespace
kubectl get namespace
```

### Step 4: Create Docker Registry Secret (if using private registry)

```bash
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=$DOCKER_USERNAME \
  --docker-password=$DOCKER_PASSWORD \
  --docker-email=$DOCKER_EMAIL \
  -n blue-green
```

---

## Deployment Steps

### Step 1: Create Kubernetes Manifests

Create `k8s/` directory:
```bash
mkdir -p k8s
```

#### Backend Deployment (`k8s/backend-deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: blue-green
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: your-username/backend:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
          name: http
        env:
        - name: PORT
          value: "5000"
        - name: MONGO_URI
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: mongo-uri
        - name: NODE_ENV
          value: "production"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
```

#### Backend Service (`k8s/backend-service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: blue-green
  labels:
    app: backend
spec:
  type: ClusterIP
  ports:
  - port: 5000
    targetPort: 5000
    protocol: TCP
    name: http
  selector:
    app: backend
```

#### Frontend Blue Deployment (`k8s/frontend-blue-deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-blue
  namespace: blue-green
  labels:
    app: frontend
    version: blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      version: blue
  template:
    metadata:
      labels:
        app: frontend
        version: blue
    spec:
      containers:
      - name: frontend-blue
        image: your-username/frontend-blue:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3100
          name: http
        env:
        - name: PORT
          value: "3100"
        - name: BACKEND_URL
          value: "http://backend:5000"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3100
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3100
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
```

#### Frontend Green Deployment (`k8s/frontend-green-deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-green
  namespace: blue-green
  labels:
    app: frontend
    version: green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      version: green
  template:
    metadata:
      labels:
        app: frontend
        version: green
    spec:
      containers:
      - name: frontend-green
        image: your-username/frontend-green:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3200
          name: http
        env:
        - name: PORT
          value: "3200"
        - name: BACKEND_URL
          value: "http://backend:5000"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3200
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3200
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
```

#### Frontend Service (`k8s/frontend-service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: blue-green
  labels:
    app: frontend
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3100
    protocol: TCP
    name: http
  selector:
    app: frontend
    version: blue
```

#### Ingress Configuration (`k8s/ingress.yaml`)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blue-green-ingress
  namespace: blue-green
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: localhost
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 5000
```

#### ConfigMap (`k8s/configmap.yaml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: blue-green
data:
  mongo-uri: "mongodb://mongo:27017/blue-green-db"
```

### Step 2: Deploy to Kubernetes

```bash
# Deploy all resources
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/backend-service.yaml
kubectl apply -f k8s/frontend-blue-deployment.yaml
kubectl apply -f k8s/frontend-green-deployment.yaml
kubectl apply -f k8s/frontend-service.yaml
kubectl apply -f k8s/ingress.yaml

# Verify deployments
kubectl get deployments -n blue-green
kubectl get services -n blue-green
kubectl get pods -n blue-green
```

### Step 3: Verify Deployment Status

```bash
# Check pod status
kubectl get pods -n blue-green -w

# Check pod logs
kubectl logs -n blue-green -l app=backend
kubectl logs -n blue-green -l app=frontend,version=blue
kubectl logs -n blue-green -l app=frontend,version=green

# Describe specific deployment
kubectl describe deployment backend -n blue-green
kubectl describe deployment frontend-blue -n blue-green
```

### Step 4: Access the Application

#### Using Minikube IP
```bash
# Get Minikube IP
MINIKUBE_IP=$(minikube ip)

# Access frontend
curl http://$MINIKUBE_IP/
curl http://$MINIKUBE_IP/health

# Access backend API
curl http://$MINIKUBE_IP/api/users
```

#### Using Port Forwarding
```bash
# Forward backend port
kubectl port-forward svc/backend 5000:5000 -n blue-green &

# Forward frontend port
kubectl port-forward svc/frontend 3100:80 -n blue-green &

# Access via localhost
curl http://localhost:3100/
curl http://localhost:5000/health
```

#### Using Ingress
```bash
# Start Minikube tunnel (in a separate terminal)
minikube tunnel

# Get ingress IP
kubectl get ingress -n blue-green

# Access via ingress IP or localhost (after tunnel)
curl http://localhost/
curl http://localhost/health
```

---

## Blue-Green Deployment Strategy

### Overview
Blue-Green deployment allows you to run two identical production environments. At any time, only one is live (receiving traffic). This enables zero-downtime deployments and quick rollback capabilities.

### Blue Environment (Currently Active)
- Frontend deployment: `frontend-blue`
- Service selector targets: `version: blue`
- Port: 3100

### Green Environment (Standby)
- Frontend deployment: `frontend-green`
- Service selector targets: `version: green` (when active)
- Port: 3200

### Deployment Process

#### Step 1: Prepare Green Environment (Standby)
```bash
# Update Green frontend image to latest version
kubectl set image deployment/frontend-green \
  frontend-green=your-username/frontend-green:v2.0.0 \
  -n blue-green

# Monitor the rollout
kubectl rollout status deployment/frontend-green -n blue-green

# Verify Green is running and healthy
kubectl get pods -l app=frontend,version=green -n blue-green
```

#### Step 2: Test Green Environment
```bash
# Port forward to green environment
kubectl port-forward svc/frontend-green 3200:3200 -n blue-green &

# Run smoke tests
curl http://localhost:3200/health
curl http://localhost:3200/

# Perform manual testing
# Access at http://localhost:3200/
```

#### Step 3: Switch Traffic to Green
```bash
# Update service selector from blue to green
kubectl patch service frontend -p '{"spec":{"selector":{"version":"green"}}}' -n blue-green

# Verify traffic is now going to green
kubectl get service frontend -n blue-green -o yaml | grep version

# Monitor for any issues
kubectl get pods -n blue-green
```

#### Step 4: Verify Green Environment in Production
```bash
# Check logs
kubectl logs -l app=frontend,version=green -n blue-green -f

# Run health checks
kubectl port-forward svc/frontend 3100:80 -n blue-green &
curl http://localhost:3100/health

# Monitor metrics
kubectl top pods -n blue-green
```

#### Step 5: Prepare Blue for Next Deployment
```bash
# Blue is now standby - update its image for next deployment
kubectl set image deployment/frontend-blue \
  frontend-blue=your-username/frontend-blue:v3.0.0 \
  -n blue-green
```

### Quick Rollback Procedure
If issues are detected after switching to Green:

```bash
# Switch traffic back to Blue immediately
kubectl patch service frontend -p '{"spec":{"selector":{"version":"blue"}}}' -n blue-green

# Verify
kubectl get service frontend -n blue-green -o yaml | grep version

# Check Blue is handling traffic correctly
kubectl logs -l app=frontend,version=blue -n blue-green -f
```

---

## Traffic Management

### Current Service Configuration
The frontend service uses selector `version: blue` to route traffic:

```yaml
selector:
  app: frontend
  version: blue
```

### Switching Active Environment

#### Switch from Blue to Green
```bash
kubectl patch service frontend \
  --type merge \
  --patch '{"spec":{"selector":{"version":"green"}}}' \
  -n blue-green
```

#### Switch from Green to Blue
```bash
kubectl patch service frontend \
  --type merge \
  --patch '{"spec":{"selector":{"version":"blue"}}}' \
  -n blue-green
```

### Verify Current Active Version
```bash
# Check which version is active
kubectl get service frontend -n blue-green -o jsonpath='{.spec.selector.version}'

# List pods handling traffic
kubectl get pods -L version -n blue-green -l app=frontend
```

### Traffic Split (Advanced - Canary Deployment)
For gradual rollout, use multiple services or a service mesh like Istio:

```bash
# Example: Manual traffic split approach
# Update service to select all frontend pods
kubectl patch service frontend \
  --type merge \
  --patch '{"spec":{"selector":{"app":"frontend"}}}' \
  -n blue-green

# This routes to both blue and green (50/50 split approximately)
```

---

## Health Checks and Monitoring

### Built-in Health Endpoints

#### Backend Health Check
```bash
curl http://localhost:5000/health
# Response: { "status": "ok", "message": "Backend API is running" }
```

#### Frontend Health Check
```bash
curl http://localhost:3100/health
# Response: { "status": "ok", "message": "Basic frontend is running", "version": "basic", "port": 3100 }
```

### Kubernetes Liveness and Readiness Probes
Both deployments include:
- **Liveness Probe**: Checks if pod is alive (restarts if failed)
- **Readiness Probe**: Checks if pod is ready to serve traffic

Configuration:
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

### Monitoring Commands

#### Check Pod Status
```bash
# Get all pods with status
kubectl get pods -n blue-green

# Get detailed pod information
kubectl describe pod <pod-name> -n blue-green

# Watch pods in real-time
kubectl get pods -n blue-green -w
```

#### Monitor Resource Usage
```bash
# Get metrics for all pods
kubectl top pods -n blue-green

# Get metrics for specific deployment
kubectl top pods -l app=backend -n blue-green

# Continuous monitoring
watch 'kubectl top pods -n blue-green'
```

#### View Logs
```bash
# Backend logs
kubectl logs -l app=backend -n blue-green

# Frontend Blue logs
kubectl logs -l app=frontend,version=blue -n blue-green

# Frontend Green logs
kubectl logs -l app=frontend,version=green -n blue-green

# Real-time logs (follow)
kubectl logs -f <pod-name> -n blue-green

# Last 100 lines
kubectl logs --tail=100 <pod-name> -n blue-green
```

#### Check Events
```bash
# Recent cluster events
kubectl get events -n blue-green

# Watch for new events
kubectl get events -n blue-green -w

# Specific pod events
kubectl describe pod <pod-name> -n blue-green
```

### Automated Monitoring Setup

#### Deploy Prometheus (Optional)
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  -n blue-green \
  --create-namespace
```

#### Deploy Grafana (Optional)
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install grafana grafana/grafana \
  -n blue-green
```

---

## Rollback Procedure

### Scenario 1: Rollback Traffic to Previous Version

#### If you switched to Green and need to rollback to Blue:
```bash
# Immediate rollback
kubectl patch service frontend \
  --type merge \
  --patch '{"spec":{"selector":{"version":"blue"}}}' \
  -n blue-green

# Verify rollback
kubectl get service frontend -n blue-green -o jsonpath='{.spec.selector.version}'

# Check logs for any errors
kubectl logs -l app=frontend,version=blue -n blue-green -f
```

### Scenario 2: Rollback Deployment (Revert image update)

#### Check rollout history
```bash
# Frontend Blue rollout history
kubectl rollout history deployment/frontend-blue -n blue-green

# Backend rollout history
kubectl rollout history deployment/backend -n blue-green
```

#### Rollback to previous version
```bash
# Rollback frontend-blue to previous revision
kubectl rollout undo deployment/frontend-blue -n blue-green

# Rollback to specific revision
kubectl rollout undo deployment/frontend-blue -n blue-green --to-revision=2

# Verify rollback is in progress
kubectl rollout status deployment/frontend-blue -n blue-green

# Check logs
kubectl logs -l app=frontend,version=blue -n blue-green -f
```

### Scenario 3: Complete Environment Rollback

If entire deployment needs rollback:
```bash
# Delete current deployments
kubectl delete deployment frontend-blue frontend-green backend -n blue-green

# Delete services
kubectl delete service frontend backend -n blue-green

# Re-deploy from previous state
kubectl apply -f k8s/

# Verify
kubectl get deployments -n blue-green
kubectl get services -n blue-green
```

### Rollback Best Practices
1. Always monitor logs during and after a rollback
2. Run health checks immediately after rollback
3. Test functionality before declaring rollback complete
4. Keep deployment revisions: `kubectl rollout history deployment/<name> -n blue-green`
5. For critical rollbacks, test in staging first

---

## Cleanup

### Remove Application from Kubernetes

#### Option 1: Delete Specific Resources
```bash
# Delete all resources in blue-green namespace
kubectl delete all -n blue-green

# Or delete specific resources
kubectl delete deployment -n blue-green --all
kubectl delete service -n blue-green --all
kubectl delete ingress -n blue-green --all
kubectl delete configmap -n blue-green --all
```

#### Option 2: Delete Entire Namespace
```bash
# Delete namespace (deletes all resources in it)
kubectl delete namespace blue-green

# Verify namespace is deleted
kubectl get namespace
```

### Stop Minikube
```bash
# Stop Minikube cluster
minikube stop

# Delete Minikube (if you want to free up resources)
minikube delete
```

### Stop Local Development Servers
```bash
# Stop processes running on local machine
# For Node.js processes:
lsof -ti:5000 | xargs kill -9    # Backend
lsof -ti:3100 | xargs kill -9    # Frontend Blue
lsof -ti:3200 | xargs kill -9    # Frontend Green
```

### Clean Docker Images
```bash
# Remove unused images
docker image prune

# Remove specific images
docker rmi your-username/backend:v1.0.0
docker rmi your-username/frontend-blue:v1.0.0
docker rmi your-username/frontend-green:v1.0.0

# Remove all images
docker image prune -a
```

### Clean Docker Containers
```bash
# Stop all running containers
docker stop $(docker ps -aq)

# Remove all containers
docker rm $(docker ps -aq)

# Remove unused volumes
docker volume prune
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: Pod fails to start
```bash
# Check pod events
kubectl describe pod <pod-name> -n blue-green

# Check logs
kubectl logs <pod-name> -n blue-green

# Common causes: Image not found, insufficient resources, environment variables
```

#### Issue 2: Service not accessible
```bash
# Verify service is created
kubectl get svc -n blue-green

# Check service endpoints
kubectl get endpoints <service-name> -n blue-green

# Verify selector matches pod labels
kubectl get pods --show-labels -n blue-green
```

#### Issue 3: Health checks failing
```bash
# Check liveness/readiness probe configuration
kubectl describe deployment <deployment-name> -n blue-green

# Manually test health endpoint
kubectl port-forward pod/<pod-name> 5000:5000 -n blue-green
curl http://localhost:5000/health
```

#### Issue 4: High resource usage
```bash
# Check resource metrics
kubectl top pods -n blue-green

# Check resource limits
kubectl get deployment -n blue-green -o yaml | grep -A 10 "resources:"

# Scale deployments
kubectl scale deployment <name> --replicas=1 -n blue-green
```

---

## Summary Checklist

- [ ] Prerequisites installed and verified
- [ ] Local development setup complete
- [ ] Docker images built and tested
- [ ] Minikube cluster started with addons enabled
- [ ] Kubernetes manifests created in `k8s/` directory
- [ ] Deployments deployed to cluster
- [ ] All pods are running and healthy
- [ ] Services are accessible
- [ ] Health endpoints respond correctly
- [ ] Ingress is configured and working
- [ ] Blue environment is active
- [ ] Green environment is standby
- [ ] Monitoring and logging configured
- [ ] Rollback procedure documented and tested
- [ ] Team trained on deployment process

---

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [Docker Documentation](https://docs.docker.com/)
- [Blue-Green Deployment Pattern](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Helm Documentation](https://helm.sh/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## Support and Contact

For issues or questions:
1. Check Kubernetes logs: `kubectl logs -n blue-green`
2. Review troubleshooting section above
3. Contact DevOps team or repository maintainer
4. Create an issue on the project repository

---

**Last Updated:** June 7, 2026
**Version:** 1.0.0
