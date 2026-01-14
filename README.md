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

![Gateway/Ingress Architecture](gateway-ingress.png)

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
- **Gateway API:** Uses Gateway, GatewayClass, HTTPRoute/HTTPS Route for the same

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

# Part 2: Deploy with Gateway API (NGINX Gateway Fabric)

### Step 1: Install Gateway API CRDs
```
kubectl kustomize \
  "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.5.1" \
  | kubectl apply -f -
kubectl get crd | grep gateway
```
---
### Step 2: Install NGINX Gateway Fabric CRDs
```
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/crds.yaml
```
---

### Step 3: Deploy NGINX Gateway Fabric Controller
```
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/nodeport/deploy.yaml
kubectl get pods -n nginx-gateway
```
---

### Step 4: Expose Fixed NodePort Values
```
kubectl patch svc nginx-gateway -n nginx-gateway --type='json' -p='[
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30080},
  {"op": "replace", "path": "/spec/ports/1/nodePort", "value": 30081}
]'
kubectl get svc -n nginx-gateway nginx-gateway
```
---

### Step 5: Create GatewayClass

`gateway-class.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
```
```
kubectl apply -f gateway-class.yaml
```
---

### Step 6: Create Gateway

`gateway.yaml`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: gateway.web.k8s.local
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    port: 443
    protocol: HTTPS
    hostname: gateway.web.k8s.local
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: web-tls-secret
        namespace: web-app
    allowedRoutes:
      namespaces:
        from: All
```
```
kubectl apply -f gateway.yaml
```
---

### Step 7: Create HTTP Routes

`http-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: web-app
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
    sectionName: http
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /green
    backendRefs:
    - name: green-svc
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /blue
    backendRefs:
    - name: blue-svc
      port: 80
```
```
kubectl apply -f http-route.yaml
```
---

### Step 8: Create HTTPS Routes

`https-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route-https
  namespace: web-app
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
    sectionName: https
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /green
    backendRefs:
    - name: green-svc
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /blue
    backendRefs:
    - name: blue-svc
      port: 80
```
```
kubectl apply -f https-route.yaml
```

### Step 9: Allow Cross-Namespace TLS Access

`reference-grant.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-to-web-app-secrets
  namespace: web-app
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: nginx-gateway
  to:
  - group: ""
    kind: Secret
    name: web-tls-secret
```
```
kubectl apply -f reference-grant.yaml
```
---

### Step 10: Verification
```
kubectl get gateway -n nginx-gateway
kubectl get httproute -n web-app
```
```
kubectl get httproute web-route -n web-app -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'
kubectl get httproute web-route-https -n web-app -o jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'
```
Expected: True
---

### Step 11: Testing

```
HTTP:
curl -H "Host: gateway.web.k8s.local" http://$NODE_IP:30080/blue
curl -H "Host: gateway.web.k8s.local" http://$NODE_IP:30080/green
```
---

```
HTTPS:

# Add host entry
echo "$NODE_IP gateway.web.k8s.local" | sudo tee -a /etc/hosts

curl -k https://gateway.web.k8s.local:30081/blue
curl -k https://gateway.web.k8s.local:30081/green
```

## Troubleshooting
ADDRESS empty → NodePort does not populate ADDRESS, PROGRAMMED=True confirms working Gateway.

HTTPRoute not accepted → check sectionName, parentRefs.namespace, TLS secret, and ReferenceGrant.

TLS listener fails → ensure secret exists and is accessible via ReferenceGrant.

Cleanup
```
kubectl delete -f web-ingress.yaml
kubectl delete -f blue-deployment.yaml
kubectl delete -f green-deployment.yaml
kubectl delete ns web-app
```

Gateway API cleanup:

``` 
kubectl delete -f reference-grant.yaml
kubectl delete -f https-route.yaml
kubectl delete -f http-route.yaml
kubectl delete -f gateway.yaml
kubectl delete -f gateway-class.yaml
kubectl delete -f nodeport/deploy.yaml
kubectl delete -f crds.yaml
```

#### Notes

NodePort is for demo; replace with LoadBalancer or MetalLB for production.

Gateway API provides better separation of concerns, cross-namespace routing, and extensibility compared to Ingress.

TLS termination happens at the gateway/ingress level.