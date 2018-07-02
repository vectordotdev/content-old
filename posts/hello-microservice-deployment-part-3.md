# Hello Microservice Deployment Part 3: CI/CD with Drone.io

Welcome back. This is the third and final part of our series on microservice deployment.

In [Part 1](http://todo) we got acquainted with Docker by building an image for a simple web app and then running that image. In [Part 2](http://todo) we got our application online by deploying it to a Kubernetes cluster that we set up ourselves on Google Cloud. We'll also deal with the basics of scaling and updating our application. So far, we have been working with [this repo](https://gitlab.com/sheena.oconnell/tutorial-timber-deploying-microservices.git).

Here, we'll get a simple CI/CD pipeline up and running so that our code changes are automatically tested and deployed as soon as we push them.

## So what's the deal with CI/CD?

Continuous Integration and Continuous Deployment are the next steps to go through if you want a production-grade microservice application. Let's revisit Webflix to make this point clear. Webflix is running a whole lot of services in K8s (Kubernetes). Each of these services is associated with some code stored in a repository somewhere. Let's say Webflix wisely chooses to use Git to store their code, and they follow a feature branching strategy.

Now, the process of deploying code to production is not so straightforward - first, we should make sure all the unit tests pass and have good coverage.   It might also be necessary to measure the performance of the new image by running some tests with a tool like [locust](https://locust.io/). The deployment process can get very complex if multiple developers are working services at the same time since we would need to keep track of version compatibility for the various functions.

CI/CD is all about automating this sort of thing.

There are loads of CI/CD tools around, and they have their ways of configuring the pipelines (a pipeline is a series of steps code needs to go through when it's pushed). Here's a general outline:

1. unit test the code
2. build the container
3. set up a test environment where the new container can run within a realistic context
4. run some tests
5. deploy to the production system
6. notify team of success/failure

If, for example, one of the test steps fails, then the code will not get deployed to production. The pipeline will skip to the end of the process and notify the team that the deployment was a failure.

You can also set up pipelines for merge/pull requests, e.g. if a developer requests a merge then execute the above pipeline but LEAVES OUT STEP 5 (deploying to production).

## Drone.io

Drone is a container based Continuous Delivery system. It's open source, highly configurable (every build step is executed by a container!) and has a lot of [plugins](http://plugins.drone.io/) available. It's also one of the easiest CI/CD systems to learn.

## Practical: Setting up Drone

In this section, we're going to set up drone on a VM in Google Cloud and get it to play nice with Gitlab. It works fine with Github and other popular Git applications as well, I just like Gitlab.

Now I'll be working on the assumption that you have been following along since part 1 of this series. We already have a K8s cluster set up on Google cloud, and it is running a Deployment containing a straightforward web app. Thus far we've been interacting with our cluster via the Google cloud shell. If you're lost, feel free to take a look at [Part 2](http://todo).


### Setup Infrastructure

The first thing we'll do is set up a VM (Google cloud calls this a compute instance) with a static IP address, and we'll make sure that Google's firewall lets in HTTP traffic.

When working with compute instances we need to continuously be aware of regions and zones. It's not too complex - in general you just want to put your compute instances close to where they will be accessed from. I'll be using `europe-west1-d` as my zone, and `europe-west1` as my region. Feel free just to copy me for this tutorial. Alternatively, take a look at [Google's documentation](https://cloud.google.com/compute/docs/regions-zones/) and pick what works best for you.


The first step is to reserve a static IP address. We have named ours `drone-ip`
```[bash]
gcloud compute addresses create drone-ip --region europe-west1
```

This outputs:
```
Created [https://www.googleapis.com/compute/v1/projects/timber-tutorial/regions/europe-west1/addresses/drone-ip].
```

Now take a look at it and take note of the actual IP address. We'll need it later:
```[bash]
gcloud compute addresses describe drone-ip --region europe-west1
```
This outputs something like:
```
address: 35.233.66.226
creationTimestamp: '2018-06-21T02:40:37.744-07:00'
description: ''
id: '431436906006760570'
kind: compute#address
name: drone-ip
region: https://www.googleapis.com/compute/v1/projects/timber-tutorial/regions/europe-west1
selfLink: https://www.googleapis.com/compute/v1/projects/timber-tutorial/regions/europe-west1/addresses/drone-ip
status: RESERVED
```
So the ip address that I just reserved is `35.233.66.226`. Yours will be different.


Now create a VM:

```[bash]
gcloud compute instances create drone-vm --zone=europe-west1-d
```
this outputs:

```
Created [https://www.googleapis.com/compute/v1/projects/timber-tutorial/zones/europe-west1-d/instances/drone-vm].
NAME      ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
drone-vm  europe-west1-d  n1-standard-1               10.132.0.6   35.195.196.332  RUNNING
```

Now we have a VM and a static IP. We need to tie them together:

First, let's look at the existing configuration for our VM:
```[bash]
gcloud compute instances describe drone-vm --zone=europe-west1-d
```
This outputs a whole lot of stuff. Most importantly:

```
networkInterfaces:
- accessConfigs:
  - kind: compute#accessConfig
    name: external-nat
    natIP: 35.195.196.332
    type: ONE_TO_ONE_NAT
```
A VM can at most have one of accessConfig. We'll need to delete the existing one and replace it with a static IP address config. First, we delete it:

```[bash]
gcloud compute instances delete-access-config drone-vm \
    --access-config-name "external-nat" --zone=europe-west1-d
```
this outputs:
```
Updated [https://www.googleapis.com/compute/v1/projects/timber-tutorial/zones/europe-west1-d/instances/drone-vm].
```

Now we add new network configuration:

```[bash]
gcloud compute instances add-access-config drone-vm \
    --access-config-name "drone-access-config" --address 35.233.66.226 --zone=europe-west1-d
```

This outputs:

```
Updated [https://www.googleapis.com/compute/v1/projects/timber-tutorial/zones/europe-west1-d/instances/drone-vm].
```

And now we need to configure the firewall to allow HTTP traffic. Google's firewall rules can be added and removed from specific instances through use of tags.

```[bash]
gcloud compute instances add-tags drone-vm --tags http-server --zone=europe-west1-d
```
This outputs:

```
Updated [https://www.googleapis.com/compute/v1/projects/timber-tutorial/zones/europe-west1-d/instances/drone-vm].
```

Awesome! Now we have a VM with a static IP address, and it can talk to the outside world via HTTP.

### Install Prerequisites

To get Drone to run, we need to install Docker and Docker-Compose. Let's do that now:

SSH into our VM. From your Google cloud shell like so:

```[bash]
export PROJECT_ID="$(gcloud config get-value project -q)"

gcloud compute --project ${PROJECT_ID} ssh --zone "europe-west1-d" "drone-vm"
```

When it asks for passphrases, you can leave them blank for this tutorial. That said, it's not good practice.

Ok, now you have a shell into your new VM. Brilliant.

Now enter:
```[bash]
uname -a
```

This will output something like
```
Linux drone-vm 4.9.0-6-amd64 #1 SMP Debian 4.9.88-1+deb9u1 (2018-05-07) x86_64 GNU/Linux
```

Next up we [install Docker](https://docs.Docker.com/install/linux/Docker-ce/debian/) and [docker-compose](https://docs.Docker.com/compose/install/#install-compose)

### Generate Gitlab oAuth credentials

In GitLab, go to your user settings and click on applications. You want to create a new application. Enter `drone` as the name. As a callback URL use `http://35.233.66.226/authorize`. The IP address there is the static IP address we just generated.

Gitlab will now output an application id and secret and some other stuff. Take note of these values, drone is going to need them.

### Create a repo on Gitlab

You'll also need to create a git repo that you own. Thus far we've been using `https://gitlab.com/sheena.oconnell/tutorial-timber-deploying-microservices.git`. You would want something like `https://gitlab.com/${YOUR_GITLAB_USER}/tutorial-timber-deploying-microservices.git` to exist. Don't put anything inside your repo just yet, we'll get to that a bit later.

### Configure Drone

First, we'll need to set up some environmental variables. Let's make a new file called `.drone_secrets.sh`

```[bash]
nano .drone_secrets.sh
```

This opens an editor.  Paste the following into the editor

```[bash]
#!/bin/sh

export DRONE_HOST=http://35.233.66.226
export DRONE_SECRET="some random string of your choosing"
export DRONE_ADMIN="sheena.oconnell"
export DRONE_GITLAB_CLIENT=<client>
export DRONE_GITLAB_SECRET=<secret>
```

You'll need to update it a little bit:

`DRONE_HOST`: this should contain your static ip address.
`DRONE_SECRET`: any random string will work. Just make something up or use a random password generator like [this one](https://www.random.org/passwords/?num=1&len=16&format=html&rnd=new)
`DRONE_ADMIN`: your gitlab username. Mine is `sheena.oconnell`.
`DRONE_GITLAB_CLIENT`: copy your gitlab application id here
`DRONE_GITLAB_SECRET`: copy your gitlab client secret here

Once you have finished editing the file then press: `Ctrl+x` then `y` then `enter` to save and exit.

Now make it executable:
```[bash]
chmod +x .drone_secrets.sh
```
And load the secrets into your environment
```
source .drone_secrets.sh
```

Cool, now let's make another file:

```[bash]
nano Docker-compose.yml
```
Paste in the following:


```[yml]
version: '2'

services:
  drone-server:
    image: drone/drone:0.8

    ports:
      - 80:8000
      - 9000
    volumes:
      - /var/lib/drone:/var/lib/drone/
    restart: always
    environment:
      - DRONE_HOST=${DRONE_HOST}
      - DRONE_SECRET=${DRONE_SECRET}
      - DRONE_ADMIN=${DRONE_ADMIN}
      - DRONE_GITLAB=true
      - DRONE_GITLAB_CLIENT=${DRONE_GITLAB_CLIENT}
      - DRONE_GITLAB_SECRET=${DRONE_GITLAB_SECRET}

  drone-agent:
    image: drone/agent:0.8
    command: agent
    restart: always
    depends_on:
      - drone-server
    volumes:
      - /var/run/Docker.sock:/var/run/Docker.sock
    environment:
      - DRONE_SERVER=drone-server:9000
      - DRONE_SECRET=${DRONE_SECRET}
```

Now, this file requires a few environmental settings to be available. Luckily we've already set those up. Save and exit just like before.

Now run

```[bash]
docker-compose up
```

There will be a whole lot of output. Open a new browser window and navigate to your drone host. In my case, that is: `http://35.233.66.226`. You will be redirected to an OAuth authorization page on Gitlab. Choose to authorize access to your account. You will then be redirected back to your drone instance, and after a little while, you will see a list of your repos.

Each repo will have a toggle button on the right of the page. Toggle whichever one(s) you want to set up CI/CD. If you have been following along, then there should be a repo called `${YOUR_GITLAB_USER}/tutorial-timber-deploying-microservices`. Go ahead and activate that one.

You can refer to the [drone documentation](http://docs.drone.io/installation/) for further instructions.

### Recap

Alright! So far so good. We've got Drone.ci all set up and talking to Gitlab.

## Practical: Giving drone access to Google Cloud

Before we get into the meat of actually running a pipeline with drone, we'll need a way for drone to authenticate with our Google project. Before we were just interacting as ourselves via the `gcloud` tooling built into the Google cloud shell (or installed locally if you wanted to do things that way). We want drone to have a subset of our user rights.

We start off by creating a service account. This is similar to a user. Like users, service accounts have credentials and rights, and they can authenticate with Google cloud. To learn all about service accounts, you can refer to [Google's official docs](https://cloud.google.com/iam/docs/creating-managing-service-accounts).

Open up another Google cloud shell and do the following:

```[bash]
gcloud iam service-accounts create drone-sa \
    --display-name "drone-sa"
```
this outputs:
```
Created service account [drone-sa].
```

Now we want to give that service account permissions. It will need to push images to the Google cloud container registry (which is based on Google Storage), and it will need to roll out upgrades to our application deployment.

```[bash]
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member serviceAccount:drone-sa@${PROJECT_ID}.iam.gserviceaccount.com --role roles/storage.admin

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member serviceAccount:drone-sa@${PROJECT_ID}.iam.gserviceaccount.com --role roles/container.developer
```

These commands output stuff like:
```
bindings:
- members:
  - serviceAccount:service-241386104325@compute-system.iam.gserviceaccount.com
  role: roles/compute.serviceAgent
- members:
  - serviceAccount:drone-sa@timber-tutorial.iam.gserviceaccount.com
  role: roles/container.developer
- members:
  - serviceAccount:service-241386104325@container-engine-robot.iam.gserviceaccount.com
  role: roles/container.serviceAgent
- members:
  - serviceAccount:241386104325-compute@developer.gserviceaccount.com
  - serviceAccount:241386104325@cloudservices.gserviceaccount.com
  - serviceAccount:service-241386104325@containerregistry.iam.gserviceaccount.com
  role: roles/editor
- members:
  - user:yourname@gmail.com
  role: roles/owner
- members:
  - serviceAccount:drone-sa@timber-tutorial.iam.gserviceaccount.com
  role: roles/storage.admin
etag: BwVvOooDQaI=
version: 1
```

If you wanted to give your drone instance access to other Google cloud functionality (for example if you needed it to interact with App Engine), you can get a full list of available roles like so:
```[bash]
gcloud iam roles list
```

Now we create some credentials for our service account. Any device with this key file will have all of the rights given to our service account. You can invalidate key files at any time. We are going to name our key `key.json`

```[bash]
gcloud iam service-accounts keys create ~/key.json \
    --iam-account drone-sa@${PROJECT_ID}.iam.gserviceaccount.com
```
this outputs:
```
created key [7ce29ec3d260c55c5ff1b32aad40a331f15edc63] of type [json] as [/home/sheenaprelude/key.json] for [drone-sa@timber-tutorial.iam.gserviceaccount.com]
```

Now we need to make the key available to drone. We'll do this by using drone's frontend. Point your browser at the drone frontend (in my case http://35.233.66.226). Then Navigate to the repository that you want to deploy. Then click on the menu button on the top right of the screen and select secrets.

Now create a new secret called `GOOGLE_CREDENTIALS`

Now in your Google cloud shell:

```[bash]
cat key.json
```
The output should be something like this:

```
{
  "type": "service_account",
  "project_id": "timber-tutorial",
  "private_key_id": "111111111111111111111111",
  "private_key": "-----BEGIN PRIVATE KEY-----\n lots and lots of stuff =\n-----END PRIVATE KEY-----\n",
  "client_email": "drone-sa@timber-tutorial.iam.gserviceaccount.com",
  "client_id": "xxxxxxxxxxxxxxxx",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://accounts.google.com/o/oauth2/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/drone-sa%40timber-tutorial.iam.gserviceaccount.com"
}
```

Copy it and paste it into the `secret value` field and click on `save`

## Practical: Finally, a pipeline!!

Now Drone has access to our Google Cloud resources (although we still need to tell it how to access the key file), and it knows about our repo. Now we need to tell drone what exactly we need done when we push code to our project.  We do this by specifying a pipeline in a file named `.drone.yml` in the root of our git repo. `.drone.yml` is written in YAML format. [Here](https://lzone.de/cheat-sheet/YAML) is a cheat-sheet that I've found quite useful.

It's time to put something in your tutorial-timber-deploying-microservices repo. We'll copy over everything from the repo created for this series of articles. In a terminal somewhere (your local computer, google cloud shell, or wherever):

```[bash]
# clone the repo if you haven't already and cd in
git clone https://gitlab.com/sheena.oconnell/tutorial-timber-deploying-microservices.git
cd tutorial-timber-deploying-microservices

# set the git repo origin to your very own repo
git remote remove origin
git remote add origin https://gitlab.com/${YOUR_GITLAB_USER}/tutorial-timber-deploying-microservices.git

# and push your changes. This will trigger the deployment pipeline already specified by my .drone.yml
git checkout master
git push --set-upstream origin master
```

This will kick off our pipeline. You can watch it happen in the drone web frontend.

Here is our full pipeline specification:

```[yaml]
pipeline:
  unit-test:
    image: python:3
    commands:
      - pip install -r requirements.txt
      - python -m pytest

  gcr:
    image: plugins/gcr
    registry: eu.gcr.io
    repo: timber-tutorial/timber-tutorial
    tags: ["commit_${DRONE_COMMIT}","build_${DRONE_BUILD_NUMBER}", "latest"]
    secrets: [GOOGLE_CREDENTIALS]
    when:
      branch: master

  deploy:
    image: Google/cloud-sdk:latest
    environment:
      PROJECT_ID: timber-tutorial
      COMPUTE_ZONE: europe-west1-d
      CLUSTER_NAME: hello-timber
    secrets: [GOOGLE_CREDENTIALS]
    commands:
      - yes | apt-get install python3
      - python3 generate_key.py key.json
      - gcloud config set project $PROJECT_ID
      - gcloud config set compute/zone $COMPUTE_ZONE
      - gcloud auth activate-service-account --key-file key.json
      - gcloud container clusters get-credentials $CLUSTER_NAME
      - kubectl set image deployment/timber-tutorial timber-tutorial=eu.gcr.io/${PROJECT_ID}/app:${DRONE_BUILD_NUMBER}
    when:
      branch: master

```

As pipelines go, it's quite a small one. We have specified three steps: `unit-test`, `gcr` and `deploy`. It helps to keep Docker-compose in mind when working with drone. Each step is run as a Docker container. So each step is based on a Docker image. And for the most part, you get to specify what happens on those containers through use of `commands`.

Let's start from the top.
```[yaml]
  unit-test:
    image: python:3
    commands:
      - pip install -r requirements.txt
      - python -m pytest
```

This step is relatively straightforward. Whenever any changes are made to the repo (on any branch) then the unit tests are run. If the tests pass then drone will proceed to the next step. In our case, all the rest of the steps only happen on the master branch, so if you are in a feature branch, the only thing this pipeline will do is run unit tests.

```[yaml]
  gcr:
    image: plugins/gcr
    registry: eu.gcr.io
    repo: timber-tutorial/timber-tutorial
    tags: ["commit_${DRONE_COMMIT}","build_${DRONE_BUILD_NUMBER}", "latest"]
    secrets: [GOOGLE_CREDENTIALS]
    when:
      branch: master
```

The `gcr` step is all about building our application Docker image and pushing it into the Google Cloud Registry (GCR). It is a special kind of step as it is based on a plugin. We won't go into detail on how plugins work here. Just think of it as an image that takes in special parameters. This one is configured to push images to `eu.gcr.io/timber-tutorial/timber-tutorial`.

The `tags` argument contains a list of tags to be applied. Here we make use of some variables supplied by Drone. `DRONE_COMMIT` is the git commit hash. And each build of each repo is numbered, so we use that as a tag too. Drone supplies a whole lot of variables, take a look [here](http://readme.drone.io/0.5/usage/environment-reference/) for a nice list.

The next thing is `secrets`. Remember that secret we copy-pasted into drone just a few minutes ago? Its name was `GOOGLE_CREDENTIALS`. This line makes sure that the contents of that secret are available to the step's container in the form of an environmental variable named `GOOGLE_CREDENTIALS`.

The last step is a little more complex:

```[yaml]
deploy:
    image: Google/cloud-sdk:latest
    environment:
      PROJECT_ID: timber-tutorial
      COMPUTE_ZONE: europe-west1-d
      CLUSTER_NAME: hello-timber
    secrets: [GOOGLE_CREDENTIALS]
    commands:
      - yes | apt-get install python3
      - python3 generate_key.py key.json
      - gcloud config set project $PROJECT_ID
      - gcloud config set compute/zone $COMPUTE_ZONE
      - gcloud auth activate-service-account --key-file key.json
      - gcloud container clusters get-credentials $CLUSTER_NAME
      - kubectl set image deployment/timber-tutorial timber-tutorial=eu.gcr.io/${PROJECT_ID}/app:${DRONE_BUILD_NUMBER}
    when:
      branch: master
```

Here our base image is supplied by Google. It gives us `gcloud` and a few bells and whistles.

`environment` lets us set up environmental variables that will be accessible in the running container, and the `secrets` work as before.

Now we have a bunch of commands. These execute in order and you should recognize most of it. The only really strange part is how we authenticate as our service account (`drone-sa`). The line that does the actual authentication is `gcloud auth activate-service-account --key-file key.json`. It requires a key file. Now ideally we would just do something like this:

```[yaml]
- echo $GOOGLE_CREDENTIALS > key.json
- gcloud auth activate-service-account --key-file key.json
```

Unfortunately, Drone completely mangles the whitespace of our secret. Thus `generate_key.py` exists to de-mangle the key, so it is useful (gee, thanks Drone!). Of course, Python needs to be available so we can run that script. Thus `yes | apt-get install python3`.

Now that everything is set up, if you make a change to your code and push it to master, then you will be able to watch the pipeline get executed by keeping an eye on the drone front-end.

Once the pipeline is complete you will be able to make sure that your deployment is updated by taking a look at the Pods on the gcloud command line:

```[bash]
kubectl get pods
```
outputs:

```
NAME                               READY     STATUS    RESTARTS   AGE
timber-tutorial-77d67cbfb8-flfxl   1/1       Running   0          49s
timber-tutorial-77d67cbfb8-gfl7z   1/1       Running   0          49s
timber-tutorial-77d67cbfb8-p9bcd   1/1       Running   0          40s
```

Now pick one and describe it:

```[bash]
kubectl describe pod timber-tutorial-77d67cbfb8-flfxl
```

We'll get a whole lot of output here. The part that is interesting to us is:
```
Containers:
  timber-tutorial:
    Image:          eu.gcr.io/timber-tutorial/timber-tutorial:commit_cb5d5ca61661954d7d139b2a1d60060cba5c4f2f

```

Now if you were to check your git log, the last commit to master that you pushed would have the commit SHA `cb5d5ca61661954d7d139b2a1d60060cba5c4f2f`. Isn't that neat?

## Conclusion

Wow, we made it! If you've worked through all the practical examples, then you've accomplished a lot.

You are now acquainted with Docker - you built an image and instantiated a container for that image. Then you got your images running on a Kubernetes cluster that you set up yourself. You then manually scaled and rolled out updates to your application.

And then in this part you got a simple CI/CD pipeline up and running from scratch by provisioning a VM, installing Drone and it's prerequisites, and getting it to play nice with Gitlab and Google Kurbenetes Engine.

On the other hand, if you worked through all the practical examples, then this is just the beginning. I sincerely hope that this article has been useful to you on your journey toward microservice mastery.

Cheers,
Sheena

## PS - Cleanup (IMPORTANT!)

Clusters cost money so it would be best to shut it down if you aren't using it. Go back to the Google Cloud Shell and do the following:

``` [bash]
kubectl delete service hello-timber-service

## now we need to wait a bit for Google to delete some forwarding rules for us. Keep an eye on them by executing this command:
gcloud compute forwarding-rules list

## once the forwarding rules are deleted then it is safe to delete the cluster:
gcloud container clusters delete hello-timber

## and delete the drone VM and it's ip address
yes | gcloud compute instances delete drone-vm --zone=europe-west1-d
yes | gcloud compute addresses delete drone-ip --region=europe-west1
```
