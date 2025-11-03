<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.2/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.4/README.md">Next ➡️</a></td>
  </tr>
</table>

# 3. Familiarizing with Docker

This section guides the reader along an initial hands-on tour to explore the world of Docker containers.
Obviously, only a very limited subset of the available Docker commands and flags are going to be presented hereafter.
In case of doubts, remember that it is always possible to issue:

```bash
docker --help
docker COMMAND --help
```

to obtain more information.

Part of the following is inspired by the first [Play with Docker](https://training.play-with-docker.com/) laboratory; additional hands-on tutorials can be found there as well.

## 3.1. Hello-world: run, show and delete container and images

First, let's recap the difference between *images* and *containers*.

A Docker image can be seen as an archive, comprised of multiple layers, which groups together all the files (executables, libraries, tools, and configuration files) needed to execute code in a Docker container.
Docker images can be either pulled from online repositories or created by means of Dockerfiles (see the corresponding section later).

Multiple independent containers can be created starting from a single image, each one characterized by its own status.
Modifications performed in a container are not automatically reflected back to the corresponding image, thus guaranteeing isolation.

### 3.1.1. Run the Hello World container

```bash
sudo docker container run hello-world
```

Observe the output, which briefly describes the steps performed by Docker to run the container.

You have probably noticed how fast Docker was to create, start, and tear down the container.
Execution speed is one of the main distinguishing characteristics of lightweight containers compared to virtual machines.
While VMs are **hardware abstractions** (requiring to boot a full operating system), containers are **application abstractions**, running on top of an already running OS.

### 3.1.2. Common Docker commands

Shows all the images locally available.
  ```bash
  sudo docker image ls
  ```
  > You should see only the `hello-world` image.

Shows the containers that are *currently* running.
  ```bash
  sudo docker container ls
  ```
  > Since none are running, this should show nothing.


Lists *all* containers ever run, including stopped ones
  ```bash
  sudo docker container ls -a
  ```
  > Shows `hello-world` with the `Exited` status.

> [!NOTE]
> If you do not assign a name to a container during creation with the `--name` option, Docker assigns a random one (e.g., `nostalgic_mccarthy`).
> It is the concatenation of an adjective and a name for easier memorization.

**Exercise:** Take note of the name assigned to your container.

### 3.1.3. Remove containers and images

Remove a container by ID or name:

```bash
sudo docker container rm <container ID>
```

> [!NOTE]
> Containers must be removed using either their *container ID* or *container name*, not their *image name* (since multiple containers may share the same image).

After running this, `docker container ls -a` should return nothing, while the image still exists (`docker image ls` shows `hello-world`).

To remove the image as well:

```bash
sudo docker rmi hello-world
# or
sudo docker image rm hello-world
```

## 3.2. Starting an Alpine Linux container

[Alpine Linux](https://alpinelinux.org/) is a lightweight Linux distribution, smaller and more resource-efficient than traditional ones.
For instance, the `alpine` image occupies less than **8 MB**, compared to **100 MB** for `ubuntu` or `debian`.

To fetch the image:

```bash
sudo docker image pull alpine:3.10
```

Then, to run it:

```bash
sudo docker container run alpine:3.10 echo "hello from alpine"
```

Docker creates a new container from the `alpine` image, runs the command, and then automatically stops the container.
The `run` command did not download the image since it was already available locally.

## 3.3. Verify the isolation of running containers

Run a shell inside a new container instance, create a file, and inspect it:

```bash
# Start the alpine container and run a shell
sudo docker container run -it alpine:3.10 /bin/sh

# Inside the container
touch test-file
ls -l
exit
```

The `-it` flag enables interactive mode.
Without it, the shell would start and immediately terminate since no command was supplied.

> [!NOTE] 
> The `/bin/sh` command must exist inside the container filesystem.

Now, create another container and list files:

```bash
sudo docker container run alpine:3.10 ls -l
```

**Question**: Is `test-file` present? Why?

### 3.3.1. Limiting CPU and Memory

To enforce CPU and memory limits:

```bash
sudo docker container run -it --cpus=1 --memory=$((100*1024*1024)) alpine:3.10 /bin/sh

# and now within the container
apk add htop stress-ng
```

Run `htop` inside the container.
Does it show the container’s or the host’s resources?
Now simulate load:

```bash
stress-ng --cpu=2 & htop
```

Observe whether the constraints are enforced.
Applications inside containers can perceive the host’s full resources even though usage is limited, which may affect their behavior.

## 3.4. Running a shell in an already running container

Sometimes it is necessary to attach a shell to a running container for debugging.

Start a long-running container:

```bash
sudo docker container run --name deadlock-app --rm --init alpine:3.10 sleep 1d
```

* `--name`: assigns a name (`deadlock-app`)
* `--rm`: automatically removes the container upon termination
* `--init`: runs an init process to handle signals properly
* `sleep 1d`: keeps the container running for one day

Open a **new terminal** and attach a shell:

```bash
sudo docker exec -it deadlock-app /bin/sh
ps -a
killall sleep
```

The container terminates automatically because of `--rm`.

> [!IMPORTANT]
> You can inspect running containers from within using `docker exec`. This greatily simplifies debugging inside containers.
>
> As a **best practice** always include `/bin/sh` (or another shell) in development containers, even if stripped from production versions.

## 3.5. Docker networking brief

Now, let’s investigate Docker’s default networking setup.

Open two terminals and start two Alpine containers with interactive shells (as done above).
Inspect IP addresses using:

```bash
ip addr
```

Then try:

* Can the two containers communicate with each other?
* Can a container ping the host?
* Can they access the Internet?
* What is the default Docker network configuration?

> [!WARNING]
> ICMP echo messages (ping) may be blocked by your network administrators.
> If testing from a VM (e.g., in the PoliTO datacenter), pinging external hosts might fail.
> Try pinging internal addresses (e.g., [www.polito.it](https://www.polito.it)) or use:
>
> ```bash
> wget https://www.polito.it
> ```
