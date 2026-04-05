# Docker for beginners: images, containers, logs, cleanup, and troubleshooting

This guide is written for someone who is new to Docker and wants a practical mental model first, then the commands that matter day to day.

## Start with the basic idea

The most common beginner mistake is mixing up **images** and **containers**.

- An **image** is a packaged template. Think of it like a read-only blueprint or snapshot.
- A **container** is a running or stopped instance created from an image.

A simple way to think about it:

- **Image** = recipe
- **Container** = meal made from the recipe

You can have one image and many containers created from it.

Examples:

- `vrnetlab/cisco_iol:17.16.01a` is an **image**
- a running lab node created from that image is a **container**

## The Docker objects you will deal with most

When people say “Docker”, they are usually working with these things:

### Images
Stored locally on your system and used to create containers.

Useful commands:

```bash
docker image ls
# shorthand
docker images
```

### Containers
Running or stopped instances created from images.

Useful commands:

```bash
docker ps
# show all, including stopped containers
docker ps -a
```

### Volumes
Persistent storage used by containers.

Useful commands:

```bash
docker volume ls
```

### Networks
Virtual networks Docker creates so containers can communicate.

Useful commands:

```bash
docker network ls
```

## Check that Docker is working

Before you troubleshoot anything complicated, confirm the Docker daemon is running.

```bash
docker version
docker info
```

A quick functional test:

```bash
docker run hello-world
```

If that works, Docker should be installed correctly.

## List images on your system

To list locally stored images:

```bash
docker image ls
```

Typical output columns:

- `REPOSITORY` — image name
- `TAG` — version or label
- `IMAGE ID` — unique identifier
- `CREATED` — when it was built or pulled
- `SIZE` — image size on disk

Examples:

```bash
docker image ls
docker image ls vrnetlab/cisco_iol
docker image ls --digests
```

If you prefer the older shorthand:

```bash
docker images
```

## Check running containers

To see only running containers:

```bash
docker ps
```

To see everything, including stopped containers:

```bash
docker ps -a
```

Important columns:

- `CONTAINER ID` — short ID for the container
- `IMAGE` — which image the container was created from
- `COMMAND` — startup command
- `STATUS` — running, exited, restarting, and so on
- `PORTS` — exposed or mapped ports
- `NAMES` — container name

Useful filters:

```bash
docker ps -a --filter name=iol
docker ps -a --filter status=exited
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Names}}'
```

## Inspect a specific image or container

When you need more detail than `docker ps` or `docker images` gives you, use `inspect`.

### Inspect an image

```bash
docker image inspect vrnetlab/cisco_iol:17.16.01a
```

### Inspect a container

```bash
docker container inspect <container_name_or_id>
```

This is very useful for checking:

- environment variables
- mounted volumes
- network settings
- entrypoint and command
- restart policies
- image metadata

## Start, stop, restart, and remove containers

### Start a stopped container

```bash
docker start <container_name_or_id>
```

### Stop a running container cleanly

```bash
docker stop <container_name_or_id>
```

If a container is slow to stop, set a timeout:

```bash
docker stop -t 30 <container_name_or_id>
```

### Restart a container

```bash
docker restart <container_name_or_id>
```

### Force kill a container

Use this only when normal stop fails.

```bash
docker kill <container_name_or_id>
```

### Remove a stopped container

```bash
docker rm <container_name_or_id>
```

### Remove a running container forcefully

```bash
docker rm -f <container_name_or_id>
```

## Remove images

To remove an image:

```bash
docker rmi vrnetlab/cisco_iol:17.16.01a
```

You can also use:

```bash
docker image rm vrnetlab/cisco_iol:17.16.01a
```

If Docker says the image is in use, it usually means one or more containers still reference it.

Find them:

```bash
docker ps -a
```

Remove the related container first, then remove the image.

If needed, force removal:

```bash
docker rmi -f vrnetlab/cisco_iol:17.16.01a
```

Be careful with `-f`. It is useful in labs, but it can hide what is actually using the image.

## View logs

Logs are one of the first places to look when a container does not behave as expected.

### Show logs once

```bash
docker logs <container_name_or_id>
```

### Follow logs live

```bash
docker logs -f <container_name_or_id>
```

### Show only the last 50 lines

```bash
docker logs --tail 50 <container_name_or_id>
```

### Show timestamps

```bash
docker logs -t <container_name_or_id>
```

### Combine useful options

```bash
docker logs -f --tail 100 -t <container_name_or_id>
```

## Get a shell inside a running container

If the container is running and has a shell installed, this is often the fastest way to understand what is happening.

```bash
docker exec -it <container_name_or_id> /bin/bash
```

If Bash is not installed:

```bash
docker exec -it <container_name_or_id> /bin/sh
```

Common uses:

- check files inside the container
- verify paths and permissions
- test commands manually
- inspect generated config files

## Watch resource usage

To check CPU, memory, network I/O, and block I/O in real time:

```bash
docker stats
```

Or for one container:

```bash
docker stats <container_name_or_id>
```

Press ctrl+c to exit

This helps when a container looks “hung” but is actually overloaded.

## See running processes inside a container

```bash
docker top <container_name_or_id>
```

This is useful for confirming whether the expected process is actually running.

## Common beginner troubleshooting workflow

When a container does not start properly, use this sequence.

### Step 1: confirm the image exists

```bash
docker image ls | grep cisco_iol
```

### Step 2: see whether the container exists

```bash
docker ps -a
```

### Step 3: read the logs

```bash
docker logs <container_name_or_id>
```

### Step 4: inspect detailed settings

```bash
docker inspect <container_name_or_id>
```

### Step 5: check whether it is constantly restarting

```bash
docker ps -a
```

Look for status messages such as:

- `Restarting`
- `Exited (1)`
- `Exited (127)`

### Step 6: enter the container if possible

```bash
docker exec -it <container_name_or_id> /bin/sh
```

### Step 7: check host-side issues

Examples:

- wrong file mounted into the container
- port conflict
- missing permissions
- not enough RAM or disk space

## 14. Useful exit codes to recognise

These are not Docker-specific, but they come up a lot.

- `Exited (0)` — the process finished successfully
- `Exited (1)` — generic application error
- `Exited (125)` — Docker itself failed to run the container
- `Exited (126)` — command invoked but cannot execute
- `Exited (127)` — command not found
- `Exited (137)` — process was killed, often due to `SIGKILL` or memory pressure

## Cleanup commands you will actually use

### Remove stopped containers

```bash
docker container prune
```

### Remove unused images

```bash
docker image prune
```

### Remove all unused objects

```bash
docker system prune
```

### Remove all unused objects including unused images, not just dangling ones

```bash
docker system prune -a
```

Be careful with prune commands. They are excellent for labs, but they can remove things you intended to keep.

## Pull images from a registry

To download an image from Docker Hub or another registry:

```bash
docker pull ubuntu:24.04
```

Then confirm:

```bash
docker image ls ubuntu
```

## Run a quick test container

A simple interactive Ubuntu shell:

```bash
docker run -it --name test-ubuntu ubuntu:24.04 /bin/bash
```

When you exit, the container stops but still exists.

See it:

```bash
docker ps -a
```

Restart it:

```bash
docker start -ai test-ubuntu
```

Remove it when done:

```bash
docker rm test-ubuntu
```

## The difference between stop, remove, and delete

These are often confused.

- `docker stop` — stops a running container, but keeps it
- `docker rm` — removes a container
- `docker rmi` — removes an image

So the normal lifecycle is often:

```bash
docker stop mycontainer
docker rm mycontainer
docker rmi myimage:tag
```

## A practical lab example

Imagine you built this image:

```bash
vrnetlab/cisco_iol:17.16.01a
```

You can confirm it exists:

```bash
docker image ls | grep cisco_iol
```

Check whether anything is running from it:

```bash
docker ps -a --filter ancestor=vrnetlab/cisco_iol:17.16.01a
```

If nothing is using it, remove it:

```bash
docker rmi vrnetlab/cisco_iol:17.16.01a
```

If Docker says it is in use:

```bash
docker ps -a
docker rm -f <container_name_or_id>
docker rmi vrnetlab/cisco_iol:17.16.01a
```

## Compose: if you later move beyond single containers

For multi-container applications, Docker Compose becomes the normal workflow.

Common commands:

```bash
docker compose ps
docker compose logs -f
docker compose stop
docker compose down
```

A simple way to think about it:

- `docker` manages one-off containers and images
- `docker compose` manages groups of related containers

## Good habits to build early

### Use explicit tags

Prefer:

```bash
docker pull ubuntu:24.04
```

instead of:

```bash
docker pull ubuntu:latest
```

Specific tags make your labs reproducible.

### Name your containers

Instead of letting Docker generate random names:

```bash
docker run --name mytest ...
```

### Read the logs before guessing

Many Docker problems become obvious once you read the logs.

### Avoid using `-f` until you know why you need it

Force flags are useful, but they can hide the underlying issue.

### Clean up intentionally

It is tempting to run `docker system prune -a` all the time. Use it when you understand what will be deleted.

## Quick command cheat sheet

### Images

```bash
docker image ls
docker image inspect <image>
docker image rm <image>
docker image prune
```

### Containers

```bash
docker ps
docker ps -a
docker start <container>
docker stop <container>
docker restart <container>
docker rm <container>
docker rm -f <container>
```

### Debugging

```bash
docker logs <container>
docker logs -f <container>
docker exec -it <container> /bin/sh
docker inspect <container>
docker stats
docker top <container>
```

### Cleanup

```bash
docker container prune
docker image prune
docker system prune
docker system prune -a
```

## Official references

These are the best places to verify syntax and behaviour.

- Docker docs home: <https://docs.docker.com/>
- Docker CLI reference: <https://docs.docker.com/reference/cli/docker/>
- `docker image ls`: <https://docs.docker.com/reference/cli/docker/image/ls/>
- `docker ps` / `docker container ls`: <https://docs.docker.com/reference/cli/docker/container/ls/>
- `docker stop`: <https://docs.docker.com/reference/cli/docker/container/stop/>
- `docker logs`: <https://docs.docker.com/reference/cli/docker/container/logs/>
- Logging overview: <https://docs.docker.com/engine/logging/>
- `docker image` subcommands: <https://docs.docker.com/reference/cli/docker/image/>
- Running containers: <https://docs.docker.com/engine/containers/run/>
- Build, tag, and publish an image: <https://docs.docker.com/get-started/docker-concepts/building-images/build-tag-and-publish-an-image/>
- What is an image?: <https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/>
- Docker Compose `ps`: <https://docs.docker.com/reference/cli/docker/compose/ps/>
- Docker Compose `logs`: <https://docs.docker.com/reference/cli/docker/compose/logs/>
- Docker Compose `stop`: <https://docs.docker.com/reference/cli/docker/compose/stop/>

## Final advice

When you are learning Docker, do not try to memorise every command.

Instead, remember this small loop:

1. list images
2. list containers
3. read logs
4. inspect details
5. stop or remove only what you understand

That alone will solve most beginner issues.
