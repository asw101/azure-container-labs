## Deploy directly to Azure Web Apps

Azure Web Apps allows you to deploy an application in a manner that requires little configuration on your behalf, and which automatically scales both "up" (the amount of resources available to an individual host) and "out" (the number of application hosts.) All you need to use Web Apps are the appropriate accounts on Azure, and your source code. The code can be
hosted in almost any location with the number of deployment options available.

The prerequisites for deploying to Azure Web Apps are:

* An Azure resource group
* An App Services plan

1. We've created a resource group for this lab, which you can get with the following CLI command:

    ```bash
    RESOURCE_GROUP=$(az group list | jq -r '[.[].name|select(. | startswith("Group-"))][0]')
    LOCATION='eastus'
    SUFFIX=$(LC_CTYPE=C tr -cd 'a-z0-9' < /dev/urandom | head -c 5)
    ```

2. Create an app services plan

    ```bash
    az appservice plan create -n deploy-lab -g $RESOURCE_GROUP --is-linux
    ```

3. Select the Web App deployment user with `az webapp deployment set`. If a user has not been created with the given user name in your tenant, one is created. The user name must be _universally unique_.

    ```bash
    WEBAPP_PASS=$(LC_CTYPE=C tr -cd 'A-Za-z0-9' < /dev/urandom | head -c 16)
    WEBAPP_USER="gopher$SUFFIX"
    az webapp deployment user set --user-name $WEBAPP_USER --password $WEBAPP_PASS
    ```

2. Create the application. This doesn't deploy it, but sets up the infrastructure in Azure and provides deployment information. For this lab, use `--deployment-local-git` which sets up a git repository on a resource in Azure that is used to build and deploy the application.

    Azure Web Apps requires that you provide a _universally unique_ application name inside of Azure, because of how applications are hosted and exposed. 

    ```bash
    APP_NAME="git-deploy-$SUFFIX"
    ```
    
    > :heavy_exclamation_mark: __IMPORTANT__
    > 
    > As of 8/21/18 the Go runtime for Azure Web Apps is currently in a closed preview, so it requires that there be an existing webapp runtime deployed
    > before being configured. This is why the runtime is initially set to `node|8.1`, to create the web app infrastructure that can
    > be reconfigured. When Azure Web Apps for Go is made publicly available, this restriction will be lifted.

    ```bash
    az webapp create --resource-group $RESOURCE_GROUP --plan deploy-lab \
        --name $APP_NAME --deployment-local-git --runtime "node|8.1"
    az webapp config set -g $RESOURCE_GROUP -n $APP_NAME --linux-fx-version "go|1"
    ```

3. Set up the redirect to port 80. Azure web apps only allow connections on ports 80 and 443.

    ```bash
    az webapp config appsettings set -g $RESOURCE_GROUP -n $APP_NAME --settings WEBSITES_PORT=8080
    ```

4. Clone the app to deploy.

    ```bash
    git clone https://github.com/aaronmsft/hello-echo.git
    ```

4. Push to the git remote for deployment. There's a local clone of the application already on
    the lab machine in the directory you're using in the directory `app`, so you can just add the remote and push.
    
    There is a hook which builds every time a push completes, and then deploys to all running instances if the build succeeds. This deploy operation is guaranteed to not interrupt availability.

    ```bash
    DEPLOY_URL=$(az webapp deployment source config-local-git -n $APP_NAME -g $RESOURCE_GROUP -o tsv)
    GIT_URL=$(echo $DEPLOY_URL | sed -e "s/$WEBAPP_USER/$WEBAPP_USER:$WEBAPP_PASS/")
    cd hello-echo
    git remote add $APP_NAME $GIT_URL 
    git push $APP_NAME master
    cd ..
    ```

5. Give a few moments for the build to complete, and then try sending an HTTP request to the endpoint.

    ```bash
    curl -X POST --data 'Hello world!' http://$APP_NAME.azurewebsites.net/echo
    ```

### Summary

Azure Web Apps is an easy way to deploy from a number of sources, but you do lose fine-grained control over the deployment. If you have a simple, lightweight API or a frontend application to deploy that still needs to scale, Azure Web Apps is a natural choice.
