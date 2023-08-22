+++
title = "Jenkins+Kubernetes"
description = ""
tags = [
    "docker",
    "kubernetes",
    "containers"
]
date = "2014-04-02"
categories = [
    "software",
    "devops"
]
draft = false
+++

# How to Setup Scalable Jenkins on Top of a Kubernetes Cluster (At Home).

> Before we start, I'll make some intro about the scenario and reasons for my choices, if you prefer, skip this part, the meat of the article starts at the __Docker__ section.
 
A few days ago I decided to start a new side project, a tool that will take care of a bunch of things I need; I pay for different ones, and yet they don't cover everything I need. Since I have another excuse, and, it's for myself, I've decided to go fully buzzword-compliant, something hard to achieve at work. 

One of the critical points is the building process of course, I could use Travis like I do some times but since the source code is private, and I have no intention to pay for Travis I've decided to spin up a Jenkins running on top of my beloved home devices, later it may change, but for now it'll be like that.

Just as a matter of fact, these are the devices I'm talking about.

![Devices](http://url/to/img.png)

Another thing that worth to mention, the version I'll describe here is the first version I got working, the simplest one, accessing the master Jenkins through `NodePort`. The final version was a bit longer road and would make this article far longer it has two services, one for the `jnlp` of type `ClusterIP` and another one, the `http` of type `LoadBalancer`.

The problem with the final version is that in my case _"I'm not sure if all this was really necessary"_ I had to use [Metallb](https://metallb.universe.tf/) to have my internal bare metal Load Balancer also because of the [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) configuration, I had to flash my Netgear and replace its original firmware by [dd-wrt](https://dd-wrt.com/) and also, a few other configurations here and there.

If was necessary or not, at least it was quite fun.

## Assumptions

I'll assume that if you're here reading about [Jenkins](https://jenkins.io/) and [Kubernetes](https://kubernetes.io/) is because you already have some knowledge about any of the container runtime environments supported by Kubernetes, since I know only about [Docker](https://www.docker.com/), I'll stick with it.

I'll assume as well that you already have your Kubernetes cluster up and running, with all the nodes ready to go, running a `kubectl get nodes -o wide` you should get enough information.

<script src="https://gist.github.com/allandequeiroz-snippets/731c7e4b54a1b06d9fb9ee0cecb2b9dd.js"></script> 

## Docker

To start with, let's take a look at the Dockerfile, the idea here is to create a basic image ready to go with the initial necessary plugins. By ready to go I mean the latest version, interesting to know that the `latest` Jenkins image at Docker Hub doesn't contain the `latest` Jenkins version, if you tried to setup Jenkins before, you know what it means, many plugins will refuse work with an outdated Jenkins version.

Lines 3 to 5 (see below) takes care of the platform update so the plugins will be happy when Jenkins starts. Luckily `updates.jenkins-ci.org` keeps a link to the `latest` bundle, so we need to point there and problem solved.

<script src="https://gist.github.com/allandequeiroz-snippets/0637e146d7b41517b83a6967aa897c06.js"></script>

To build this image you need to execute the following commands, please replace `allandequeiroz/jenkins` by `<your user at docker hub>/<the name you prefer>`.

<script src="https://gist.github.com/allandequeiroz-snippets/237860290b7e6e4c07358271f01c5456.js"></script>

The commands above will build and push the `latest` version. Alternatively, you can set a specific version as well.

<script src="https://gist.github.com/allandequeiroz-snippets/a12ecd09befca96e0672930938af72cb.js"></script>

## Kubernetes

As mentioned before, for the sake of simplicity, I'll place here my initial working version, without load balancing capabilities, for this version you'll access your Jenkins instance at port `30000` of a fixed host, defined by a tag.

> If you prefer, you can see an alternative version [here](https://gist.github.com/allandequeiroz-snippets/ec4d9457d03eb619b4b7e425839aeba3). If what you need is more likely the alternative version, please read the first part of the article, before "Assumptions", because you may need some more steps if you're deploying it at home.

<script src="https://gist.github.com/allandequeiroz-snippets/9287b002e32b2a938610740a2c5f00ec.js"></script>

A bit of explanation about this file:

__Lines: 16-17__ - We're telling Jenkins to skip the initial wizard so we won't need to install any other plugin, set any password or get any hash from the logs right now when it starts, we'll go straight to business.

__Lines: 26-28__ - We're mounting a volume at the master instance to keep our configurations alive across different deployments.

__Lines: 31-32__ - These are the lines that define where Kubernetes will place your Jenkins master instance.

Since you have `nodeSelector`, before you execute this file, you need to pick one of your nodes to host the Jenkins master, to do so, execute the lines below. Please replace `boss` by the name of your node.

<script src="https://gist.github.com/allandequeiroz-snippets/b0bb3ba04098b3e26f1fd0f9b60eb77f.js"></script>

The first line adds the label; the second one will give you a description about the node, look for `Labels:` you should see amongst the labels a line like `jenkins=master`.

Assuming that you've saved the `yaml` above in a file called `jenkins.yaml`, is time to apply it; to do so, you must be at your __master__ host.

> This process sometimes turns to be a bit repetitive and tedious, if you prefer, use a shell script to make it a bit easier, [here](https://gist.github.com/allandequeiroz-snippets/e4cd2bd63841f3b1878e709c47ca38fd) you'll find an example.
 
<script src="https://gist.github.com/allandequeiroz-snippets/947511b90004cb4258678e8195ce2848.js"></script>

If you've executed [this script](https://gist.github.com/allandequeiroz-snippets/e4cd2bd63841f3b1878e709c47ca38fd) you'll see an output similar to the one below, if not, copy and paste the first lines after the `echoes`, and I'll manage to see what you have so far.

<script src="https://gist.github.com/allandequeiroz-snippets/90bdcc80cee9595a311a315ea42fa83f.js"></script>

Notice that the service doesn't have any information about its public address, but we've specified a host where Jenkins should be deployed __the same one you labelled before__, with this in hand and the `nodePort` we've specified we have enough information to access Jenkins. In my case [http://boss:30000](http://boss:30000)

## Jenkins

Before jumping into the Kubernetes plugin configuration, let's take care of something essential, credentials. First, let's create a service account and fetch the secret.

<script src="https://gist.github.com/allandequeiroz-snippets/c8a0a3314fdfede28b95fc7785ae9d32.js"></script>

Copy the whole content printed at the console by the third command and go to `Jenkins > Credentials > System > Global credentials > Add Credentials`, change the `Kind` drop-down options to `Secret text` and past into `Secret`, set an `ID` to make it easy to pick it up later. There are many ways to handle credentials, but I've picked this approach because I think it would be easier to set up different environments later.

Now let's create our configuration at `Jenkins > Manage Jenkins > Configure System > Add a new cloud > Kubernetes`

Now, we'll need a few pieces of information to set up our Jenkins cluster:

* __Kubernetes URL:__ At the master node executes `kubectl cluster-info | grep master` and copy the URL you get there, for example, `https://192.168.1.101:6443`.
* __Credentials:__ Select the credentials we've created at the previous step.
* __Jenkins tunnel:__ This information is critical to avoid falling into problems such as `connection refused` or `jnlp is not enabled`, you'll need to point to your service endpoint, do you remember the name of the service we've provided at the `jenkins.yaml`? In this case it was `jenkins-master-svc` if you changed, get back there and double-check, we'll need it to build the service FQDN `<service name>.<namespace>.svc.cluster.local:<port>`, in this case, `jenkins-master-svc.default.svc.cluster.local:50000`.

Anything else you can copy from the screenshots below at this time, after that, play around with the configurations until you get what you need.

![Cloud Kubernetes Block](http://url/to/img.png) 

![Kubernetes Pod Template Block](http://url/to/img.png) 

Ok, now let's create some dummy build projects, a `Freestyle project` will do, at `Build > Execute shell` add something silly to start with, for example `sleep 30`, this job will take a while to finish, and we manage to see multiple instances working in parallel. Create two of three to see it working for real.

![Kubernetes Pod Template Block](http://url/to/img.png) 

Back to the main screen start building all of them and once you see new entries at `Build Executor Status` execute the following command to see their real status and the node where they are running at.

<script src="https://gist.github.com/allandequeiroz-snippets/c632aabbe3a3b66e2930216fb92ecde4.js"></script>

By the end, you should see the sunny Jenkins interface.

![Kubernetes Pod Template Block](http://url/to/img.png) 

From this point you have your Jenkins cluster up and running, is up to you know to play with the many configuration options provided by the Kubernetes plugin and to set up your build projects. 