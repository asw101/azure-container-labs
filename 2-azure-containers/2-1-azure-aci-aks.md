# Deploy to Azure Container Instance (ACI) and Azure Kubernetes Service (AKS)

The Azure Kubernetes Service (AKS) allows for spinning up a Kubernetes instance to deploy your application infrastructure to. Once this instance is available, you can deploy your application to Kubernetes using the `kubectl` command-line tool as you would with any other Kubernetes instance.

## Setup

1. Configure our environment variables and login to Azure CLI (if neccessary):

```bash
# RESOURCE_GROUP='Group-...'
RESOURCE_GROUP=$(az group list | jq -r '[.[].name|select(. | startswith("Group-"))][0]')
LOCATION='eastus'
if [ -z "$RANDOM_STR" ]; then RANDOM_STR=$(openssl rand -hex 3); else echo $RANDOM_STR; fi
# RANDOM_STR='9f889c'
CONTAINER_REGISTRY=acr${RANDOM_STR}
CONTAINER_IMAGE='spring-music:v1'
KUBERNETES_SERVICE=aks${RANDOM_STR}

# If we're not yet logged in:
az login

# Create the resource group, if required:
az group create --name $RESOURCE_GROUP --location $LOCATION
```

2. Create [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli#create-a-container-registry) and [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#create-aks-cluster)

```bash
az acr create -g $RESOURCE_GROUP -n $CONTAINER_REGISTRY --sku Basic

az aks create -g $RESOURCE_GROUP -n $KUBERNETES_SERVICE --node-count 3 --generate-ssh-keys

# install cli and connect to aks cluster
az aks install-cli
az aks get-credentials -g $RESOURCE_GROUP -n $KUBERNETES_SERVICE
```

3. [Grant Azure Kubernetes Service access to Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aks#grant-aks-access-to-acr)


```bash
# get the id of the service principal configured for AKS, get the ACR registry resource id, and create role assignment
CLIENT_ID=$(az aks show -g $RESOURCE_GROUP -n $KUBERNETES_SERVICE --query "servicePrincipalProfile.clientId" --output tsv)
CONTAINER_REGISTRY_ID=$(az acr show -g $RESOURCE_GROUP -n $CONTAINER_REGISTRY --query "id" --output tsv)
az role assignment create --assignee $CLIENT_ID --scope $CONTAINER_REGISTRY_ID --role Reader
```

## Build application using Azure Container Registry Build

See: [Azure Container Registry Tutorial](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-quick-build)

```bash
az acr build --registry $CONTAINER_REGISTRY --image $CONTAINER_IMAGE .
```

## Deploy application to Azure Container Instances (ACI)

See: [Azure Container Instances Quickstart](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-quickstart#create-a-container)

```bash
# enable admin credentials
az acr update -n $CONTAINER_REGISTRY --admin-enabled true
CONTAINER_REGISTRY_PASSWORD=$(az acr credential show -n $CONTAINER_REGISTRY | jq -r .passwords[0].value)

# create container instance
az container create --resource-group $RESOURCE_GROUP --location $LOCATION \
    --name aci${RANDOM_STR} \
    --image "${CONTAINER_REGISTRY}.azurecr.io/${CONTAINER_IMAGE}" \
    --registry-login-server "${CONTAINER_REGISTRY}.azurecr.io" \
    --registry-username $CONTAINER_REGISTRY \
    --registry-password $CONTAINER_REGISTRY_PASSWORD \
    --cpu 1 \
    --memory 1 \
    --ports 8080 \
    --dns-name-label aci${RANDOM_STR}

# show container events
az container show -g $RESOURCE_GROUP -n aci${RANDOM_STR} | jq .containers[0].instanceView.events[]

CONTAINER_INSTANCE_FQDN=$(az container show -g $RESOURCE_GROUP -n aci${RANDOM_STR} | jq -r .ipAddress.fqdn)

curl "${CONTAINER_INSTANCE_FQDN}:8080"

open "http://${CONTAINER_INSTANCE_FQDN}:8080"
```

## Deploy application to Azure Kubernetes Service

1. Set `image: ...` in [kubernetes-deploy.yaml](kubernetes-deployment.yaml) to the fully qualified image name which can be found via:

```bash
echo "${CONTAINER_REGISTRY}.azurecr.io/${CONTAINER_IMAGE}"
```

2. Deploy the application to Azure Kuberentes Service:

```bash
# create deployment
kubectl apply -f kubernetes-deployment.yaml

kubectl get deploy --watch

kubectl port-forward deploy/hello-echo 9090:8080

kubectl describe deploy hello-echo

kubectl get pods

kubectl logs ...

kubectl scale deployment hello-echo --replicas=1

# create service
kubectl apply -f kubernetes-service.yaml

kubectl port-forward service/hello-echo 9090:80

kubectl get service --watch

IP_ADDRESS=$(kubectl get service hello-echo -o json | jq -r .status.loadBalancer.ingress[0].ip)

curl $IP_ADDRESS

open "http://${IP_ADDRESS}"
```

## Resources

- https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment
