# Working containers with Registries

In this lab you will be introduced Docker Repositories.  In the previous lab we created a custom image using Asp.Net Core and a Dockerfile.  This is great for development but it's not useful if we can share that image with the rest of the team or our other environments.  In this lab we will create a private Container Registry in Azure and push our images to it.

## Let's get started

1. Using the Azure Cli 2.0 create a resource group:

    ```
    az group create --name dockerworkshop
    ```

2. Create a registry:

    ```
    az acr create -n workshopRegistry -g dockerworkshop -l eastus
    ```

7.  Assign an Azure Active Directory Service Principal to the registry using the ```id``` that was output in the previous command:

    ```
    az ad sp create-for-rbac --scopes /subscriptions/<your-registry-id>/resourcegroups/myresourcegroup/providers/Microsoft.ContainerRegistry/registries/workshopRegistry --role Owner --password <your-strong-password>
    ```

8. We will use the image we created in the previous lab.  First we will tage it to be something more meaning full:

    ```
    $ docker tag hellodocker myregistry.azurecr.io/dockerworkshop/hellodocker:1.0
    ```

    If we run ```docker images``` we should see dockerworkshop/hellodocker:1.0 listed.

9. Next we will authenticate and push to our repository:

    ```
    docker login myregistry.azurecr.io -u <service-principal-id> -p <your-strong-password>
    docker push myregistry.azurecr.io/dockerworkshop/hellodocker:1.0
    ```

    If you log into the Azure Portal and navigate to your registry you should see the image:


10. Finally you can remove your local image and pull it locally to verify it works:

    ```
    docker rmi myregistry.azurecr.io/dockerworkshop/hellodocker:1.0
    docker images
    ```

    Verify the image is no longer there.

    Now pull it from the registry and run it:

    ```
    docker pull myregistry.azurecr.io/dockerworkshop/hellodocker:1.0
    docker run -d -p 8080:80 myregistry.azurecr.io/dockerworkshop/hellodocker:1.0
    ```

     Navigate to navigate to ```localhost:8080``` to see your project. 


## Wrap up
To wrap up the command kill all the running containers and clean up.  Note the ```docker system prune``` should only be used in environments where you want to throw away old containers.

```
$ docker stop $(docker ps -q)  #on windows use: FOR /f "tokens=*" %i IN ('docker ps -q') DO docker stop %i
$ docker system prune
```

