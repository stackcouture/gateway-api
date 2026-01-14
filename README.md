# Kubernetes Blue/Green Sample Web Application Deployment

This repository demonstrates how to deploy a **sample web application** in Kubernetes using two approaches:

1. **NGINX Ingress Controller**  
2. **Kubernetes Gateway API with NGINX Gateway Fabric**

Both setups route traffic to **Blue** and **Green** deployments and demonstrate **HTTP/HTTPS routing with TLS**, including cross-namespace secret access.

---

## Table of Contents

- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Prerequisites](#prerequisites)
- [Part 1: Deploy Sample Web App with NGINX Ingress](#part-1-deploy-sample-web-app-with-nginx-ingress)
  - [Step 1: Create Namespace](#step-1-create-namespace)
  - [Step 2: Deploy Green App](#step-2-deploy-green-app)
  - [Step 3: Deploy Blue App](#step-3-deploy-blue-app)
  - [Step 4: Create Self-Signed TLS Secret](#step-4-create-self-signed-tls-secret)
  - [Step 5: Install Helm & NGINX Ingress Controller](#step-5-install-helm--nginx-ingress-controller)
  - [Step 6: Create Ingress Resource](#step-6-create-ingress-resource)
  - [Step 7: Test Local Access](#step-7-test-local-access)
- [Part 2: Deploy with Gateway API](#part-2-deploy-with-gateway-api)
  - [Step 1: Install Gateway API CRDs](#step-1-install-gateway-api-crds)
  - [Step 2: Install NGINX Gateway Fabric CRDs](#step-2-install-nginx-gateway-fabric-crds)
  - [Step 3: Deploy NGINX Gateway Fabric Controller](#step-3-deploy-nginx-gateway-fabric-controller)
  - [Step 4: Expose Fixed NodePort Values](#step-4-expose-fixed-nodeport-values)
  - [Step 5: Create GatewayClass](#step-5-create-gatewayclass)
  - [Step 6: Create Gateway](#step-6-create-gateway)
  - [Step 7: Create HTTP Routes](#step-7-create-http-routes)
  - [Step 8: Create HTTPS Routes](#step-8-create-https-routes)
  - [Step 9: Allow Cross-Namespace TLS Access](#step-9-allow-cross-namespace-tls-access)
  - [Step 10: Verification](#step-10-verification)
  - [Step 11: Testing](#step-11-testing)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Notes](#notes)

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
```text
       +----------------+
       |     Client     |
       +----------------+
         |        |
   HTTP :30080  HTTPS :30081
         |        |
         v        v
+--------------------------+
| NGINX Gateway / Ingress |
+--------------------------+

/blue → blue-svc → blue deployment
/green → green-svc → green deployment
```

- **Ingress:** Routes traffic via path-based rules to services
- **Gateway API:** Uses Gateway, GatewayClass, HTTPRoute/HTTPSRoute for the same

---

## Prerequisites
- Kubernetes cluster (v1.26+ recommended)
- `kubectl` configured
- `openssl` for self-signed certificates
- Helm v3+ for NGINX Ingress
- NodePort access enabled for local testing

---

# Part 1: Deploy Sample Web App with NGINX Ingress

### Step 1: Create Namespace

`namespace.yaml`:

```yaml
apiVersion: v1 
kind: Namespace 
metadata: 
  name: web-app

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


# Self Signed Certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=gateway.web.k8s.local" \
  -addext "subjectAltName = DNS:gateway.web.k8s.local"

# Install helm 
wget https://get.helm.sh/helm-v3.10.3-linux-amd64.tar.gz
tar -zxf helm-v3.10.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin

# Setup the Ingress-controller
helm install ingress-nginx \
    --set controller.service.type=NodePort \
    --set controller.service.nodePorts.http=30082 \
    --set controller.service.nodePorts.https=30443 \
    --repo https://kubernetes.github.io/ingress-nginx \
    ingress-nginx


# Add an entry to /etc/hosts for local testing
echo "$(kubectl get ingress web -n web-app -o jsonpath='{.status.loadBalancer.ingress[0].ip}') gateway.web.k8s.local" | sudo tee -a /etc/hosts

# Test HTTP access
curl -k http://gateway.web.k8s.local/blue
curl -k https://gateway.web.k8s.local/blue


curl -k http://gateway.web.k8s.local/green
curl -k https://gateway.web.k8s.local/green



# Install Gateway API Resources
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.5.1" | kubectl apply -f -

# Verify installation
kubectl get crd | grep gateway

## Configure NGINX Gateway Fabric

# Deploy NGINX Gateway Fabric CRDs
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/crds.yaml

# Deploy NGINX Gateway Fabric
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/nodeport/deploy.yaml

# Verify the deployment
kubectl get pods -n nginx-gateway


# Update the service to expose specific nodePort values
kubectl patch svc nginx-gateway -n nginx-gateway --type='json' -p='[
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30080},
  {"op": "replace", "path": "/spec/ports/1/nodePort", "value": 30081}
]'

# Verify the service has been updated
kubectl get svc -n nginx-gateway nginx-gateway


## Create GatewayClass and Gateway Resources

kubectl get gateway -n nginx-gateway
kubectl get httproute -n web-app


## Verify the Gateway API Configuration
# Check the Gateway status
kubectl describe gateway nginx-gateway -n nginx-gateway

# Check the HTTPRoute status
kubectl describe httproute web-route -n web-app

# Check if the HTTPRoute is properly bound to the Gateway
kubectl get httproute web-route -n web-app -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'
kubectl get httproute web-route-https -n web-app -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'



## Test the Gateway API Configuration
# Test the / endpoint
curl -v -H "Host: gateway.web.k8s.local" http://$NODE_IP:30080/blue
curl -v -H "Host: gateway.web.k8s.local" http://$NODE_IP:30080/green

# Add the entry in /etc/hosts
# ${NODE_IP}   gateway.web.k8s.local

# Test the / endpoint
curl -v -k https://gateway.web.k8s.local:30081/blue
curl -v -k https://gateway.web.k8s.local:30081/green 