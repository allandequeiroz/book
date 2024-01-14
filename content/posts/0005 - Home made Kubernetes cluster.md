+++
title = "Home made Kubernetes cluster"
description = ""
tags = [
    "docker",
    "kubernetes",
    "containers"
]
date = "2017-06-10"
categories = [
    "software",
    "devops"
]
draft = false
+++

# Homemade Kubernetes Cluster

In my previous post, I wrote about how to set up a scalable Jenkins using Kubernetes at home on top of home devices, in my case, 3 Raspberries, 2 Rock64 and a NUC. Since it wasn't a 5 minutes task I'll describe here the steps I went through to get it working. 

## Pre-checks

Before installing Kubernetes you better check which Docker version is supported by the Kubernetes version you intend to install, a few days ago I upgraded my Kubernetes cluster from version 12 to 15 so, version 15 is what I'll be used as a reference here.

If you go to the [Kubernetes Changelogs](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.15.md) page and look for __Docker__ you'll spot the __"validated docker versions"__, as we can see, I could have upgraded to the version __18.09__, but in the end, I've decided by __18.6.2__.

Said that before installing Docker and Kubernetes it might be a good idea to upgrade your whole system if it is safe to do so.

<script src="https://gist.github.com/allandequeiroz-snippets/f14c30414898d411272ccea734a397ac.js"></script>

## Docker

Once you determined which Docker version to install, before moving forward you need to keep a few details in mind such as the device architectures you also have the OS version. You may need to edit these details at your `sources.list` otherwise the downloaded packages won't work.

- If you have any device which the architecture __is not__ `amd64`, remove `[arch=amd64]` from __sources.list__.
- Instead of $(lsb_release -cs) you can specify the OS release you want to target by hand if you want, __eg: `xenial`__.
- Package version may look strange but it's correct, __eg: `docker-ce=18.06.2~ce~3-0~ubuntu`__.

Once you're done with the steps above you can add the repository keys so you won't get any harmful package by accident.

<script src="https://gist.github.com/allandequeiroz-snippets/6236d41a8555816bbc62e7951b639c44.js"></script>

Now is time to include the sources list per se, please notice the two options below, the first one is for anything that is __not__ `ARM`, the second one for `ARM` architectures.

__Anything but ARM__
<script src="https://gist.github.com/allandequeiroz-snippets/28e8e36dac729be8155612de9ab1ca14.js"></script>

__ARM__
<script src="https://gist.github.com/allandequeiroz-snippets/03decdaa56f006a087004416b2090f79.js"></script>

Install Docker
<script src="https://gist.github.com/allandequeiroz-snippets/8d2b54148e5fd3031bec2d245d21929d.js"></script>

Setup Docker daemon
<script src="https://gist.github.com/allandequeiroz-snippets/2905dfc061af89d71261cd3832950e75.js"></script>

Restart Docker
<script src="https://gist.github.com/allandequeiroz-snippets/57fb2b1da63012fec122cb168b2921f1.js"></script>

At this point, we should have a Docker version fully compliant with Kubernetes, up and running.

### Docker Compose

Since we're installing Docker, just in case you decide to spin up a `docker-compose` for some reason, let's install it as well. The points of attention are similar to the one used during the Docker installation.

__Anything but ARM__
<script src="https://gist.github.com/allandequeiroz-snippets/5c3569a267beba69a4e01116c63b0045.js"></script>

__ARM__

The process to have `docker-compose` at the `ARM` platform was a bit more complicated, at least for me, but is not so complicated, here are the steps I went through:

<script src="https://gist.github.com/allandequeiroz-snippets/96fd18d139a32001d4efc12892a4f2a7.js"></script> 

### Docker Compose bash autocompletion

Now that you have `docker-compose` everywhere is time to set the autocompletion to spare a bit of your patience down the road with typing.

<script src="https://gist.github.com/allandequeiroz-snippets/79aa7d478cf2bd1cf3c39a30c29ecf61.js"></script>

## Kubernetes

With a fully happy Docker running across all the devices, it is time to install Kubernetes.

<script src="https://gist.github.com/allandequeiroz-snippets/bb27bd76309beb4cd2ed2d3f69240968.js"></script>


### Initialising Master Kubernetes

This step will be easier if you save the script below at your master host, execute, and let it do its job. You can run step-by-step by hand if you want, just like the ones before, this is up to you.

<script src="https://gist.github.com/allandequeiroz-snippets/e4d14fb168dbc1f52da780bb9f355b1e.js"></script>

By the end you'll see amongst the logged lines, one starting with `kubeadm join`, copy this whole line and execute across the other devices you have. The line I'm talking about is similar to this:

`kubeadm join 192.168.1.2:6443 --token zd07uq.d91cxeg22nhwl6ti --discovery-token-ca-cert-hash sha256:551f848676f99621bbed06810d15532f69019398d18d475a0cbaa9e69ee9d195`

At this point you should have all your devices communicating with the manager, to check how is it going execute `kubectl get nodes -o wide` at the manager node, maybe the nodes are not ready yet, give it some time, they need to perform some tasks as well. Once you see something like the lines below, you're ready to go.

<script src="https://gist.github.com/allandequeiroz-snippets/731c7e4b54a1b06d9fb9ee0cecb2b9dd.js"></script>

### Kubernetes autocompletion

Last but not least, `kubectl` completion, believe me, this one is a must-have, especially about the Pod's names, you won't want to copy and paste or even worse, type the names Kubernetes gives to the Pods.

echo "source <(kubectl completion bash)" >> ~/.bashrc 

## Conclusion

It was a short step-by-step on how to set up your Kubernetes cluster; I believe it'll be useful, especially if you have devices with different architectures. It may not be fully compatible with the scenario, but for sure it'll give you some ideas to solve your problems.

Another thing, if you're looking to go beyond and have your bare metal Load balanced cluster, take a look at [MetalLb](https://metallb.universe.tf/). And do not forget to take a look at the [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/) page as well, the `kubernetes_master.sh` installed [Wave Net](https://www.weave.works/oss/net/) for you but, you may want a different one.

Hope it helps!

