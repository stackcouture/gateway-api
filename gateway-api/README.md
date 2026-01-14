# Kubernetes Blue/Green Sample Web Application Deployment

This repository demonstrates how to deploy a **sample web application** in Kubernetes using two approaches:

 **Kubernetes Gateway API with NGINX Gateway Fabric**

Both setups route traffic to **Blue** and **Green** deployments and demonstrate **HTTP/HTTPS routing with TLS**, including cross-namespace secret access.

---

## Table of Contents

- [Deploy with Gateway API](#deploy-with-gateway-api)
  - [Step 1: Create Namespace](#step-1-create-namespace)
  - [Step 2: Deploy Green App](#step-2-deploy-green-app)
  - [Step 3: Deploy Blue App](#step-3-deploy-blue-app)
  - [Step 4: Create Self-Signed TLS Secret](#step-4-create-self-signed-tls-secret)
  - [Step 5: Install Gateway API CRDs](#step-5-install-gateway-api-crds)
  - [Step 6: Install NGINX Gateway Fabric CRDs](#step-6-install-nginx-gateway-fabric-crds)
  - [Step 7: Deploy NGINX Gateway Fabric Controller](#step-7-deploy-nginx-gateway-fabric-controller)
  - [Step 8: Expose Fixed NodePort Values](#step-8-expose-fixed-nodeport-values)
  - [Step 9: Create GatewayClass](#step-9-create-gatewayclass)
  - [Step 10: Create Gateway](#step-10-create-gateway)
  - [Step 11: Create HTTP Routes](#step-11-create-http-routes)
  - [Step 12: Create HTTPS Routes](#step-12-create-https-routes)
  - [Step 13: Allow Cross-Namespace TLS Access](#step-13-allow-cross-namespace-tls-access)
  - [Step 14: Verification](#step-14-verification)
  - [Step 15: Testing](#step-15-testing)


  #  Deploy with Gateway API (NGINX Gateway Fabric)

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
---

### Step 5: Install Gateway API CRDs
```
kubectl kustomize \
  "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.5.1" \
  | kubectl apply -f -
kubectl get crd | grep gateway
```
---
### Step 6: Install NGINX Gateway Fabric CRDs
```
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/crds.yaml
```
---

### Step 7: Deploy NGINX Gateway Fabric Controller
```
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.1/deploy/nodeport/deploy.yaml
kubectl get pods -n nginx-gateway
```
---

### Step 8: Expose Fixed NodePort Values
```
kubectl patch svc nginx-gateway -n nginx-gateway --type='json' -p='[
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30080},
  {"op": "replace", "path": "/spec/ports/1/nodePort", "value": 30081}
]'
kubectl get svc -n nginx-gateway nginx-gateway
```
---

### Step 9: Create GatewayClass

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

### Step 10: Create Gateway

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

### Step 11: Create HTTP Routes

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

### Step 12: Create HTTPS Routes

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

### Step 13: Allow Cross-Namespace TLS Access

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

### Step 14: Verification
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

### Step 15: Testing

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
```text
ADDRESS empty → NodePort does not populate ADDRESS, PROGRAMMED=True confirms working Gateway.

HTTPRoute not accepted → check sectionName, parentRefs.namespace, TLS secret, and ReferenceGrant.

TLS listener fails → ensure secret exists and is accessible via ReferenceGrant.
```
---

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