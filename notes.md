https://docs.microsoft.com/en-us/azure/app-service/tutorial-custom-container?pivots=container-linux

## Clone or download the sample app

    git clone https://github.com/Azure-Samples/docker-django-webapp-linux.git --config core.autocrlf=input
    cd docker-django-webapp-linux


## Build and test the image locally

    docker build --tag appsvc-tutorial-custom-image .
    docker run -p 8000:8000 appsvc-tutorial-custom-image


## Create a resource group

    az group create --name AppSvc-DockerTutorial-rg --location westeurope


## Push the image to Azure Container Registry

    acr_name="dockertestacr"

    az acr create --name "${acr_name}" --resource-group AppSvc-DockerTutorial-rg --sku Basic --admin-enabled true
    az acr credential show --resource-group AppSvc-DockerTutorial-rg --name "${acr_name}"

    docker login "${acr_name}".azurecr.io --username dockertestacr
    docker tag appsvc-tutorial-custom-image "${acr_name}".azurecr.io/appsvc-tutorial-custom-image:latest
    docker push "${acr_name}".azurecr.io/appsvc-tutorial-custom-image:latest
    az acr repository list -n "${acr_name}"


## Configure App Service to deploy the image from the registry

    web_app_name="django-test-webapp"

    az appservice plan create --name AppSvc-DockerTutorial-plan \
        --resource-group AppSvc-DockerTutorial-rg --is-linux
    
    az webapp create --resource-group AppSvc-DockerTutorial-rg \
        --plan AppSvc-DockerTutorial-plan \
        --name "${web_app_name}" \
        --deployment-container-image-name "${acr_name}".azurecr.io/appsvc-tutorial-custom-image:latest

    az webapp config appsettings set --resource-group AppSvc-DockerTutorial-rg \
        --name "${web_app_name}" \
        --settings WEBSITES_PORT=8000

    webapp_principal_id="$(az webapp identity assign --resource-group AppSvc-DockerTutorial-rg \
        --name "${web_app_name}" \
        --query principalId \
        --output tsv)"; echo "${webapp_principal_id}"

    subscription_id="$(az account show --query id --output tsv)"; echo "${subscription_id}"

    az role assignment create --assignee "${webapp_principal_id}" \
        --scope /subscriptions/${subscription_id}"/resourceGroups/AppSvc-DockerTutorial-rg/providers/Microsoft.ContainerRegistry/registries/${acr_name}" \
        --role "AcrPull"


## Deploy the image and test the app

    az webapp config container set --name "${web_app_name}" \
        --resource-group AppSvc-DockerTutorial-rg \
        --docker-custom-image-name "${acr_name}".azurecr.io/appsvc-tutorial-custom-image:latest\
         --docker-registry-server-url https://"${acr_name}".azurecr.io

Go to https://django-test-webapp.azurewebsites.net/


## Access diagnostic logs

    az webapp log config --name "${web_app_name}" \
        --resource-group AppSvc-DockerTutorial-rg \
        --docker-container-logging filesystem

    az webapp log tail --name "${web_app_name}" --resource-group AppSvc-DockerTutorial-rg


## Open SSH connection to container

Go to https://django-test-webapp.scm.azurewebsites.net/webssh/host

Or run this command:
    
    az webapp ssh --resource-group AppSvc-DockerTutorial-rg --name "${web_app_name}"


## Clean up resources

    az group delete --no-wait
