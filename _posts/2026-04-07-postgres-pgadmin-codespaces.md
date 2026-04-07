---
title: 'Spin Up PostgreSQL and pgAdmin in GitHub Codespaces'
date: 2026-04-07
collection: posts
permalink: /posts/2026/04/postgres-pgadmin-codespace/
toc: true
tags:
  - docker
  - postgres
  - github-codespaces
---
Setting up and running a database development environment shouldn't be a project in itself. GitHub Codespaces, with its pre-configured container support, turns this process from a tedious task into a single-click operation. In this post, we'll cover using GitHub Codespaces to define and run PostgreSQL and pgAdmin in isolated containers.

---

## What is GitHub Codespaces?

In this guide, we will run two containers in a GitHub codespaces environment with the following architecture.

![alt text](/images/posts/postgres-pgadmin-codespaces/1-container-architecture.png "Container Architecture")

However, before diving into the database setup, let’s quickly demystify the engine that makes it all possible: [GitHub Codespaces](https://docs.github.com/en/codespaces/about-codespaces/what-are-codespaces).

GitHub Codespaces is our instant development Sandbox. Codespaces acts like our personal, pre-configured development computer. We can access it instantly using our browser or VS Code Desktop. It eliminates the classic **"it works on my machine"** problem. Every developer gets an identical, cloud-hosted environment by using the *Configuration-as-Code* practice. Instead of manual setups, we define our project’s tools, runtimes, and dependencies (e.g., Python, PostgreSQL, pgAdmin) in simple configuration files that persist in our repository.

When we launch a Codespace from our repository, GitHub reads the configs, spins up a fresh Linux-based container with everything pre-installed, and presents us with a ready-to-code workspace. It’s a repeatable, consistent, and disposable development sandbox.

### Built on Dev Containers

These configured environments are powered by Development Containers (also known as dev containers), which are essentially Docker containers optimized for coding. We can start with a default container that includes common tools, or fully customize it by adding a `.devcontainer` folder to our project. This lets us craft the perfect environment, ensuring everyone works with the exact same versions of everything.

### How Do We Use It?

Getting started is straightforward:

1. **Create a Codespace**:
   We can launch one directly from any branch of our GitHub repository, from a template, or even using GitHub’s command-line tool (CLI).
2. **Work Anywhere**:
   Connect and code from our web browser, through the full-featured Visual Studio Code desktop application, or a JetBrains IDE.
3. **Powerful & Scalable**:
   With Codespaces running on virtual machines, we can scale to match our needs, from a standard 2-core machine for light work up to a powerful 32-core machine for heavy-duty tasks.

For personal use, GitHub Codespaces includes a monthly free tier with every GitHub account. We can experiment, build, and learn (like following this very guide!) without any initial cost or payment details. It’s the ultimate risk-free way to try new tools and streamline our workflow.

---

### Step 1: Launch the Cloud Development Environment

Before configuring our database, we need to set up GitHub Codespaces, which provides a cloud-based development environment with tools like Python and Docker pre-installed.

Here’s how to get started:

**1.1 Create a New Repository**

First, create a new repository to host our Codespace configuration:

![alt text](/images/posts/postgres-pgadmin-codespaces/2-create-repository.png "Create a Repository")

1. Go to our GitHub account and click the **"+"** icon in the top-right, then select **"New repository"**.
2. Configure it with these settings:
   * **Repository name:** `postgres-pgadmin-codespaces` (or a name of our choice)
   * **Description:** *Containerized PostgreSQL and pgAdmin for data ingestion.*
   * **Visibility:** Select **"Public"**.
   * **Initialize this repository with:** Check **"Add a README file"**.
   * **.gitignore template:** Select **"Python"**.
3. Click **"Create repository"**.

**1.2 Launch Our Codespace**

With our repository created, launching the cloud environment takes one click:

1. Navigate to the main page of our new repository.
2. Click the **"Code"** button (green button near the top), then select the **"Codespaces"** tab.
3. Click **"Create codespace on main"**.

![alt text](/images/posts/postgres-pgadmin-codespaces/3-create-codespace.png "Create Codespace")

GitHub will build our cloud-based container (running on a Linux virtual machine) and open a full VS Code-like editor directly in our browser. This may take a minute the first time.

---

### Step 2: Switch to VS Code Desktop (Optional)

While the browser experience is excellent, you might prefer working in your local **Visual Studio Code** application. The integration is seamless as we simply select `Open in VS Code Desktop` in the options to open a remote window after clicking the Codespaces button on the bottom left side.

![alt text](/images/posts/postgres-pgadmin-codespaces/4-vscode-codespace.png "Open Codespace in VS Code Desktop")

To have all future Codespaces open automatically in your desktop VS Code:

1. Go to your GitHub [**Personal Settings**](https://github.com/settings/profile).
2. In the left sidebar, click **"Codespaces"**.
3. Under **"Editor preference"**, select **"Visual Studio Code"**.

![alt text](/images/posts/postgres-pgadmin-codespaces/5-codespace-editor.png "Codespace Editor Preference")

---

### Step 3: Resuming Our Codespaces

You can reopen any of your active or stopped codespaces on GitHub or Visual Studio Code.

* **To see all your codespaces:** Visit [**github.com/codespaces**](https://github.com/codespaces).
* **To quickly resume the last-used codespace for a repo:** On any repository page, press the **comma key ","**. A "Resume codespace" prompt will appear.

![alt text](/images/posts/postgres-pgadmin-codespaces/6-resume-codespace.png "Resume Codespace")

* **To reconnect from VS Code Desktop:** Open the **Remote Explorer** sidebar (`Ctrl+Shift+P` / `Cmd+Shift+P` and search for it if not visible), find your codespace under "GitHub Codespaces", right-click it, and select **"Connect to Codespace"**.

---

### Step 4: Verify the Environment

Once your Codespace is running, open a new terminal in VS Code Desktop (`Terminal` > `New Terminal`) and run a quick check to confirm the core tools are ready:

```bash
python3 --version
docker --version
```

You should see output confirming that both Python and Docker are installed and available.

---

## Create .env File

We'll run PostgreSQL using configuration stored in a `.env` file with the following variables:

```bash
POSTGRES_USER=root
POSTGRES_PASSWORD=root
POSTGRES_DB=db_test
```

## Create a Docker Volume

First, let's create a named volume for persistent data storage.

```bash
docker volume create postgres_data
```

## Create a Docker Network

Next, we create a network to attach a container to another container's networking stack directly, using the `--network container:<name|id>` flag format.

```bash
docker network create postgres_network
```

## Run Postgres Container

In this section, we'll simply run the command to spin up a Postgres container. For a deeper dive into Docker containerization concepts and configuration details, please refer to my dedicated post about [Scaling PostgreSQL with Docker Containerization](https://tantikristanti.github.io/posts/2026/03/postgres-docker/).

```bash
docker run -it --rm \
  --env-file=.env \
  -v postgres_data:/var/lib/postgresql \
  -p 5432:5432 \
  --network=postgres_network \
  --name postgres_service \
  postgres:18
```

The parameters used for the container are as follows:

| Parameter                                     | What It Does                                                                               |
| --------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `-d`                                        | Runs container in detached mode (background)                                               |
| `-it`                                       | Runs container in interactive mode with terminal output                                    |
| `--rm`                                      | Auto-removes container when stopped                                                        |
| `--name postgres_container`                 | Gives container a human-readable name                                                      |
| `-e POSTGRES_USER=root`                     | Sets the database superuser                                                                |
| `-e POSTGRES_PASSWORD=root`                 | Sets the password for the superuser                                                        |
| `-e POSTGRES_DB=db_test`                    | Creates a default database on startup                                                      |
| `-v postgres_data:/var/lib/postgresql/data` | Persists data in a Docker volume                                                           |
| `-p 5432:5432`                              | Maps ports from container (on the right side of `:`) to host (on the left side of `:`) |
| `postgres:18`                               | Uses PostgreSQL version 18 image                                                           |

## Run pgAdmin Container

With a running PostgreSQL database, let's add a visual management interface with pgAdmin to complete our development toolkit. pgAdmin 4 is the most popular open-source GUI tool for managing PostgreSQL databases. It provides an intuitive visual interface for designing tables, running queries, and managing complex databases.

Open a new terminal and run the following command to spin up the pgAdmin container. It will connect to the Postgres server over the same network (`postgres_network`) by referencing the container name `postgres_service`.

```bash
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -v pgadmin_data:/var/lib/pgadmin \
  -p 8085:80 \
  --network=postgres_network \
  --name pgadmin \
  dpage/pgadmin4
```

The parameters for configuring pgAdmin are as follows:

| Parameter                                      | What It Does                                                                                                                                   |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `-it`                                        | Runs container in interactive mode with terminal output                                                                                        |
| `-e PGADMIN_DEFAULT_EMAIL="admin@admin.com"` | Sets the default login email for pgAdmin                                                                                                       |
| `-e PGADMIN_DEFAULT_PASSWORD="root"`         | Sets the default password for pgAdmin login                                                                                                    |
| `-v pgadmin_data:/var/lib/pgadmin`           | Persists pgAdmin configuration in a Docker volume                                                                                              |
| `-p 8085:80`                                 | Maps ports from container to host. It makes pgAdmin accessible at `http://localhost:8085` by mapping container's port 80 to host's port 8085 |
| `--network=postgres_network`                 | Connects container to a custom Docker network                                                                                                  |
| `--name pgadmin`                             | Gives container a human-readable name                                                                                                          |
| `dpage/pgadmin4`                             | Uses latest stable version of the official pgAdmin 4 Docker image                                                                              |

## Check the Running Containers

Verify that both containers, Postgres and pgAdmin, are running with the `docker ps` command. Open a new terminal to check:

```bash
docker ps
```

![alt text](/images/posts/postgres-pgadmin-codespaces/7-running-containers.png "Running Containers")

## Login to pgAdmin

Once both containers (Postgres and pgAdmin) are running, open pgAdmin at `http://localhost:8085`. Log in with the credentials specified when configuring the container:

- **Email:** admin@admin.com
- **Password:** root

### Create a Server

Create a new server by right-clicking **Server → Register → Server**. Fill in the form:

- **General tab:**
  - Name: `pgadmin`
- **Connection tab:**
  - Host: `postgres_service` (the Postgres container name)
  - Port: `5432`
  - Username: `root`
  - Password: `root`
- Click **Save**

![alt text](/images/posts/postgres-pgadmin-codespaces/8-pgadmin-settings.png "pgAdmin")

Once the server is connected, verify that the `db_test` database defined in the `.env` file has been created.

![alt text](/images/posts/postgres-pgadmin-codespaces/9-db-test.png "db-test")

## Orchestrating Services with Docker Compose

Manually running Docker commands is great for learning, but managing multiple containers becomes a chore. With a Docker compose, we can:

- **Launch the entire stack with one command** (`docker compose up`)
- **Automatically create a shared network** so our containers can communicate seamlessly.
- **Manage configuration centrally** in a version-controlled file.
- **Define the infrastructure as code** for easy replication and sharing

In this step, we'll consolidate our PostgreSQL and pgAdmin setup into a clean, reusable configuration file. Create a new file named `docker-compose.yaml` containing the following configuration to build the previously built containers automatically. Here, we don't need to define a Docker network because all containers built with Docker compose will be on the same network.

```bash
services:
  postgres_service:
    image: postgres:18
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=db_test
    volumes:
      - "postgres_data:/var/lib/postgresql:rw"
    ports:
      - "5432:5432"
  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=root
    volumes:
      - "pgadmin_data:/var/lib/pgadmin"
    ports:
      - "8085:80"

volumes:
  postgres_data:
  pgadmin_data:
```

### Compose Up

To run a Docker Compose file, simply run the `docker-compose up` command. We need to ensure that no containers are conflicting by stopping and shutting them down first.

```bash
docker stop $(docker ps)
docker-compose up
```

The first time we launch the stack with Docker Compose, we need to log in to pgAdmin and reconnect to the PostgreSQL server connection. The good news is, once we do this, Docker Compose will remember those settings. When we stop and restart the container, we only need to log back in because the server, database, and table configurations will be preserved.

### Compose Down

To stop Docker containers, simply run the `docker-compose down` command.

```bash
docker-compose down
```

## References

1. [GitHub Codespaces Documentation](https://docs.github.com/en/codespaces)
2. [Docker documentation](https://docs.docker.com/)
3. [pgAdmin](https://www.pgadmin.org/)
4. [DataTalksClub Data Engineering Zoomcamp Repository](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main)
