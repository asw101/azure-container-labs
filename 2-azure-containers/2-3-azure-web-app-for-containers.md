## Deploy as a container to Azure Web Apps

Azure Web Apps also supports deployment through containers. This section of the lab will demonstrate
how to deploy from an Azure Container Registry (ACR).

To deploy a container to Azure Web apps, you need:

* A resource group
* An Azure Web Apps plan using Linux hosts (created with `--is-linux`)
* A Docker image available from a container registry

We'll be using the App Services Plan created in the last section, so if you skipped it, go back and run the setup code to get the group and set up the plan. For containers, a deployment user is not necessary.

> :heavy_exclamation_mark: __NOTE__
>
> If you completed the [Build and Containerize a Go Application](/1-app-hello-echo) lab, then you can skip the steps
> 1 and 2, and use that container registry instead. If you don't have the environment variables for it
> available in your current terminal session, you can get the registry name again with: 
>
> ```bash
> REGISTRY_NAME=$(az acr list | jq -r '[.[].name|select(. | startswith("acr2018"))][0]')
> ```

1. Create an ACR to deploy the container image to.

```bash
REGISTRY_NAME="acr2018${SUFFIX}"
az acr create -g $RESOURCE_GROUP -l $LOCATION --name $REGISTRY_NAME --sku Basic --admin-enabled true
```

2. Build and deploy the container image in the repository.

```bash
az acr build -r $REGISTRY_NAME -t hello-echo --file Dockerfile .
```

3. Create the application. The same restrictions apply for containerized applications as others, namely the
application name must be _universally unique_. The Azure CLI also does not yet support deployment directly
from ACR, so a "scratch" container will be used for the initial creation.

    ```bash
    APP_NAME="container-deploy-$SUFFIX"
    az webapp create -g $RESOURCE_GROUP -p deploy-lab -n $APP_NAME \
        --deployment-container-image-name alpine
   ```

4. Deploy the ACR image.

    ```bash
   ACR_PASS=$(az acr credential show -g $RESOURCE_GROUP -n $REGISTRY_NAME -o tsv --query 'passwords[0].value')
   az webapp config container set --name $APP_NAME --resource-group $RESOURCE_GROUP \
        --docker-custom-image-name $REGISTRY_NAME.azurecr.io/hello-echo \
        --docker-registry-server-url https://$REGISTRY_NAME.azurecr.io \
        --docker-registry-server-user $REGISTRY_NAME \
        --docker-registry-server-password $ACR_PASS 
    ```

5. Configure the port redirect settings.

    ```bash
    az webapp config appsettings set -g $RESOURCE_GROUP -n $APP_NAME --settings  WEBSITES_PORT=8080
    ```

6. Wait a few moments for the container to deploy and restart. Then send a request to the remote application:

    ```bash
    curl -X POST --data 'Hello world!' http://$APP_NAME.azurewebsites.net/echo
    ```

### Summary

If you have an application which doesn't require a lot of manual configuration, is already containerized, and is reachable from a Docker registry, it's a good candidate for deployment with web apps. Using a private registry is possible but requires additional configuration. You can also set up a web app to support multiple containers with orchestration, but it is recommended that you set up your own AKS instance since this is a preview feature and subject to change.

You cannot use a free-tier app service plan, or a plan which allows for Windows hosts, when deploying a container as a web app.
