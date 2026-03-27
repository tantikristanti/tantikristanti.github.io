---
title: 'Dockerizing PostgreSQL'
date: 2026-03-26
collection: posts
permalink: /posts/2026/03/postgres-docker/
toc: true
tags:
  - docker
  - postgres
---
Docker has become a widely adopted tool for developers due to its capacity to standardize development environments, streamline resource management, accelerate deployment workflows, and ensure application isolation through containerization. By packaging applications into standardized, self-sufficient units called containers, Docker ensures all necessary components, such as code, libraries, tools, and runtime environments, are included, enabling seamless execution across diverse environments. This guide walks through the process of dockerizing PostgreSQL. PostgreSQL is an open-source object-relational database system that builds upon SQL functionality while offering advanced features to securely manage and scale complex data operations.

## Install and Run Docker Desktop

Installing Docker is a straightforward process. The official [Docker Desktop](https://docs.docker.com/get-started/introduction/get-docker-desktop/) documentation provides step-by-step instructions for installing it on various operating systems, including macOS, Windows, and Linux.

After installation, verify that Docker is working properly by running the version or info command.

```bash
docker version
docker info
```

## Pull Docker Images from Docker Hub

There are different methods to pull Docker images: using the Docker Desktop, CLI, and API.

### Docker Desktop

While offering an easy-to-use development environment, downloading images with Docker Desktop is also convenient.

1. Launch Docker Desktop and navigate to the **Images** section.
2. Use the search bar to locate the desired image.
3. Select the image and click **Pull** to initiate the download.

![alt text](/images/posts/postgres-docker/1-docker-pull.png "Pull Postgres image")

Additionally, we can pull the latest version of an image from Docker Hub directly within the Images view by selecting the image, clicking the **More options** button, and choosing **Pull**.

### Command-Line Interface (CLI)

To pull the [Postgres image](https://hub.docker.com/_/postgres) from the Docker Hub in the terminal, run the following command:

If no tag is provided, Docker Engine uses the `:latest` tag as a default.

```bash
docker pull postgres 
```

This action downloads the image and its associated layers to our local system.
We can explicitly specify the version  by using the `:tag`.

```bash
docker pull postgres:14.22-trixie 
```

### API

Docker images can also be pulled using the API for integration into various programming languages.

```python
import docker  
client = docker.from_env()  
image = client.images.pull("postgres")  
print(f"ID: {image.id}, Tags: {image.tags}")  
```

### List the Images

> To list the images that are stored locally, use the docker images command.

```bash
docker images
```

## Run Docker Containers

To launch and manage images within isolated containers, we use the `docker run` command by specifying various arguments.
To see a list of arguments, including the use of environment variables and how the container should be run, we can use the help command.

```bash
docker run --help 
```

The following command runs the `postgres` image as a container named *postgres-docker* by setting the environment variables (`-e`) with the username and password to connect to Postgres. The container runs in the background in detached mode (`-d`). We will expose the Postgres server on our host system on port 5431 by mapping it from port 5432.

```bash
docker run --name posgtres-docker \
-e POSTGRES_USER=postgres \
-e POSTGRES_PASSWORD=postgres \
-p 5431:5432  \
-d postgres
```

Some of the details we specify are:

- The name of the Postgres container can be anything.
- The username and password can also be anything, but by default, here's a list we can use as a reference.

| Username | Password |
| :------: | :------: |
| postgres | postgres |
| postgres | password |
| postgres |  admin  |
|  admin  |  admin  |
|  admin  | password |

- By default, Postgres runs on port 5432, but here we'll map it to port 5431 on our host system. We can create multiple Postgres services running concurrently on different ports by ensuring that the container names are not the same. For example, we can run the Postgres services on ports 5433, 5434, and so on by mapping the ports using this syntax:
  `-p [host_port] : [container_port]`.

```bash
docker run --name posgtres-docker-1 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=secret -p 5433:5432  -d postgres

docker run --name posgtres-docker-2 -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=admin -p 5434:5432  -d postgres
```

### List the Containers

To list the running containers, use the `docker ps` or `docker container ls` command.

```bash
docker ps
```

Or

```bash
docker container ls
```

![alt text](/images/posts/postgres-docker/2-docker-ps.png "Containers")

To display all containers, including those currently running and those that have stopped, add the all `-a` argument.

```bash
docker ps -a
```

### Stop the Containers

To stop the running containers, use the `docker stop` command.

```bash
docker stop posgtres-docker-1
docker stop posgtres-docker-2
```

### Remove the Containers

To remove the running containers, use the `docker rm` command.

```bash
docker rm posgtres-docker-1
docker rm posgtres-docker-2
```

## Create .env File

Up to this point, we have successfully run the Postgres service. However, when creating a container, we have to enter environment variables one by one on the command line, which is impractical if we have many environment variables to set. Therefore, we create a separate file specifically for these environment variables, named `.env`.

To create this file in the terminal, we can use any text editor we like (e.g., VIM, Nano).

```bash
vi .env 
```

- Type `i` to switch from escape mode to insert mode.
- Enter the following variables:

```bash
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

- Esc and save the file `:wq!`

We need to ensure that the `.env` file is added to `.gitignore` so that it won't be committed by the version control system when pushing the project to the GitHub repository.

Once the file is created, we can configure the container using this `.env` file by specifying it in the `--env-file` argument.

```bash
docker run --name posgtres-docker-3 --env-file=.env -p 5435:5432  -d postgres
```

## Create Docker Compose Yaml File

Running containers manually in the terminal can be cumbersome, especially as the number of containers to be handled increases. To address this issue, we can create a `docker-compose.yml` file containing the configuration for running containers. This way, whenever we need to start a container, we simply run `compose up`, and conversely, when we need to stop it, we run `compose down` using the YAML file.

Let's create the `docker-compose.yml` file using any text editor we like (e.g., VIM, Nano). The YAML configuration file contains the following configuration:

```bash
services:
  postgres:
    image: postgres
    container_name: postgres-docker-4
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5436:5432" 
```

### Run the Service

Run `compose up` on the path where the Yaml file is located to run the service in the background or in detached `-d` mode.

```bash
docker compose up -d
```

### Stop the Service

Run `compose down` to stop the service.

```bash
docker compose down
```

## Connect to the Postgres Server

There are several ways to connect to a running Postgres server, including using the terminal, pgAdmin, and connecting it as part of our programming code. In this guide, we'll connect to the server using the terminal and pgAdmin.

### Exec Into the Posgtres Container

One way to ensure the PostgreSQL server is working properly is to run a command inside the running container using `docker exec` in interactive mode `-it` by specifying the command itself (e.g., psql) and the username `-U` to access the server. `psql` is an interface we can access through the terminal to interact with the Postgres database.

```bash
docker exec -it posgtres-docker psql -U postgres
```

### Connect to Postgres Server using pgAdmin

To use pgAdmin, we first need to install the tool according to our operating system from the [pgAdmin](https://www.pgadmin.org/download/) website.
Then, we create a new server connection and configure it with the container name, username, and password specified during the container configuration.

![alt text](/images/posts/postgres-docker/3-create-server-connection.png "Postgres Server")

## Create Database and Tables

We will test our Postgres server by creating a database and table in the container terminal and check whether the results are synchronized in pgAdmin.

![alt text](/images/posts/postgres-docker/4-create-database-table.png "Database and Table")

## Create Persistent Volume

When a container is destroyed, the data will also be lost. To prevent this, we can store the data in a persistent volume by specifying the data location in the source or host machine `postgres-data` and the target or container `/var/lib/postgresql/data`. Therefore, the data will be mounted from the source to the target when a new container that specifies the volume is started. Volumes are persistent data stores for a container.

### Create a Volume

First, create a volume with any name, for example, `postgres-data`, using the `volume` command.

```bash
docker volume create postgres-data
```

Then, configure the container with the volume by mapping the source to the target volume. By default, Docker will store data under `/var/lib/postgresql`. The command `-v [host_volume_location]:[container_data_location]` means to mount the volume on the host to the container data location.

```bash
docker run --name postgres-docker-4 --env-file=.env -p 5436:5432 -v postgres-data:/var/lib/postgresql -d postgres
```

### Test the Persistent Volume

We'll test the volume by first creating a new database and table.

```bash
docker exec -it postgres-docker-4 psql -U postgres
```

```sql
CREATE DATABASE enterprise_b;
SELECT datname FROM pg_database;
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name VARCHAR,
  position VARCHAR
);
```

List the table using `\dt` and quit using `\q`.

Then, we stop and delete the container.

```bash
docker stop postgres-docker-4
docker rm postgres-docker-4
```

Last, we recreate the container with the same configuration as the deleted one and check if the data persists.

```bash
docker run --name postgres-docker-4 --env-file=.env -p 5436:5432 -v postgres-data:/var/lib/postgresql -d postgres
```

![alt text](/images/posts/postgres-docker/5-volume-persist.png "Volume")

### Configure the Volume in the Docker Compose File

To add the volume in the `docker-compose.yml` file, we need to specify the source and target under the `volumes` and refer to it in the configuration.

```bash
services:
  postgres:
    image: postgres
    container_name: postgres-docker-4
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5436:5432" 
    volumes:
      - postgres-data:/var/lib/postgresql
volumes:
  postgres-data:
```

## References

1. [Docker documentation](https://docs.docker.com/)
2. [pgAdmin](https://www.pgadmin.org/)
