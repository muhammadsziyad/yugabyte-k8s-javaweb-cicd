# Deploying YugabyteDB and Java Web Application with Authentication and CI/CD

## Overview

This project involves deploying YugabyteDB and a Java web application on Kubernetes. YugabyteDB will be deployed using StatefulSet to ensure data persistence and high availability. The Java web application will be deployed using Deployment for scalability and managed through Kubernetes Services. Ingress will handle external access, and CI/CD pipelines will automate the deployment process. Authentication will be integrated into the Java web application.

## Objectives

1.  **Deploy YugabyteDB** using StatefulSet for high availability and persistence.
2.  **Deploy a Java web application** with Deployment for scalability and load balancing.
3.  **Integrate authentication** within the Java web application.
4.  **Set up CI/CD pipelines** to automate the build, test, and deployment processes.
5.  **Expose services** using Kubernetes Services and Ingress for external access.
6.  **Manage configuration and credentials** using ConfigMaps and Secrets.

## 1. Installation

**Prerequisites:**

-   Kubernetes cluster (e.g., Minikube, cloud provider, etc.)
-   `kubectl` CLI tool installed
-   Docker installed
-   Helm (optional, for managing Kubernetes applications)
-   Jenkins or GitHub Actions for CI/CD

### 1.1 Install Minikube (Optional, if you don’t have a Kubernetes cluster)

Follow the official Minikube installation guide to install Minikube on your system.

### 1.2 Start Minikube


```bash
minikube start
``` 

### 1.3 Install Helm (optional)

Follow the official Helm installation guide to install Helm.

## 2. Kubernetes Configuration

### 2.1 YugabyteDB StatefulSet and Service

**File: `yugabyte-db-statefulset.yaml`**


```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: yugabyte-db
spec:
  serviceName: "yugabyte-db"
  replicas: 3
  selector:
    matchLabels:
      app: yugabyte-db
  template:
    metadata:
      labels:
        app: yugabyte-db
    spec:
      containers:
      - name: yugabyte
        image: yugabytedb/yugabyte:latest
        ports:
        - containerPort: 5433
        volumeMounts:
        - name: data
          mountPath: /mnt/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
``` 

**File: `yugabyte-db-service.yaml`**


```yaml
apiVersion: v1
kind: Service
metadata:
  name: yugabyte-db
spec:
  ports:
  - port: 5433
    targetPort: 5433
  clusterIP: None
  selector:
    app: yugabyte-db
``` 

### 2.2 Java Web Application Deployment and Service

**File: `java-webapp-deployment.yaml`**


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-webapp
  template:
    metadata:
      labels:
        app: java-webapp
    spec:
      containers:
      - name: java-webapp
        image: your-java-webapp-image:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db-url
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
``` 

**File: `java-webapp-service.yaml`**


```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-webapp-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: java-webapp
``` 

### 2.3 Ingress Configuration

**File: `java-webapp-ingress.yaml`**


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-webapp-ingress
spec:
  rules:
  - host: java-webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-webapp-service
            port:
              number: 80
``` 

### 2.4 ConfigMap and Secret

**File: `app-config.yaml`**


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  db-url: yugabyte-db.yugabyte-db.svc.cluster.local
``` 

**File: `db-secrets.yaml`**


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
data:
  username: <base64-encoded-username>
  password: <base64-encoded-password>
``` 

_Note: Replace `<base64-encoded-username>` and `<base64-encoded-password>` with base64 encoded values._

## 3. CI/CD Integration

### 3.1 Jenkins Pipeline Example

**File: `Jenkinsfile`**


```
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    docker.build('your-java-webapp-image:latest')
                }
            }
        }
        stage('Test') {
            steps {
                // Add your test steps here
                echo 'Running tests...'
            }
        }
        stage('Deploy') {
            steps {
                script {
                    // Push Docker image to registry
                    docker.withRegistry('https://your-docker-registry', 'docker-credentials-id') {
                        docker.image('your-java-webapp-image:latest').push('latest')
                    }
                    
                    // Deploy to Kubernetes
                    kubectl.apply('-f java-webapp-deployment.yaml')
                    kubectl.apply('-f java-webapp-service.yaml')
                    kubectl.apply('-f java-webapp-ingress.yaml')
                }
            }
        }
    }
}
``` 

### 3.2 GitHub Actions Example

**File: `.github/workflows/deploy.yml`**


```
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          
      - name: Build with Gradle
        run: ./gradlew build
        
      - name: Build Docker image
        run: docker build -t your-java-webapp-image:latest .
        
      - name: Push Docker image
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker tag your-java-webapp-image:latest your-docker-registry/your-java-webapp-image:latest
          docker push your-docker-registry/your-java-webapp-image:latest
        
      - name: Deploy to Kubernetes
        uses: azure/setup-kubectl@v1
        with:
          version: '1.21.0'
        
      - name: Apply Kubernetes manifests
        run: |
          kubectl apply -f java-webapp-deployment.yaml
          kubectl apply -f java-webapp-service.yaml
          kubectl apply -f java-webapp-ingress.yaml
``` 

## 4. Apply the Configuration

1.  **Deploy YugabyteDB:**

    
    ```
    kubectl apply -f yugabyte-db-statefulset.yaml
    kubectl apply -f yugabyte-db-service.yaml
    ``` 
    
2.  **Deploy Java Web Application:**

    
    ```
    kubectl apply -f java-webapp-deployment.yaml
    kubectl apply -f java-webapp-service.yaml
    kubectl apply -f java-webapp-ingress.yaml
    ``` 
    
3.  **Create ConfigMap and Secret:**

    
    ```
    kubectl apply -f app-config.yaml
    kubectl apply -f db-secrets.yaml
    ``` 
    
4.  **Add Hostname to `/etc/hosts` (for local testing):**

    
    ```
    echo "$(minikube ip) java-webapp.local" | sudo tee -a /etc/hosts
    ``` 
    

## 5. Verify and Access the Application

1.  **Verify Pods and Services:**

    
    ```
    kubectl get pods
    kubectl get services
    kubectl get ingress
    ``` 
    
2.  **Access the Java Web Application:**
    
    Open your browser and navigate to `http://java-webapp.local`. You should see your Java web application running and connected to YugabyteDB.