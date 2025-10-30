<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.5/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../README.md">End ✅</a></td>
  </tr>
</table>

# 6. Additional “Do It Yourself” Exercises

This section presents a set of **advanced, self-guided exercises** designed to:

1. Strengthen your understanding of containerization concepts.
2. Test your ability to continue experimenting independently.

## 6.1. The Problem of Cooperative Applications

As discussed earlier, **best practices** recommend running **a single process per container**.
This design provides multiple benefits:

* **Separation of Concerns:**
  Containers with only one process are simpler, easier to orchestrate, and support horizontal scaling and rolling upgrades.

* **Self-Healing:**
  The Docker engine can automatically restart failed containers. However, it cannot detect crashes in auxiliary processes inside a multi-process container.

* **Troubleshooting:**
  Centralized logs become more readable when each container hosts only one process.

This principle, however, poses challenges for **legacy monolithic applications** that are hard to decompose.
In practice, you may need **multiple cooperating containers** to emulate tightly coupled behaviors.
Common scenarios include:

* **Configuration Reload:**
  A sidecar watches configuration files, rebuilds them, and signals the main process to reload.

* **Metrics Exposure:**
  A sidecar exports metrics for a main process that does not natively support monitoring.

* **Proxying or TLS Termination:**
  A sidecar acts as a reverse proxy to expose the main service via HTTPS.

Such containers often need to **share Linux namespaces** (network, PID, etc.) to interact effectively.
In the next sections, you will explore how to share namespaces and understand their relevance in orchestration.

## 6.2. Sharing Network Namespaces with Docker

Containers can share their **network namespace**, allowing them to see the same interfaces and communicate over `localhost`.

Normally, containers can reuse the same port numbers because each one has its own isolated network namespace.
When they share a namespace, they share interfaces and port bindings — so two containers cannot listen on the same port simultaneously.

### 6.2.1. Example

```bash
docker container run -d --name=nginx nginx:1.23
docker container run -it --network=container:nginx alpine:3.10
```

Here, the second container joins the **network namespace** of `nginx`.
Inside the Alpine shell, you can run:

```bash
wget -O - -q localhost
```

This fetches the Nginx landing page via `localhost`, showing how both containers share the same network stack.

In Docker Compose, this can be defined using the `network_mode` field:

```yaml
version: "3.7"
services:
  nginx:
    image: nginx:1.23
  alpine:
    image: alpine:3.10
    command: wget -O - -q localhost
    network_mode: service:nginx
```

> [!TIP]
> Ensure there are **no spaces** after the colon in `service:nginx`.

## 6.3. Multi-Stage Builds

When building Docker images, keeping them **small and clean** is crucial.
Large images increase download times and may expose more vulnerabilities.

To achieve lean production images, Docker supports **multi-stage builds**, where:

1. The first stage builds the application (with all necessary toolchains).
2. The second stage copies only the resulting binaries or artifacts.

This technique drastically reduces image size and attack surface.

For example, consider a small **C application** using the `libpcap` library:

```c
#include <pcap.h>
#include <stdio.h>

int main() {
  pcap_if_t *alldevs;
  char errbuf[PCAP_ERRBUF_SIZE];
  if (pcap_findalldevs(&alldevs, errbuf) == -1) {
    fprintf(stderr, "Error: %s\n", errbuf);
    return 1;
  }
  for (pcap_if_t *d = alldevs; d != NULL; d = d->next)
    printf("%s\n", d->name);
  pcap_freealldevs(alldevs);
  return 0;
}
```

### 6.3.1. Multi-stage Dockerfile

```dockerfile
# Build stage
FROM alpine:3.18 AS builder
RUN apk add --no-cache build-base libpcap-dev
COPY main.c .
RUN gcc -o interface-lister main.c -lpcap

# Runtime stage
FROM alpine:3.18
COPY --from=builder /interface-lister /usr/local/bin/interface-lister
ENTRYPOINT ["interface-lister"]
```

After building:

```bash
docker build -t interface-lister:0.1 .
docker image ls
```

Output:

```
REPOSITORY          TAG           IMAGE ID       SIZE
interface-lister    0.1           098dc03a096a   106MB
<none>              <none>        de7abdd44708   1.26GB
```

As seen, the final image is **ten times smaller** than the build stage image.

## 6.4. Building a Container Runtime from Scratch in Go

A great way to understand Docker internals is to build a minimal container engine yourself.
In [this tutorial by Liz Rice](https://www.youtube.com/watch?v=Utf-A4rODH8), you can follow how to implement a simple container runtime in Go.

No prior Go knowledge is required — the tutorial explains each step in detail, showing how Docker interacts with Linux kernel primitives.

## 6.5. From Simple Containers to Pods

This exercise demonstrates the path from **single containers** to **Kubernetes Pods**.

Kubernetes Pods are groups of containers that share some **namespaces and resources** (e.g., network, IPC).
Internally, Kubernetes uses a lightweight **pause container** to provide a shared environment.

### 6.5.1. Example

Start the pause container:

```bash
docker run -d --ipc=shareable --name pause -p 8080:80 \
    gcr.io/google_containers/pause-amd64:3.0
```

Then, run an **Nginx proxy** and a **Ghost blog** container, both sharing the pause container’s namespaces.

```nginx
events {}
http {
  server {
    listen 80;
    location / {
      proxy_pass http://localhost:2368;
    }
  }
}
```

### 6.5.2. Containers

```bash
docker run -d --name nginx -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
    --net=container:pause --ipc=container:pause nginx:1.23

docker run -d --name ghost -e NODE_ENV=development \
    --net=container:pause --ipc=container:pause ghost:5.22
```

Now visit [http://localhost:8080](http://localhost:8080) —
you should see the Ghost blog served through Nginx.

## 6.6. PIDs in Linux

In Linux, processes form a **tree structure**. Each process has a **Parent Process ID (PPID)**, except for `init` (PID 1).
Processes can spawn new ones via the `fork` syscall, and replace themselves via `exec`.

### 6.6.1. Zombie Processes

A **zombie** is a process that has terminated but still exists in the process table because its parent has not collected its exit status using the `wait` syscall.
Normally, the `init` process (PID 1) adopts or reaps orphaned processes to prevent PID exhaustion.

## 6.7. The Problem of Zombies in Containers

To demonstrate the zombie process issue, run two containers sharing a single **PID namespace**.

```bash
docker run -d --name sleep --ipc=shareable alpine:3.10 sleep 1d
docker run -d --name nginx --ipc=container:sleep --pid=container:sleep nginx:1.23
```

Then, inspect the processes:

```bash
docker exec -it sleep /bin/sh
ps -o pid,ppid,stat,args
```

You will see that:

* `sleep 1d` is PID 1 (the “init” process).
* The `nginx` master and worker processes share the same PID namespace.
* Killing the `nginx` master will leave zombie workers since `sleep` does not reap them.
