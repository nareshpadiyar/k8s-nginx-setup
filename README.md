# Dynamic NGINX Web Application on Kubernetes

A step-by-step guide and reference project for building a custom NGINX Docker image that dynamically injects and displays its container hostname, and deploying it onto an IBM Cloud Kubernetes Service (IKS) cluster with load balancing.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture & Tech Stack](#-architecture--tech-stack)
- [Prerequisites](#-prerequisites)
- [Repository Structure](#-repository-structure)
- [Part 1: Docker Image Creation](#part-1-docker-image-creation)
  - [1. Configuration Files](#1-configuration-files)
  - [2. Build & Push Image](#2-build--push-image)
- [Part 2: Kubernetes Deployment](#part-2-kubernetes-deployment)
  - [1. Cluster Setup & Connectivity](#1-cluster-setup--connectivity)
  - [2. Manifests & Deployment](#2-manifests--deployment)
  - [3. Verification & Access](#3-verification--access)
- [Clean Up](#-clean-up)

---

## 🎯 Overview

This project demonstrates how to:
1. Build a lightweight NGINX image using a custom shell script (`entrypoint.sh`) that dynamically replaces a template variable (`{{HOSTNAME}}`) with the running pod/container's hostname at launch time.
2. Publish the container image to Docker Hub.
3. Deploy a multi-replica deployment on Kubernetes (specifically tested on **IBM Kubernetes Service - IKS**).
4. Expose the deployment via a Kubernetes `LoadBalancer` Service to observe request handling across distinct pod hostnames.

---

## 🛠 Architecture & Tech Stack

- **Container Runtime / Engine:** Docker (x86 / `amd64` architecture recommended for compatibility)
- **Web Server:** NGINX (Base Image: `nginx:latest`)
- **Orchestration:** Kubernetes (IKS Cluster)
- **Image Registry:** Docker Hub (`nareshpadiyar/naresh-app:v1`)

---

## ⚡ Prerequisites

Before beginning, ensure you have the following installed and configured:

- [Docker Engine / Docker Desktop](https://www.docker.com/) (Ensure you are building on an **x86_64** architecture host, or use `docker buildx` if on Apple Silicon/ARM to target `linux/amd64`).
- [kubectl CLI](https://kubernetes.io/docs/tasks/tools/)
- [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cli-getting-started) (with `ks` plugin installed)
- Active **Docker Hub** account.
- Running **IBM Cloud Kubernetes Service (IKS)** cluster with public inbound/outbound connectivity.

---

## 📁 Repository Structure

```text
.
├── Dockerfile          # Docker image specification
├── index.html          # HTML template with {{HOSTNAME}} placeholder
├── entrypoint.sh       # Script to substitute container hostname into index.html
├── deployment.yaml     # Kubernetes Deployment manifest (3 replicas)
└── service.yaml        # Kubernetes LoadBalancer Service manifest
```

---

## Part 1: Docker Image Creation

> **Note on Architecture:** Build this image on an x86 processor architecture machine, or compile using cross-platform build flags if working on macOS (ARM/Apple Silicon), to prevent platform architecture mismatches on Kubernetes worker nodes.

### 1. Configuration Files

#### `index.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Kubernetes Demo</title>
    <style>
        body {
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: Arial, sans-serif;
            font-size: 24px;
            background-color: #c1f80a;
        }
    </style>
</head>
<body>
    <center>
        <h1>
            Hello from NGINX Docker Container: {{HOSTNAME}}
        </h1>
    </center>
</body>
</html>
```

#### `entrypoint.sh`
```bash
#!/bin/sh
# Dynamically replace {{HOSTNAME}} placeholder with actual hostname
sed -i "s/{{HOSTNAME}}/$(hostname)/g" /usr/share/nginx/html/index.html
exec "$@"
```

#### `Dockerfile`
```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

---

### 2. Build & Push Image

1. **Authenticate with Docker Hub:**
   ```bash
   docker login docker.io
   ```

2. **Build the image:**
   ```bash
   docker build -t nareshpadiyar/naresh-app:v1 .
   ```

3. **Verify local image creation:**
   ```bash
   docker images | grep naresh-app
   ```

4. **Push the image to Docker Hub:**
   ```bash
   docker push nareshpadiyar/naresh-app:v1
   ```

5. **Verify online repository:**
   Confirm the image tag exists at: `https://hub.docker.com/repositories/nareshpadiyar`

---

## Part 2: Kubernetes Deployment

### 1. Cluster Setup & Connectivity

1. **Provision an IKS Cluster:** Ensure public Inbound and Outbound access are enabled.
2. **Connect to your IKS Cluster:**
   ```bash
   ibmcloud login -a cloud.ibm.com --sso
   ibmcloud ks cluster config --cluster <cluster-name>
   ```

3. **Create a target namespace:**
   ```bash
   kubectl create namespace naresh-webapp
   ```

---

### 2. Manifests & Deployment

#### Step A: Deploy Application

Create `deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: naresh-webapp
  namespace: naresh-webapp
  labels:
    app: naresh-webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: naresh-webapp
  template:
    metadata:
      labels:
        app: naresh-webapp
    spec:
      containers:
        - name: naresh-webapp
          image: nareshpadiyar/naresh-app:v1
          ports:
            - containerPort: 80
```

Apply the deployment manifest:
```bash
kubectl apply -f deployment.yaml
```

Verify deployment and pods status:
```bash
kubectl get deployments -n naresh-webapp
kubectl get pods -n naresh-webapp
```

---

#### Step B: Expose Application via LoadBalancer

Create `service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: naresh-webapp-service
  namespace: naresh-webapp
spec:
  selector:
    app: naresh-webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

Apply the service manifest:
```bash
kubectl apply -f service.yaml
```

---

### 3. Verification & Access

1. **Check the Service status:**
   ```bash
   kubectl get svc -n naresh-webapp
   ```

2. **Retrieve External IP / Ingress Domain:**
   ```bash
   kubectl describe svc naresh-webapp-service -n naresh-webapp
   ```

3. **Check all created resources in the namespace:**
   ```bash
   kubectl get all -n naresh-webapp
   ```

4. **Access the Web Application:**
   Open a browser or run a `curl` request pointing to the LoadBalancer IP/Domain:
   ```text
   http://<LoadBalancer_IP_or_Domain>
   ```
   *Refreshing the browser will demonstrate traffic load balancing across the 3 backend pod instances, showing different container hostnames.*

---

## 🧹 Clean Up

To remove all created resources from your Kubernetes cluster, run:

```bash
kubectl delete namespace naresh-webapp
```