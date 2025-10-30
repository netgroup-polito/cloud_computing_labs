<table style="width:100%">
  <tr>
    <td align="left"><a href="../1.4/README.md">⬅️ Previous</a></td>
    <td align="right"><a href="../1.6/README.md">Next ➡️</a></td>
  </tr>
</table>

# 5. Using Docker Compose to Coordinate Multiple Containers

You have now successfully started a basic **LAMP stack** based on Dockerized components.

However, you might have noticed how cumbersome it quickly becomes to create multiple correlated containers, each with its own configuration parameters, when using bare `docker run` commands.
To simplify container lifecycle management, several **orchestrators** have been developed. Among the most popular are:

* [Docker Compose](https://docs.docker.com/compose/)
* [Docker Swarm](https://docs.docker.com/swarm/overview/)
* [Kubernetes](https://kubernetes.io/)

In this section, you will experiment with **Docker Compose**.

> **Official definition:**
> “Compose is a tool for defining and running multi-container Docker applications.
> With Compose, you use a YAML file to configure your application's services.
> Then, with a single command, you create and start all the services from your configuration.”

Unlike Swarm or Kubernetes, Docker Compose operates on a **single node**.
Swarm and Kubernetes, instead, are designed for **clusters** that can include thousands of nodes, offering advanced scaling and orchestration features.

## 5.1. Defining the LAMP Stack in `docker-compose.yml`

Create a new file called `docker-compose.yml` to replicate the same setup you manually created in the previous section.

```yaml
# Example docker-compose.yml (see snippet 3D-DockerCompose.sh)
version: "3.7"

services:
  apache:
    image: dockerlab-apache-php
    ports:
      - "8080:80"
    volumes:
      - ./docker-lab/apache-php/html:/var/www/localhost/htdocs:ro
    networks:
      - dockerlab-network

  mariadb:
    image: mariadb:10.7
    environment:
      MYSQL_DATABASE: database
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_USER: testuser
      MYSQL_PASSWORD: testpass
    volumes:
      - ./docker-lab/mariadb/database:/var/lib/mysql
      - ./docker-lab/mariadb/scripts:/docker-entrypoint-initdb.d:ro
    networks:
      - dockerlab-network

networks:
  dockerlab-network:
```

> [!WARNING]
> YAML files rely heavily on **indentation** for structure.
> Do *not* copy directly from a PDF file — it will likely break indentation.
> Always double-check your layout using a text editor such as `nano docker-compose.yml` or `vim docker-compose.yml`.

The file defines the containers (services) to be started, along with:

* The image or Dockerfile to use
* Volume mounts
* Port mappings
* Environment variables
* Networking configurations

For an exhaustive list of options, refer to the [Compose file reference](https://docs.docker.com/compose/compose-file/).

## 5.2. Running and Stopping the Stack

Once your `docker-compose.yml` file is ready, you can start and stop the full stack with just two commands:

```bash
# Start the containers defined in docker-compose.yml (in background)
docker-compose up -d

# Stop the running containers
docker-compose stop
```