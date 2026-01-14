# Kubernetes Blue/Green Sample Web Application Deployment

This repository demonstrates how to deploy a **sample web application** in Kubernetes using two approaches:

1. **NGINX Ingress Controller**  

Both setups route traffic to **Blue** and **Green** deployments and demonstrate **HTTP/HTTPS routing with TLS**, including cross-namespace secret access.

---

## Table of Contents
- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Prerequisites](#prerequisites)
- [Deploy Sample Web App with NGINX Ingress](#deploy-sample-web-app-with-nginx-ingress)
  - [Step 1: Create Namespace](#step-1-create-namespace)
  - [Step 2: Deploy Green App](#step-2-deploy-green-app)
  - [Step 3: Deploy Blue App](#step-3-deploy-blue-app)
  - [Step 4: Create Self-Signed TLS Secret](#step-4-create-self-signed-tls-secret)
  - [Step 5: Install Helm & NGINX Ingress Controller](#step-5-install-helm--nginx-ingress-controller)
  - [Step 6: Create Ingress Resource](#step-6-create-ingress-resource)
  - [Step 7: Test Local Access](#step-7-test-local-access)

---
## Overview

This project demonstrates **modern Kubernetes traffic management**:

- Blue/Green application deployments
- HTTP/HTTPS routing
- TLS termination at gateway/ingress
- Cross-namespace secret access using `ReferenceGrant` (Gateway API)
- NodePort for local testing (can be replaced with LoadBalancer/MetalLB for production)

---

## Architecture Diagram

![Gateway/Ingress Architecture](gateway-ingress.png)


```
- **Ingress:** Routes traffic via path-based rules to services
```
---

## Prerequisites
- Kubernetes cluster (v1.26+ recommended)
- `kubectl` configured
- `openssl` for self-signed certificates
- Helm v3+ for NGINX Ingress
- NodePort access enabled for local testing

---

# Deploy Sample Web App with NGINX Ingress

### Step 1: Create Namespace

`namespace.yaml`:

```yaml
apiVersion: v1 
kind: Namespace 
metadata: 
  name: web-app
```
```
kubectl apply -f namespace.yaml
kubectl get ns
```
### Step 2: Deploy Green App
`green-deployment.yaml`:
```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata:
  name: green-deployment
  namespace: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: green-app
  template:
    metadata:
      labels:
        app: green-app
    spec:
      containers:
      - name: green-container
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
        env:
        - name: GREETING
          value: "Hello from the Green App!"

---
apiVersion: v1 
kind: Service 
metadata:
  name: green-svc
  namespace: web-app
spec:
  selector:
    app: green-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```
```
kubectl apply -f green-deployment.yaml
kubectl get pods -n web-app
kubectl get svc -n web-app
```
---
### Step 3: Deploy Blue App

`blue-deployment.yaml`:

```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata:
  name: blue-deployment
  namespace: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blue-app
  template:
    metadata:
      labels:
        app: blue-app
    spec:
      containers:
      - name: blue-container
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
        env:
        - name: GREETING
          value: "Hello from the Blue App!"

---
apiVersion: v1 
kind: Service 
metadata:
  name: blue-svc
  namespace: web-app
spec:
  selector:
    app: blue-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```
```
kubectl apply -f blue-deployment.yaml
kubectl get pods -n web-app
kubectl get svc -n web-app
```
---
### Step 4: Create Self-Signed TLS Secret
```
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=gateway.web.k8s.local" \
  -addext "subjectAltName = DNS:gateway.web.k8s.local"
```

`secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-tls-secret
  namespace: web-app
type: kubernetes.io/tls
stringData:
  tls.crt: |
    <paste tls.crt here>
  tls.key: |
    <paste tls.key here>
```
```
kubectl apply -f secret.yaml
kubectl get secret -n web-app
```

### Step 5: Install Helm & NGINX Ingress Controller
```
wget https://get.helm.sh/helm-v3.10.3-linux-amd64.tar.gz
tar -zxf helm-v3.10.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin
helm version
```

## Install NGINX Ingress Controller (NodePort):

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30082 \
  --set controller.service.nodePorts.https=30443
```
---

### Step 6: Create Ingress Resource

`web-ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: web-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - gateway.web.k8s.local
    secretName: web-tls-secret
  rules:
  - host: gateway.web.k8s.local
    http:
      paths:
      - path: /blue
        pathType: Prefix
        backend:
          service:
            name: blue-svc
            port:
              number: 80
      - path: /green
        pathType: Prefix
        backend:
          service:
            name: green-svc
            port:
              number: 80
```
```
kubectl apply -f web-ingress.yaml
kubectl get ingress -n web-app
```
---

### Step 7: Test Local Access

Add host entry:
```
echo "$(kubectl get ingress web -n web-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}') gateway.web.k8s.local" | sudo tee -a /etc/hosts
```
```
Test routes:

curl -k http://gateway.web.k8s.local/green
curl -k https://gateway.web.k8s.local/green
curl -k http://gateway.web.k8s.local/blue
curl -k https://gateway.web.k8s.local/blue
```
---
