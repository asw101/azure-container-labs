# Deploy your application to Azure

In this short lab, you're given a small echo server to deploy onto Azure. You'll learn how to
deploy an application with the Azure CLI as a standalone Azure Web App, in a container for Azure Web Apps for Containers, and deploying to Azure Container Instances. Along the way you'll learn the advantages and disadvantages of each deployment model. If you're willing to spend some more time with the lab, there is also a longer, individual article about deploying to Azure Kubernetes Service (AKS).

## The sample application

The sample application is contained in this directory as a `main.go` file. You can test it by running
locally and sending any HTTP request with a body to `localhost:8080/echo`. This echo server repeats back the POST body of the request, and then prints it locally on the server to stdout.

```bash
go run main.go &
curl -X POST --data 'Hello world!' http://localhost:8080/echo
kill %1
```

In the output, you should see `Hello world!` echoed back at you.

This is the same application that is used in one of the other labs, [Build and Containerize a Go Application](/1-app-hello-echo).
If you have already built and deployed the container to Azure Container Registry (ACR) from this earlier lab,
you'll be able to skip some steps here.

## Deploy to:

1. [Azure Container Instance and Azure Kubernetes Service](2-1-azure-aci-aks.md)
1. [Azure Container Instance](2-2-azure-container-instance.md)
1. [Azure Web App for Containers](2-3-azure-web-app-for-containers.md)
1. [Azure Web App](2-4-azure-web-app.md)
