## Configure Azure CLI to access your subscription

```
az login
az account set --subscription <subscriptionId>
```

## Create resource group for you cluster

```
az group create --name <resource-group> --location <region>
```

## Create AKS cluster

```
az aks create --resource-group <resource-group> \
    --name <cluster-name> \
    --location <region> \
    --kubernetes-version 1.15.3 \
    --enable-addon monitoring
```

## Verify connectivity from kubectl
```
az aks get-credentials --resource-group <resource-group> --name <cluster-name>

kubectl get nodes
```

## Install Helm
```
kubectl apply -f helm-rbac.yaml
helm init --service-account tiller
```

## Deploy MongoDB using Helm
```
helm install stable/mongodb --name orders-mongo --set mongodbUsername=orders-user,mongodbPassword=orders-password,mongodbDatabase=akschallenge
```

## Create MongoDB secret
```
kubectl create secret generic mongodb --from-literal=mongoHost="orders-mongo-mongodb.default.svc.cluster.local" --from-literal=mongoUser="orders-user" --from-literal=mongoPassword="orders-password"
```

## Deploy backend and create service
```
kubectl apply -f captureorder-deployment.yaml
kubectl apply -f captureorder-service.yaml
```
Wait for couple of minutes for ALB to assign a public IP
```
kubectl get service captureorder -o jsonpath="{.status.loadBalancer.ingress[*].ip}" -w
```

## Verify connection between backend service and mongo
```
curl -d '{"EmailAddress": "email@domain.com", "Product": "prod-1", "Total": 100}' -H "Content-Type: application/json" -X POST http://[Your Service Public LoadBalancer IP]/v1/order
```

## Deploy frontend application
Edit `CAPTUREORDERSERVICEIP` in `frontend-deployment.yaml`, then deploy it

```
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

## Install ingress
```
helm repo update

helm upgrade --install ingress stable/nginx-ingress --namespace ingress
```

Wait for a couple of minutes and get public IP of ingress

```
kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}"
```

## Configure Ingress
Edit `frontend-ingress.yaml` with your Ingress' IP address and deploy it

```
kubectl apply -f frontend-ingress.yaml
```

## Check your site
http://frontend.YOUR-INGRESS-IP-ADDRESS.nip.io

## Enable TLS on Ingress
Install certmanager
```
helm install stable/cert-manager --name cert-manager --set ingressShim.defaultIssuerName=letsencrypt --set ingressShim.defaultIssuerKind=ClusterIssuer --version v0.5.2
```

Update Ingress configuration with TLS (remember to edit the file first)
```
kubectl apply -f frontend-ingress-tls.yaml
```

Verify it's up and running
```
kubectl describe certificate frontend
```

## Run load test
Start the test
```
az container create -g <resource-group> -n loadtest --image azch/loadtest --restart-policy Never -e SERVICE_IP=<public ip of order capture service>
```

Check how it goes
```
az container logs -g <resource-group> -n loadtest
```

Stop it
```
az container delete -g <resource-group> -n loadtest
```

## Create Horizontal Pod Autoscaler
```
kubectl apply -f captureorder-hpa.yaml
```

Run the test again.
