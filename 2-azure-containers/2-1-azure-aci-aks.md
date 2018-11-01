# Deploy to Azure Container Instance (ACI) and Azure Kubernetes Service (AKS)

The Azure Kubernetes Service (AKS) allows for spinning up a Kubernetes instance to deploy your application infrastructure to. Once this instance is available, you can deploy your application to Kubernetes using the `kubectl` command-line tool as you would with any other Kubernetes instance.

# Login to Azure Portal and Configure Cloud Shell
Log into the Azure Portal to access the Subscription. Once you are logged in you will access and setup Cloud Shell to get a Bash environment to run the commands.
> In this session you can use [Cloud Shell](https://docs.microsoft.com/en-ca/azure/cloud-shell/quickstart) ...

# Clone the Azure Container Labs repo from GitHub:
> Azure-Samples/azure-container-labs or similar

git clone https://github.com/asw101/azure-container-labs.git

cd azure-container-labs/

# Configure environment variables

Configure our environment variables and login to Azure CLI (if neccessary):

```bash
# RESOURCE_GROUP='Group-...'
RESOURCE_GROUP=$(az group list | jq -r '[.[].name|select(. | startswith("Group-"))][0]')
LOCATION='eastus'
if [ -z "$RANDOM_STR" ]; then RANDOM_STR=$(openssl rand -hex 3); else echo $RANDOM_STR; fi
# RANDOM_STR='9f889c'
## CONTAINER_IMAGE=$(...) # automatic: check existing RG
CONTAINER_REGISTRY=acr${RANDOM_STR}
CONTAINER_IMAGE='hello-echo:latest'
##KUBERNETES_SERVICE=$(...) # automatic: check shared RG
KUBERNETES_SERVICE=aks${RANDOM_STR}
```

# Setup (optional)
> If you are working in a "workshop" environment, you won't need to do this.

# If we're not yet logged in:
az login

# Create the resource group, if required:
az group create --name $RESOURCE_GROUP --location $LOCATION

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


# Docker multi-stage build using Azure Container Registry (ACR) Build

In our [Dockerfile](Dockerfile) we are building our application in the `FROM golang:rc-alpine as builder` stage, and copying the `hello-echo` binary to a `FROM scratch` image in the second stage (note that we also build with `CGO_ENABLED=0` when using a `scratch` image).

> **Note: Our Azure Container Registry's name must also be globally unique, so we are also setting CONTAINER_REGISTRY to 'acr2018' with 5 character random suffix.**

## Create an Azure Container Registry

Now let's use the Azure CLI (az) to [create an Azure Container Registry]((https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli#create-a-container-registry)) and build an image in the cloud.

```bash
az acr create -g $RESOURCE_GROUP -n $CONTAINER_REGISTRY --sku Basic --enable-admin
```

## Build application using Azure Container Registry Build

See: [Azure Container Registry Tutorial](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-quick-build)

To build using ACR Build, which will push our Dockerfile and code to the cloud, build the image, and store it in our Azure Container Registry, we use the [az acr build](https://docs.microsoft.com/en-us/cli/azure/acr?#az-acr-build) command:

    az acr build -r $CONTAINER_REGISTRY -t $CONTAINER_IMAGE --file Dockerfile .

The fully-qualified name of the image in our Container Registry will then be `$CONTAINER_REGISTRY'.azurecr.io/hello-echo:latest'`. Let's output this with:

    echo "${CONTAINER_REGISTRY}.azurecr.io/${CONTAINER_IMAGE}"

Congratulations, you have compiled your Go application and built your first image inside Azure Container Registry, using ACR Build and Docker multi-stage builds. This can be used on local and remote machines via `az acr login -n $REGISTRY_NAME` and `docker run -it $REGISTRY_NAME'.azurecr.io/hello-echo:latest'`.


## Deploy application to Azure Kubernetes Service

1. Set `image: ...` in [kubernetes-deploy.yaml](kubernetes-deployment.yaml) to the fully qualified image name which can be found via:

```bash
echo "${CONTAINER_REGISTRY}.azurecr.io/${CONTAINER_IMAGE}"
```

> If you are running in Azure Cloud Shell you can open it with our integrated 'code' editor via: ```code kubernetes-deployment.yaml```

> 1. Create a imagePullSecret that will allow AKS to pull from ACR
> Note: this is not required if you are using your own cluster with the xxx above.

```bash
CONTAINER_REGISTRY_PASSWORD=$(az acr credential show --name $CONTAINER_REGISTRY | jq -r .passwords[0].value)

kubectl create secret docker-registry regcred --docker-server=$CONTAINER_REGISTRY'.azurecr.io' \
    --docker-username=$CONTAINER_REGISTRY \
    --docker-password=$CONTAINER_REGISTRY_PASSWORD \
    --docker-email=user@example.org
```

2. Deploy the application to Azure Kuberentes Service:

```bash
# create deployment
kubectl apply -f kubernetes-deployment.yaml

kubectl get deploy --watch
# note: ctrl + c to stop watch

# > note: works locally
# kubectl port-forward deploy/hello-echo 9090:8080

kubectl describe deploy hello-echo

kubectl get pods

# get the first pod within our deployment and show the logs
> tweak sample to get the pod from the label name
export POD=$(kubectl get pods -l app.kubernetes.io/name=jaeger-operator --output name)
kubectl logs $POD

# scale our deployment from one instance to two
kubectl scale deployment hello-echo --replicas=2

> use http routing
# create service (of type LoadBalancer) that will expose our deployment to the world via an Azure Load Balancer witha  public IP 
kubectl apply -f kubernetes-service.yaml

> remove (or comment)
# NO longer needed if we do HTTP APP ROUTING
#kubectl port-forward service/hello-echo 9090:80

# wait for our service to become available
kubectl get service --watch

> change this to fqdn for ingress
# get the public IP of our service
IP_ADDRESS=$(kubectl get service hello-echo -o json | jq -r .status.loadBalancer.ingress[0].ip)

# test the service in the terminal via curl
curl $IP_ADDRESS

# you may also open the following URL in your web browser
echo "http://${IP_ADDRESS}"

```

## Deploy application to Azure Container Instances (ACI)

See: [Azure Container Instances Quickstart](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-quickstart#create-a-container)

```bash
# get our container registry password
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

# test the service in the terminal via curl
curl "${CONTAINER_INSTANCE_FQDN}:8080"

# you may also open the following URL in your web browser
echo "http://${CONTAINER_INSTANCE_FQDN}:8080"
```

## Resources

- https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment
