---
layout: post
title: "Running tasks with cronjob in Azure Kubernetes Service."
slug: kubernetes-cronjob-tutorial
description: How to deploy cron job in kubernetes, configure kubernetes cluster, scale cluster to zero with autoscaler, run cron job manually as one time job, automate deployment with makefile.
keywords: kubernetes cronjob autoscaler makefile docker slack azure-container-registry azure-kubernetes-service
---

This tutorial provides a guide on how to deploy cron jobs to Azure Kubernetes Service (AKS), automate the 
deployment with Makefile, and configure a cluster with multiple node pools.

You will learn how to: 

* Prepare a Python app for deployment to AKS and build a docker image.
* Create an Azure container registry and push images to the registry.
* Create and configure a Kubernetes cluster, scale it down to zero with autoscaler.
* Schedule and deploy jobs to Kubernetes cluster.
* Run cron job manually as a one time job.   
* Automate the deployment process with Makefile.

[1]: https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial "kubernetes-cronjob-tutorial"

<br/>
<div class="blog-card">
<h7 class="m-1">source code repository: <a href="https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial">viktorsapozhok/kubernetes-cronjob-tutorial</a></h7><br/><br/>
<a class="github-button" href="https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star viktorsapozhok/kubernetes-cronjob-tutorial on GitHub">Star</a>
<a class="github-button" href="https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/fork" data-icon="octicon-repo-forked" data-size="large" data-show-count="true" aria-label="Fork viktorsapozhok/kubernetes-cronjob-tutorial on GitHub">Fork</a>
<a class="github-button" href="https://github.com/viktorsapozhok" data-size="large" data-show-count="true" aria-label="Follow @viktorsapozhok on GitHub">Follow @viktorsapozhok</a>
</div>

## 1. Clone and install the application

The application used in this tutorial to demonstrate the deployment process is 
a simple Python app printing messages to stdout and a slack channel.

Clone the application to your development environment.

```shell
$ git clone https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial.git
```

Then go to the project directory and install the application.

```shell
$ cd kubernetes-cronjob-tutorial
$ pip install .
```

When its installed, you can verify the installation calling `myapp` from the command line.

```shell
$ myapp --help
Usage: myapp [OPTIONS]

  Demo app printing current time and job name.

Options:
  --job TEXT  Job name
  --slack     Send message to slack
  --help      Show this message and exit.
```

You can run it without sending messages to the slack channel, only printing to console.
In this case, don't pass the `slack` flag.

```shell
$ myapp --job JOB-1 
14:00:26: JOB-1 started
```

To integrate it with slack, you need to configure an incoming webhook for your channel. Read [here][2] 
how to do this. Add a Webhook URL you will get to an environment variable `SLACK_TEST_URL`.

[2]: https://api.slack.com/messaging/webhooks# "Sending messages using incoming Webhooks"

Verify that it works by passing `slack` flag to `myapp` command.

```bash
$ myapp --job JOB-1 --slack 
14:00:34: JOB-1 started
```

If everything is correct then you should receive the same message in your slack channel.

<img src="https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/docs/source/images/slack.png?raw=true" width="700">

Application is installed to your development environment, and we can start preparing it to the deployment.

## 2. Install docker and docker-compose

Skip this section and go to the section 3, if you already have docker installed.

Update software repositories to make sure you’ve got access to the latest revisions.
Then install docker engine.

```bash
$ sudo apt-get update
$ sudo apt install docker.io
```

Now we set up docker service to be running at startup.

```bash
$ sudo systemctl start docker
$ sudo systemctl enable docker
```

You can test the installation verifying docker version.

```bash
$ sudo docker --version
```

By default, the docker daemon always run as `root` user and other users can access it
only with `sudo` privileges. To be able to run docker as non-root user, create a new group
called `docker` and add your user to it.

```bash
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
```

Log out and log back in so that group membership is re-evaluated.
You can also issue the following command to activate changes.

```bash
$ newgrp docker
```

Verify if docker can be run as non-root.

```bash
$ docker run hello-world
```

Reboot if you got error.

Now, after docker engine is installed, we install docker compose, a tool for
defining and running multi-container docker applications. 

Follow the [official installation guide][3] and test the installation verifying compose version.

[3]: https://docs.docker.com/compose/install/ "Install Docker Compose"

```bash
$ docker-compose --version
```

## 3. Create docker image for your Python app

Now that we have installed our application and docker software, we can build a docker image 
for the application and run it inside the container. To do that, we create a Dockerfile, a text file
that contains a list of instructions, which describes how a docker image is built.

We store [Dockerfile][4] in the project root directory and add the following instructions to it:

[4]: https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/Dockerfile "Dockerfile"

```dockerfile
FROM python:3.9-slim

ARG SLACK_TEST_URL

COPY requirements.txt .
RUN pip install --no-cache-dir -r ./requirements.txt \
    && rm -f requirements.txt

RUN groupadd --gid 1000 user \
    && useradd --uid 1000 --gid 1000 --create-home --shell /bin/bash user

COPY . /home/user/app
WORKDIR /home/user/app

RUN pip install --no-cache-dir . \
    && chown -R "1000:1000" /home/user 

USER user
CMD tail -f /dev/null
```

Here we create a non-root user, as it's not recommended running the container as a root due 
to some security issues.

We intentionally keep the list of requirements outside `setup.py` to be able to install it
in a separate layer, before installing the application. The point is that docker is caching 
every layer (each `RUN` instruction will create a layer) and checks the previous builds
to use the untouched layers as cache.

In case we don't keep the requirements in a separate file, but instead list them directly in `setup.py`,
docker will install it again every time we change something in the application code
and rebuild the image. Therefore, if we want to reduce the building time, we use two
layers for installation, one for requirements, one for application. 

We also modify [setup.py][5] to dynamically read the list of requirements from file:

[5]: https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/setup.py "setup.py"

```python
from setuptools import setup


def get_requirements():
    r = []
    with open("requirements.txt") as fp:
        for line in fp.read().split("\n"):
            if not line.startswith("#"):
                r += [line.strip()]
    return r


setup(
    name="app",
    packages=["app"],
    include_package_data=True,
    zip_safe=False,
    install_requires=get_requirements(),
    entry_points={
        "console_scripts": [
            "myapp=app.cli:main",
        ]
    },
)
```

We can now build the docker image using `docker build` command.

```bash
$ docker build --tag app:v0 .
```

When building process is over, you can find your image in local image store.

```bash
$ docker images
REPOSITORY   TAG        IMAGE ID       CREATED             SIZE
app          v0         73ac1e524c0e   35 seconds ago      123MB
python       3.9-slim   609da079b03a   About an hour ago   115MB
```

All right, now let's run our application inside the container.

```bash
$ docker run app:v0 myapp --job JOB-1 
20:56:33: JOB-1 started
```

The container has been started and application was successfully running inside the container.
Container stops after the application has finished. Docker containers are 
not automatically removed when you stop them. To remove one or more containers, use 
`docker container rm` command specifying container IDs you want to remove.

You can view the list of all containers using `docker container ls --all` command.

```bash
$ docker container ls --all
CONTAINER ID   IMAGE   COMMAND               CREATED             STATUS                       
c942c2424719   app     "myapp"               3 seconds ago       Exited (0) 2 seconds ago
0d311b2708e4   app     "myapp --job JOB-1"   8 seconds ago       Exited (0) 7 seconds ago
```

From the list above, you can see the `CONTAINER ID`. Pass it to `docker container rm` command to delete 
the containers.

```bash
# remove two containers
$ docker container rm c942c2424719 0d311b2708e4
c942c2424719
0d311b2708e4

# remove one container
$ docker container rm c942c2424719
c942c2424719
```

To remove all stopped containers, use `docker container prune` command.

Note, that you can start the container with `--rm` flag meaning that the container
will be automatically removed after stop.

```bash
$ docker run --rm app:v0 myapp --job JOB-1 
```

To run application with `slack` option, you need to pass Webhook URL via environment variable in docker.
To do this, use `--env` option with `docker run` command.

```bash
$ docker run --rm --env SLACK_TEST_URL=$SLACK_TEST_URL app:v0 myapp --job docker-job --slack  
13:19:16: docker-job started
```

If everything works, you will receive message in slack channel.

<img src="https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/docs/source/images/slack_2.png?raw=true" width="700">

## 4. Push docker images to the registry

Azure Container Registry (ACR) is a private registry for container images, it allows
you to build, store, and manage container images. In this tutorial, we deploy an ACR instance
and push a docker image to it. This requires that you have Azure CLI installed. Follow the
[official guide][6] if you need to install it.

[6]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli "Install Azure CLI"

To create an ACR instance, we need to have a resource group, a logical container that
includes all the related resources to your solution. Create a new group with `az group create` command or
use the existing group if you already have one.

```bash
$ az group create --name myResourceGroup --location westeurope
```

Now we can create an Azure Container Registry with `az acr create` command. We use the `Basic` SKU, which
includes 10 GiB storage. Service tier can be changed at any time, you can use `az acr update` command to 
switch between service tiers. 

```bash
$ az acr create \
  --resource-group myResourceGroup \
  --name vanillacontainerregistry \
  --sku Basic \
  --location westeurope
```

To push our docker image to the registry, it has to be tagged with the server address `vanillacontainerregistry.azurecr.io`.
You can find the address on Azure Portal. Use lowercase letters in registry name to avoid some warning messages.

```bash
$ docker tag app:v0 vanillacontainerregistry.azurecr.io/app:v0

$ docker images
REPOSITORY                                TAG   IMAGE ID       CREATED        SIZE
vanillacontainerregistry.azurecr.io/app   v0    46894e5479b8   25 hours ago   123MB
app                                       v0    46894e5479b8   25 hours ago   123MB
```

Now, we can log in to the registry and push our container image.

```bash
$ az acr login --name vanillacontainerregistry
Login Succeeded

$ docker push vanillacontainerregistry.azurecr.io/app:v0
The push refers to repository [vanillacontainerregistry.azurecr.io/app]
d4f6821c5d53: Pushed 
67a0bfcd1c19: Pushed 
1493f3cb6eb5: Pushed 
d713ef9ef160: Pushed 
b53f0c01f700: Pushed 
297b05241274: Pushed 
677735e8b7e0: Pushed 
0315c2e53dfa: Pushed 
98a85d041f35: Pushed 
02c055ef67f5: Pushed 
v0: digest: sha256:2b11fb037d0c3606dd32daeb95e355655594159de6a8ba11aa0046cad0e93838 size: 2413
```

That's it. We built a docker image for our Python application, created an Azure Container Registry and
pushed the image to the repository. You can view the repository with `az acr repository` command or via portal.

```bash
$ az acr repository show-tags --name vanillacontainerregistry --repository app --output table
Result
--------
v0
```

All good, we move on to the next step.

## 5. Create and configure Kubernetes cluster

In Azure Kubernetes Service (AKS), nodes having the same configuration are combined into node pools.
Each node pool contains underlying VMs that run your apps. AKS offers a feature called the cluster autoscaler
to automatically scale node pools. Autoscaler saves costs by starting infrastructure before demand increases
and releasing resources when demand decreases. In case we are running scheduled jobs without running any permanent
workloads, we need to scale the whole cluster down to zero when there are no jobs running. However, at least
one node must always be available in the cluster as it's used to run the system pods. Therefore, our strategy
for running jobs in the cluster will be to keep only one node running idle when there are no jobs, add nodes before 
job startup and shut them down after job has finished.

Let's create an AKS cluster with `az aks create` command.

```bash
$ az aks create \
  --resource-group myResourceGroup \
  --name vanilla-aks-test \
  --node-count 1 \
  --attach-acr vanillacontainerregistry \
  --location westeurope
```

Specifying `node-count` option as 1, we created a cluster with default node pool that contains only one node.
By default, it contains 3 nodes. The number of nodes can be changed after cluster creating with `az aks scale`
command. We also granted the cluster identity the right to pull images from our container registry 
using`attach-acr` option.

To connect to the cluster from local machine we use Kubernetes client `kubectl`, it's included in Azure CLI and 
should be already installed. To configure `kubectl`, use the `az aks get-credentials` command.

```bash
$ az aks get-credentials --resource-group myResourceGroup --name vanilla-aks-test
Merged "vanilla-aks-test" as current context in /home/user/.kube/config
```

Now we can connect to cluster and display nodes information.

```bash
$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-72918754-vmss000000   Ready    agent   20m   v1.19.9

$ kubectl get nodes -L agentpool -L node.kubernetes.io/instance-type 
NAME                                STATUS   ROLES   AGE   VERSION   AGENTPOOL   INSTANCE-TYPE
aks-nodepool1-72918754-vmss000000   Ready    agent   37m   v1.19.9   nodepool1   Standard_DS2_v2
```

Cluster has one node with VM size Standard_DS2_v2 (2 vCPUs, 7 GB RAM, 14 GB storage). This will
generate about 100 usd/month costs. 

We can check what is running on the node with `kubectl get pods` command. So far, it has only
system processes (pods) running.

```bash
$ kubectl get pods
No resources found in default namespace.

$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   azure-ip-masq-agent-55qmp             1/1     Running   0          26m
kube-system   coredns-76c97c8599-8qxlz              1/1     Running   0          27m
kube-system   coredns-76c97c8599-sv2g4              1/1     Running   0          26m
kube-system   coredns-autoscaler-599949fd86-2dq4b   1/1     Running   0          27m
kube-system   kube-proxy-xqgjl                      1/1     Running   0          26m
kube-system   metrics-server-77c8679d7d-2x26x       1/1     Running   0          27m
kube-system   tunnelfront-6dcdcd4f8d-pcgcb          1/1     Running   0          27m
```

We can update node pool (we still have a single pool called `nodepool1`) to activate 
autoscaler and enable it to increase the number of nodes up to 5 if needed.

```bash
$ az aks nodepool update \
  --resource-group myResourceGroup \
  --cluster-name vanilla-aks-test \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5
```

This configuration will be enough for running a variety of lightweight jobs. Let's try to 
deploy our application to the cluster.

## 6. Deploy cron jobs to Azure Kubernetes Service

To deploy our application, we will use the `kubectl apply` command. This command parses 
the manifest file and creates the defined Kubernetes objects. We start from creating such
a manifest file for our application cron job.

In root directory, we create a file called `aks-manifest.yml` and specify it as follows:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: app-job-1
  namespace: app
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: app
            image: vanillacontainerregistry.azurecr.io/app:v0
            env:
              - name: SLACK_TEST_URL
                value: my-webhook-url
            command: ["/bin/sh", "-c"]
            args: ["myapp --job AKS-JOB-1 --slack"]
            resources:
              requests:
                cpu: "0.5"
                memory: 500Mi
              limits:
                cpu: "1"
                memory: 1000Mi
          restartPolicy: Never
      backoffLimit: 2
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 2
```

Here we specified the job name `app-job-1` and the namespace `app`. Namespaces in Kubernetes provide a scopes for names
and can be used to divide cluster resources between users or projects. Not necessary to use namespaces when you don't feel that
you need it, but in case you want to logically separate one bunch of jobs from the other, it might be useful. Note, that namespace
`app` still doesn't exist, and we need to create it before applying the manifest.

```bash
$ kubectl create namespace app
namespace/app created
```

Next, we specify the crontab expression used as a schedule for our job. Expression `*/5 * * * *` means that job is 
supposed to run on every 5th minute. We pass the name of the docker image to be pulled from 
container registry attached to cluster, and specify the container name as `app`. Remove environment 
variable spec given by `env` key from manifest if you didn't integrate application with slack channel. 

Now we can apply the deployment to cluster.

```bash
$ kubectl --namespace app apply -f ./aks-manifest.yml  
cronjob.batch/app-job-1 created
```

You can view some short information about the job with `kubectl get` command.

```bash
$ kubectl get cronjob --namespace app
NAME        SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
app-job-1   */5 * * * *   False     0        18s             4m49s
```

Application is running in Kubernetes cluster and sending messages to slack.

<img src="https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/docs/source/images/slack_3.png?raw=true" width="700">

To retrieve cron job logs from Kubernetes, you can use `kubectl logs` command

```bash
$ kubectl --namespace app get pods
NAME                        READY   STATUS      RESTARTS   AGE
app-job-1-1622391900-4crp8   0/1     Completed   0          13s

$ kubectl logs --namespace app app-job-1-1622391900-4crp8
13:20:10: JOB-1 started
```

Note, that we hardcoded constants such as job name, schedule, etc. in the manifest file. 
This configuration works in case of our demo job, but in real life project you get a setup with 
many jobs and need therefore to create a manifest for every task. This is the option
we want to avoid, so let's delete our job from cluster and redeploy it in more general way.

```bash
$ kubectl --namespace app delete cronjob app-job-1
cronjob.batch "app-job-1" deleted
```

## 7. Automate deployment to AKS with Makefile

Suppose that we have 3 jobs with different schedules, and we want 
to deploy them with a single command. To do this, we can replace the constants in the manifest
by placeholders and substitute them in a loop with corresponding values based on our deployment setup file.

First, let's create a file called [deployment.yml][7] and place it in the root directory. In this file,
we will store all the parameters used for deployment process.

[7]: https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/deployment.yml "deployment.yml"

```yaml
rg: 
  name: myResourceGroup
acr:
  name: vanillacontainerregistry
  url: vanillacontainerregistry.azurecr.io
aks:
  cluster_name: vanilla-aks-test
  namespace: app
jobs:
  job1:
    schedule: "*/5 * * * *"
    command: "myapp --job JOB-1 --slack"
  job2:
    schedule: "*/10 * * * *"
    command: "myapp --job JOB-2 --slack"
  job3:
    schedule: "*/20 * * * *"
    command: "myapp --job JOB-3 --slack"
```

As long as we are going to aggregate multiple shell commands into single command, we 
need a [Makefile][8]. Let's create one in the project's root directory. To read the contents
of a YAML file from Makefile, we can use `yq` utility. Documentation on it can be found [here][9].

[8]: https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/Makefile "Makefile"
[9]: https://mikefarah.gitbook.io/yq/ "Lightweight and portable command-line YAML processor"

On linux it can be installed via snap. We need `v4` version.

```bash
$ snap install yq
```

In [Makefile][8], we create a function `get_param` reading data from YAML file. It can be organized as follows:

```makefile
get_param = yq e .$(1) deployment.yml

AKS_NAME := $(shell $(call get_param,aks.cluster_name))

print-cluster-name:
	echo aks.cluster_name: $(AKS_NAME)
```

Now, if we run `make print-cluster-name` from project's root the name of AKS cluster from YAML file
will be placed into `AKS_NAME` variable and displayed in stdout.

```bash
$ make print-cluster-name
aks.cluster_name: vanilla-aks-test
```

We want to run a deployment script for each of our 3 jobs in a loop. This can be done
with following Makefile setup:

```makefile
JOB ?=

get_param = yq e .$(1) deployment.yml

JOBS := $(shell yq eval '.jobs | keys | join(" ")' deployment.yml)

deploy:
	$(eval SCHEDULE := $(shell $(call get_param,jobs.$(JOB).schedule)))
	$(eval COMMAND := $(shell $(call get_param,jobs.$(JOB).command)))

	echo deploying $(JOB)
	echo schedule: "$(SCHEDULE)"
	echo command: "$(COMMAND)"

deploy-all:
	for job in $(JOBS); do \
		$(MAKE) JOB=$$job deploy; \
		echo ""; \
	done
```

Now we can deploy any individual job defined in `deployment.yml`, or deploy all the jobs with 
a single command.

```bash
$ make deploy JOB=job1
deploying job1
schedule: */5 * * * *
command: myapp --job JOB-1 --slack

$ make deploy-all
deploying job1
schedule: */5 * * * *
command: myapp --job JOB-1 --slack

deploying job2
schedule: */10 * * * *
command: myapp --job JOB-2 --slack

deploying job3
schedule: */20 * * * *
command: myapp --job JOB-3 --slack
```

We are almost there. Now, all we need is to replace constants in manifest by placeholders
and dynamically substitute these placeholders by corresponding values in a loop.

Here is how [aks-manifest.yml][10] looks after replacement the constants:

[10]: https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/aks-manifest.yml "aks-manifest.yml"

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: $NAME
  namespace: $NAMESPACE
spec:
  schedule: "$SCHEDULE"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: $CONTAINER
            image: $IMAGE
            env:
              - name: SLACK_TEST_URL
                value: $SLACK_TEST_URL
            command: ["/bin/sh", "-c"]
            args: ["$COMMAND"]
            resources:
              requests:
                cpu: "0.5"
                memory: 500Mi
              limits:
                cpu: "1"
                memory: 1000Mi
          restartPolicy: Never
      backoffLimit: 2
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 2
```

Note, that you need to set environment variable `SLACK_TEST_URL` if you are going to run
demo jobs with `slack` option, otherwise just remove `env` instruction.

To dynamically set all other placeholders, we can use `envsubst` utility.

```makefile
JOB ?=
VERSION = 0.1

get_param = yq e .$(1) deployment.yml

aks.namespace := $(shell $(call get_param,aks.namespace))
acr.url := $(shell $(call get_param,acr.url))

docker.tag = app
docker.container = app
docker.image = $(acr.url)/$(docker.tag):v$(VERSION)

job.name = $(aks.namespace)-$(subst _,-,$(JOB))
job.schedule = "$(shell $(call get_param,jobs.$(JOB).schedule))"
job.command = $(shell $(call get_param,jobs.$(JOB).command))
job.manifest.template = ./aks-manifest.yml
job.manifest = ./concrete-aks-manifest.yml

create-manifest:
	touch $(job.manifest)

	NAME=$(job.name) \
	NAMESPACE=$(aks.namespace) \
	CONTAINER=$(docker.container) \
	IMAGE=$(docker.image) \
	SCHEDULE=$(job.schedule) \
	COMMAND="$(job.command)" \
	envsubst < $(job.manifest.template) > $(job.manifest)
```

Now, running `make create-manifest` from the root directory, you will create a new file called 
`concrete-aks-manifest.yml` having job configuration parameters instead of placeholders.

```bash
$ make create-manifest JOB=job2

$ cat concrete-aks-manifest.yml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: app-job2
  namespace: app
spec:
  schedule: "*/10 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: app
            image: vanillacontainerregistry/azurecr.io/app:v0.1
            env:
              - name: SLACK_TEST_URL
                value: https://hooks.slack.com/...
            command: ["/bin/sh", "-c"]
            args: ["myapp --job JOB-2 --slack"]
            resources:
              requests:
                cpu: "0.5"
                memory: 500Mi
              limits:
                cpu: "1"
                memory: 1000Mi
          restartPolicy: Never
      backoffLimit: 2
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 2
```

Now putting it all together, we get the following [Makefile][8] setup:

```makefile
JOB ?=
VERSION = 0.1

get_param = yq e .$(1) deployment.yml

aks.namespace := $(shell $(call get_param,aks.namespace))
acr.url := $(shell $(call get_param,acr.url))

docker.tag = app
docker.container = app
docker.image = $(acr.url)/$(docker.tag):v$(VERSION)

jobs := $(shell yq eval '.jobs | keys | join(" ")' deployment.yml)

job.name = $(aks.namespace)-$(subst _,-,$(JOB))
job.schedule = "$(shell $(call get_param,jobs.$(JOB).schedule))"
job.command = $(shell $(call get_param,jobs.$(JOB).command))
job.manifest.template = ./aks-manifest.yml
job.manifest = ./concrete-aks-manifest.yml

_create-manifest:
	touch $(job.manifest)

	NAME=$(job.name) \
	NAMESPACE=$(aks.namespace) \
	CONTAINER=$(docker.container) \
	IMAGE=$(docker.image) \
	SCHEDULE=$(job.schedule) \
	COMMAND="$(job.command)" \
	envsubst < $(job.manifest.template) > $(job.manifest)

# Delete cronjob, (`-` means to continue on error)
delete-job:
	-kubectl --namespace $(aks.namespace) delete cronjob $(job.name)

delete-all:
	for job in $(jobs); do \
		echo "removing $$job"; \
		$(MAKE) JOB=$$job delete-job; \
		echo ""; \
	done

deploy-job:
	make delete-job
	make _create-manifest
	kubectl apply -f $(job.manifest)
	rm $(job.manifest)

deploy-all:
	for job in $(jobs); do \
		echo "deploying $$job"; \
		$(MAKE) JOB=$$job deploy-job; \
		echo ""; \
	done
```

That's it. Let's deploy `job1`.

```bash
$ kubectl --namespace app get cronjobs
No resources found in app namespace.

$ make deploy-job JOB=job1
cronjob.batch "app-job1" deleted
cronjob.batch/app-job1 created
```

Job is now deployed and running in Kubernetes.

```bash
$ kubectl --namespace app get cronjobs
NAME       SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
app-job1   */5 * * * *   False     0        <none>          30s
```

Let's deploy all jobs together.

```bash
$ make deploy-all
deploying job1
Error from server (NotFound): cronjobs.batch "app-job1" not found
make[2]: [Makefile:41: delete-job] Error 1 (ignored)
cronjob.batch/app-job1 created

deploying job2
Error from server (NotFound): cronjobs.batch "app-job2" not found
make[2]: [Makefile:41: delete-job] Error 1 (ignored)
cronjob.batch/app-job2 created

deploying job3
Error from server (NotFound): cronjobs.batch "app-job3" not found
make[2]: [Makefile:41: delete-job] Error 1 (ignored)
cronjob.batch/app-job3 created
```

All jobs have been deployed.

```bash
$ kubectl --namespace app get cronjobs
NAME       SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
app-job1   */5 * * * *    False     0        <none>          38s
app-job2   */10 * * * *   False     0        <none>          37s
app-job3   */20 * * * *   False     0        <none>          31s
```

Here is what we receive in slack channel.

<img src="https://github.com/viktorsapozhok/kubernetes-cronjob-tutorial/blob/master/docs/source/images/slack_4.png?raw=true" width="700">

## 8. Run cron job manually as a one time job

Sometimes, we need to restart a failed job, or start it manually for testing purposes. 
To do this, we can use `kubectl create job` command as simply as following:

```bash
$ kubectl create job --namespace app --from=cronjob/app-job1 app-job1-run-now  
job.batch/app-job1-run-now created
```

We can also append the current time to job's name to display when exactly it was running, and
create a new target in Makefile.

```makefile
JOB ?=

get_param = yq e .$(1) deployment.yml

aks.namespace := $(shell $(call get_param,aks.namespace))
job.name = $(aks.namespace)-$(subst _,-,$(JOB))
time.now = $(shell date +"%Y.%m.%d.%H.%M")

run-job-now:
	kubectl create job \
	--namespace $(aks.namespace) \
	--from=cronjob/$(job.name) $(job.name)-$(time.now)
```

Now issuing `make run-job-now JOB=job1` from the root directory, you will start `job1`. 

## 9. Multiple node pools setup

Until now, we have used only one node pool, contained nodes with Standard_DS2_v2 size.
In case, you have a job required more resources, e.g. high memory utilization, NVIDIA GPUs etc, 
you need another node pool with nodes satisfied your requirements. The second node pool
will be more expensive than the default pool, and we need to configure the cluster in a way
that only jobs required more resources will be running in a second pool, whereas all other 
jobs continue running in the default pool.

Let's add a new job `job4` to our deployment config. 

```yaml
jobs:
  ...
  job4:
    schedule: "0 12 * * *"
    command: "myapp --job TURBO-JOB --slack"
```

Assume, it will be running once a day at 12:00 UTC on a machine with Standard_DS5_v2 size 
having 16 CPUs and 56 GB RAM. Running on such a machine for 730 hours a month will generate 
costs about 800 Usd/month, but if the job takes 1-2 hours to complete we can significantly 
reduce our costs using autoscaler, immediately releasing expensive resources after the 
job has completed.

Add a new pool to cluster with `az aks nodepool add` command, specifying pool's name as `turbo`.

```bash
$ az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name vanilla-aks-test \
  --name turbo \
  --node-count 1 \
  --min-count 0 \
  --max-count 1 \
  --enable-cluster-autoscaler \
  --max-pods 16 \
  --node-vm-size Standard_DS5_v2
```

Configuring pool in such a way, it will be scaled down to 0 after `job4` has completed.
For more final tweaking, you can configure autoscaler settings provided by `cluster-autoscaler-options`.

```bash
$ az aks update \
  --resource-group myResourceGroup \
  --name vanilla-aks-test \
  --cluster-autoscaler-profile scale-down-delay-after-add=3m scale-down-unneeded-time=3m
```

You can always delete a pool with `az aks nodepool delete` command.

```bash
$ az aks nodepool delete \
  --resource-group myResourceGroup \
  --cluster-name vanilla-aks-test \
  --name turbo
```

As a final touch, we add a mapping between jobs and pools specifying `nodeSelector` instruction
in manifest file.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
spec:
  jobTemplate:
    spec:
      template:
        spec:
          ...
          nodeSelector:
            agentpool: $AGENTPOOL
          ...
```

Define `agentpool` for every job in deployment config.

```yaml
jobs:
  job1:
    schedule: "*/5 * * * *"
    command: "myapp --job JOB-1 --slack"
    agentpool: nodepool1
  job2:
    schedule: "*/10 * * * *"
    command: "myapp --job JOB-2 --slack"
    agentpool: nodepool1
  job3:
    schedule: "*/20 * * * *"
    command: "myapp --job JOB-3 --slack"
    agentpool: nodepool1
  job4:
    schedule: "0 12 * * *"
    command: "myapp --job TURBO-JOB --slack"
    agentpool: turbo
```

Integrate `AGENTPOOL` variable to Makefile.

```makefile
JOB ?=
VERSION = 0.1

get_param = yq e .$(1) deployment.yml

aks.namespace := $(shell $(call get_param,aks.namespace))
acr.url := $(shell $(call get_param,acr.url))

docker.tag = app
docker.container = app
docker.image = $(acr.url)/$(docker.tag):v$(VERSION)

jobs := $(shell yq eval '.jobs | keys | join(" ")' deployment.yml)

job.name = $(aks.namespace)-$(subst _,-,$(JOB))
job.schedule = "$(shell $(call get_param,jobs.$(JOB).schedule))"
job.command = $(shell $(call get_param,jobs.$(JOB).command))
job.agentpool = $(shell $(call get_param,jobs.$(JOB).agentpool))
job.manifest.template = ./aks-manifest.yml
job.manifest = ./concrete-aks-manifest.yml

_create-manifest:
	touch $(job.manifest)

	NAME=$(job.name) \
	NAMESPACE=$(aks.namespace) \
	CONTAINER=$(docker.container) \
	IMAGE=$(docker.image) \
	SCHEDULE=$(job.schedule) \
	COMMAND="$(job.command)" \
	AGENTPOOL=$(job.agentpool) \
	envsubst < $(job.manifest.template) > $(job.manifest)
```

Let's deploy all four jobs and see what happens when `job4` is running.

```bash
$ make deploy-all

$ kubectl --namespace app get cronjobs
NAME       SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
app-job1   */5 * * * *    False     0        <none>          12s
app-job2   */10 * * * *   False     0        <none>          10s
app-job3   */20 * * * *   False     0        <none>          9s
app-job4   0 12 * * *     False     0        <none>          7s
```

Now we start `job4`.

```bash
$ make run-job-now JOB=job4

$ kubectl --namespace app get pods
NAME                              READY   STATUS      RESTARTS   AGE
app-job1-1622408700-svbgq         0/1     Completed   0          3m10s
app-job4-2021.05.30.23.07-gbrfs   0/1     Pending     0          15s
```

We see that `job1` has finished and `job4` is pending. Let's check what happens on the pod.

```bash
$ kubectl describe pod app-job4-2021.05.30.23.07-gbrfs --namespace app
Events:
  Type     Reason            Age                    From                Message
  ----     ------            ----                   ----                -------
  Normal   TriggeredScaleUp  6m9s                   cluster-autoscaler  pod triggered scale-up: [{aks-turbo-72918754-vmss 0->1 (max: 1)}]
  Warning  FailedScheduling  4m53s (x3 over 6m13s)  default-scheduler   0/1 nodes are available: 1 node(s) didnt match node selector.
  Warning  FailedScheduling  4m27s (x2 over 4m36s)  default-scheduler   0/2 nodes are available: 1 node(s) didnt match node selector, 1 node(s) had taint {node.kubernetes.io/not-ready: }, that the pod didnt tolerate.
  Normal   Scheduled         4m16s                  default-scheduler   Successfully assigned app/app-job4-2021.05.30.23.07-gbrfs to aks-turbo-72918754-vmss000001
  Normal   Pulling           4m15s                  kubelet             Pulling image "vanillacontainerregistry.azurecr.io/app:v0.1"
  Normal   Pulled            4m11s                  kubelet             Successfully pulled image "vanillacontainerregistry.azurecr.io/app:v0.1" in 4.337866521s
  Normal   Created           4m11s                  kubelet             Created container app
  Normal   Started           4m11s                  kubelet             Started container app
```

We see that there are no nodes which would tolerate node selector, and autoscaler triggered a scale-up event.
It started the node in `turbo` pool, assigned job to it, and started the container. Autoscaler deactivates
the node after job has finished and there are 0 active nodes in `turbo` pool.

If we look at pod events for `job1`, we see that job has been assigned to the node in our default pool `nodepool1`.
The node was already available (as it's a default pool and one node is always running) and autoscaler 
didn't trigger a scale-up event.

```bash
$ kubectl describe pod app-job1-1622408700-svbgq --namespace app 
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  39s   default-scheduler  Successfully assigned app/app-job1-1622408700-svbgq to aks-nodepool1-72918754-vmss000000
  Normal  Pulled     38s   kubelet            Container image "vanillacontainerregistry.azurecr.io/app:v0.1" already present on machine
  Normal  Created    38s   kubelet            Created container app
  Normal  Started    38s   kubelet            Started container app
```

That's it. We have successfully configured our kubernetes cluster for the case of multiple pools.
