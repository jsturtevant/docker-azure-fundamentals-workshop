# Docker CLI Intro

In this lab you will be introduced the useful Docker CLI commands.  This will set the foundation for working with Docker in the rest of the labs.

This is the first part in Lab1:
    - Hands on with Containers (this file)
    - [Hands on with Dockerfiles](Hands-on-with-dockerfiles.md)
    - [Hands on with Registries](Hands-on-with-registries.md)

## Let's get started

We will start with the classic Hello World, Docker style:

1. Type the following command in the terminal: 

	```
	$ docker run hello-world
	```

	If Docker is configured correctly you should see an output like so.
	
	```
	Unable to find image 'hello-world:latest' locally
	latest: Pulling from library/hello-world
	b901d36b6f2f: Pull complete
	0a6ba66e537a: Pull complete
	Digest: sha256:517f03be3f8169d84711c9ffb2b3235a4d27c1eb4ad147f6248c8040adb93113
	Status: Downloaded newer image for hello-world:latest
	
	Hello from Docker.
	This message shows that your installation appears to be working correctly.
	
	To generate this message, Docker took the following steps:
	1. The Docker client contacted the Docker daemon.
	2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
	3. The Docker daemon created a new container from that image which runs the
		executable that produces the output you are currently reading.
	4. The Docker daemon streamed that output to the Docker client, which sent it
		to your terminal.
	
	To try something more ambitious, you can run an Ubuntu container with:
	$ docker run -it ubuntu bash
	
	Share images, automate workflows, and more with a free Docker Hub account:
	https://hub.docker.com
	
	For more examples and ideas, visit:
	https://docs.docker.com/userguide/
	```
	
	First the ```hello-world``` image is downloaded to your docker machine. Then ```hello-world``` container ran it's start up command and exited.  It is important to note that when a container's startup command (specified implicitly in ```hello-world```. We will learn to specify startup commands later) stops the container stops.
    
    It is the same as doing a ```pull``` operation.  The difference between ```pull``` and ```run``` is that ```run```, after retrieving the latest version (if not specified) the container is started.
	
2. Now let's try to ```pull``` the latest Ubuntu images to see the difference and have a bit more fun:

	```
	$ docker pull ubuntu
	```

	It will download the latest [Ubuntu](https://hub.docker.com/_/ubuntu/) image from [Docker Hub](https://hub.docker.com/).  Notice it did not run a container this time.

3. List the all the current images you have on your local machine:

	```
	$ docker images
	```

    Your output should look something simliar too:

    ```
    REPOSITORY                                             TAG                     IMAGE ID            CREATED             SIZE
    ubuntu                                                 latest                  6a2f32de169d        5 weeks ago         117 MB
    hello-world                                            latest                  48b5124b2768        4 months ago        1.84 kB
    ```

4. Since we only pulled the image local we now need to start up a container using the Ubuntu image:

	```
	$ docker run -it ubuntu /bin/bash
	```

    You are now at at an Ubuntu bash prompt.  The  ```-i```  flag starts an interactive container meaning that the prompt we enter will be from within the container. The  ```-t```  flag creates a pseudo-TTY that attaches  stdin  and  stdout.  The ```bin/bash``` tells the container the startup command to run the bash when it starts.

    Give it a try by running something like:

    ```
    # apt-get update
    # apt-get install fortune 
    # /usr/games/fortune
    ```
	
	At this point you are running commands inside the Ubuntu container.  To leave the container and get back to your local machine prompt you can use the escape sequence  ```Ctrl-p  +  Ctrl-q```.  This will detach the TTY without exiting the shell; The container will continue to exist in a stopped state once exited. 

5. To see a list of all the containers running:

	```	
	$ docker ps
	```

    You should see output similar too:

    ```
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    ef587c7c7b73        ubuntu              "/bin/bash"         23 seconds ago      Up 22 seconds                           heuristic
    ```

    ```docker ps``` shows all the running containers, in this case the Ubuntu bash prompt we just exited.  But we started and stopped a ```hello-world``` container earlier.  To see see all containers, both running and stopped, us the following command:

    ```	
	$ docker ps -a
	```

    You should see output similar too:

    ```
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
    ef587c7c7b73        ubuntu              "/bin/bash"         2 minutes ago       Up 2 minutes                                    heuristic
    2ccb5af5704b        hello-world         "/hello"            16 minutes ago      Exited (0) 16 minutes ago                       confident
    ```

    Notice that the status for the Ubuntu image is ```Up``` and the status for ```hello-world``` is ```Exited```.

7. We can also re-attach to any running containers.  Let's re-attach the the running Ubuntu image, add a file, then create a new custom image.  This will demonstrate how you work with containers and is useful in one off scenarios but not so much in        production.  We will see a better way to modify container contents in a later lab.  For now this will get you use to working with containers. 

    Use the output from the previous ```docker ps``` command to get the ```container id```.  In my case it is ```ef587c7c7b73``` but yours will be unique.  Note you only have to specify enough values from the id to be unique across containers running on your machine, in this case I used just ```ef5```:

    ```
    $ docker attach ef5
    ```

    You will now be running bash inside the container.  Next create a file and verify it's contents"

    ```
    root@ef587c7c7b73:/# echo "hello from ubuntu container" > hello.txt
    root@ef587c7c7b73:/# cat hello.txt
    

    hello from ubuntu container
    ```

    Exit the container using  ```Ctrl-p  +  Ctrl-q```.  We modified the contents of the container.  If we were to re-attach to the container the ```hello.txt``` file would still be there but if we were to create a new container from ```docker run -it ubuntu /bin/bash``` the file would not be there (**go ahead and give it a try**).

    What if we wanted that file to be there when started a new container?  We can 'commit' our changes and create a new image:

    ```
    $ docker commit ef5 my-custom-ubuntu

    $ docker images
    ```

    Here we use ```docker commit``` and passed the container id (found in previous step) and gave the image a new name.  Your output should look like:

    ```
    REPOSITORY                                             TAG                     IMAGE ID            CREATED             SIZE
    my-custom-ubuntu                                       latest                  47b846a5c1f6        18 seconds ago      118 MB
    ubuntu                                                 latest                  6a2f32de169d        5 weeks ago         117 MB
    hello-world                                            latest                  48b5124b2768        4 months ago        1.84 kB
    ```

    Finally we can now create a container from the new image and verify the contents:

    ```
    $ docker run -it my-custom-ubuntu /bin/bash
    root@ebfedfe1fcf5:/# cat hello.txt
    hello from ubuntu container
    ```

    Exit the container by using ```Ctrl-p  +  Ctrl-q```.

6. You can also create long running containers and inspect their logs.  Let's create a long running container and assign it a name

	```
	# Start a long running process
	$ docker run -d --name=my-ubuntu ubuntu /bin/sh -c "while true; do echo Hello world; sleep 1; done"
	
	# Collect the output of the job so far
	$ docker logs my-ubuntu
    ```	

    Exit the container by using ```Ctrl-p  +  Ctrl-q```.

8.  For containers that did not have an interactive prompt, like the ```my-ubuntu``` machine we created, we can still attach to them using ```docker exec```,  this is useful when you are in development but if you find yourself doing anything like this     in production you should probably reconsider your approach.  

    ```docker exec``` will attach to the running container in a way that is similar to ```ssh``` into a machine,  all of the process will still be running and you can use it to inspect the current state of the container.    Let's attach to the 

    ```
    $ docker exec -it my-ubuntu /bin/bash
    ```

    This will give you a prompt on the running machine, while the main process still runs.

    Exit the container by using ```Ctrl-p  +  Ctrl-q```.

7. Lastly, we can take a low-level dive into our Docker container using the docker inspect command. It returns a JSON hash of useful configuration and status information about Docker containers.  We won't go into detail here but it is a useful command to know.  Take a look at it. What information to you recognize?

	```
	$ docker inspect my-ubuntu
	```

## Wrap up
To wrap up the command kill all the running containers and clean up.  Note the ```docker system prune``` should only be used in environments where you want to throw away old containers.

```
$ docker stop $(docker ps -q)  #on windows use: FOR /f "tokens=*" %i IN ('docker ps -q') DO docker stop %i
$ docker system prune
```

## Next Part 
Proceed to next lab which will show you how to work with [Dockerfiles](Hands-on-with-dockerfiles.md).

## Credits
This lab was adapted from https://github.com/Lybecker/ContainerWorkshop.