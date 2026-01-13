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