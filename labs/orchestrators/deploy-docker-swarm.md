# Docker Swarm Mode with Azure
In this lab you will play around with the container orchestration features of Docker. You will deploy a simple application to a single host and learn how that works. Then, you will configure Docker Swarm Mode, and learn to deploy the same simple application across multiple hosts. You will then see how to scale the application and move the workload across different hosts easily.

> **Tasks**:
>
> * [Section #0 - Prerequisites/Creating VMs](#prerequisites)
> * [Section #1 - What is Orchestration](#basics)
> * [Section #2 - Configure Swarm Mode](#start-cluster)
> * [Section #3 - Deploy applications across multiple hosts](#multi-application)
> * [Section #4 - Drain a node and reschedule the containers](#drain)
> * [Cleaning Up](#cleanup)
> * [Next Steps/Learn More](#nextsteps)

## Document conventions

When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value. 

For instance if you see `ssh <username>@<hostname>` you would actually type something like `ssh sysadmin@swarm-leader.ivaf2i2atqouppoxund0tvddsa.jx.internal.cloudapp.net`

# <a name="prerequisites"></a>Section 0: Prerequisites/Creating VMs

For this lab we will need to create three virtual machines referred to as __swarm-leader__, __swarm-node-1__ and __swarm-node-2__. We will use Azure and a custom bash script that uses the Azure CLI to create all the VMs in one command.

## Step 0.1 - Install Azure CLI 2.0

You will need to install [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) for this lab.

> Note for Windows users: Please run all the commands from the ```Git Bash``` prompt that is installed with [Git For Windows](https://git-for-windows.github.io/).  You can download it from the page if you do not have it.  Use all the defaults when installing.

## Step 0.2 - Login to the Azure CLI

Before you continue please login into the Azure CLI and get your subscription Id.

**Login to the Azure CLI**

    az login

You should see output similar to below. Follow the instructions printed on the command shell.
When that process completes, the command shell completes the log in process. It might look something like:

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

**Get your subscription Id**

    az account list

This will print out a JSON block similar to the below one showing the accounts you are currently logged into. The subscription id is found in the **id** field.

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

Copy your subscription id because we will use it a few times.
    
**Confirm your Azure Subscription**

    az account set --subscription <YOUR SUB ID>

## Step 0.3 - Create docker hosts with Azure Virtual Machines

This topic uses three VMs, but you can use any number you want. 

**Build Swarm VMs**

Clone this repository and move to the deployment folder:

    $ git clone https://github.com/billpratt/docker-azure-workshop.git
    $ cd docker-azure-workshop/deployment

Run the deployment script and you will be prompted for your ```Subscription Id or Name```, ```Resource Group Name```, and ```Admin Password```.:

    $ . ./deploy.sh
    
    Subscription Id:
    <YOUR SUB ID>
    
    ResourceGroupName:
    dockerswarm

    
    Enter an admin password for vms
    Admin Password:
    <YOUR PASSWORD>

**Important** Use a password you will remember because you will use it to SSH into the VMs later.

This will take 5-10 minutes.  

When you're done you should be able to use **az vm list** to see your Azure VMs and get the public IPs to SSH into:

    $ az vm list --resource-group <Resource Group Name Used> --output table --show-details

    Name          ResourceGroup    PowerState    PublicIps      Location
    ------------  ---------------  ------------  -------------  ----------
    swarm-leader  dockerswarm      VM running    11.11.111.111  eastus
    swarm-node-1  dockerswarm      VM running    11.11.111.112  eastus
    swarm-node-2  dockerswarm      VM running    11.11.111.113  eastus
    swarm-node-3  dockerswarm      VM running    11.11.111.114  eastus

# <a name="basics"></a>Section 1: What is Orchestration

So, what is Orchestration anyways? Well, Orchestration is probably best described using an example. Lets say that you have an application that has high traffic along with high-availability requirements. Due to these requirements, you typically want to deploy across at least 3+ machines, so that in the event a host fails, your application will still be accessible from at least two others. Obviously, this is just an example and your use-case will likely have its own requirements, but you get the idea.

Deploying your application without Orchestration is typically very time consuming and error prone, because you would have to manually SSH into each machine, start up your application, and then continually keep tabs on things to make sure it is running as you expect.

But, with Orchestration tooling, you can typically off-load much of this manual work and let automation do the heavy lifting. One cool feature of Orchestration with Docker Swarm, is that you can deploy an application across many hosts with only a single command (once Swarm mode is enabled). Plus, if one of the supporting nodes dies in your Docker Swarm, other nodes will automatically pick up load, and your application will continue to hum along as usual.

If you are typically only using `docker run` to deploy your applications, then you could likely really benefit from using Docker Compose, Docker Swarm mode, and both Docker Compose and Swarm.

# <a name="start-cluster"></a>Section 2: Configure Swarm Mode

Real-world applications are typically deployed across multiple hosts as discussed earlier. This improves application performance and availability, as well as allowing individual application components to scale independently. Docker has powerful native tools to help you do this.

A swarm comprises one or more *Manager Nodes* and one or more *Worker Nodes*. The manager nodes maintain the state of swarm and schedule application containers. The worker nodes run the application containers. As of Docker 1.12, no external backend, or 3rd party components, are required for a fully functioning swarm - everything is built-in!

In this part of the demo you will use all three of the nodes in your lab. __swarm-leader__ will be the Swarm manager, while __swarm-node-1__ and __swarm-node-2__ will be worker nodes. Swarm mode supports a highly available redundant manager nodes, but for the purposes of this lab you will only deploy a single manager node.

## Step 2.1 - Create a Manager node

Note for Windows users please use the ```Git Bash``` prompt that is installed with [Git For Windows](https://git-for-windows.github.io/).

If you haven't already done so, please SSH in to **swarm-leader**.

```
$ ssh sysadmin@<swarm-leader public IP address>
```

In this step you'll initialize a new Swarm, join a single worker node, and verify the operations worked.

Run `docker swarm init` on **swarm-leader**.

```
$ docker swarm init
Swarm initialized: current node (6dlewb50pj2y66q4zi3egnwbi) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1wxyoueqgpcrc4xk2t3ec7n1poy75g4kowmwz64p7ulqx611ih-68pazn0mj8p4p4lnuf4ctp8xy \
    10.0.0.5:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

You can run the `docker info` command to verify that **swarm-leader** was successfully configured as a swarm manager node.

```
$ docker info
Containers: 2
 Running: 0
 Paused: 0
 Stopped: 2
Images: 2
Server Version: 17.03.1-ee-3
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 13
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
Swarm: active
 NodeID: rwezvezez3bg1kqg0y0f4ju22
 Is Manager: true
 ClusterID: qccn5eanox0uctyj6xtfvesy2
 Managers: 1
 Nodes: 1
 Orchestration:
  Task History Retention Limit: 5
 Raft:
  Snapshot Interval: 10000
  Number of Old Snapshots to Retain: 0
  Heartbeat Tick: 1
  Election Tick: 3
 Dispatcher:
  Heartbeat Period: 5 seconds
 CA Configuration:
  Expiry Duration: 3 months
 Node Address: 10.0.0.5
 Manager Addresses:
  10.0.0.5:2377
<Snip>
```

The swarm is now initialized with **swarm-leader** as the only Manager node. In the next section you will add **swarm-node-1** and **swarm-node-2** as *Worker nodes*.

##  Step 2.2 - Join Worker nodes to the Swarm

You will perform the following procedure on **swarm-node-1** and **swarm-node-2**. Towards the end of the procedure you will switch back to **swarm-leader**.

Open a new SSH session to __swarm-node-1__ (Keep your SSH session to **swarm-leader** open in another tab or window).

```
$ ssh sysadmin@<swarm-node-1 public IP address>
```

Now, take that entire `docker swarm join ...` command we copied earlier from `swarm-leader` where it was displayed as terminal output. We need to paste the copied command into the terminal of **swarm-node-1** and **swarm-node-2**.

It should look something like this for **swarm-node-1**. By the way, if the `docker swarm join ...` command scrolled off your screen already, you can run the `docker swarm join-token worker` command on the Manager node to get it again.

```
$ docker swarm join \
    --token SWMTKN-1-1wxyoueqgpcrc4xk2t3ec7n1poy75g4kowmwz64p7ulqx611ih-68pazn0mj8p4p4lnuf4ctp8xy \
    10.0.0.5:2377
```

Again, ssh into **swarm-node-2** and it should look something like this.

```
$ ssh sysadmin@<swarm-node-2 public IP address>
```

```
$ docker swarm join \
    --token SWMTKN-1-1wxyoueqgpcrc4xk2t3ec7n1poy75g4kowmwz64p7ulqx611ih-68pazn0mj8p4p4lnuf4ctp8xy \
    10.0.0.5:2377
```

Once you have run this on **swarm-node-1** and **swarm-node-2**, switch back to **swarm-leader**, and run a `docker node ls` to verify that both nodes are part of the Swarm. You should see three nodes, **swarm-leader** as the Manager node and **swarm-node-1** and **swarm-node-2** both as Worker nodes.

```
$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
6dlewb50pj2y66q4zi3egnwbi *  swarm-leader   Ready   Active        Leader
ym6sdzrcm08s6ohqmjx9mk3dv    swarm-node-2   Ready   Active
yu3hbegvwsdpy9esh9t2lr431    swarm-node-1   Ready   Active
```

The `docker node ls` command shows you all of the nodes that are in the swarm as well as their roles in the swarm. The `*` identifies the node that you are issuing the command from.

Congratulations! You have configured a swarm with one manager node and two worker nodes.

# <a name="multi-application"></a>Section 3: Deploy applications across multiple hosts

Now that you have a swarm up and running, it is time to deploy our really simple web application.

You will perform the following procedure from **swarm-leader**.

## Step 3.1 - Deploy the application components as Docker services

We will use the concept of *Services* to scale our application easily and manage many containers as a single entity. Read more about [Docker services here].(https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/)

> *Services* are a new concept in Docker 1.12. They work with swarms and are intended for long-running containers.

You will perform this procedure from **swarm-leader**.

Lets deploy `pet-web-app` as a *Service* across our Docker Swarm.

```
$ docker service create --replicas 1 --name pet-web-app --publish 80:5000 chrch/docker-pets
```

Verify that the `service create` has been received by the Swarm manager.

```
$ docker service ls
ID            NAME       MODE        REPLICAS  IMAGE
of5rxsxsmm3a  pet-web-app  replicated  1/1      chrch/docker-pets:latest
```

The state of the service may change a couple times until it is running. The image is being downloaded from Docker Hub to the other engines in the Swarm. Once the image is downloaded the container goes into a running state on one of the three nodes.

At this point it may not seem that we have done anything very differently than just running a `docker run ...`. We have again deployed a single container on a single host. The difference here is that the container has been scheduled on a swarm cluster.

Well done. You have deployed the pet-web-app to your new Swarm using Docker services. 

## Step 3.2 - Scale the app

Demand is crazy! Everybody loves your `pet-web-app` app! It's time to scale out.

One of the great things about *services* is that you can scale them up and down to meet demand. In this step you'll scale the service up and then back down.

You will perform the following procedure from **swarm-leader**.

Scale the number of containers in the **pet-web-app** service to 7 with the `docker service update --replicas 7 pet-web-app` command. `replicas` is the term we use to describe identical containers providing the same service.

```
$ docker service update --replicas 7 pet-web-app
```
	
The Swarm manager schedules so that there are 7 `pet-web-app` containers in the cluster. These will be scheduled evenly across the Swarm members.

We are going to use the `docker service ps pet-web-app` command. If you do this quick fast enough after using the  `--replicas` option you can see the containers come up in real time.

```
$ docker service ps pet-web-app
ID            NAME         IMAGE          NODE     DESIRED STATE  CURRENT STATE          ERROR  PORTS
7k0flfh2wpt1  pet-web-app.1  ubuntu:latest  swarm-leader  Running        Running 9 minutes ago
wol6bzq7xf0v  pet-web-app.2  ubuntu:latest  swarm-node-2  Running        Running 2 minutes ago
id50tzzk1qbm  pet-web-app.3  ubuntu:latest  swarm-node-1  Running        Running 2 minutes ago
ozj2itmio16q  pet-web-app.4  ubuntu:latest  swarm-node-2  Running        Running 2 minutes ago
o4rk5aiely2o  pet-web-app.5  ubuntu:latest  swarm-node-1  Running        Running 2 minutes ago
35t0eamu0rue  pet-web-app.6  ubuntu:latest  swarm-node-1  Running        Running 2 minutes ago
44s8d59vr4a8  pet-web-app.7  ubuntu:latest  swarm-leader  Running        Running 2 minutes ago
```

Notice that there are now 7 containers listed. It may take a few seconds for the new containers in the service to all show as **RUNNING**.  The `NODE` column tells us on which node a container is running.

## Step 3.3 - Access the app from the browser

You will need the public IP address of the **swarm-leader** in order to access the `pet-web-app` from the browser.

Get the public IP address by running the command below on a local terminal (not one you have SSHed into):

```
$ az vm list --resource-group dockerswarm --output table --show-details

Name          ResourceGroup    PowerState    PublicIps      Location
------------  ---------------  ------------  -------------  ----------
swarm-leader  dockerswarm      VM running    11.11.111.111  eastus
swarm-node-1  dockerswarm      VM running    11.11.111.112  eastus
swarm-node-2  dockerswarm      VM running    11.11.111.113  eastus
swarm-node-3  dockerswarm      VM running    11.11.111.114  eastus
```

Find the IP address for **swarm-leader** and copy/paste into a browser. 

Click the button on the page or refresh your browser a few times. Watch the container id change under `Processed by container ID`. You should be load-balancing between the 7 containers deployed for your `pet-web-app` service.

# <a name="drain"></a>Section 4: Drain a node and reschedule the containers

## Step 4.1 - Bring a node down for maintenance and add it back into the swarm.

Your pet-web-app has been doing amazing but now it's time to do some maintenance on one of your servers so you will need to gracefully take a server out of the swarm without interrupting service to your customers.

Take a look at the status of your nodes again by running `docker node ls` on **swarm-leader**.  

```
$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
6dlewb50pj2y66q4zi3egnwbi *  swarm-leader   Ready   Active        Leader
ym6sdzrcm08s6ohqmjx9mk3dv    swarm-node-2   Ready   Active
yu3hbegvwsdpy9esh9t2lr431    swarm-node-1   Ready   Active
```

You will be taking **swarm-node-1** out of service for maintenance.

If you haven't already done so, please SSH in to **swarm-node-1**.

```
$ ssh ubuntu@<swarm-leader IP address>
```

Then lets see the containers that you have running there.

```
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED              STATUS                        PORTS               NAMES
1a2042e69061        chrch/docker-pets:latest   "/bin/sh -c 'pytho..."   About a minute ago   Up About a minute (healthy)                       pet-web-app.3.ijlezlna8emfc7gp2xc61b3sd
11ca4df91856        chrch/docker-pets:latest   "/bin/sh -c 'pytho..."   6 minutes ago        Up 6 minutes (healthy)                            pet-web-app.2.g9tj6nupthdnvkdhuwe55f0sl
```

You can see that we have a few of the `pet-web-app` containers running here (your output might look different though).

Now lets jump back to **swarm-leader** (the Swarm manager) and take **swarm-node-1** out of service. To do that, lets run `docker node ls` again.

```
$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
6dlewb50pj2y66q4zi3egnwbi *  swarm-leader   Ready   Active        Leader
ym6sdzrcm08s6ohqmjx9mk3dv    swarm-node-2   Ready   Active
yu3hbegvwsdpy9esh9t2lr431    swarm-node-1   Ready   Active
```

We are going to take the **ID** for **swarm-node-1** and run `docker node update --availability drain yu3hbegvwsdpy9esh9t2lr431`. We are using the **swarm-node-1** host **ID** as input into our `drain` command. 

```
$ docker node update --availability drain yu3hbegvwsdpy9esh9t2lr431
```
Check the status of the nodes

```
$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
6dlewb50pj2y66q4zi3egnwbi *  swarm-leader   Ready   Active        Leader
ym6sdzrcm08s6ohqmjx9mk3dv    swarm-node-2   Ready   Active
yu3hbegvwsdpy9esh9t2lr431    swarm-node-1   Ready   Drain
```

Node **swarm-node-1** is now in the `Drain` state. 


Switch back to **swarm-node-1** and see what is running there by running `docker ps`.

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

 **swarm-node-1** does not have any containers running on it.

Next, check the service again on **swarm-leader** to make sure that the container were rescheduled. You should see all four containers running on the remaining two nodes.

```
$ docker service ps pet-web-app
ID            NAME             IMAGE          NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
7k0flfh2wpt1  pet-web-app.1      ubuntu:latest  swarm-leader  Running        Running 25 minutes ago
wol6bzq7xf0v  pet-web-app.2      ubuntu:latest  swarm-node-2  Running        Running 18 minutes ago
s3548wki7rlk  pet-web-app.6      ubuntu:latest  swarm-node-2  Running        Running 3 minutes ago
35t0eamu0rue   \_ pet-web-app.6  ubuntu:latest  swarm-node-1  Shutdown       Shutdown 3 minutes ago
44s8d59vr4a8  pet-web-app.7      ubuntu:latest  swarm-leader  Running        Running 18 minutes ago
```

Finally, after maintenance is complete, add the **swarm-node-1** back into the swarm
```
$ docker node update --availability active ykxalgc687g4f70jw6grccx31
ykxalgc687g4f70jw6grccx31
```

# <a name="cleanup"></a>Cleaning Up

Execute the `docker service rm pet-web-app` command on **swarm-leader** to remove the service called *myservice*.

```
$ docker service rm pet-web-app
```

Execute the `docker ps` command on **swarm-leader** to get a list of running containers.

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
044bea1c2277        ubuntu              "sleep infinity"    17 minutes ago      17 minutes ag                           distracted_mayer
```

Finally, lets remove swarm-leader, swarm-node-1, and swarm-node-2 from the Swarm. We can use the `docker swarm leave --force` command to do that. 

Lets run `docker swarm leave --force` on **swarm-leader**.

```
$ docker swarm leave --force
```

Then, run `docker swarm leave --force` on **swarm-node-1**.

```
$ docker swarm leave --force
```

Finally, run `docker swarm leave --force` on **swarm-node-2**.

```
$ docker swarm leave --force
```

Congratulations! You've completed this lab. You now know how to build a swarm, deploy applications as collections of services, and scale individual services up and down.

## Important Note

In order to save credits in your Azure subscription or reduce costs, it's recommended that you STOP or DELETE the VMs after the completion of the lab.  If you stop the VMs and turn them back on at a future time, the external IP addresses will likely change.

# <a name="nextsteps"></a>Next steps

1. Interactive Docker tutorials at Katacoda - [https://www.katacoda.com/courses/docker](https://www.katacoda.com/courses/docker)
2. Learn more about Docker Swarm Mode - [https://docs.docker.com/engine/swarm/](https://docs.docker.com/engine/swarm/) , or perhaps a [video](https://www.youtube.com/watch?v=KC4Ad1DS8xU)
3. Azure Container Service to manage your container orchestrator nodes  - [https://azure.microsoft.com/en-us/services/container-service/](https://azure.microsoft.com/en-us/services/container-service/)


