# Hello Microservice Deployment Part 2: Kubernetes and Google Cloud

## Introduction

Hi and welcome back. This is the second part of a three part series. In [part 1](http://todo) we got acquainted with Docker by building an image for a simple web app, and then running that image. We have thus far been working with [this repo](https://gitlab.com/sheena.oconnell/tutorial-timber-deploying-microservices.git).

In this part we'll be taking the docker image from part 1 and getting it to run on a Kurbenetes cluster that we set up on Google cloud.

In [Part 3](http://todo) we'll use Drone.io to set up a simple CI/CD pipeline. It will test our application and roll out any changes we make to the master branch of our repo.

## So what is Kubernetes (K8s) for?

Ok, so what is this container orchestration thing about? Let's revisit Webflix for this one. Let's say Webflix has gone down the microservices path (for their organization it is a pretty good course of action) and they have isolated a few aspect of their larger application into individual services. Each of those services are then built as images and need to be deployed in the form of live containers.

![Webflix Services](../images/hello_micro_deploy/deploy-micro-orchestrated-services.png)

The diagram here shows just a small portion of the Webflix application. Let's assume it's peak watching time, the little numbers next to the various services are examples of the number of live containers in play. In theory you would be able to make this happen through use of the `docker run` command.

Now let's say one of the recommend_service containers hits an unrecoverable error and falls over. If we were managing this through `docker run` then we would need to notice this and then restart the offending container. Or suddenly we are no longer in peak time and the numbers of containers needs to be scaled down. Or you want to deploy a new version of the recommend_service image? Then there are other concerns like container startup time and aliveness probes. And what if you have a bunch of machines that you want to run your containers on, you would need to keep track of what resources are available on each machine, and what replicas are running on each machine so you can scale up and scale down when you need to.

That is where container orchestration comes in.

Please note, the author of this post is terribly biased and thinks K8s is the shizzle. So that is what we will cover here. There are alternative platforms that you might want to look into, such as Docker Swarm and Mesos/Marathon.

Ok, back to the K8s goodness.

K8s was born and raised and battle hardened at Google. Google used it internally to run huge production workloads, and then they open-sourced it because they are lovely people. So K8s is a freely available, highly scalable, extendable and generally solid platform. Another thing that is really useful is that Google cloud has first class K8s support, so if you have a K8s cluster on Google cloud then there are all sorts of tools available that make managing your application easier.

K8s defines a bunch of concepts, there are entire books written on the subject so what follows is a surface level introduction. Personally I've found [this book](https://www.amazon.com/gp/product/1491935677/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1491935677&linkCode=as2&tag=sheena0d-20&linkId=e5bf88f134212c4beba73c601598d7ff") seriously useful.


Kubernetes lets you run your workload on a cluster of machines. It installs a daemon (a deamon is just a long running background process) on each of these machine and the daemons manage the Kubernetes objects. K8s job is to try to make the object configuration represent reality. Eg if you state that there should be three instances of the container recommend_service:1.0.1 then K8s will create those containers. If one of the containers dies (even if you manually kill it yourself) then K8s will make the configuration true again by recreating the killed container somewhere on the cluster.

There are a lot of different objects defined within K8s. I'll introduce you to just a few. This will be a very very brief introduction to K8s objects and their capabilities. Again there are [entire books](https://www.amazon.com/gp/product/1491935677/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1491935677&linkCode=as2&tag=sheena0d-20&linkId=e5bf88f134212c4beba73c601598d7ff") written on the subject and if you want to become an expert on the topic I would urge you to do some reading on your own.

### A *pod* is a group of one of more containers

Think about peas in a pod, or a Pod of whales. Containers in a pod are managed as a group - they are deployed, replicated, started and stopped together. We will run our simple service as a single container in a pod. For more information on Pods take a look at the [k8s documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod/). It's really quite good.

If you deploy a pod to a k8s cluster then all containers in that specific pod will run on the same node in the cluster. Also all the containers in a single pod can address each other as though they are on the same computer (they share ip address and port space).This means that they can address each other via `localhost`. And of course their ports can't overlap.

### A *deployment* is a group of one or more pods

Now say Webflix has a Pod that runs their recommendation service and they hit a spike in traffic. They would need to create a few new Pods to handle the traffic. A common approach is to use a Deployment for this.

A Deployment specifies a Pod as well as rules around how that Pod is replicated. Eg it could say that we want 3 replicas of a pod containing recommend_service:1.2.0. It also does clever things like help with rolling out upgrades - if recommend_service:1.3.0 becomes available then you can instruct k8s to take down the recommend_service pods and bring up new pods in a sensible way.

Deployments add a lot of power to Pods. To learn more about them I would like to refer you to the [official documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

### A *service* defines networking rules

Containers in a Pod can talk to each other as though they are on the same host. But you'll often need Pods to talk to other Pods, and you'll also need to expose some of your Pods to the big bad Internet because users outside your cluster will likely need to interact with your applications. This is achieved through Services. Services defines the network interfaces exposed by Pods. Again I would like to refer you to the [official documentation](https://kubernetes.io/docs/concepts/services-networking/service/) if you need more info.


## Practical: Deploying our application to Google cloud

Now we know why a tool like K8s is useful... let's start using it. I've chosen to proceed by making use of Google's Kubernetes engine offering. You can use a different service provider if you want to, or even run your own cluster, or simulate a cluster using [minikube](https://kubernetes.io/docs/setup/minikube/). For the most part the tutorial will be exactly the same except for the initial cluster setup and the final cluster cleanup.

### Set up our cluster

Visit the [Kubernetes engine page ](https://console.cloud.google.com/projectselector/kubernetes). If you are new to Google cloud you'll need to sign in with your Google account.

Create or select a project - their user interface is fairly intuitive. You'll need to wait a little while for the K8s API and related services to be activated.

Now visit the [Google Cloud Console](https://console.cloud.google.com). At the top of the page you will see a little terminal button. Click on it to activate the Google cloud shell.

If you want to use a terminal on your own computer then you can, it just requires a bit more setup. You would need to install the Google Cloud SDK and the kubectl component and perform some extra configuration that's outside the scope of this text. If you want to go down that route take a look at [this](https://cloud.google.com/sdk/docs/quickstarts). You'll be interacting with your cluster through use of the gcloud command line utility.

Ok, so back to the shell: Run the following command to create your cluster:
```[bash]
gcloud container clusters create hello-timber --num-nodes=3 --zone=europe-west1-d
```

This will create a three node cluster called hello-timber. It might take a few minutes. Each node is a compute instance (VM) managed by Google. I've chosen here to put the nodes in west Europe. If you want to see a full list of available zones you can use the command `gcloud compute zones list`. Alternatively you can refer to [this document](https://cloud.google.com/compute/docs/regions-zones/) for a list of available zones.

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

```[bash]
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

In [part 1](http://todo) we covered creating a docker image. Now we need to make our image available to our cluster. This means we need to put our image into a [container registry](https://cloud.google.com/container-registry/) that Google can access. Google cloud has a container registry built in.

In order to make use of Google's registry you will need to tag your images appropriately. In the commands below you will notice that our image tags include `eu.gcr.io`. This is because we are using a zone in Europe. If you chose to put your cluster in a different zone then just refer to [this document](https://cloud.google.com/container-registry/docs/pushing-and-pulling) and update your build and push commands appropriately.

On the Google cloud shell:

```[bash]
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
docker build -t eu.gcr.io/${PROJECT_ID}/timber-tutorial:v1 .
## notice the funny name. eu.gcr.io refers to google's container registry in
## Europe. If your cluster is not in Europe (eg you chose a US zone when creating
## your cluster) then you must use a different url. Refer to this document for
## details: https://cloud.google.com/container-registry/docs/pushing-and-pulling

## once you are finished building the image, push it
docker push eu.gcr.io/${PROJECT_ID}/timber-tutorial:v1
```

Now we have our application built as an image, and that image is available to our cluster! The next step is to create a deployment and then expose that deployment to the outside world as a service.

### Get your application to run as a deployment

Inside the Google cloud shell do the following:

```[bash]
kubectl run timber-tutorial --image=eu.gcr.io/${PROJECT_ID}/timber-tutorial:v1 --port 8080
```

The output should be something like:

```
deployment "timber-tutorial" created
```
As in: You just created a K8s Deployment!

Now let's take a look at what we have done. First take a look at the deployment we just created.

```[bash]
kubectl get deployments
```
This will give you a brief summary of the current deployments. The output will look like:

```
NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
timber-tutorial   1         1         1            1           57s
```

Now a deployment manages pods. So let's look at those:

```[bash]
kubectl get pods
```
This outputs:

```
NAME                              READY     STATUS    RESTARTS   AGE
timber-tutorial-99f796786-r92fv   1/1       Running   0          1m
```

Alright! Now we need to expose our application to the Internet by using a K8s service. This is similarly easy:

```[bash]
kubectl expose deployment timber-tutorial --type=LoadBalancer --port 80 --target-port 8080
```

This outputs:
```
service "timber-tutorial" exposed
```

Now this is a bit more complex. What it does is tell Google that we want a LoadBalancer. Loadbalancers decide which Pods should get incident traffic. So if I have a bunch of replicas of my pod running, and a bunch of clients trying to access my application then Google will use it's infrastructure to spread the traffic over my pods. This is a very simplified explanation. And there are a few more kinds of services you might want to know about if you are actually building a microservices project.

Remember this line from our Dockerfile?
```[dockerfile]
CMD ["gunicorn", "-b", "0.0.0.0:8080", "main:__hug_wsgi__"]
```

Our container is expecting to communicate with the outside world via port 8080. That's the `--target-port`. The port we want to communicate with is `80`. So we just tell the service that we want to tie those ports together.

Let's take a look at how our service is doing
```[bash]
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

```[bash]
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

```[bash]
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

```[bash]
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

```[bash]
git checkout -b version2 origin/version_2
```

Now let's build and push a new version of our image:

```[bash]
docker build -t eu.gcr.io/${PROJECT_ID}/timber-tutorial:v2 .
docker push eu.gcr.io/${PROJECT_ID}/timber-tutorial:v2
```

Now we tell the deployment to run our new image:

```[bash]
kubectl set image deployment/timber-tutorial timber-tutorial=eu.gcr.io/${PROJECT_ID}/timber-tutorial:v2
```

And watch our pods:
```[bash]
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

```[bash]
kubectl get pods
```

this outputs:

```
NAME                               READY     STATUS    RESTARTS   AGE
timber-tutorial-66c6545dd9-2lnlt   1/1       Running   0          2m
timber-tutorial-66c6545dd9-h7vfv   1/1       Running   0          1m
timber-tutorial-66c6545dd9-s44xm   1/1       Running   0          2m
```

So what just happened? We started off with three pods of running `timber-tutorial:v1` now we have three pods running `timber-tutorial:v2`. K8s doesn't just kill all the running pods and create all the new ones all at the same time. It is possible to make that happen but it would mean that the application would be down for a little while. Instead, K8s will start bringing new pods online while terminating old ones and re balancing traffic. That means that anyone accessing your application will be able to keep accessing it while the update is being rolled out. This is of course only part of the story, K8s is capable of performing all sorts of health and readiness checks along the way, and can be configured to roll things out in different ways.

Let's go back to our local terminal and see what our application is doing how:

```[bash]
curl ${EXTERNAL_IP}/index
```

This outputs:

```
{"timber": "so delicious", "why": "because logging shouldn't be hard"}
```

## IMPORTANT

Clusters cost money! So you'll want to shut yours down when you are done with it. We'll talk about cleanup at the end of the [next part of this series](http://todo). If you aren't keen to dive into CI/CD just yet then that is totally fine, just skip to the section on cleanup.

## Conclusion

Well done for getting this far! Seriously, you've learned a lot. In this part of the series we set up K8s clusters on Google's cloud; and we covered some basic K8s object configuration skills including scaling and upgrading pods on the fly. That's a lot of ground to cover! Give yourself a pat on the back.

That said, the examples in this article are of a hello-world variety. There is a lot more worth learning. From architecting and testing; to versioning and monitoring of live applications. If you want more of a deep dive into those kinds of topics, I highly recommend [this book](https://www.amazon.com/gp/product/1491950358/ref=as_li_tl?ie=UTF8&tag=sheena0d-20&camp=1789&creative=9325&linkCode=as2&creativeASIN=1491950358&linkId=af50b90636d80798e5e85c43a83d7b5f). I've personally found it quite useful.

The [final post](http://todo) on this series will cover setting up a basic CI/CD pipeline for our application, as well as the all important cleanup.



