# Hello Microservices

This article is all about deploying microservices. We'll cover everything from the creation of a single container, to scaling an application on a Kubernetes cluster that we set up on Google cloud. This article will not cover every aspect of designing, deploying, testing and monitoring your services (there are [entire books](https://www.amazon.com/gp/product/1492034029/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1492034029&linkCode=as2&tag=sheena0d-20&linkId=5574de8e1ec2bdcab78e56b31e09910f) written on that topic), but it will give you a pretty good idea of how to get started.

## Micro-what?

If you are familiar with the concepts of microservices then feel free to skip this section.

In the beginning there were monoliths. Imagine you have an application running on the cloud somewhere that handles authentication; authorization; sending emails to members; keeping track of fees and membership details; distributing large volumes of entertaining and thought-provoking videos; keeping track of who has seen which video; making recommendations to members based on their viewing history. Let's call our totally hypothetical application Webflix.

Let's say we write the entire Webflix application as a single application - a monolith. There is quite a lot going on in there, which means quite a lot of testing would need to happen every time any changes are made. Eg: if I make a change to the code responsible for recommendation then I should run the entire unit test suite - after all there might be some coupling between the recommendation code and some other part of the system. Now once my tests pass I can deploy the code but that is quite a pain as well because there sure is a lot of it. I would probably need to rebuild absolutely everything and deploy a big thing, or introduce some complexity by introducing a kind of packaging system for different code units. Rolling back is similarly inefficient or complex.

And once our application is live then there are further inefficiencies. Let's say we have a few people signed up and paid up and they are binge watching their respective guilty pleasures one fine Saturday afternoon. With an application like Webflix, one would expect there to be more people watching videos at certain times and on certain days so it would make sense to build in the capability to scale up and down. But when we scale up then we scale up everything since it is a monolith. Wouldn't it be nice to just scale up those parts of the system that are actually under pressure during peak times? For exAMPLE the subsystem responsible for resetting forgotten passwords can probably chill out, but the subsystem responsible for actually serving videos should be scaled.

Enter Microservices.

The idea with microservices is that many applications could benefit from being broken down into smaller applications (called services) that play well together. So going back to Webflix, we would have one service for recommendations; one service for distributing videos; one service for managing payments; one service for authentication; one for authorization... wait, maybe the authentication and authorization service should be lumped into one service. And then the forgot password functionality could maybe be incorporated into that service too? Or should it be separate? The forgot password function would need to send emails, and emails have no place in authorization (unless we build some kind of 2FA system that uses emails).  What about the service that keeps track of payments? That feeds into the authorization system too because no-pay-no-watch. Hmmm... So we see that sometimes the boundaries between services are pretty clear and obvious. Sometimes they can be a little fuzzy. And the services need to communicate effectively.

Alright, let's say we've managed to break things down into all the different services we need. Now if I make a change to the recommendation system I just need to run the unit tests for that one system, I can deploy and roll back that service individually and I can scale it individually as needed. Winning! But wait, there are other systems involved - the recommendation system needs to be accessed by a end-user facing front-end, and it needs to know about what the user has already watched. That introduces some complexity because there is a chance that a change to the recommendation service will break communication with the functionality that it talks to. There is only so much that can be done with mocks in unit tests. That means that we'll need to build a new layer of testing that makes sure that the various services do in fact play nice.

And what if we have multiple developers working on various services at different paces? [Which seems like a reasonable assumption](https://en.wikipedia.org/wiki/Conway%27s_law) Let's say we are running recommend_service:1.2.0 and video_history_service:1.3.0 and those play nice. But then recommend_service:1.3.0 is created and that one breaks the overall system. So a patch is created to fix it (recommend_service:1.3.1) but in the meantime recommend_service:1.4.1 is released... this is starting to resemble a certain kind of hell. Special attention will need to be paid to which versions of the various services are compatible. And it would obviously make sense to put effort into making sure that exposed APIs are as stable as possible. And as self-documenting and generally discoverable as possible too.

Alrighty, so lets say we get everything deployed. Now these services need to communicate effectively. We could just use HTTP but what if the network is just slightly unreliable? What if a service is in the middle of an upgrade? What if the traffic is lost or replayed? Some thought will need to go in there. And what if something breaks without raising an exception? For example, what if "Barney the Dinosaur" is recommended to a hard code zombie fan? The history, user-preference and recommendations services should all be examined. And what if there is a spike in latency? Any number of individual services could be at fault.

So there are pros and cons to both microservices and monoliths and they are appropriate for different projects. But whichever route you go down, you will need to deploy your code. And that is really what this tutorial is about.

## Introducing Docker

If you are comfortable with Docker then feel free to skip this section :)

![Bare Metal](../images/hello_micro_deploy/deploy-micro-metal.png)

You run applications on your computer all the time. Your computer is a bare-metal machine. Applications run on top of your operating system (OS) and the OS manages the hardware. But deploying applications to bare metal machines can be a bit of a pain because if you want to change the specifications (eg: add or remove RAM) then you need to physically access the machine and install them. And running multiple applications on one machine means that they can mess with each other. There are a whole host of problems with deploying applications to bare metal. Many of those problems are overcome by Virtual Machines(VMs).

![Virtual Machine](../images/hello_micro_deploy/deploy-micro-VM.png)

A VM runs on a hypervisor, the hypervisor runs on the host OS and the host OS controls the hardware. The difference here is that there can be multiple VMs on top of a single hypervisor. The hypervisor's job is primarily to allocate resources to the various VMs. Now your application would run on top of an OS installed onto a VM. The VMs are isolated from each other and can all be running different operating systems. This means that you can run your application on one VM and some other organization can run theirs on a seperate VM on the same bare metal machine.

But the VMs are quite heavy - wouldn't it be nice to strip away all those extra OSs? Containers do that. A container can be thought of as a really lightweight VM.

![Containers](../images/hello_micro_deploy/deploy-micro-containers.png)

Containers are muuuch smaller than VMs by default, and require a lot less in terms of processing power in order to run - they tend to be mere megabytes in size and take just seconds to start. VMs tend to be Gigabytes in size and can take minutes to start. Containers make use of libraries and packages on the host operating system in order to access resources; then make use of their own libraries and packages in order to emulate seperate operating systems as needed. Eg: you could run an Ubuntu image on your Windows machine.

This is nice for a lot of reasons. Speed is great, size is great. Management is also great: If you are running VMs then that means you need to make sure that all the different operating systems in play are up to date and looked after, and VMs (like bare metal) tend to accumulate undocumented bloat as utilities are installed, updated, and generally messed with on the fly. Containers are fully specified in code and easy enough to re-create that manually messing around with their internals usually isn't needed at all.

And their smallness is of course really great when it comes to scaling applications. If we were hosting recommend_service as a VM and had a spike in traffic then users would just have to deal with some extra latency and perhaps some errors while we brought extra VMs online. On the other hand if we were using containers we would be able to bring them online much faster, and potentially even keep extra instances running just in case of spikes because containers are cheap.

[Docker](https://www.docker.com/) is the de facto industry standard container platform and that's the one that we'll be dealing with in this article.

## Practical: lets make an api

Docker allows the creation of images, images are instantiated to create containers (if you are familiar with object orientated programming then images are like classes, and containers are like objects). In this section we will create and run a container, the container will contain a service we wish to deploy.

We'll start off by making a simple Python [Hug](http://www.hug.rest/) application and running it locally. This tutorial assumes you can install python3, virtualenvwrapper and git on your own.

We don't actually *need* to install and run our app locally, but it may add some clarity to the discussion around creating our image.
```
## create and activate your virtual environment. Application dependencies will be installed here instead of globally. This has nothing to do with containers really, it's just a special directory and path configuration. Also it is good practice

mkvirtualenv --python=`which python3` timber_deployment_tutorial

## clone the application

git clone https://gitlab.com/sheena.oconnell/tutorial-timber-deploying-microservices.git

## install the application requirements

cd tutorial-timber-deploying-microservices
pip install -r requirements.txt

## run the application on your bare metal

hug -f main.py

```
You should get an output something like this:

```
hug -f main.py                                                                                                                                          [11:03]

/#######################################################################\
          `.----``..-------..``.----.
         :/:::::--:---------:--::::://.
        .+::::----##/-/oo+:-##----:::://
        `//::-------/oosoo-------::://.       ###    ###  ###    ###    #####
          .-:------./++o/o-.------::-`   ```  ###    ###  ###    ###  ##
             `----.-./+o+:..----.     `.:///. #########  ###    ### ##
   ```        `----.-::::::------  `.-:::://. ###    ###  ###    ### ###   ####
  ://::--.``` -:``...-----...` `:--::::::-.`  ###    ###  ###   ###   ###    ##
  :/:::::::::-:-     `````      .:::::-.`     ###    ###    #####     ######
   ``.--:::::::.                .:::.`
         ``..::.                .::         EMBRACE THE APIs OF THE FUTURE
             ::-                .:-
             -::`               ::-                   VERSION 2.4.0
             `::-              -::`
              -::-`           -::-
\########################################################################/

 Copyright (C) 2016 Timothy Edmund Crosley
 Under the MIT License


Serving on :8000...
127.0.0.1 - - [07/Jun/2018 11:04:27] "GET /index HTTP/1.1" 200 26
```

Let's query the index page to make sure the code actually runs. Open a new terminal and then:

```
curl 0.0.0.0:8000/index
```
This should return the response:

```
{"timber": "so delicious"}
```

Isn't that nice?

## Practical: lets run our api as an image

Now that you know that the application basically works, it's time to build an image. Notice that the repo contains a Dockerfile. Inside the Dockerfile you'll find something this:

```
FROM python:3
EXPOSE 8080

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "-b", "0.0.0.0:8080", "main:__hug_wsgi__"]
```

The line that says `FROM` specifies a base image. The base image in this case is `python:3`. When you build this image then docker will check if you already have the image downloaded. If the python3 image is not present then docker will download it from dockerhub. Dockerhub is full of all sorts of useful images. You can also push your images there if you want to make them publicly available. If you don't yet have a dockerhub account then now is a good time to make one.

If you want to learn more about pulling and pushing images then take a look at [the docs](https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/pull-and-push-images/#pull-an-image).

The `CMD` line at the end of the Dockerfile is for launching the main process of the container. If that process dies then the container dies too. Notice that during local development we used `hug -f main.py` and inside the container we are using [gunicorn](http://gunicorn.org/). The reason is that hug's built in server isn't really production ready.

Ok, now we can build the image:

```
docker build -t timber_tutorial:1 .
```

And then to run the image (that is, create a container):

```
docker run -p 8080:8080 timber_tutorial:1
```

You should see output like so:

```
[2018-06-15 07:11:59 +0000] [1] [INFO] Starting gunicorn 19.8.1
[2018-06-15 07:11:59 +0000] [1] [INFO] Listening at: http://127.0.0.1:8000 (1)
[2018-06-15 07:11:59 +0000] [1] [INFO] Using worker: sync
[2018-06-15 07:11:59 +0000] [9] [INFO] Booting worker with pid: 9
```

Open up another terminal then curl the api and make sure it still all works:

```
curl 0.0.0.0:8080/index
```

This outputs:
```
{"timber": "so delicious"}
```

Notice we are using port 8080 instead of 8000 here. You can specify whatever port you want.

### Summary

So now we have an image that we can run on any computer (bare metal or VM) that can run docker images. Our image has a name and a version number. If we wanted to make any changes to the functionality of our image then we would specify those changes in code and then re build the image with a different version tag. eg:
```
docker build -t timber_tutorial:2 .
```

There is quite a lot to be said about Docker that is outside the scope of this text. I suggest you take a look at the [official documentation](https://docs.docker.com/) if you need more details.

## Intro to Container Orchestration with Kubernetes (K8s)

Brace yourself, shit's about to get real...

So we have a container and we can run it locally. We would even be able to run it on a run of the mill VM somewhere on the cloud if we wanted to. But we're not going to do that. We're going to use K8s instead. For our simple application K8s is a bit overkill but it is the de facto standards when it comes to container orchestration.

Ok, so what is this container orchestration thing about? Let's revisit Webflix for this one. Let's say Webflix has gone down the microservices path (for their organization it is a pretty good course of action) and they have isolated a few aspect of their larger application into individual services. Each of those services are then built as images and need to be deployed in the form of live containers.

![Webflix Services](../images/hello_micro_deploy/deploy-micro-orchestrated-services.png)

The diagram here shows just a small portion of the Webflix application. Let's assume it's peak watching time, the little numbers next to the various services are examples of the number of live containers in play. In theory you would be able to make this happen through use of the `docker run` command.

Now let's say one of the recommend_service containers hits an unrecoverable error and falls over. If we were managing this through `docker run` then we would need to notice this and then restart the offending container. Or suddenly we are no longer in peak time and the numbers of containers needs to be scaled down. Or you want to deploy a new version of the recommend_service image? Then there are other concerns like container startup time and aliveness probes. And what if you have a bunch of machines that you want to run your containers on, you would need to keep track of what resources are available on each machine, and what replicas are running on each machine so you can scale up and scale down when you need to.

That is where container orchestration comes in.

Please note, the author of this post is terribly biased and thinks K8s is the shizzle. So that is what we will cover here. There are alternative platforms that you might want to look into, such as Docker Swarm and Mesos/Marathon.

Ok, back to the K8s goodness.

K8s was born and raised and battle hardened at Google. Google used it internally to run huge production workloads, and then they open-sourced it because they are lovely people. So K8s is a freely available, highly scalable, extendable and generally solid platform. Another thing that is really useful is that Google cloud has first class K8s support, so if you have a K8s cluster on Google cloud then there are all sorts of tools available that make managing your application easier.

K8s defines a bunch of concepts, there are entire books written on the subject so what follows is a surface level introduction. Personally I've found [this book](https://www.amazon.com/gp/product/1491935677/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1491935677&linkCode=as2&tag=sheena0d-20&linkId=e5bf88f134212c4beba73c601598d7ff") seriously useful.

K8s is all about managing certain objects. These objects can be created and configured in a bunch of ways. We'll be using the command line utility kubectl. K8s lets you run your workload on a cluster of machines, K8s installs a daemon on each of these machines, the daemons manage the K8s objects. Object configuration is communicated to the cluster. K8s job is to try to make the object configuration represent reality. Eg if you state that there should be three instances of the container recommend_service:1.0.1 then K8s will create those containers. If one of the containers dies (even if you manually kill it yourself) then K8s will make the configuration true again by recreating the killed container somewhere on the cluster.

There are a lot of different objects defined within K8s. I'll introduce you to just a few:

A *pod* is a group of containers (think of a pod of whales). The smallest pod contains just one container. Containers in a pod are managed as a group - they are deployed, replicated, started and stopped together. We will run our simple service as a single container in a pod. For more information on Pods take a look at the [k8s documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod/). It's really quite good.

If you deploy a pod to a k8s cluster then all containers in that specific pod will run on the same node in the cluster. Also all the containers in a single pod can address each other as though they are on the same computer (they share ip address and port space).This means that they can address each other via localhost. And of course their ports can't overlap.

Now say Webflix has a Pod that runs their recommendation service and they hit a spike in traffic. They would need to create a few new Pods to handle the traffic. A common approach is to use a Deployment for this.

A Deployment specifies a Pod as well as rules around how that Pod is replicated. Eg it could say that we want 3 replicas of a pod containing recommend_service:1.2.0. It also does clever things like help with rolling out upgrades - if recommend_service:1.3.0 becomes available then you can instruct k8s to take down the recommend_service pods and bring up new pods in a sensible way.

Deployments add a lot of power to Pods. To learn more about them I would like to refer you to the [official documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

The last object I'd like to cover is the Service. Containers in a Pod can talk to each other as though they are on the same host. But you'll often need Pods to talk to other Pods, and you'll also need to expose some of your Pods to the big bad Internet because users outside your cluster will likely need to interact with your applications. This is achieved through Services. Services defines the network interfaces exposed by Pods. Again I would like to refer you to the [official documentation](https://kubernetes.io/docs/concepts/services-networking/service/) if you need more info.

This was a very very brief introduction to K8s objects and their capabilities. Again there are [entire books](https://www.amazon.com/gp/product/1491935677/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1491935677&linkCode=as2&tag=sheena0d-20&linkId=e5bf88f134212c4beba73c601598d7ff") written on the subject and if you want to become an expert on the topic I would urge you to do some reading on your own.

## Practical: Deploying our application to Google cloud

Now we know why a tool like K8s is useful... let's start using it. I've chosen to proceed by making use of Google's Kubernetes engine offering. You can use a different service provider if you want to, or even run your own cluster, or simulate a cluster using [minikube](https://kubernetes.io/docs/setup/minikube/). For the most part the tutorial will be exactly the same except for the initial cluster setup and the final cluster cleanup.

### Set up our cluster

Visit the Kubernetes engine page (https://console.cloud.google.com/projectselector/kubernetes). If you are new to Google cloud you'll need to sign in with your Google account. You can sign up for $300 worth of free credits while you are at it. If you aren't new to Google and have already used up your free credits then the cluster will cost money (not a lot though).

Create or select a project - their user interface is fairly intuitive. You'll need to wait a little while for the K8s API and related services to be activated.

Now visit the [Google Cloud Console](https://console.cloud.google.com). At the top of the page you will see a little terminal button. Click on it to activate the Google cloud shell.

If you want to use a terminal on your own computer then you can, it just requires a bit more setup. You would need to install the Google Cloud SDK and the kubectl component and perform some extra configuration that's outside the scope of this text. If you want to go down that route take a look at [this](https://cloud.google.com/sdk/docs/quickstarts). You'll be interacting with your cluster through use of the gcloud command line utility.

Ok, so back to the shell: Run the following command to create your cluster:
```
gcloud container clusters create hello-timber --num-nodes=3 --zone=europe-west1-d
```

This will create a three node cluster called hello-timber. It might take a few minutes. Each node is a compute instance (VM) managed by Google. I've chosen here to put the nodes in west Europe. If you want to see a full list of available zones you can use the command `gcloud compute zones list`. Alternatively you can refer to [this document](https://cloud.google.com/compute/docs/regions-zones/) for a list of available zones.

In general it is good practice to choose a zone that is close to whatever clients are consuming your application. Although it is possible to deploy an application on multiple clusters in multiple zones around the world, that is waaaay outside the scope of this tutorial.

Once your cluster has been created successfully you should see something like this:

```
WARNING: Currently node auto repairs are disabled by default. In the future this will change and they will be enabled by default. Use `--[no-]enable-autorepair` flag  to suppress this warning.
WARNING: Starting in Kubernetes v1.10, new clusters will no longer get compute-rw and storage-ro scopes added to what is specified in --scopes (though the latter will remain included in the default --scopes). To use these scopes, add them
 explicitly to --scopes. To use the new behavior, set container/new_scopes_behavior property (gcloud config set container/new_scopes_behavior true).
Creating cluster hello-timber...done.
Created [https://container.googleapis.com/v1/projects/timber-tutorial/zones/europe-west1-d/clusters/hello-timber].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/europe-west1-d/hello-timber?project=timber-tutorial
kubeconfig entry generated for hello-timber.
NAME          LOCATION        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
hello-timber  europe-west1-d  1.8.10-gke.0    35.205.54.147  n1-standard-1  1.8.10-gke.0  3          RUNNING
```
To see the individual nodes:

```
gcloud compute instances list
```

This will output something like:

```
NAME                                         ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-hello-timber-default-pool-a471ab46-c0q8  europe-west1-d  n1-standard-1               10.132.0.3   35.205.110.255  RUNNING
gke-hello-timber-default-pool-a471ab46-jv7n  europe-west1-d  n1-standard-1               10.132.0.4   35.189.247.55   RUNNING
gke-hello-timber-default-pool-a471ab46-mt54  europe-west1-d  n1-standard-1               10.132.0.2   35.190.195.64   RUNNING
```

Achievement unlocked: K8s cluster!

### Upload your Docker image

We have a docker image on our local machine. We need to make sure it is available to our cluster. This means we need to put our image into a [container registry](https://cloud.google.com/container-registry/) that Google can access. Google cloud has a container registry built in.

In order to make use of Google's registry you will need to tag your images appropriately. In the commands below you will notice that our image tags include `eu.gcr.io`. This is because we are using a zone in Europe. If you chose to put your cluster in a different zone then just refer to [this document](https://cloud.google.com/container-registry/docs/pushing-and-pulling) and update your build and push commands appropriately.

On the Google cloud shell:
```
## clone your repo since it's not yet available to this shell. Of course if you
## are using a local install of gcloud then you already have this code

git clone https://gitlab.com/sheena.oconnell/tutorial-timber-deploying-microservices.git
cd tutorial-timber-deploying-microservices

## get your project id, we'll need it later

export PROJECT_ID="$(gcloud config get-value project -q)"

## configure docker so it can push properly
gcloud auth configure-docker
## just say yes to whatever it asks

## next up, build the image
docker build -t eu.gcr.io/${PROJECT_ID}/timber_tutorial:v1 .
## notice the funny name. eu.gcr.io refers to google's container registry in
## Europe. If your cluster is not in Europe (eg you chose a US zone when creating
## your cluster) then you must use a different url. Refer to this document for
## details: https://cloud.google.com/container-registry/docs/pushing-and-pulling

## once you are finished building the image, push it
docker push eu.gcr.io/${PROJECT_ID}/timber_tutorial:v1
```

Now we have our application built as an image, and that image is available to our cluster! The next step is to create a deployment and then expose that deployment to the outside world as a service.

### Get your application to run as a deployment

Inside the Google cloud shell do the following:

```
kubectl run timber-tutorial --image=eu.gcr.io/${PROJECT_ID}/timber_tutorial:v1 --port 8080
```

The output should be something like:
```
deployment "timber-tutorial" created
```
As in: You just created a K8s Deployment!

Now let's take a look at what we have done. First take a look at the deployment we just created.

```
kubectl get deployments
```
This will give you a brief summary of the current deployments. The output will look like:

```
NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
timber-tutorial   1         1         1            1           57s
```

Now a deployment manages pods. So let's look at those:

```
kubectl get pods

```
This outputs:

```
NAME                              READY     STATUS    RESTARTS   AGE
timber-tutorial-99f796786-r92fv   1/1       Running   0          1m
```

Alright! Now we need to expose our application to the Internet by using a K8s service. This is similarly easy:

```
kubectl expose deployment timber-tutorial --type=LoadBalancer --port 80 --target-port 8080
```
This outputs:
```
service "timber-tutorial" exposed
```

Now this is a bit more complex. What it does is tell Google that we want a LoadBalancer. Loadbalancers decide which Pods should get incident traffic. So if I have a bunch of replicas of my pod running, and a bunch of clients trying to access my application then Google will use it's infrastructure to spread the traffic over my pods. This is a very simplified explanation. And there are a few more kinds of services you might want to know about if you are actually building a microservices project.

Remember this line from our Dockerfile?
```
CMD ["gunicorn", "-b", "0.0.0.0:8080", "main:__hug_wsgi__"]
```

Our container is expecting to communicate with the outside world via port 8080. That's the `--target-port`. The port we want to communicate with is `80`. So we just tell the service that we want to tie those ports together.

Let's take a look at how our service is doing
```
kubectl get service -w
```
Notice the `-w` here. Services take a bit of time to get going because LoadBalancers need some time to wake up. The `-w` stands for `watch` whenever the service is updated with new information then that information will get printed to the terminal.

The output is like this:

```
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes        ClusterIP      10.47.240.1     <none>        443/TCP        44m
timber-tutorial   LoadBalancer   10.47.241.244   <pending>     80:30262/TCP   40s
## time passes before the next line is printed out
timber-tutorial   LoadBalancer   10.47.241.244   35.205.44.169   80:30262/TCP   55s
```
Press `Ctrl+C` to stop watching the services.

Now let's access the application from your local computer.

```
## copy the external IP address from the service output above
export EXTERNAL_IP="35.205.44.169"
curl ${EXTERNAL_IP}/index
```

This outputs:
```
{"timber": "so delicious"}
```

Awesome! so we have our image running as a container in a cluster hosted on Google Kurbenetes engine! I don't know about you but I find this pretty exciting. But we have only just started to scratch the surface of what K8s can do.

### Scale Up

Let's say our application has proven to be seriously delicious. And so a lot of traffic is coming our way. Cool! Time to scale up. We need to update our deployment object so that it expects 3 pods.

```
## first tell k8s to scale it up
kubectl scale deployment timber-tutorial --replicas=3

## now take a look at your handiwork
kubectl get deployment timber-tutorial
```

The output is:
```
NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
timber-tutorial   3         3         3            3           23m
```

And take a look at the pods:

```
kubectl get pods
```

The output is:
```
NAME                              READY     STATUS    RESTARTS   AGE
timber-tutorial-99f796786-2x964   1/1       Running   0          54s
timber-tutorial-99f796786-r92fv   1/1       Running   0          23m
timber-tutorial-99f796786-tlnck   1/1       Running   0          54s
```

Brilliant! So now when someone accesses our K8s service's external IP address then the traffic will be routed by the LoadBalancer to one of the pods. So we can handle three times the amount of traffic we could before.

### Deploy a new version of your application

Now let's make a new version of our application. Go back to your Google cloud shell and checkout the `version2` branch of our application:

```
git checkout -b version2 origin/version_2
```

Now let's build and push a new version of our image:

```
docker build -t eu.gcr.io/${PROJECT_ID}/timber_tutorial:v2 .
docker push eu.gcr.io/${PROJECT_ID}/timber_tutorial:v2
```

Now we tell the deployment to run our new image:

```
kubectl set image deployment/timber-tutorial timber-tutorial=eu.gcr.io/${PROJECT_ID}/timber_tutorial:v2
```

And watch our pods:
```
kubectl get pods -w
```

And a whole lot of stuff happens over time:

```
NAME                               READY     STATUS              RESTARTS   AGE
timber-tutorial-66c6545dd9-2lnlt   0/1       ContainerCreating   0          42s
timber-tutorial-66c6545dd9-s44xm   0/1       ContainerCreating   0          42s
timber-tutorial-99f796786-2x964    1/1       Running             0          24m
timber-tutorial-99f796786-r92fv    1/1       Running             0          47m
timber-tutorial-66c6545dd9-2lnlt   1/1       Running   0         46s
timber-tutorial-99f796786-2x964   1/1       Terminating   0         24m
timber-tutorial-66c6545dd9-h7vfv   0/1       Pending   0         1s
timber-tutorial-66c6545dd9-h7vfv   0/1       Pending   0         1s
timber-tutorial-66c6545dd9-h7vfv   0/1       ContainerCreating   0         1s
timber-tutorial-99f796786-2x964   0/1       Terminating   0         24m
timber-tutorial-99f796786-2x964   0/1       Terminating   0         24m
timber-tutorial-66c6545dd9-s44xm   1/1       Running   0         48s
timber-tutorial-99f796786-r92fv   1/1       Terminating   0         47m
timber-tutorial-99f796786-r92fv   0/1       Terminating   0         47m
timber-tutorial-66c6545dd9-h7vfv   1/1       Running   0         4s
timber-tutorial-99f796786-2x964   0/1       Terminating   0         24m
timber-tutorial-99f796786-2x964   0/1       Terminating   0         24m
timber-tutorial-99f796786-r92fv   0/1       Terminating   0         47m
timber-tutorial-99f796786-r92fv   0/1       Terminating   0         47m
```

Eventually new things stop happening. Now `Ctrl+C` and run get pods again:

```
kubectl get pods
```

this outputs:

```
NAME                               READY     STATUS    RESTARTS   AGE
timber-tutorial-66c6545dd9-2lnlt   1/1       Running   0          2m
timber-tutorial-66c6545dd9-h7vfv   1/1       Running   0          1m
timber-tutorial-66c6545dd9-s44xm   1/1       Running   0          2m
```

So what just happened? We started off with three pods of running `timber_tutorial:v1` now we have three pods running `timber_tutorial:v2`. K8s doesn't just kill all the running pods and create all the new ones all at the same time. It is possible to make that happen but it would mean that the application would be down for a little while. Instead, K8s will start bringing new pods online while terminating old ones and re balancing traffic. That means that anyone accessing your application will be able to keep accessing it while the update is being rolled out. This is of course only part of the story, K8s is capable of performing all sorts of health and readiness checks along the way, and can be configured to roll things out in different ways.

Let's go back to our local terminal and see what our application is doing how:

```
curl ${EXTERNAL_IP}/index
```

This outputs:

```
{"timber": "so delicious", "why": "because logging shouldn't be hard"}
```

## Conclusion

Well done for getting this far! Seriously, you've learned a lot. Packaging a web application as a docker image; pulling and pushing docker images; creating docker containers locally; setting up K8s clusters on Google's cloud; some basic K8s object configuration skills including scaling and upgrading your pods on the fly. That's a lot of ground to cover, give yourself a pat on the back.

That said, the examples in this article are of a hello-world variety. There is a lot more worth learning. From architecting and testing; to versioning and monitoring of live applications. I sincerely hope that this tutorial is a valuable step in your journey toward microservice mastery.

The next post in this series will cover setting up a basic CI/CD pipeline for our application. Stay tuned :)

## One more thing - Cleanup

Clusters cost money so it would be best to shut it down if you aren't using it. Go back to the Google Cloud Shell and do the following:

```
kubectl delete service hello-timber-service

## now we need to wait a bit for google to delete some forwarding rules for us. Keep an eye on them by executing this command:
gcloud compute forwarding-rules list

## once the forwarding rules are deleted then it is safe to delete the cluster:
gcloud container clusters delete hello-timber
```







