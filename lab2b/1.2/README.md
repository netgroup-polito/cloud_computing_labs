<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.1/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.3/README.md">Next ➡️</a></td>
  </tr>
</table>

# 2. Docker

## 2.1. Introduction

This section provides a **practical introduction to [Docker](https://www.docker.com/)** — a software platform that enables the easy creation of **lightweight**, **portable**, and **self-contained containers**.

Throughout this lab, we will use a **multi-container application** that simulates a typical real-world deployment. It consists of two main components:

* A **Web server** (`apache`) with the **PHP interpreter**
* A **Database server** (`mariadb`)

---

### 2.1.1. Docker in Brief

Unlike full virtualization, **containerization** leverages **OS-level virtualization** to isolate and limit the execution scope of an application.
In Linux, this isolation is achieved through **cgroups** and **namespaces**.
Instead of having access to the entire host system, an application inside a container only sees the resources allocated to that container.

First released in **2013**, **Docker Engine** is an open-source containerization technology for building and running applications within containers.
It has rapidly become the most popular engine for managing containers.

**Docker Engine provides:**

* **Container image format and build system:**
  Docker defines a standard image format and provides tools such as `Dockerfile` and `docker build` to create images.
  Each Docker image is a standalone package containing everything needed to run an application — code, runtime, system tools, and libraries.

* **Command-Line Interface (CLI):**
  The Docker CLI allows you to manage images (`docker images`, `docker rmi`, etc.) and container instances (`docker ps`, `docker rm`, etc.).
  The CLI acts as the client to a system-level **Docker daemon**, similar in architecture to **libvirt**.

* **Image distribution via registries:**
  Docker supports pushing and pulling images from remote repositories called **registries**.
  By default, the registry points to **[Docker Hub](https://hub.docker.com)**, but custom registries can be used (e.g., [port.us.org](https://port.us.org), [quay.io](https://quay.io)).

## 2.2. Docker Setup

It is assumed that **Docker Engine** is already installed on your system.
You can verify this with:

```bash
docker --version
```

If it is not installed, follow the official installation guide:
👉 [https://docs.docker.com/install/](https://docs.docker.com/install/)

Since a later section of the lab will use **Docker Compose**, verify its presence as well:

```bash
docker-compose -v
```

If needed, install it following the instructions here:
👉 [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)

> [!WARNING]
> By default, the `docker` command requires root privileges.
> If you wish to run Docker commands without `sudo`, add your user to the `docker` group as described in the official guide:
> [https://docs.docker.com/install/linux/linux-postinstall/](https://docs.docker.com/install/linux/linux-postinstall/)

## 2.3. Adjusting MTU of Containers

In some **virtualized environments**, the host network interface may have an **MTU smaller than 1500**.
In such cases, you must explicitly configure the MTU for Docker network interfaces to match the host configuration.

### 2.3.1. Check the Host MTU

Display all local network interfaces and their MTU values using:

```bash
ip link
```

Example output:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether uu:vv:ww:xx:yy:zz brd ff:ff:ff:ff:ff:ff
```

If the outgoing interface (e.g., `ens3`) has an MTU smaller than 1500, you need to configure Docker to ensure that the **virtual network interfaces of containers** use a matching MTU.

### 2.3.2. Configure Docker Daemon

Create or edit the file `/etc/docker/daemon.json` as follows:

```json
{
  "mtu": 1400
}
```

In this example, the MTU value `1400` corresponds to that of the host’s `ens3` interface.
After applying this configuration, **restart the Docker daemon** so the change takes effect for newly created containers.

```bash
sudo systemctl restart docker
```

> [!NOTE]
> Docker Compose creates a **new bridged network** for every Compose environment by default.
> Therefore, ensure that each new network respects the configured MTU setting.
