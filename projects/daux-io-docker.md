# Publish and maintain markdown written documentations with daux.io, Docker and GitHub

As many of us will agree: Markdown is a great way to simply write documentations easy and quickly. [daux.io](http://daux.io/) is a documentation generator that uses a simple folder structure and Markdown files to create custom documentation on the fly. It helps you to create great looking documentation in a developer friendly way.

Daux.io is easy to install and easy to use. Nevertheless I was thinking about a solution how I can split up the documentation's source files from presenting the documentation with Daux.io. I came up with a simple approach where Daux.io is running in a Docker container and the documentation sources are hosted on GitHub. The container exposes an additional URL for GitHub WebHooks to trigger an update process of the documentation on git-push events.

---

![Solution Overview](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-09-05_daux-overview.png)

The docker image including a running Daux.io installation and the WebHook is available on DockerHub:

```
docker pull cokeschlumpf/daux.io
```

The sources are also published on [GitHub](http://github.com/cokeschlumpf/daux.io).

Following I'll explain step by step how to setup the environment for this documentation solution on [IBM Bluemix](https://console.ng.bluemix.net).

---

## Step 1: Clone the Docker image and publish to your private Bluemix registry

First, clone the image from DockerHub:

```
docker pull cokeschlumpf/daux.io
```

To make the image available on your organizations Bluemix instance, tag the image with your organizations credentials:

```
docker tag cokeschlumpf/daux.io:latest registry.ng.bluemix.net/<your_namespace>/daux.io
```

Afterwards push the image to your Bluemix private repository:

```
docker push registry.ng.bluemix.net/<your_namespace>/daux.io
```

For further information see [IBM Container Docs](https://console.ng.bluemix.net/docs/containers/container_images_pulling.html).

## Step 2: Create & run container on Bluemix

Start the container on Bluemix as single container or container group. Optionally you can bind a Bluemix or custom URL to your container. For further details see [IBM Container Docs](https://console.ng.bluemix.net/docs/containers/container_index.html).

## Step 3: Create a repository with a WebHook for your documentation

Simply create a repository on GitHub including your documentation or fork my sample repository [cokeschlumpf/daux.io-sample-docs](http://github.com/cokeschlumpf/daux.io-sample-docs).

Finally you'll need to create a WebHook in your repository which calls your container's update endpoint on every push:

```
http://<CONTAINER_IP>/update-webhook.php
```

![GitHub Web Hook configuration](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/master/images/2016-09-05_daux-github-webhook.png)

The first time `update-webhook.php` is called by GitHub it's initialized for the GitHub repository, only this repository can now effectively call the endpoint. When it's called, the repository's master branch is pulled from GitHub and the documentation will be updated in the container.
