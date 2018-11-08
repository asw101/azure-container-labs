# Deploy to Azure Container Instance (ACI) and Azure Kubernetes Service (AKS)

Azure Kubernetes Service (AKS) is Azure's fully managed Kubernetes service. Once we have deployed our instance, and an Azure Container Registry (ACR) for our Docker image, we can deploy our application to Kubernetes using its `kubectl` command-line tool.

## Login to Azure Portal and open Cloud Shell
Log into the Azure Portal to access the Subscription. Once you are logged, open [Cloud Shell](https://docs.microsoft.com/en-ca/azure/cloud-shell/quickstart) which provides a full bash environment to run the commands below.

## Clone the Azure Container Labs repo from GitHub

Once inside cloud shell, clone this repository using the below command and change to the azure-container-labs directory:

```bash
git clone https://github.com/asw101/azure-container-labs.git
cd azure-container-labs/
```

## Configure environment variables

Below we will set some environment variables which will be across the commands we run:

```bash
# RESOURCE_GROUP='Group-...'
RESOURCE_GROUP=$(az group list | jq -r '[.[].name|select(. | startswith("Group-"))][0]')
LOCATION='eastus'
if [ -z "$RANDOM_STR" ]; then RANDOM_STR=$(openssl rand -hex 3); else echo $RANDOM_STR; fi
# RANDOM_STR='9c445c'
## CONTAINER_IMAGE=$(...) # automatic: check existing RG
CONTAINER_REGISTRY=acr${RANDOM_STR}
CONTAINER_IMAGE='hello-echo:latest'
##KUBERNETES_SERVICE=$(...) # automatic: check shared RG
KUBERNETES_SERVICE=aks${RANDOM_STR}
```

## Optional setup

If you are running this lab *in your own subscription* and not in a workshop environment, you will need to run the following commands. Otherwise, continue to [Docker multi-stage build using Azure Container Registry (ACR) Build]().

If you choose to run the Azure CLI locally, rather than in Cloud Shell, you will need to run the `az login` command.

```bash
az login
```

Create a Resource Group

```bash
az group create --name $RESOURCE_GROUP --location $LOCATION
```

Create [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli#create-a-container-registry) and [Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough#create-aks-cluster)

```bash
az acr create -g $RESOURCE_GROUP -n $CONTAINER_REGISTRY --sku Basic --admin-enabled

az aks create -g $RESOURCE_GROUP -n $KUBERNETES_SERVICE --node-count 3 --generate-ssh-keys
```

Install the kubectl CLI tool and connect to your cluster

```bash
az aks install-cli
az aks get-credentials -g $RESOURCE_GROUP -n $KUBERNETES_SERVICE
```

[Grant Azure Kubernetes Service access to Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aks#grant-aks-access-to-acr)

```bash
# get the id of the service principal configured for AKS, get the ACR registry resource id, and create role assignment
CLIENT_ID=$(az aks show -g $RESOURCE_GROUP -n $KUBERNETES_SERVICE --query "servicePrincipalProfile.clientId" --output tsv)
CONTAINER_REGISTRY_ID=$(az acr show -g $RESOURCE_GROUP -n $CONTAINER_REGISTRY --query "id" --output tsv)
az role assignment create --assignee $CLIENT_ID --scope $CONTAINER_REGISTRY_ID --role Reader
```

## Docker multi-stage build using Azure Container Registry (ACR) Build

In our [Dockerfile](Dockerfile) we build our application in the `FROM golang:rc-alpine as builder` stage, and copy the `hello-echo` binary to a `FROM scratch` image in the second stage (note that we also build with `CGO_ENABLED=0` when using a `scratch` image).

## Create an Azure Container Registry

Now let's use the Azure CLI (az) to [create an Azure Container Registry]((https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli#create-a-container-registry)) and build an image in the cloud.

Note: Our Azure Container Registry's name must also be globally unique, so we have set `CONTAINER_REGISTRY` to 'acr' with a random alphanumeric suffix.

```bash
az acr create -g $RESOURCE_GROUP -n $CONTAINER_REGISTRY --sku Basic --admin-enabled
```

## Build application using Azure Container Registry Build

See: [Azure Container Registry Tutorial](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-quick-build)

To build using ACR Build, which will push our Dockerfile and code to the cloud, build the image, and store it in our Azure Container Registry, we use the [az acr build](https://docs.microsoft.com/en-us/cli/azure/acr?#az-acr-build) command:

```bash
az acr build -r $CONTAINER_REGISTRY -t $CONTAINER_IMAGE --file Dockerfile .
```

The fully-qualified name of the image in our Container Registry will then be `$CONTAINER_REGISTRY'.azurecr.io/hello-echo:latest'`. Let's output this with:

```bash
echo "${CONTAINER_REGISTRY}.azurecr.io/${CONTAINER_IMAGE}"
```

Congratulations, you have compiled your Go application and built your first image inside Azure Container Registry, using ACR Build and Docker multi-stage builds. This can be used used on any machine running Doocker, locally, or in the cloud, via `az acr login -n $REGISTRY_NAME` followed by `docker run -it $REGISTRY_NAME'.azurecr.io/hello-echo:latest'`.

## Prepare application for deployment to Kubernetes

Open [kubernetes-deploy.yaml](kubernetes-deployment.yaml) in a text editor and set `image: ...` to the fully qualified image name which can be found via:

```bash
echo "${CONTAINER_REGISTRY}.azurecr.io/${CONTAINER_IMAGE}"
```

Note: If you are running in Azure Cloud Shell you can edit `kubernetes-deployment.yaml` right in your browser with our integrated 'code' editor via: `code kubernetes-deployment.yaml`.

Create a Kubernetes Secret with the credentials neccessary to [pull an image from a private registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/). 

Note: this is *not* required if you have setup your own cluster under [Optional setup]() above.

```bash
CONTAINER_REGISTRY_PASSWORD=$(az acr credential show --name $CONTAINER_REGISTRY | jq -r .passwords[0].value)

kubectl create secret docker-registry regcred --docker-server="${CONTAINER_REGISTRY}.azurecr.io" \
    --docker-username=$CONTAINER_REGISTRY \
    --docker-password=$CONTAINER_REGISTRY_PASSWORD \
    --docker-email=user@example.org
```

Enable HTTP Application Routing (see: [Use HTTP Application Routing](https://docs.microsoft.com/en-us/azure/aks/http-application-routing))

```bash
az aks enable-addons --resource-group $RESOURCE_GROUP --name $KUBERNETES_SERVICE --addons http_application_routing

APPLICATON_ROUTING_ZONE=$(az aks show --resource-group $RESOURCE_GROUP --name $KUBERNETES_SERVICE | jq -r .addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName)
```

Open `kubernetes-service-ingress.yaml` and replace `...` in (`  - host: ...` under the `Ingress` object) with the result of:

```bash
echo "hello-echo-${RANDOM_STR}.${APPLICATON_ROUTING_ZONE}"
```

## Deploy application to Azure Kubernetes Service

Install the kubectl CLI tool and connect to your cluster (if required)

```bash
az aks install-cli
az aks get-credentials -g $RESOURCE_GROUP -n $KUBERNETES_SERVICE
```

Below we will run a series of `kubectl` commands to deploy our application, interact with our deployment and expose it on the public internet.

```bash
# create deployment with kubectl apply
kubectl apply -f kubernetes-deployment.yaml

# we use kubectl get deploy to check the status of the deployment
kubectl get deploy --watch
# note: ctrl + c to stop watch

# if we are running the kubectl on our local machine (not inside cloudshell) we can forward port 9090 on our local machine to 8080 so that we can interact with our application before we expose it to the world.
# kubectl port-forward deploy/hello-echo 9090:8080
# open http://localhost:9090
# ctrl + c to exit port-forward

# describe provides more details of our deployment
kubectl describe deploy hello-echo

# get pods will get all the running pods, including those in our deployment
kubectl get pods

> tweak sample to get the pod from the label name
# get the name of a specific pod by its label, and view its logs
POD=$(kubectl get pods -l app=hello-echo -o json | jq -r '.items[0].metadata.name')
kubectl logs $POD

# scale our deployment from one instance to two
kubectl scale deployment hello-echo --replicas=2
```

Next we will deploy a `Service` so that your application can be reached from the outside world.

Service + Azure HTTP Application Routing

```bash
# create kubernetes service, and an ingress with the annotation for HTTP Application Routing
kubectl apply -f kubernetes-service-ingress.yaml

# wait for our service to become available
kubectl get service --watch

# the public URL for our service will be 
SERVICE_FQDN="http://hello-echo-${RANDOM_STR}.${APPLICATON_ROUTING_ZONE}"

# test the service in the terminal via curl
curl $SERVICE_FQDN

# you may also open the following URL in your web browser
echo $SERVICE_FQDN
```

Kubernetes Service

```bash
> use http routing
# create service (of type LoadBalancer) that will expose our deployment to the world via an Azure Load Balancer witha  public IP 
kubectl apply -f kubernetes-service.yaml

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

Congratulations, you have successfully deployed your application to Kubernetes on Azure Kubernetes Service!

## Deploy application to Azure Container Instances (ACI)

We can deploy the same application as a single stand-alone container to Azure Container Instance below. See also: [Azure Container Instances Quickstart](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-quickstart#create-a-container).

Ensure you have run the [Configure environment variables]() section above, as we are re-using these below.

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

# get the fully qualified domain of our container instance, set --dns-name-label above
CONTAINER_INSTANCE_FQDN=$(az container show -g $RESOURCE_GROUP -n aci${RANDOM_STR} | jq -r .ipAddress.fqdn)

# test the service in the terminal via curl
curl "${CONTAINER_INSTANCE_FQDN}:8080"

# you may also open the following URL in your web browser
echo "http://${CONTAINER_INSTANCE_FQDN}:8080"
```

## Additional container labs

1. [Azure Container Instance](2-2-azure-container-instance.md)
1. [Azure Web App for Containers](2-3-azure-web-app-for-containers.md)
1. [Azure Web App](2-4-azure-web-app.md)

## Resources

- https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment