# Kubernetes on Azure Container Service

In this lab, you will create a Kubernetes cluster using Azure Container Service, deploy an application, scale it up and down, and hopefully get a good sense on why Kubernetes is a good choice for container orchestration. By the end, you will learn just how easy it is to create and maintain a Kubernetes cluster with Azure Container Service.

> **Tasks**:
>
> * [Section #0 - Prerequisites](#prerequisites)
> * [Section #1 - What is Orchestration](#basics)
> * [Section #2 - Configure Kubernetes cluster using Azure Container Service](#start-cluster)
> * [Section #3 - Deploy an application in Kubernetes](#deploy-app)
> * [Section #4 - Scale the application](#scale-app)
> * [Section #5 - Kubernetes Dashboard](#dashboard)
> * [Cleaning Up](#cleanup)

## Document conventions

When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value. 

For instance if you see `az account set --subscription <YOUR SUB ID>` you would actually type something like `az account set --subscription 1111-1111-11111`

# <a name="prerequisites"></a>Section 0: Prerequisites

## Step 0.1 - Azure CLI 2.0

This lab will use a Docker container that has the Azure CLI pre-installed. When you run the container it will download the image from Docker Hub and at the end you will be on a Bash command prompt.

To run the Docker container:

```
docker run -it -p 8001:8001 azuresdk/azure-cli-python
```

## Step 0.2 - Login to the Azure CLI

Before you continue please login into the Azure CLI and get your subscription Id.

**Login to the Azure CLI**

```
az login
```

Follow the instructions printed on the command shell.
When that process completes, the command shell completes the log in process. It might look something like:

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

**Get your subscription Id**

```
az account list
```

This will print out a JSON block similar to the below one showing the accounts you are currently logged into. The subscription id is found in the ```id``` field.

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
    
**Confirm your Azure Subscription**

```
az account set --subscription <YOUR SUB ID>
```

# <a name="basics"></a>Section 1: What is Orchestration

So, what is Orchestration anyways? Well, Orchestration is probably best described using an example. Lets say that you have an application that has high traffic along with high-availability requirements. Due to these requirements, you typically want to deploy across at least 3+ machines, so that in the event a host fails, your application will still be accessible from at least two others. Obviously, this is just an example and your use-case will likely have its own requirements, but you get the idea.

Deploying your application without Orchestration is typically very time consuming and error prone, because you would have to manually SSH into each machine, start up your application, and then continually keep tabs on things to make sure it is running as you expect.

But, with Orchestration tooling, you can typically off-load much of this manual work and let automation do the heavy lifting. One cool feature of Orchestration with Kubernetes, is that you can deploy an application across many hosts with only a single command. Plus, if one of the supporting nodes dies in your Kubernetes cluster, other nodes will automatically pick up load, and your application will continue to hum along as usual.

If you are typically only using `docker run` to deploy your applications, then you could likely really benefit from using Kubernetes.

# <a name="start-cluster"></a>Section 2: Configure Kubernetes cluster using Azure Container Service

Real-world applications are typically deployed across multiple hosts as discussed earlier. This improves application performance and availability, as well as allowing individual application components to scale independently. Kubernetes is a powerful orchestrator that has tools to help you do this, and Azure Container Service (ACS) makes creating a new Kubernetes cluster super simple.

A Kubernetes cluster comprises of one or more *Manager Nodes* and one or more *Worker Nodes*. The manager nodes maintain the state of the cluster and schedule application containers. The worker nodes run the application containers.

## Step 2.1 - Create a new Azure resource group

A nice feature of Azure is the ability to group all of the resources that make up our Kubernetes cluster in one central location for easier visability and maintenance. This grouping is called a ```resource group```.

Simply type this into the command line to create the group. You will specify a name for the group as well as what Azure region (location) it will be in.

```
az group create --name kube-demo --location eastus
```

Since we will need to enter the name of the resource group a few times, lets set it as a default to avoid having to type it over and over.

```
az configure --defaults group=kube-demo location=eastus
```

If you need to clear these defaults later, you can type (dont do until the end of this lab):

```
az configure --defaults group='' location=''
```

## Step 2.2 - Create Kubernetes cluster using ACS

Create a Kubernetes cluster in your resource group by using the ```az acs create``` command with ```--orchestrator-type=kubernetes```. For command syntax, see the ```az acs create --help```

This version of the command automatically generates the SSH RSA keys and service principal for the Kubernetes cluster. The SSH keys will allow you to remotely connect to your cluster nodes and the service principal will allows Kubernetes to create Azure resources, such as load balancers, on your behalf. 

```
az acs create \
--orchestrator-type=kubernetes \
--name=kube-demo-cluster \
--dns-prefix=<unique dns name> \
--generate-ssh-key
```

After several minutes, the command completes, and you should have a working Kubernetes cluster with one *manager* node and three *worker* nodes.

To view everything that was created, go to the [Azure Portal](https://portal.azure.com).

Click ```Resource Groups```

![](images/resource-group.png)

Click on the resource group named ```kube-demo```

![](images/resource-group-list.png)

You should see all of the resources that were created for our Azure Container Service Kubernetes cluster.

## Step 2.3 Connect to the cluster

To connect and interact with the Kubernetes cluster from your client computer, you use ```kubectl```, the Kubernetes command-line client. You can install it with:

```
az acs kubernetes install-cli
```

Then, run the following command to download the master Kubernetes cluster configuration to the ```~/.kube/config``` file. This will allow you to work with the cluster using ```kubectl```.

```
az acs kubernetes get-credentials --name=kube-demo-cluster
```

Verify you are working against the correct cluster with this command:

```
kubectl config current-context
```

You should see the name of the cluster you just created with the name you provided for ```<unique dns name>```

```
kube-demo
```

Finally, verify you can connect to the cluster by viewing the cluster ```nodes```.

```
kubectl get nodes
```

You should see 1 *master* node and 3 *worker* (agent) nodes similar to this

```
NAME                    STATUS                     AGE       VERSION
k8s-agent-3fb13719-0    Ready                      27m       v1.5.3
k8s-agent-3fb13719-1    Ready                      27m       v1.5.3
k8s-agent-3fb13719-2    Ready                      27m       v1.5.3
k8s-master-3fb13719-0   Ready,SchedulingDisabled   27m       v1.5.3
```

# <a name="deploy-app"></a>Section 3: Deploy an application in Kubernetes

Now that we have a Kubernetes cluster up and running, it is time to deploy our really simple web application.

## Step 3.1 - Deploy the application as a Kubernetes deployment

A Kubernetes *Deployment* provides a declarative way to deploy and update your application. This includes the name of your application, how many replicas, rolling update strategy and much more.

Lets deploy an application named `hello-world` as a *Deployment* across our Kubernetes cluster with ```1 replica``` using the Docker image ```billpratt/netcore-hello-world``` from Docker Hub.

```
kubectl run hello-world --image billpratt/netcore-hello-world --replicas 1
```

If successful, you should see:

```
deployment "hello-world" created
```

## Step 3.2 - Get details of the Deployment

To verify the deployment is running, run

```
kubectl get deployment hello-world
```

Output should look similar to

```
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-world   1         1         1            0           1m
```

Notice, the ```Desired``` and ```Current``` columns. We have *declared* that we *desire* 1 replica, or instance, of the application and we *currently* have 1 running.

You can see the full details of your deployment in ```yaml``` or ```json``` format by adding ```--output yaml``` or ```--output json``` to the same command.

```
kubectl get deployment hello-world --output yaml
```

Output in YAML

```
apiVersion: v1
items:
- apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    annotations:
      deployment.kubernetes.io/revision: "1"
    creationTimestamp: 2017-05-31T01:34:44Z
    generation: 1
    labels:
      run: hello-world
    name: hello-world
    namespace: default
    resourceVersion: "1664"
    selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/hello-world
    uid: 52dcef2e-45a1-11e7-a409-000d3a1bceae
  spec:
    replicas: 1
    selector:
      matchLabels:
        run: hello-world
    strategy:
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 1
      type: RollingUpdate
    template:
      metadata:
        creationTimestamp: null
        labels:
          run: hello-world
      spec:
        containers:
        - image: billpratt/netcore-hello-world
          imagePullPolicy: Always
          name: hello-world
          resources: {}
          terminationMessagePath: /dev/termination-log
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        securityContext: {}
        terminationGracePeriodSeconds: 30
  status:
    availableReplicas: 1
    conditions:
    - lastTransitionTime: 2017-05-31T01:34:44Z
      lastUpdateTime: 2017-05-31T01:34:44Z
      message: Deployment has minimum availability.
      reason: MinimumReplicasAvailable
      status: "True"
      type: Available
    observedGeneration: 1
    replicas: 1
    updatedReplicas: 1
kind: List
metadata: {}
resourceVersion: ""
selfLink: ""
```

# Step 3.3 - Kubernetes pods

Kubernetes run an instance of your application as a container in a Kubernetes ```Pod```. A Kubernetes Deployment is responsible for maintaining the state of pods including what version of the container to run inside the pod and how many replicas (pods) should be running based on what the state of the Deployment you have declared.

In the previous step, you deployed 1 replica. This replica is running in a *pod*. To see which cluster node is running this pod, run:

```
kubectl get pods -o wide
```

You should see something similar to

```
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
hello-world-2680727081-4p8th   1/1       Running   0          17m       10.244.3.2   k8s-agent-eda0b42-0
```

In the above example, our one pod is running on the node ```k8s-agent-eda0b42-0```

# Step 3.4 - Expose application to the world

A Kubernetes ```Service``` is an abstraction layer on top of the Deployment. It is responsible for exposing your application either internally or externally, depending on how you *declare* it, as well as maintaining routes to the application replicas. When accessing your application in Kubernetes, you go through the Service. A more detailed description of a Kubernetes Service can be read [here](https://kubernetes.io/docs/concepts/services-networking/service/).

The easiest way to expose your application to external traffic, is to create a *Service* with type ```Load Balancer```.

```
kubectl expose deployments hello-world --port=80 --type=LoadBalancer
```

This command creates a Kubernetes Service of type Load Balancer with port 80 exposed. Since we are running our cluster in Azure Container Service, a new Azure Load Balancer will be created and *wired up* to point to and expose our application.

Run the following command to *watch* our new service for changes. 

```
kubectl get svc hello-world --watch
```

With output

```
kubectl get svc hello-world --watch
NAME          CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
hello-world   10.0.20.8    <pending>     80:30969/TCP   19s
```

This process can take a few minutes so please be patient. We are looking for the ```EXTERNAL-IP``` column to change. Once we have an IP address, the Azure Load Balancer is ready and we should be able to hit our application using the provided IP Address. After a minute you should see something like:

```
NAME          CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
hello-world   10.0.88.111   <pending>     80:30504/TCP   3s
hello-world   10.0.88.111   40.71.92.176   80:30504/TCP   4m
```

Once we have an IP address you can stop watching for changes with ```CTRL-C``` to bring you back to the command line.

**Access your application**

Copy the external IP Address provided from ```kubectl get svc hello-world``` and paste into your browser. You should see a simple ```Hello World``` page load.

Congratulations, you just created your first application in Kubernetes and exposed it to the world!


# <a name="scale-app"></a>Section 4: Scale the application

One of the great things about *deployments* is that you can scale the pods up and down to meet demand. In this step you'll scale the service up and then back down.

Scale the number of pods in the **hello-world** service to 7 with the `kubectl scale deployment hello-world --replicas 7` command. `replicas` is the term we use to describe identical pods running our application as a container.

```
kubectl scale deployment hello-world --replicas 7
```

To view the results, get the details of the deployment

```
kubectl get deployment hello-world
```

You should see 7 *Desired* and 7 *Current* pods (replicas)

```
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-world   7         7         7            1           32m
```

View which cluster nodes are running our pods

```
kubectl get pods -o wide
```

```
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
hello-world-2680727081-12rcf   1/1       Running   0          1m        10.244.2.5   k8s-agent-7f099eb2-0
hello-world-2680727081-2jt6g   1/1       Running   0          8m        10.244.0.5   k8s-agent-7f099eb2-1
hello-world-2680727081-9kf1z   1/1       Running   0          1m        10.244.2.3   k8s-agent-7f099eb2-0
hello-world-2680727081-m8j32   1/1       Running   0          1m        10.244.2.4   k8s-agent-7f099eb2-0
hello-world-2680727081-s5g8w   1/1       Running   0          1m        10.244.3.4   k8s-agent-7f099eb2-2
hello-world-2680727081-txj6h   1/1       Running   0          1m        10.244.0.6   k8s-agent-7f099eb2-1
hello-world-2680727081-xvrfd   1/1       Running   0          1m        10.244.3.3   k8s-agent-7f099eb2-2
```

Lets get crazy and scale up even further. A Kubernetes cluster can handle lots of running pods, depending on the resources of your cluster nodes.

```
kubectl scale deployment hello-world --replicas 20
```

Running ```kubectl get pods -o wide``` again will show you how the pods are distributed across the cluster.

Finally, lets scale the number of replicas back down

```
kubectl scale deployment hello-world --replicas 3 
```

If you run ```kubectl get pods -o wide``` fast enough, you should see pods ```Terminating``` while keeping three running.

```
NAME                           READY     STATUS        RESTARTS   AGE       IP           NODE
hello-world-2680727081-12rcf   1/1       Terminating   0          2m        10.244.2.5   k8s-agent-7f099eb2-0
hello-world-2680727081-2jt6g   1/1       Running       0          10m       10.244.0.5   k8s-agent-7f099eb2-1
hello-world-2680727081-45n0s   1/1       Terminating   0          1m        10.244.0.9   k8s-agent-7f099eb2-1
hello-world-2680727081-5sgt2   1/1       Terminating   0          1m        10.244.3.9   k8s-agent-7f099eb2-2
hello-world-2680727081-9kf1z   1/1       Running       0          2m        10.244.2.3   k8s-agent-7f099eb2-0
hello-world-2680727081-9w077   1/1       Terminating   0          1m        10.244.3.8   k8s-agent-7f099eb2-2
hello-world-2680727081-fb9df   1/1       Terminating   0          1m        10.244.0.7   k8s-agent-7f099eb2-1
hello-world-2680727081-jhm5d   1/1       Terminating   0          1m        10.244.2.8   k8s-agent-7f099eb2-0
hello-world-2680727081-k3bgz   1/1       Terminating   0          1m        10.244.2.7   k8s-agent-7f099eb2-0
hello-world-2680727081-kksp1   1/1       Terminating   0          1m        10.244.3.5   k8s-agent-7f099eb2-2
hello-world-2680727081-m8j32   1/1       Terminating   0          2m        10.244.2.4   k8s-agent-7f099eb2-0
hello-world-2680727081-pj1f1   1/1       Terminating   0          1m        10.244.3.6   k8s-agent-7f099eb2-2
hello-world-2680727081-s5g8w   1/1       Terminating   0          2m        10.244.3.4   k8s-agent-7f099eb2-2
hello-world-2680727081-t5fj1   1/1       Terminating   0          1m        10.244.2.9   k8s-agent-7f099eb2-0
hello-world-2680727081-txj6h   1/1       Running       0          2m        10.244.0.6   k8s-agent-7f099eb2-1
hello-world-2680727081-vrftc   1/1       Terminating   0          1m        10.244.2.6   k8s-agent-7f099eb2-0
hello-world-2680727081-xvrfd   1/1       Terminating   0          2m        10.244.3.3   k8s-agent-7f099eb2-2
hello-world-2680727081-zzwjj   1/1       Terminating   0          1m        10.244.3.7   k8s-agent-7f099eb2-2
```

# <a name="dashboard"></a>Section 5: Kubernetes Dashboard

Kubernetes provides a nice dashboard UI to visualize and maintain applications in our cluster. With Azure Container Service, the dashboard is pre-installed.

You can access the dashboard by setting up a local proxy on port 8001 inside the container we are running in.

```
kubectl proxy --address 0.0.0.0
```

**Note**: ```--address 0.0.0.0``` is not needed if running outside of a container.

```
Starting to serve on 127.0.0.1:8001
```

In a browser go to [http://localhost:8001/ui](http://localhost:8001/ui)

You should see the ```hello-world``` *deployment*. Feel free to click around and explore. 

![alt text](images/kube-dashboard-screenshot.png)

Back in the container, exit out of the proxy with ```CTRL-C```

# <a name="cleanup"></a>Section 5: Cleanup

To remove ```hello-world``` from cluster, you must delete both the *deployment* and the *service*.  

To delete the deployment

```
kubectl delete deployment hello-world
```

To delete the service

```
kubectl delete service hello-world
```

**Important Note**: You should delete the ACS cluster if you are no longer using it. Go to the [Azure Portal].(https://portal.azure.com)

Click ```Resource Groups```

![](images/resource-group.png)

Click on the resource group named ```kube-demo```

![](images/resource-group-list.png)

In the ```Overview``` tab, click the Delete button and follow the instructions.

![](images/resource-group-delete.png)
