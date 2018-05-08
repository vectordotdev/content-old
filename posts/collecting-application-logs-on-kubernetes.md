# Collecting Application Logs on Kubernetes

Logs are a commonly used source of data to track, verify, and diagnose the state of a system. Regarding [the three pillars of observability](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html), logs typically offer the richest context and are the closest to the source of an event.

In the current landscape, typically all application and system logs are collected in centralized logging systems and exposed via APIs and Web UIs. These systems are deployed in clusters and receive logs from either the applications themselves or agent processes running on the host.
Centralized logging tools are responsible for parsing, indexing, and analyzing log data to produce on demand insights for their consumers. These insights can range from providing search for error debugging to building reports tracking monthly business performance metrics.

![elk](https://res.cloudinary.com/timber/image/upload/c_scale,w_400/v1526999168/elk_knmiyh.png)

With the rise in popularity of containers, companies are looking at how they can migrate their existing workflows into Docker and onto Kubernetes. Part of that migration is preserving the existing frameworks in place for running software including log collection and analytics. We are going to walk through a few examples of how you can collect and ship your Docker Container logs running on Kubernetes to your centralized logging solution. Whether you are already running a Kubernetes cluster or thinking about it, hopefully, you will learn something new.

_Kubernetes provides excellent high-level documentation on its [logging and log collection strategies](https://kubernetes.io/docs/concepts/cluster-administration/logging/#system-component-logs). We are going to touch on some of the concepts presented while working through our example. We will be focusing on application logs only, leaving system logs for another time._

## Launching our local Kubernetes Cluster

For the following examples, we will be running a local Kubernetes cluster via `minikube`.  If you are not familiar with the [`minikube`](https://github.com/kubernetes/minikube) project, I highly recommend taking a look as it is useful for working with and developing against Kubernetes.

We will also be using an [example go application](https://github.com/timberio/docker-logging-app) that logs the current timestamp every second. This application will make it easy to view and verify our work.

After following the `minikube` [installation instructions](https://github.com/kubernetes/minikube#installation), you should be able to get up and running via the  `start` command.

_The following minikube examples also require [Virtualbox](https://www.virtualbox.org/wiki/Downloads) (5.+). We will be using the latest minikube (v0.27.0) and supported version of Kubernetes (v1.10.0)._

```bash
minikube start --vm-driver virtualbox
```

```text
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```

Once complete, we should be able to use `kubectl` to interact with our cluster.

```bash
kubectl version
```

```text
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.2", GitCommit:"81753b10df112992bf51bbc2c2f85208aad78335", GitTreeState:"clean", BuildDate:"2018-05-12T04:12:12Z", GoVersion:"go1.9.6", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"fc32d2f3698e36b93322a3465f63a14e9f0eaead", GitTreeState:"clean", BuildDate:"2018-03-26T16:44:10Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
```

## Running our `logging app` Pod

_All of the Kubernetes manifest files live [here](https://github.com/timberio/examples/tree/master/kubernetes)._

To run our `logging-app` container, we need to create a Kubernetes Pod. A Pod is a collection of containers run in a shared context and is the smallest deployable Kubernetes resource.

Kubernetes resources are most commonly written as YAML files called manifests, which are evaluated with `kubectl`. In this case, we will be using `kubectl apply` to create resources, and `kubectl delete` to delete resources. Both commands accept a file as an argument allowing for simple setup and teardown.

First, we will create a Pod that runs our `logging-app` container writes its logs to stdout and stderr.

_Writing to stdout and stderr is recommended for processes running in Docker Containers._

```bash
kubectl apply -f https://raw.githubusercontent.com/timberio/examples/master/kubernetes/reading-log-files-from-containers/logging-app-pod.yaml
```

If successful, you should see this message:

```text
pod "logging-app-pod" created
```

You should also be able to see the running Pod:

```bash
kubectl get pods
```

```text
NAME              READY     STATUS    RESTARTS   AGE
logging-app-pod   1/1       Running   0          7s
```

And you should be able to view the Pod's logs:

```bash
kubectl logs pod logging-app-pod
```

```text
Starting timer...
2018-05-17 21:25:26.141379027 +0000 UTC m=+1.001544440
2018-05-17 21:25:27.142059053 +0000 UTC m=+2.002224427
2018-05-17 21:25:28.141096826 +0000 UTC m=+3.001262208
2018-05-17 21:25:29.141103465 +0000 UTC m=+4.001268857
2018-05-17 21:25:30.141240541 +0000 UTC m=+5.001405913
2018-05-17 21:25:31.14116725 +0000 UTC m=+6.001332633
...
```

## What Just Happened?

We accomplished a couple of things since we booted up our cluster. We successfully launched a Pod running our `logging-app` and viewed its logs. But where did those logs come from?

By default in Kubernetes, Docker is configured to write a container's stdout and stderr to a file under `/var/log/containers` on the host system. What we created looks like this:

![kubernetes-logging-node-level](https://d33wubrfki0l68.cloudfront.net/59b1aae2adcfe4f06270b99a2789012ed64bec1f/4d0ad/images/docs/user-guide/logging/logging-node-level.png)

We can verify our logs by ssh-ing into the `minikube` virtual machine and looking at this file.

```bash
minikube ssh
cd /var/log/containers
ls | grep logging-app
```

```text
logging-app-pod_default_logging-app-CONTAINER-ID.log
```

_The container id is a unique alphanumeric string._

Something important to notice about this file is the structure of its name. It turns out it is of the form `POD-NAME_NAMESPACE_CONTAINER-NAME`. All log files written by Kubernetes adhere to this format, which you can see by looking at other files in this directory.

If we look at the contents of that file, we should see the same output that `kubectl logs` gave us, since `kubectl logs` actually reads this file.

```bash
sudo head -n7 logging-app-pod_default_logging-app-CONTAINER-ID.log
```

```text
{"log":"Starting timer...\n","stream":"stdout","time":"2018-05-17T21:25:25.140994702Z"}
{"log":"2018-05-17 21:25:26.141379027 +0000 UTC m=+1.001544440 \n","stream":"stdout","time":"2018-05-17T21:25:26.141762383Z"}
{"log":"2018-05-17 21:25:27.142059053 +0000 UTC m=+2.002224427 \n","stream":"stdout","time":"2018-05-17T21:25:27.14231103Z"}
{"log":"2018-05-17 21:25:28.141096826 +0000 UTC m=+3.001262208 \n","stream":"stdout","time":"2018-05-17T21:25:28.141450911Z"}
{"log":"2018-05-17 21:25:29.141103465 +0000 UTC m=+4.001268857 \n","stream":"stdout","time":"2018-05-17T21:25:29.141328534Z"}
{"log":"2018-05-17 21:25:30.141240541 +0000 UTC m=+5.001405913 \n","stream":"stdout","time":"2018-05-17T21:25:30.141901061Z"}
{"log":"2018-05-17 21:25:31.14116725 +0000 UTC m=+6.001332633 \n","stream":"stdout","time":"2018-05-17T21:25:31.141442254Z"}
```

So we recognize our log lines, but these lines are inside JSON blobs under the field name `log`. This format is the default Docker logging format written by the Docker default logging driver `json-file`. More information about Docker logging drivers can be found [here](https://docs.docker.com/config/containers/logging/configure/).

## Shipping the Logs

Now that we have an app writing logs, it's time to ship them to a centralized logging system. Our implementation will be similar to how existing centralized logging systems are set up, with an agent running on each host but as Kubernetes resources.

Running a logging agent on every host in a Kubernetes cluster can be achieved through the use of a DaemonSet. A DaemonSet is a Kubernetes Controller that ensures a set of Nodes (in this case all) runs a copy of a Pod. The Pod, in this case, will include a container running the logging agent and will mount the necessary host file paths for reading and shipping the log files. This configuration is the most common and encouraged approach for application log collection on Kubernetes.

The following commands will create a DaemonSet running the Timber Agent and ship the logs contained in `/var/log/containers` to timber.io.

__*Note: A [timber.io](https://timber.io/) account and api key are required.*__

We first create a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) to store our Timber API Key:

_If you are still in an ssh session with the minikube virtual machine, you can end it with `exit` or `CTRL-D`._

```bash
kubectl create secret generic timber --from-literal=timber-api-key=TIMBER_API_KEY
```

```text
secret "timber" created
```

Then we launch the Timber Agent DaemonSet:

```bash
kubectl apply -f https://github.com/timberio/agent/blob/kubernetes/support/scripts/kubernetes/timber-agent-daemonset-rbac.yaml
```

```text
serviceaccount "timber-agent" created
clusterrole.rbac.authorization.k8s.io "timber-agent" created
clusterrolebinding.rbac.authorization.k8s.io "timber-agent" created
configmap "timber-agent" created
daemonset.extensions "timber-agent" created
```

_We are running the [RBAC](https://kubernetes.io/docs/admin/authorization/rbac/) version of the Timber Agent DaemonSet as this aligns with latest Kubernetes best practices._

And verify it is running as expected:

```bash
kubectl get pods
```

```text
NAME                 READY     STATUS    RESTARTS   AGE
logging-app-pod      1/1       Running   0          22m
timber-agent-54thq   2/2       Running   0          3m
```

Behold our logs!

![timber-ui-pod](https://cl.ly/0P3z3L003K0X/Image%202018-05-18%20at%2016.05.34.png)

_To avoid excess log shipping, the Timber Agent only ships new logs and not the contents of the file on boot._

## What if the logs are not written on stdout and stderr?

It is not uncommon to have Docker Containers running a piece of software which only logs to files on disk (e.g., nginx, although there are probably ways around it). In this case, we have to write the logs to a location the agent can read. Fortunately, we have two ways of doing this, but ideally, the container would log to stdout and stderr.

### 1. Write files to the host at a path being watched by logging agent

We can accomplish this configuring our `logging-app` to write to a location that our Pod has mapped to `/var/log/containers`. In this way, the file resides at a path currently being watched by the logging-agent. The Pod container path to host map is a feature of Docker called [volumes](https://docs.docker.com/storage/volumes/).

First, let's tear down our existing logging-app:

```bash
kubectl delete -f https://raw.githubusercontent.com/timberio/examples/master/kubernetes/reading-log-files-from-containers/logging-app-pod.yaml
```

```text
pod "logging-app-pod" deleted
```

```bash
kubectl get pods
```

Only the logging-agent pod should be running:

```text
NAME                 READY     STATUS    RESTARTS   AGE
timber-agent-54thq   2/2       Running   0          9m
```

And let's launch our `logging-app` that writes it logs to disk inside its container:

```bash
kubectl apply -f https://raw.githubusercontent.com/timberio/examples/master/kubernetes/reading-log-files-from-containers/logging-app-pod-host-log.yaml
```

```text
pod "logging-app-pod-host-log" created
```

Once up and running, we can verify the logs are being written to our host:

```bash
minikube ssh
cd /var/log/containers
sudo tail -n7 logging-app.log
```

```text
2018-05-18 20:09:45.498647066 +0000 UTC m=+145.001202735
2018-05-18 20:09:46.498552442 +0000 UTC m=+146.001108107
2018-05-18 20:09:47.500480854 +0000 UTC m=+147.003036530
2018-05-18 20:09:48.498736217 +0000 UTC m=+148.001291871
2018-05-18 20:09:49.49846095 +0000 UTC m=+149.001016602
2018-05-18 20:09:50.49995019 +0000 UTC m=+150.002505851
2018-05-18 20:09:51.498797206 +0000 UTC m=+151.001352863
```

And to timber.io:

![timber-ui-pod-host-log](https://cl.ly/243g0A0p2k3w/Image%202018-05-18%20at%2016.10.14.png)

Note that the log lines are no longer JSON. The different format comes from writing the logs directly to the host and bypassing the Docker JSON logging driver. Writing the logs outside of Docker also mean we cannot view them with kubectl:

```bash
kubectl logs logging-app-pod-host-log # Should return nothing
```

### 2. Streaming log file to stdout and stderr of a sidecar container

Our second option is to collect and ship our `logging-app` logs using a sidecar container. A sidecar container is a secondary container that performs a supporting function for the primary container. In Kubernetes, sidecar containers are run within the same Pod.

To implement this pattern, we will first create a volume for our Pod to be shared by all containers in the Pod. Then we will configure our `logging-app` to write to a file in that volume and configure our sidecar container to read and write that file to its stdout. Since the sidecar is itself a container, its stdout will be written to a file under `/var/log/containers` and subsequently shipped via our agent to [timber.io](https://timber.io/).

As before, we will delete our existing `logging-app` Pod:

```bash
kubectl delete -f https://raw.githubusercontent.com/timberio/examples/master/kubernetes/reading-log-files-from-containers/logging-app-pod-host-log.yaml
```

```bash
kubectl get pods
```

```text
NAME                 READY     STATUS    RESTARTS   AGE
timber-agent-54thq   2/2       Running   0          20m
```

And start our Pod configured with a streaming sidecar container:

```bash
kubectl apply -f https://raw.githubusercontent.com/timberio/examples/master/kubernetes/reading-log-files-from-containers/logging-app-pod-sidecar-log-streamer.yaml
```

```text
pod "logging-app-pod-sidecar-log-streamer" created
```

```bash
kubectl get pods
```

```text
NAME                                   READY     STATUS    RESTARTS   AGE
logging-app-pod-sidecar-log-streamer   2/2       Running   0          26s
timber-agent-54thq                     2/2       Running   0          21m
```

Our Kubernetes Cluster now looks something like this:

![kubernetes-logging-wit-streaming-sidecar](https://d33wubrfki0l68.cloudfront.net/c51467e219320fdd46ab1acb40867b79a58d37af/b5414/images/docs/user-guide/logging/logging-with-streaming-sidecar.png)

In this case, we should see our logs in three places.

With kubectl:

```bash
kubectl logs logging-app-pod-sidecar-log-streamer streamer
```

```text
Starting timer...
2018-05-18 20:18:36.429165162 +0000 UTC m=+1.002015650
2018-05-18 20:18:37.43020559 +0000 UTC m=+2.003056082
2018-05-18 20:18:38.429455731 +0000 UTC m=+3.002306228
2018-05-18 20:18:39.429220164 +0000 UTC m=+4.002070664
2018-05-18 20:18:40.430212368 +0000 UTC m=+5.003062864
2018-05-18 20:18:41.429767779 +0000 UTC m=+6.002618277
```

On the host (minikube):

```bash
minikube ssh
cd /var/log/containers
ls | grep _streamer
```

```text
logging-app-pod-sidecar-log-streamer_default_streamer-CONTAINER_ID.log
```

```bash
sudo head -n7 logging-app-pod-sidecar-log-streamer_default_streamer-CONTAINER_ID.log
```

And the Timber UI:
![timber-ui-pod-sidecar-log-streamer](https://cl.ly/3p0S0D3B2r3e/Image%202018-05-18%20at%2016.24.46.png)

There are a few things to notice here. First, we are looking for logs from the sidecar container as our Pod is now running two containers. This changes out kubectl command as well as our log file path. We can see the
 `POD-NAME_NAMESPACE_CONTAINER-NAME` filename format. Second, the log lines are JSON again, as stdout of the sidecar container is captured and written by Docker and its JSON logging driver. Last, logs from our `logging-app` are hidden as they are not being written to the containers stdout.

```bash
kubectl logs logging-app-pod-sidecar-log-streamer # Should return nothing
```

There are two concerns with this approach: resource usage and log tagging. While the resource consumption of our sidecar should be small, it does increase the overall resource usage of the Pod or Pods using this approach. Log tagging or log tracking is another issue, as the logs from the `logging-app` container will appear to be coming from the sidecar container. There are ways around this, such as having a unique sidecar container per file, but this only exacerbates the resource consumption issue.

## What if I can't run the DaemonSet?

Not being able to run the logging agent as a DaemonSet is an unfortunate circumstance and hopefully an extremely uncommon one. No matter, we can still ship our logs. In this case, we also have two solutions.

### 1. Run the logging agent as sidecar container

If the logging agent cannot be run as a DaemonSet, it can be run as a sidecar container. This approach is similar to the streaming sidecar container, but we can configure the agent to tag the logs correctly. However, resource consumption is even more of a concern as every application will be running a dedicated logging agent.

You should know the flow by now, so first let's delete our existing `logging-app` pod:

```bash
kubectl delete -f https://raw.githubusercontent.com/timberio/examples/master/kubernetes/reading-log-files-from-containers/logging-app-pod-sidecar-log-streamer.yaml
```

```text
pod "logging-app-pod-sidecar-log-streamer" deleted
```

And also let's tear down our Timber Agent DaemonSet and Secret:

```bash
kubectl delete -f https://github.com/timberio/agent/blob/kubernetes/support/scripts/kubernetes/timber-agent-daemonset-rbac.yaml
```

```text
serviceaccount "timber-agent" deleted
clusterrole.rbac.authorization.k8s.io "timber-agent" deleted
clusterrolebinding.rbac.authorization.k8s.io "timber-agent" deleted
configmap "timber-agent" deleted
daemonset.extensions "timber-agent" deleted
```

```bash
kubectl delete secret timber
```

```text
secret "timber" deleted
```

Then let's launch our logging-app Pod with our sidecar Timber Agent:

_The steps here required are slightly different as we need to edit the manifest to include our Timber API Key._

```bash
curl -s https://raw.githubusercontent.com/timberio/examples/master/kubernetes/reading-log-files-from-containers/logging-app-pod-sidecar-log-agent.yaml -O logging-app-pod-sidecar-log-agent.yaml > /dev/null
sed -i -e 's/TIMBER_API_KEY/your_api_key/' logging-app-pod-sidecar-log-agent.yaml
kubectl apply -f logging-app-pod-sidecar-log-agent.yaml
```

```text
configmap "logging-app-pod-sidecar-log-agent-config-map" created
pod "logging-app-pod-sidecar-log-agent" created
```

```bash
kubectl get pods
```

```text
NAME                                READY     STATUS    RESTARTS   AGE
logging-app-pod-sidecar-log-agent   2/2       Running   0          5s
```

Our setup now looks something like this:

![kubernetes-logging-with-sidecar-agent](https://d33wubrfki0l68.cloudfront.net/d55c404912a21223392e7d1a5a1741bda283f3df/c0397/images/docs/user-guide/logging/logging-with-sidecar-agent.png)

We can verify the logs are where we expect them to be, which is in our centralized logging system.

![timber-ui-app-sidecar-log-agent](https://cl.ly/3Q1R2S3l1T2O/Image%202018-05-18%20at%2017.00.13.png)

If we look on our Kubernetes cluster, we should be able to view the agent logs but not the `logging-app` logs.

_The agent logs to stdout by default._

```bash
kubectl logs logging-app-pod-sidecar-log-agent timber-agent
```

```text
time="2018-05-18T20:59:17Z" level=info msg="Timber agent starting"
time="2018-05-18T20:59:17Z" level=info msg="Opened configuration file at /timber/config.toml"
time="2018-05-18T20:59:17Z" level=info msg="Log collection endpoint: https://logs.timber.io/frames"
time="2018-05-18T20:59:17Z" level=info msg="Using filesystem polling: %!s(bool=false)"
time="2018-05-18T20:59:17Z" level=info msg="Maximum time between sends: 3 seconds"
time="2018-05-18T20:59:17Z" level=info msg="File count: 1"
time="2018-05-18T20:59:17Z" level=info msg="File 1: /var/log/logging-app/app.log (api key: ...5c41)"
...
```

```bash
minikube ssh
cd /var/log/containers/
ls | grep agent
```

```text
logging-app-pod-sidecar-log-agent_default_timber-agent-CONTAINER-ID.log
```

```bash
sudo head -n7 logging-app-pod-sidecar-log-agent_default_timber-agent-CONTAINER-ID.log
```

```text
{"log":"time=\"2018-05-18T20:59:17Z\" level=info msg=\"Timber agent starting\"\n","stream":"stderr","time":"2018-05-18T20:59:17.434971968Z"}
{"log":"time=\"2018-05-18T20:59:17Z\" level=info msg=\"Opened configuration file at /timber/config.toml\"\n","stream":"stderr","time":"2018-05-18T20:59:17.435274088Z"}
{"log":"time=\"2018-05-18T20:59:17Z\" level=info msg=\"Log collection endpoint: https://logs.timber.io/frames\"\n","stream":"stderr","time":"2018-05-18T20:59:17.435284199Z"}
{"log":"time=\"2018-05-18T20:59:17Z\" level=info msg=\"Using filesystem polling: %!s(bool=false)\"\n","stream":"stderr","time":"2018-05-18T20:59:17.435287049Z"}
{"log":"time=\"2018-05-18T20:59:17Z\" level=info msg=\"Maximum time between sends: 3 seconds\"\n","stream":"stderr","time":"2018-05-18T20:59:17.435289412Z"}
{"log":"time=\"2018-05-18T20:59:17Z\" level=info msg=\"File count: 1\"\n","stream":"stderr","time":"2018-05-18T20:59:17.435291694Z"}
{"log":"time=\"2018-05-18T20:59:17Z\" level=info msg=\"File 1: /var/log/logging-app/app.log (api key: ...5c41)\"\n","stream":"stderr","time":"2018-05-18T20:59:17.435294015Z"}
```

```bash
kubectl logs logging-app-pod-sidecar-log-agent logging-app # Should return nothing
```

### 2. Configure applications to write logs directly to logging backend

![kubernetes-logging-from-application](https://d33wubrfki0l68.cloudfront.net/0b4444914e56a3049a54c16b44f1a6619c0b198e/260e4/images/docs/user-guide/logging/logging-from-application.png)

All of our previous examples have run the logging agent in a container outside separate from our application.  An entirely different approach is to ship the logs directly to the logging backend from the application. This workflow bypasses all of our host system logic but requires application changes, which may or may not be an option. Additionally, it introduces its own set of challenges, mainly around reliability.

- If our logging backend is down, what happens to the logs?
- If our application crashes, what happens to the logs?

It is not always trivial to answer these questions, and as such, this is our least recommended approach. Performance and resource utilization are also present, but this implementation could end up costing less than our logging agent sidecar approach.

## Let's Wrap Up

Now that we have discussed the basics of logging and application log collection on Kubernetes, you should have not only learned something new but also be able to implement whichever strategy fills your own application logging needs.

There will also be a follow-up post addressing log augmentation on Kubernetes, or how to include contextual data to get more value from your logs.

Go forth and build... and log!

Until next time,

mark@timber.io
