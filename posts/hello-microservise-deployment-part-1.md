# Hello Microservice Deployment Part 1: Docker

This article is the first of a three part series all about deploying microservices. In this part we'll cover the basics of what problems microservices solve (and cause), and we'll create a docker image and run it locally as a container.

In [Part 2](http://todo) we'll get our application online by deploying it to a Kubernetes cluster that we set up ourselves on Google Cloud. We'll also deal with the basics of scaling and updating our application.
In [Part 3](http://todo) we'll use Drone.io to set up a simple CI/CD pipeline. It will test our application and roll out any changes we make to the master branch of our repo.

## Micro-what?

If you are familiar with the concepts of microservices then feel free to skip this section.

In the beginning there were monoliths. Imagine you have an application running on the cloud somewhere that handles authentication; authorization; sending emails to members; keeping track of fees and membership details; distributing large volumes of entertaining and thought-provoking videos; keeping track of who has seen which video; making recommendations to members based on their viewing history. Let's call our totally hypothetical application Webflix.

Let's say we write the entire Webflix application as a single application - a monolith. There is quite a lot going on in there, which means quite a lot of testing would need to happen every time any changes are made. Eg: if I make a change to the code responsible for recommendation then I should run the entire unit test suite - after all there might be some coupling between the recommendation code and some other part of the system. Now once my tests pass I can deploy the code but that is quite a pain as well because there sure is a lot of it. I would need to rebuild absolutely everything and deploy a big thing, or introduce some complexity by introducing a kind of packaging system for different code units. Rolling back is similarly inefficient or complex.

Wouldn't it be nice to be able to scale up (or down) Webflix to demand? For example at peak watching time you would want to scale up the subsystem responsible for serving videos but you might not want to scale up anything else. In a monolith architecture, scaling is all or nothing.

Enter Microservices.

The idea with microservices is that many applications could benefit from being broken down into smaller applications (called services) that play well together. So going back to Webflix, we would have one service for recommendations; one service for distributing videos; one service for managing payments. Each service is defined by its purpose and responsibilities. Sometimes, however, things are not so clear. For example, should authentication and authorization be one service or two? And beyond that, Webflix needs to enforce a no-pay-no-watch policy. This means that the authorization system needs to be aware of the payment system. And the payment and forgot-password and user-registration systems all need to be able to send emails.

So we see that sometimes the boundaries between services are pretty clear and obvious. But sometimes they can be a little fuzzy. And the services need to communicate effectively.

After breaking things down as above, let's zoom in onto a single well-defined service: the recommendation system. If I make a change to the code of the recommendation service I just need to run the unit tests for that one system, I can deploy and roll back that service individually and I can scale it individually as needed. I can even deploy multiple versions for a touch of A/B testing. Winning! But wait, there are other systems involved - the recommendation system needs to be accessed by a end-user facing front-end, and it needs to know about what the user has already watched. That introduces some complexity because there is a chance that a change to the recommendation service will break communication with the functionality that it talks to. There is only so much that can be done with mocks in unit tests. That means that we'll need to build a new layer of testing that makes sure that the various services do in fact play nice.

Then there is the problem of version compatibility: Let's say we are running recommend_service:1.2.0 and video_history_service:1.3.0 and those play nice. But then recommend_service:1.3.0 is created and that one breaks the overall system. So a patch is created to fix it (recommend_service:1.3.1) but in the meantime recommend_service:1.4.1 is released.

Alrighty, so lets say we get everything deployed. Now these services need to communicate effectively. We could just use HTTP but what if the network is just slightly unreliable? What if a service is in the middle of an upgrade? What if the traffic is lost or replayed? Some thought will need to go in there. And what if something breaks without raising an exception? For example, what if "Barney the Dinosaur" is recommended to a hard code zombie fan? The history, user-preference and recommendations services should all be examined. And what if there is a spike in latency? Any number of individual services could be at fault.

So there are pros and cons to both microservices and monoliths (monoliths of course don't introduce the complexities of microservices - communication, deployment and testing are much easier). The two styles are appropriate for different projects. But whichever route you go down, you will need to deploy your code. And that is really what this tutorial is about.

## Introducing Docker

If you are comfortable with Docker then feel free to skip this section :)

![Bare Metal](/images/hello_micro_deploy/deploy-micro-metal.png)

You run applications on your computer all the time. Your computer is a bare-metal machine. Applications run on top of your operating system (OS) and the OS manages the hardware. There are a whole host of problems with deploying applications to bare metal which I wont get into here. Many of those problems are overcome by Virtual Machines(VMs).

![Virtual Machine](/images/hello_micro_deploy/deploy-micro-VM.png)

A VM runs on a hypervisor, the hypervisor runs on the host OS and the host OS controls the hardware. There can be multiple unrelated and isolated VMs on top of a single hypervisor. The hypervisor's job is primarily to allocate resources to the various VMs. Now your application would run on top of an OS installed onto a VM.

But the VMs are quite heavy - wouldn't it be nice to strip away all those extra OSs? Containers do that. A container can be thought of as a really lightweight VM.

![Containers](/images/hello_micro_deploy/deploy-micro-containers.png)

Containers are much smaller than VMs by default - a container doesn't contain a full operating system, whereas a VM does. Containers thus require a lot less in terms of processing power in order to run - they tend to be mere megabytes in size and take just seconds to start. VMs tend to be Gigabytes in size and can take minutes to start, operating systems are big!  Containers make use of libraries and packages on the host operating system in order to access resources; then make use of their own libraries and packages in order to emulate seperate operating systems as needed instead of installing unnecessary bloat. Eg: you could run an Ubuntu image on your Windows machine without needing to install Ubuntu.

VMs (like bare metal) tend to accumulate undocumented bloat as utilities are installed, updated, and generally messed with on the fly. Containers are fully specified in code, and easy enough to re-create that manually messing around with their internals usually isn't needed at all.

The smallness of containers is of course really great when it comes to scaling applications. If we were hosting recommend_service as a VM and had a spike in traffic then users would just have to deal with some extra latency and perhaps some errors while we brought extra VMs online. On the other hand if we were using containers we would be able to bring them online much faster, and potentially even keep extra instances running just in case of spikes because containers are cheap.

[Docker](https://www.docker.com/) is the de facto industry standard container platform and that's the one that we'll be dealing with in this article.

## Practical: lets make an api

Docker allows the creation of images, images are instantiated to create containers (if you are familiar with object orientated programming then images are like classes, and containers are like objects). In this section we will create and run a container, the container will contain a service we wish to deploy.

We'll start off by making a simple Python [Hug](http://www.hug.rest/) application and running it locally. Hug is a framework for creating super-fast, self-documenting APIs no matter how those APIs are exposed. In our case we'll we using it to expose a simple API via `HTTP`. Basically you create plain old Python functions and then define how (and if) you want them to be exposed through use of decorators. It is fairly new on the scene but considered production ready.

This tutorial assumes you can install python3, [virtualenvwrapper](http://virtualenvwrapper.readthedocs.io/en/latest/) and git on your own. YOu can learn more about Python 3 virtual environments [here](https://docs.python.org/3/library/venv.html). Virtualenvwrapper simply provides tooling to make the management of your virtual environments easier.

We're going to start off by running our app locally. We don't actually *need* to install and run our app locally, but it may add some clarity to the discussion around creating our image.

```[bash]
## create and activate your virtual environment. Application dependencies will be installed here instead of globally. This has nothing to do with containers really, it's just a special directory and path configuration. Also it is good practice

mkvirtualenv --python=`which python3` timber_deployment_tutorial

## clone the application

git clone https://gitlab.com/sheena.oconnell/tutorial-timber-deploying-microservices.git

## install the application requirements

cd tutorial-timber-deploying-microservices
pip install -r requirements.txt

## run the application

hug -f main.py

```
You should get an output something like this:

```[bash]
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

```[bash]
curl 0.0.0.0:8000/index
```
This should return the response:

```
{"timber": "so delicious"}
```

Isn't that nice?

## Practical: lets run our api as a Docker container

Now that you know that the application basically works, it's time to build an image. Notice that the repo contains a Dockerfile. Inside the Dockerfile you'll find something this:

```[dockerfile]
FROM python:3
EXPOSE 8080

WORKDIR /usr/src/app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "-b", "0.0.0.0:8080", "main:__hug_wsgi__"]
```

If you want to learn more about pulling and pushing images then take a look at [the docs](https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/pull-and-push-images/#pull-an-image).

Ok, now we can build the image:

```[bash]
docker build -t timber-tutorial:1 .
```

And then to run the image (that is, create a container):

```[bash]
docker run -p 8080:8080 timber-tutorial:1
```

You should see output like so:

```
[2018-06-15 07:11:59 +0000] [1] [INFO] Starting gunicorn 19.8.1
[2018-06-15 07:11:59 +0000] [1] [INFO] Listening at: http://127.0.0.1:8000 (1)
[2018-06-15 07:11:59 +0000] [1] [INFO] Using worker: sync
[2018-06-15 07:11:59 +0000] [9] [INFO] Booting worker with pid: 9
```

Open up another terminal then curl the api and make sure it still all works:

```[bash]
curl 0.0.0.0:8080/index
```

This outputs:
```
{"timber": "so delicious"}
```

Notice we are using port 8080 instead of 8000 here. You can specify whatever port you want.

### A very brief introduction to versions

So now we have an image that we can run on any computer (bare metal or VM) that can run docker images. Our image has a name and a version number. If we wanted to make any changes to the functionality of our image then we would specify those changes in code and then re build the image with a different version tag. eg:

```[bash]
docker build -t timber-tutorial:2 .
```

## Summary

Well done :) You've managed to build and tag a docker image and run it as a container.

There is quite a lot more to be said about Docker that is outside the scope of this text. I suggest you take a look at the [official documentation](https://docs.docker.com/) if you need more details. Or if you need something more structured then there are a lot of truly excellent books like [this one](https://www.amazon.com/gp/product/1521822808/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1521822808&linkCode=as2&tag=sheena0d-20&linkId=b8572b1cb6f525a7e11977cdfaf953ba) and [this one](https://www.amazon.com/gp/product/1491950358/ref=as_li_tl?ie=UTF8&tag=sheena0d-20&camp=1789&creative=9325&linkCode=as2&creativeASIN=1491950358&linkId=af50b90636d80798e5e85c43a83d7b5f).

Are you ready for the next step? In [part 2](http://todo) we'll be deploying, scaling and updating our little application on the cloud!


