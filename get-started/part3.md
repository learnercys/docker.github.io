---
title: "Get Started, Part 3: Services"
keywords: services, replicas, scale, ports, compose, compose file, stack, networking
description: Learn how to define load-balanced and scalable service that runs containers.
---
{% include_relative nav.html selected="3" %}

## Prerequisites

- [Install Docker version 1.13 or higher](/engine/installation/index.md).

- Get [Docker Compose](/compose/overview.md). On [Docker for
Mac](/docker-for-mac/index.md) and [Docker for
Windows](/docker-for-windows/index.md) it's pre-installed, so you're good-to-go.
On Linux systems you will need to [install it
directly](https://github.com/docker/compose/releases). On pre Windows 10 systems
_without Hyper-V_, use [Docker
Toolbox](https://docs.docker.com/toolbox/overview.md).

- Read the orientation in [Part 1](index.md).

- Learn how to create containers in [Part 2](part2.md).

- Make sure you have published the `friendlyhello` image you created by
[pushing it to a registry](/get-started/part2.md#share-your-image). We'll
use that shared image here.

- Be sure your image works as a deployed container. Run this command,
slotting in your info for `username`, `repo`, and `tag`: `docker run -p 80:80
username/repo:tag`, then visit `http://localhost/`.

## Introduction

In part 3, we scale our application and enable load-balancing. To do this, we
must go one level up in the hierarchy of a distributed application: the
**service**.

- Stack
- **Services** (you are here)
- Container (covered in [part 2](part2.md))

## About services

In a distributed application, different pieces of the app are called "services."
For example, if you imagine a video sharing site, it probably includes a service
for storing application data in a database, a service for video transcoding in
the background after a user uploads something, a service for the front-end, and
so on.

Services are really just "containers in production." A service only runs one
image, but it codifies the way that image runs&#8212;what ports it should use,
how many replicas of the container should run so the service has the capacity it
needs, and so on. Scaling a service changes the number of container instances
running that piece of software, assigning more computing resources to the
service in the process.

Luckily it's very easy to define, run, and scale services with the Docker
platform -- just write a `docker-compose.yml` file.

## Your first `docker-compose.yml` file

A `docker-compose.yml` file is a YAML file that defines how Docker containers
should behave in production.

### `docker-compose.yml`

Save this file as `docker-compose.yml` wherever you want. Be sure you have
[pushed the image](/get-started/part2.md#share-your-image) you created in [Part
2](part2.md) to a registry, and update this `.yml` by replacing
`username/repo:tag` with your image details.

```yaml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet
networks:
  webnet:
```

This `docker-compose.yml` file tells Docker to do the following:

- Pull [the image we uploaded in step 2](part2.md) from the registry.

- Run 5 instances of that image as a service
  called `web`, limiting each one to use, at most, 10% of the CPU (across all
  cores), and 50MB of RAM.

- Immediately restart containers if one fails.

- Map port 80 on the host to `web`'s port 80.

- Instruct `web`'s containers to share port 80 via a load-balanced network
  called `webnet`. (Internally, the containers themselves will publish to
  `web`'s port 80 at an ephemeral port.)

- Define the `webnet` network with the default settings (which is a
  load-balanced overlay network).


> Wondering about Compose file versions, names, and commands?
>
Notice that we set the Compose file to `version: "3"`. This essentially makes it
[swarm mode](/engine/swarm/index.md) compatible. We can make use of the [deploy
key](/compose/compose-file/index.md#deploy) (only available on [Compose file
formats version 3.x](/compose/compose-file/index.md) and up) and its sub-options
to load balance and optimize performance for each service (e.g., `web`). We can
run the file with the `docker stack deploy` command (also only supported on
Compose files version 3.x and up). You could use `docker-compose up` to run
version 3 files with _non swarm_ configurations, but we are focusing on a stack
deployment since we are building up to a swarm example.
>
You can name the Compose file anything you want to make it logically meaningful
to you; `docker-compose.yml` is simply a standard name. We could just as easily
have called this file `docker-stack.yml` or something more specific to our
project.

## Run your new load-balanced app

Before we can use the `docker stack deploy` command we'll first run:

```shell
docker swarm init
```

>**Note**: We'll get into the meaning of that command in [part 4](part4.md).
> If you don't run `docker swarm init` you'll get an error that "this node is not a swarm manager."

Now let's run it. You have to give your app a name. Here, it is set to
`getstartedlab`:

```shell
docker stack deploy -c docker-compose.yml getstartedlab
```

Our single service stack is running 5 container instances of our deployed image
on one host. Let's investigate.

Get the service ID for the one service in our application:

```shell
docker service ls
```

You'll see output for the `web` service, prepended with your app name. If you
named it the same as shown in this example, the name will be
`getstartedlab_web`. The service ID is listed as well, along with the number of
replicas, image name, and exposed ports.

Docker swarms run tasks that spawn containers. Tasks have state and their own
IDs. Let's list the tasks:

```shell
docker service ps <service>
```

>**Note**: Docker's support for swarms is built using a project
called SwarmKit. SwarmKit tasks do not need to be containers, but
Docker swarm tasks are defined to spawn them.

Let's inspect one of these tasks, and limit the output to container ID:

```shell
docker inspect --format='{% raw %}{{.Status.ContainerStatus.ContainerID}}{% endraw %}' <task>
```

Vice versa, you can inspect a container ID, and extract the task ID.

First run `docker container ls` to get container IDs, then:

```shell
docker inspect --format="{% raw %}{{index .Config.Labels \"com.docker.swarm.task.id\"}}{% endraw %}" <container>
```

Now list all 5 containers:

```shell
docker container ls -q
```

You can run `curl http://localhost` several times in a row, or go to that URL in
your browser and hit refresh a few times.

![Hello World in browser](images/app80-in-browser.png)

Either way, you'll see the container ID change, demonstrating the
load-balancing; with each request, one of the 5 replicas is chosen, in a
round-robin fashion, to respond. The container IDs will match your output from
the previous command (`docker container ls -q`).

(Windows 10 PowerShell should already have `curl` available, but if not you can
grab a Linux terminal emulater like [Git
BASH](https://git-for-windows.github.io/){: target="_blank" class="_"} if you
want to try it out. It isn't critical to the taskflow here.)

>**Note**: At this stage, it may take up to 30 seconds for the containers
to respond to HTTP requests. This is not indicative of Docker or
swarm performance, but rather an unmet Redis dependency that we will
address later in the tutorial. For now, the visitor counter isn't working
for the same reason; we haven't yet added a service to persist data.


## Scale the app

You can scale the app by changing the `replicas` value in `docker-compose.yml`,
saving the change, and re-running the `docker stack deploy` command:

```shell
docker stack deploy -c docker-compose.yml getstartedlab
```

Docker will do an in-place update, no need to tear the stack down first or kill
any containers.

Now, re-run `docker container ls -q` to see the deployed instances reconfigured.
If you scaled up the replicas, more tasks, and hence, more containers, are
started.

### Take down the app and the swarm

Take the app down with `docker stack rm`:

```shell
docker stack rm getstartedlab
```

This removes the app, but our one-node swarm is still up and running (as shown
by `docker node ls`). Take down the swarm with `docker swarm leave --force`.

It's as easy as that to stand up and scale your app with Docker. You've taken a
huge step towards learning how to run containers in production. Up next, you
will learn how to run this app as a bonafide swarm on a cluster of Docker
machines.

> **Note**: Compose files like this are used to define applications with Docker, and can be uploaded to cloud providers using [Docker
Cloud](/docker-cloud/), or on any hardware or cloud provider you choose with
[Docker Enterprise Edition](https://www.docker.com/enterprise-edition).

[On to "Part 4" >>](part4.md){: class="button outline-btn"}

## Recap and cheat sheet (optional)

Here's [a terminal recording of what was covered on this page](https://asciinema.org/a/b5gai4rnflh7r0kie01fx6lip):

<script type="text/javascript" src="https://asciinema.org/a/b5gai4rnflh7r0kie01fx6lip.js" id="asciicast-b5gai4rnflh7r0kie01fx6lip" speed="2" async></script>

To recap, while typing `docker run` is simple enough, the true implementation
of a container in production is running it as a service. Services codify a
container's behavior in a Compose file, and this file can be used to scale,
limit, and redeploy our app. Changes to the service can be applied in place, as
it runs, using the same command that launched the service:
`docker stack deploy`.

Some commands to explore at this stage:

```shell
docker stack ls                                            # List stacks or apps
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker service ls                 # List running services associated with an app
docker service ps <service>                  # List tasks associated with an app
docker inspect <task or container>                   # Inspect task or container
docker container ls -q                                      # List container IDs
docker stack rm <appname>                             # Tear down an application
```
