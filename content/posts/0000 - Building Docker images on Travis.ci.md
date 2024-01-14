+++
title = "Building Docker images on Travis.ci"
description = ""
tags = [
    "docker",
    "container",
    "travis.ci",
]
date = "2016-01-02"
categories = [
    "software",
    "devops",
]
draft = false
+++

Inspired by the posts from [Viktor Adam's](https://blog.viktoradam.net/) blog, when I have some spare time I have been playing with Docker and some open source tools to build this blog.

So far I have my two little devices a [Raspberry Pi](https://www.raspberrypi.org/) and a [Rock64](https://www.armbian.com/rock64/) hosting this blog, yes, that's right, you're reading this straight from a [Ghost](https://ghost.org/) blog behind an [NGinx](https://nginx.org/en/) hosted at my home, be nice and say hi!

![Allan's home devices](https://i.imgur.com/iu1rpSO.png)

Appart of some details like sensible public data management and external CI usage to generate and publish Docker images `that is the subject of this post,` this is how my stack looks like right now. It will evolve soonish :]

![Initial home stack](https://i.imgur.com/DY0FwHq.png)

## Sensible public data

As we can see from the description of [this repo](https://hub.docker.com/r/allandequeiroz/cloudflare-ddns/) of mine on Docker hub, some environment variables are expected as parameters to create containers. Some sensible data like Cloudflare API key goes to a [public repository](https://github.com/allandequeiroz/cloudflare-ddns) on Github, the `.env` file in binary form instead of row text.

I could keep it somewhere on my local file system and use it to create containers but if I've learned something from past events is, maintaining files on my machine is the same of losing them sooner or later so, better to have them on the cloud where smarter people than me are taking care of backups.

However, how to keep this kind of data safe in a public space? If you are using `git` a good option is to use [Git Crypt](https://www.agwa.name/projects/git-crypt).

## Git Crypt 

First, install it, following this [guide](https://github.com/AGWA/git-crypt/blob/master/INSTALL.md).

Once you have installed it you need a key to `lock` and `unlock` the files you want, there are two options, the `Symmetric Mode` where you export a key and refer it to unlock your files and `GPG Mode` where you create a key and import the public key wherever you need to unlock files. I have decided to use `Symmetric Mode,` this time __*"after few disappointments with the GPG Mode"*__ . The setup is easy and do not require further steps after you export your key.

## Setting up git-crypt and exporting a symmetric secret key

Now on the git repository initiate, `git-crypt` like with a regular `git` init and export your symmetric key. To accomplish this you only need to perform the steps below.

<pre class="line-numbers language-bash">
<code>git-crypt init
git-crypt export-key /lab/security/git-cript-key
</code></pre>

Now you need to create the `.gitattributes` at the root of your project and list the files you want to encrypt, the syntax is the same used by `.gitignore` files. Eg: `*.env filter=git-crypt diff=git-crypt`

From this point `.env` will be automatically encrypted before being pushed to the repository. To unlock it in different machines, you need to run `git-crypt unlock` referring the secret key that you've just exported, in my case `git-crypt unlock /lab/security/git-crypt-key`

> If you have the intention to unlock this files somewhere else, you need to have access to this key there too, the process is merely the same, unlock the files with the same command above.


## Setting up a Trevis.ci account

At this point, you probably have something [like that](https://github.com/allandequeiroz/cloudflare-ddns) on your GitHub. The next step is to build your images and publish them automatically when you update your repository.

Using your GitHub account, Sign Up to [Trevis.ci](https://travis-ci.org/) and enable the integration for this repo.

## Setting up the .travis.yml file

To tell Travis what to do, you need a `.travis` file on your repository, you can learn about it [here](https://docs.travis-ci.com/) to create one that works for you. Mine looks like that:

<pre class="line-numbers language-yaml">
<code>sudo: required

services:
- docker

script:
# setup QEMU
- docker run --rm --privileged multiarch/qemu-user-static:register --reset
# build images
- docker build -t ddns:$DOCKER_TAG -f Dockerfile.$DOCKER_ARCH .
# push images
- docker login -u="$DOCKER_HUB_LOGIN" -p="$DOCKER_HUB_PASSWORD"
- docker tag ddns:$DOCKER_TAG allandequeiroz/cloudflare-ddns:$DOCKER_TAG
- docker push allandequeiroz/cloudflare-ddns:$DOCKER_TAG
- >
  if [ "$LATEST" == "true" ]; then
    docker tag ddns:$DOCKER_TAG allandequeiroz/cloudflare-ddns:latest
    docker push allandequeiroz/cloudflare-ddns:latest
  fi

env:
  matrix:
  - DOCKER_TAG=arm       DOCKER_ARCH=arm       LATEST=true
  - DOCKER_TAG=amd64     DOCKER_ARCH=amd64
</code></pre>

Since I'm using Raspberry Pi and Rock64, I need to build images for `arm` architecture, and since I want to be nice to people, I'm generating `amd` ones even that they are useless for me.

You can ask `Travis` to do that defining the `matrix` section, this way the `script` part will be executed once for each line of the `matrix`, in this case, every time I push something new to GitHub, two new images will be generated and pushed to Docker hub.

Another interesting detail here is the ability to define environment variables, as you may notice my Docker hub credentials to push images: `docker login -u="$DOCKER_HUB_LOGIN" -p="$DOCKER_HUB_PASSWORD"`

`Travis` provides a `settings` interface where you can define environment variables and use them instead of keeping it public on your repository.

Ok. Now you need to push something new to your repository or Trigger the build from `More options` on Travis interface, by the end you'll see something like that on Travis, and your images will be available on Docker hub.

## Travis badges

After all this effort you deserve some badges, you can get one straight from your `Travis account`, just ask for the `.svg` file related to your repository. Travis update it after each build so take care to not break the build :)

`https://api.travis-ci.org/allandequeiroz/cloudflare-ddns.svg`
