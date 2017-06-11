# Docker Azure Fundamentals Workshop

## Description 
Welcome to the Container Workshop where you will get hands-on with containers. We will start the day with running Docker containers locally and by the end of the day having them running in a deploying them to the Azure Cloud.

## Requirements

- Azure Subscription
   - [Free Trial](https://azure.microsoft.com/en-us/free/)
   - [Visual Studio Dev Essentials](https://azure.microsoft.com/en-us/pricing/member-offers/vs-dev-essentials/)
- Docker (choose your platform below)
   - [Docker for Windows](https://docs.docker.com/docker-for-windows/install/) 
   - [Docker for Mac](https://docs.docker.com/docker-for-mac/install/)
   - [Linux machine with Docker install](https://docs.docker.com/engine/installation/#supported-platforms) - choose your linux flavor
   - If none of these are an option you can [create Docker VM](https://github.com/Azure/azure-quickstart-templates/tree/master/docker-simple-on-ubuntu) by clicking ```Deploy to Azure``` button

The following are optional:

- [Azure Cli 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) (optional)
- [Bash On Ubuntu On Windows](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) (optional)
- [VS Code](https://code.visualstudio.com/) with [Docker plugin](https://marketplace.visualstudio.com/items?itemName=PeterJausovec.vscode-docker) (optional)

## Agenda 
### Introduction Docker Containers ([presentation](https://github.com/jsturtevant/docker-azure-fundamentals-workshop/blob/master/presentations/Introduction-to-Docker.pptx))
- Containers 101
- DockerFile’s 
- Repository’s 
   
### Hands on with Containers Lab 
- [Hands on with Containers](labs/lab1/Hands-on-with-containers.md)
- [Hands on with Dockerfiles](labs/lab1/Hands-on-with-dockerfiles.md)
- [Hands on with Registries](labs/lab1/Hands-on-with-registries.md)
    
### Creating Effective Docker Images ([presentation](https://github.com/billpratt/docker-azure-workshop/blob/master/slides/CreatingEffectiveDockerImages.pptx))

### Orchestrators ([presentation](https://github.com/jsturtevant/docker-azure-fundamentals-workshop/blob/master/presentations/Orchestrators.pptx))
- Intro 
- Swarm/Kubernetes/Service Fabric 
- Swarm API 
    
### Orchestrators Hands On 
- [Create Kubernetes Cluster with Azure Container Service](labs/orchestrators/deploy-kubernetes-acs.md)
- Bonus: 
    - [Create Swarm Cluster In Azure](labs/orchestrators/deploy-docker-swarm.md)

## Credits

Some of the labs for this workshop were adapted from https://github.com/Lybecker/ContainerWorkshop and https://github.com/tripdubroot/ContainerCamp.  
