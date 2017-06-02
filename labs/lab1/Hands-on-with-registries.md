# Working with Registries

In this lab you will be introduced Docker Repositories.  In the previous lab we created a custom image using Asp.Net Core and a Dockerfile.  This is great for development but it's not useful if we can share that image with the rest of the team or our other environments.  In this lab we will create a private Container Registry in Azure and push our images to it.

This is the third part in Lab1:

- [Hands on with Containers](Hands-on-with-containers.md)
- [Hands on with Dockerfiles](Hands-on-with-dockerfiles.md)
- Hands on with Registries (this file)


## Let's get started

0. If you don't have the Azure Cli 2.0 installed locally run the following command to download the docker image with it installed.  Steps with the ```#``` should be run with in the Azure Cli contianer.  You can open two terminals for this lab, one for the Docker Cli container and one for running Docker on your local machine.

    ```
    $ docker run -it azuresdk/azure-cli-python 
    ```

1. Before you continue please login into the Azure CLI and get your subscription Id.

    Login to the Azure CLI
    
    ```
    # az login
    ```

    You should see output similar to below. Follow the instructions printed on the command shell. When that process completes, the command shell completes the log in process. It might look something like:

    ```
    [
        {
            "cloudName": "AzureCloud",
            "id": "...",
            "isDefault": true,
            "name": "Free Trial",
            "state": "Enabled",
            "tenantId": "...",
            "user": {
            "name": "...",
            "type": "user"
            }
        }
    ] 
    ```

    Get your subscription Id
    
    ```
    # az account list
    ```

    This will print out a JSON block similar to the below one showing the accounts you are currently logged into. The subscription id is found in the id field.

    ```
    [
        {
            "cloudName": "AzureCloud",
            "id": ".....",
            "isDefault": true,
            "name": "Free Trial",
            "state": "Enabled",
            "tenantId": "....",
            "user": {
                "name": "....",
                "type": "user"
            }
        }
    ]
    ```

    Copy your subscription id because we will use it a few times.

    Confirm your Azure Subscription

    ```
    # az account set --subscription <YOUR SUB ID>.  
    ```

2. Using the Azure Cli 2.0 create a resource group:

    ```
    # az group create --name dockerworkshop --location eastus
    ```

3. Create a registry.  The registry name must be unique accross all of azure as it will be apart of the url to access your registry.  Give it a unique name with your intials at the end.  This step will take a few minutes.

    ```
    # az acr create -n workshopRegistry<your-initials> -g dockerworkshop -l eastus --sku Basic
    ```

4.  Assign an Azure Active Directory Service Principal to the registry using the ```id``` that was output in the previous command.  The previous command gives you the command that you can copy paste to create the AD Service Principal.  It will look similiar:

    ```
    # az ad sp create-for-rbac -n acrworkshop --scopes /subscriptions/<your-registry-id>/resourcegroups/dockerworkshop/providers/Microsoft.ContainerRegistry/registries/workshopRegistry<your-initials> --role Owner 
    ```

    After a few  moments you should see output similiar too:

    ```
    Retrying role assignment creation: 1/36
    {
    "appId": "7e837398-27e1-44cd-a52f-afb9671214d3",
    "displayName": "acrworkshop",
    "name": "http://azure-cli-2017-06-02-12-45-19",
    "password": "10d38737e-9d36-43f6-82d2-30a632ddde36",
    "tenant": "464858f3-e0d2-4c88-a99d-caa6229c81a6"
    }
    ```

    Take note of the ```appId``` and ```password``` in the output we will be using it in a later step.

> note: The rest of the commands in the lab should be run on your local machine. Not inside the docker container with the Azure Cli (step 0).

5. We will use the image we created in the previous lab.  First we will tage it to be something more meaning full:

    ```
    $ docker tag hellodocker  workshopRegistry<your-initials>.azurecr.io/dockerworkshop/hellodocker:1.0
    ```

    If we run ```docker images``` we should see ``` workshopRegistry<your-initials>/dockerworkshop/hellodocker:1.0``` listed.

6. Next we will authenticate and push to our repository.  Use the values from ```appId``` and ```password``` from the previous step:

    ```
    $ docker login  workshopRegistry<your-initials>.azurecr.io -u <service-principal-appId> -p <password> 
    $ docker push works workshopRegistry<your-initials>hopregistry.azurecr.io/dockerworkshop/hellodocker:1.0
    ```

7. Finally you can remove your local image and pull it locally to verify it works:

    ```
    $ docker rmi  workshopRegistry<your-initials>.azurecr.io/dockerworkshop/hellodocker:1.0
    $ docker images
    ```

    Verify the image is no longer there.

    Now pull it from the registry and run it:

    ```
    $ docker pull  workshopRegistry<your-initials>.azurecr.io/dockerworkshop/hellodocker:1.0
    $ docker run -d -p 8080:80  workshopRegistry<your-initials>.azurecr.io/dockerworkshop/hellodocker:1.0
    ```

     Navigate to navigate to ```localhost:8080``` to see your project. 

## Wrap up
To wrap up the command kill all the running containers and clean up.  Note the ```docker system prune``` should only be used in environments where you want to throw away old containers.

```
$ docker stop $(docker ps -q)  #on windows use: FOR /f "tokens=*" %i IN ('docker ps -q') DO docker stop %i
$ docker system prune
```

You can also login into the portal and remove the registry if you will not be using it in the future.

