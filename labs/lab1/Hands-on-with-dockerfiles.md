# Working containers with Dockerfile and Registries

In this lab you will be introduced the DockerFile and Docker Repositories.  In the previous lab we created a container, added a file and committed.  Although this was a good exercise to get us use to working with containers, in real systems this would quickly become tedious.  Instead there is a concept of a docker file, which is a script that enables use to define what container should look like.  

## Let's get started
1. If you haven't already clone ths repository at ```https://github.com/jsturtevant/docker-azure-fundamentals-workshop``` and move into the directory ```docker-azure-fundamentals-workshop\labs\lab1\src\hellodocker```.  You can optionally open the code in VS Code or your favorite editor.

    ``` 
    $ git clone https://github.com/jsturtevant/docker-azure-fundamentals-workshop.git
    $ cd docker-azure-fundamentals-workshop\labs\lab1\src\hellodocker
    ```

    This is a simple hello word in ASP.NET.  

2. Add a file in the root of the repository called ```Dockerfile``` (```touch Dockerfile``` or through your editor)
    
    Paste the follow into the file

    ```
    FROM microsoft/aspnetcore:1.1.1
    WORKDIR /app
    EXPOSE 80
    ENTRYPOINT ["dotnet", "WebFront.dll"]

    ## copy last as this is the most likely thing that will change
    ## reduces build time/disk usage/bandwidth
    COPY . .
    ```

3. Run the command:

    ```
    docker build -t my-aspnetcore .
    ```
    This will create the new image based on the public image ```my-aspnetcore```


4. Run your new docker container with ```docker run -d -p 8080:80 my-aspnetcore```

5. Navigate to localhost:8080

6. Create [Azure Container Registry](https://portal.azure.com/#create/Microsoft.ContainerRegistry):

![Create Azure Container Registry in portal](images/create-azure-container-registry.png)

7.  Get credentials

8. sign in 

9. push 

10. pull.