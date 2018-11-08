# Deploy to Azure Container Instances (ACI)

If you have a container that you want to deploy like a web application, but have more control over its resource allocation and scaling, ACI may be the right option. Using ACI does __not__ require setting up any App Services plans.

The requirements for deploying with ACI are:

* An Azure resource group
* A Docker image available from a registry

For this lab you will use the ACR created in the last step, or if you did the previous [Build and Containerize a Go Application](/1-app-hello-echo) lab, the ACR from that. If you have not set up your environment or created an ACR, go back to the previous steps and create one.

1. Create the container instance and deploy. Unlike application names, ACI names only need to be unique within their resource group.

    ```bash
    az container create -g $RESOURCE_GROUP --name aci-deploy --image $REGISTRY_NAME.azurecr.io/hello-echo --registry-password $ACR_PASS --registry-username $REGISTRY_NAME --ip-address public --ports 8080
    ```

    Note that even if the container image specifies that certain ports should be exposed, they must additionally be exposed in the ACI creation. Container creation can take a few minutes while the instance spins up and runs the image.

2. Get the container's public IP address. You may need to wait a few moments for it to become available.

    ```bash
    PUBLIC_IP=$(az container show -g $RESOURCE_GROUP -n aci-deploy --query 'ipAddress.ip' --out tsv)
    ```

3. Connect to the running container.

    ```bash
    curl -X POST --data 'Hello world!' http://$PUBLIC_IP:8080/echo
    ```

### Summary

If you have a container that you want more direct management over, ACI may be the right deploy method for you. This gives you more flexibility with the resources consumed and the costs incurred, at the expense of requiring user intervention instead of selecting a pre-set plan from app services. Using ACI also requires a little more knowledge about the details of scaling on Azure to effectively manage, but is better for applications which need more than one port or which you want to keep private within the Azure infrastructure.
