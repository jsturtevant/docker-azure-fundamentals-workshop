# Docker Intro

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

    You are now at at an Ubuntu bash prompt.  The  ```-i```  flag starts an interactive container meaning that the prompt we enter will be from within the container. The  ```-t```  flag creates a pseudo-TTY that attaches  stdin  and  stdout.  The ```bin/bash``` tells the container to run the bash command when it starts up.

    Give it a try by running something like:

    ```
    $ apt-get install fortune 
    $ fortune
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

6. You can also create long running containers and inspect their logs.  Let's create a long running container and assign it a name

	```
	# Start a long running process
	$ docker run -d --name=my-ubuntu ubuntu /bin/sh -c "while true; do echo Hello world; sleep 1; done"
	
	# Collect the output of the job so far
	$ docker logs my-ubuntu
	
	# Kill the job
	$ docker kill my-ubuntu	
    ```	

7. Inspecting our container
	Lastly, we can take a low-level dive into our Docker container using the docker inspect command. It returns a JSON hash of useful configuration and status information about Docker containers.

	```
	$ docker inspect container_name
	```

### Credits
This lab was adapted from https://github.com/Lybecker/ContainerWorkshop.