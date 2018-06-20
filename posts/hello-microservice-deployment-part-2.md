TODO: this is not close to finished.

## Intro to CI/CD

Continuous Integration and Continuous Deployment are the next step to go through if you want a production grade micro-service application. Let's revisit Webflix to make this point clear. Webflix is running a whole lot of services in k8s. Each of these services is associated with some code stored in a repository somewhere. Let's say Webflix wisely chooses to use Git to store their code and they follow a feature branching strategy. Branching stratergies are a bit out side the scope of this article but basically what this means is that if a developer wants to make a new feature then they create that feature on a new git branch. Once the developer is confident that their feature is complete then they request that their feature branch gets merged into the master branch.  And once the code is merged to the master branch it should mean that it is ready to be deployed into production.

The process of deploying code to production is not so straightforward - first we should make sure all the unit tests pass and have good coverage. Then since we are working with microservices there is probably a docker image to build and push. Once that is done then it would be good to make sure that the docker image actually works by doing some tests against a live container (maybe a group of live containers). It might also be necessary to measure the performance of the new image by running some tests with a tool like [locust](https://locust.io/). The deployment process can get very complex if there are multiple developers working on multiple services at the same time since we would need to keep track of version compatibility for the various services.

CI/CD is all about automating this sort of thing.

There are loads of CI/CD tools around and they have their own ways of configuring their pipelines (a pipeline is a series of steps code needs to go through when it is pushed). There are whole books dedicated to designing deployment pipelines but in general you'll want to do something like this:

1. unit test the code
2. build the container
3. set up a test environment where the new container can run within a realistic context
4. run some integration tests
5. maybe run a few more tests eg: saturation tests with locust or similar
6. deploy to the production system
7. notify team of success/failure

If, for example, one of the test steps fails, then the code will not get deployed to production. The pipeline will skip to the end of the process and notify the team that the deployment was a failure.

You can also set up pipelines for merge/pull requests, eg if a developer requests a merge then execute the above pipeline but LEAVE OUT STEP 6 (deploying to production).


## Practical: Configuring a CI/CD pipeline with Drone.io

Drone.io is a docker based CI/CD platform. Each step in your build pipeline is specified as a bunch of commands to be executed on a docker image.

We're going start off by installing drone on a VM on Google cloud.
