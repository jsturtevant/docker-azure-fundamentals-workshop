# Working with Dockerfiles

In this lab you will be introduced the [DockerFile](https://docs.docker.com/engine/reference/builder/) and [Docker Compose](https://docs.docker.com/compose/).  In the previous lab we created a container, added a file and committed.  Although this was a good exercise to get us use to working with containers, in real systems this would quickly become tedious.  Instead there is a concept of a Dockerfile, which is a script that enables use to define what container should look like.  

We will use a ```Dockerfile``` to build and run an [Asp.Net Core](https://www.microsoft.com/net/core/platform) application.  Note that you will **not** need anything other than docker installed to do this lab.

This is the second part in Lab1:

- [Hands on with Containers](Hands-on-with-containers.md)
- Hands on with Dockerfiles (this file)
- [Hands on with Registries](Hands-on-with-registries.md)

## Let's get started
1. If you haven't already clone ths repository at https://github.com/jsturtevant/docker-azure-fundamentals-workshop and move into the directory ```docker-azure-fundamentals-workshop\labs\lab1\src\hellodocker```.  You can optionally open the code in VS Code or your favorite editor.

    ``` 
    $ git clone https://github.com/jsturtevant/docker-azure-fundamentals-workshop.git
    $ cd docker-azure-fundamentals-workshop\labs\lab1\src\hellodocker
    ```

    This is a simple hello word in ASP.NET.  

2. Add a file in the root of the repository called ```Dockerfile``` (```touch Dockerfile``` or through your editor)
    
    Paste the follow into the file

    ```
    FROM microsoft/aspnetcore:1.1.2

    WORKDIR /app
    EXPOSE 80
    ENTRYPOINT ["dotnet", "hellodocker.dll"]

    ## copy last as this is the most likely thing that will change
    ## reduces build time/disk usage/bandwidth
    COPY ./bin/Release/netcoreapp1.1/publish .
    ```

    You will notice that this copies the files at ```bin/Release/netcoreapp1.1/``` to the container.  We have not yet built our project so those file will not be there.  We could use the command line to restore the packages, build the source and then use the above Dockerfile.
    
    > Note: *if you don't have dotnet core installed skip to step 3.*  Read the steps below to understand how it would work.
    
    ```
    $ dotnet restore
    $ dotnet publish -c Release
    ```  

    Finally we can build the image by running:

    ```
    $ docker build -t hellodocker .
    ```

    Run your new docker container with ```docker run -d -p 8080:80 hellodocker``` and navigate to navigate to http://localhost:8080.

    Stop your container with ```docker stop <contianerid>``` 

3. The above steps required you to have [.Net Core](https://www.microsoft.com/net/core/platform) installed.  It also required several manual steps. To automate the process and also eliminate the need for external dependencies (very helpful on a build machine) we can introduce a second container that will have the dependencies and tooling installed needed for building .NET Core.  We will do this using a compose file.

     Add a file in the root of the repository called ```docker-compose.build.yml``` (```touch docker-compose.build.yml``` or through your editor) and paste the following in:

    ```
    version: '2'

    services:
        build-image:
            build: .
            image: hellodocker

        build:
            image: microsoft/aspnetcore-build:1.1.2
            volumes:
                - .:/src

            working_dir: /src
            command: /bin/bash -c "dotnet restore  && dotnet publish -c Release"

    ```

    This uses a compose file to add two new services that help automate the build process.  The ```build-image``` service simply uses the Dockerfile that we already created.  The second service ```build``` uses asp.net core build image that has all the dependencies needed to build a asp.net core project. If you do not have asp.net core installed this will still enable you to build the project using this docker image.

4. To use the compose file to build the project:

    ```
    $ docker-compose -f docker-compose.build.yml up build
    ```

    You should see output similar too:

    ```
    Creating network "hellodocker_default" with the default driver
    Creating hellodocker_build_1
    Attaching to hellodocker_build_1
    build_1        |   Restoring packages for /src/hellodocker.csproj...
    build_1        |   Generating MSBuild file /src/obj/hellodocker.csproj.nuget.g.props.
    build_1        |   Writing lock file to disk. Path: /src/obj/project.assets.json
    build_1        |   Restore completed in 994.66 ms for /src/hellodocker.csproj.
    build_1        |
    build_1        |   NuGet Config files used:
    build_1        |       /root/.nuget/NuGet/NuGet.Config
    build_1        |
    build_1        |   Feeds used:
    build_1        |       https://api.nuget.org/v3/index.json
    build_1        | Microsoft (R) Build Engine version 15.1.1012.6693
    build_1        | Copyright (C) Microsoft Corporation. All rights reserved.
    build_1        |
    build_1        |   hellodocker -> /src/bin/Release/netcoreapp1.1/hellodocker.dll
    hellodocker_build_1 exited with code 0
    ```

    This will build the project ***inside the ```microsoft/aspnetcore-build:1.1.2```*** container.  There is a volume map that maps the current directory (all of the asp.net source) to the containers ```/src``` folder.  When the build is finished you should find the resulting build at on your computer ```docker-azure-fundamentals-workshop\labs\lab1\src\hellodocker\bin\Release\netcoreapp1.1\publish```.  

5. Now you can build the image and run your project:

    ```
    $ docker-compose -f docker-compose.build.yml build build-image
    $ docker run -d -p 8080:80 hellodocker
    ```

    This step is equivalent to running ```docker build -t hellodocker .``` manually.  The advantage of putting inside the compose file is that you now can track the image version name and everything is in one place.

    Navigate to navigate to http://localhost:8080 to see your project. 

## Wrap up
To wrap up the command kill all the running containers and clean up.  Note the ```docker system prune``` should only be used in environments where you want to throw away old containers.

```
$ docker-compose -f docker-compose.build.yml down
$ docker stop $(docker ps -q)  #on windows use: FOR /f "tokens=*" %i IN ('docker ps -q') DO docker stop %i
$ docker system prune
```

## Next Part 
Proceed to next lab which will show you how to work with [registries](Hands-on-with-registries.md).