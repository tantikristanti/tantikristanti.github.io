---
title: 'From Script to Container: Dockerizing Your Python Data Pipeline'
date: 2026-04-10
collection: posts
permalink: /posts/2026/04/dockerizing-structured-data-ingestion/
toc: true
tags:
  - dockerizing-data-pipeline
  - postgres
  - docker
  - github-codespaces
---
In data engineering, reproducibility and scalability are paramount. What begins as a simple Python script can quickly evolve into a critical pipeline component that needs to run consistently across environments. The gap between **"it works on my machine"** and **"it works everywhere"** is where containerization becomes indispensable.

In this guide, we'll transform a Python data ingestion application into a portable, scalable Docker image. Building on our previous setup with PostgreSQL and pgAdmin in Docker Compose, we'll now complete the picture by containerizing the data pipeline itself. This evolution from hybrid architecture to fully containerized ecosystem represents more than just a technical exercise; it's a fundamental shift toward reproducibility, scalability, and operational reliability in data workflows.

Whether you're managing pipelines for analytics, machine learning, or application data, mastering this transition will give you consistent data solutions. Let's get started and containerize our data ingestion pipeline, creating a self-contained unit that can run on your laptop, in a cloud environment, or anywhere Docker can go.

---

## Prerequisites and Prior Setup

To follow along with this guide, you'll need a few tools and resources ready. If you're jumping in here without having completed the earlier steps, don't worry, the previous guide will get you up to speed quickly.

### What You'll Need

- *Docker* [[1]] installed and running on your system.
- *A working PostgreSQL [[2]] and pgAdmin [[3]] environment* already set up from [Mastering Data Pipelines with Postgres &amp; pgAdmin in GitHub Codespaces](https://tantikristanti.github.io/posts/2026/04/structured-data-ingestion-postgres-pgadmin/).
- *The data ingestion script*, available here: [`data-ingestion.py`](https://github.com/tantikristanti/postgres-pgadmin-codespaces/blob/main/data-ingestion.py).

> ⭐️ **Note:** If you haven't yet configured your database environment, please start with the [previous tutorial](https://tantikristanti.github.io/posts/2026/04/structured-data-ingestion-postgres-pgadmin/) before continuing here. It provides the essential foundation we’ll build upon in this guide.

### Quick Start: Running the Postgres and pgAdmin Containers

With everything installed, start your database environment with a single command:

```bash
docker-compose up
```

If you need to run in detached mode:

```bash
docker-compose up -d
```

### Login to pgAdmin

Once both containers are running, you can access the [pgAdmin](https://www.pgadmin.org/) web interface by navigating to **[http://localhost:8085](http://localhost:8085)** in your browser.

![alt text](/images/posts/2026-04-10-dockerizing-data-pipeline/1-pgadmin.png "pgAdmin")

On your first login, you'll need to create a server connection to link pgAdmin with your [PostgreSQL](https://www.postgresql.org/) instance. Use the database credentials stored in your `.env` file for this setup.

> 🔄 **Good news:** Because we configured the pgAdmin container using a Docker volume to persist its data, this server configuration will be saved automatically. On subsequent visits, you won’t need to set up the connection again, just log in and start managing your database.

---

## Understanding Our Docker Compose

Let's take a closer look at our Docker Compose configuration, a setup designed to orchestrate two essential database services and prepare the groundwork for our upcoming data ingestion pipeline. This configuration establishes two interconnected services:

- A *PostgreSQL* database container named `postgres`.
- A *pgAdmin* web management interface container named `pgadmin`.

![alt text](/images/posts/2026-04-10-dockerizing-data-pipeline/2-docker-compose.png "Docker compose")

### State Persistence with Docker Volumes

Both services use Docker volumes (`postgres_data` and `pgadmin_data`) to preserve state across container sessions. This means:

- Database tables and records remain intact.
- pgAdmin configurations and server connections are saved.
- You can restart, rebuild, or update containers without losing data.

### Network Strategy: Explicit Naming

Although Docker Compose automatically creates a shared network for all services within a configuration, we’ve explicitly named ours `postgres_network`. Here’s why:

- *Default naming* (`[project]_default`) works but can be less intuitive, especially when connecting additional containers from other projects.
- *Explicit naming* improves clarity and makes it straightforward for future services (such as our data ingestion app) to join the same network.
- *Service discovery* becomes more intentional, any container on `postgres_network` can locate the PostgreSQL database simply by referencing the service name `postgres`.

You can view and inspect your Docker networks using the following commands:

```bash
# List all Docker networks
docker network ls

# Inspect a specific network
docker network inspect [network]
```

---

## Evolving Our Architecture: A Fully Containerized Ecosystem

We're evolving our system design from a hybrid approach to a *unified container architecture*, where every component (PostgreSQL, pgAdmin, and our data ingestion application) resides within the same Docker network [[4]]. This transition from architecture **A** to **B** represents a significant shift in how our services communicate and interact.

In our [previous guide](https://tantikristanti.github.io/posts/2026/04/structured-data-ingestion-postgres-pgadmin/), the data ingestion application ran on the host machine and connected to PostgreSQL via `localhost:5432`. This worked because we mapped the container's port 5432 to the host's port 5432.

However, once we containerize the data ingestion application, `localhost` within that container refers to its own isolated network interface, not the PostgreSQL service running in another container. This architectural shift requires a configuration change:

**Update your `.env` file:**

```env
# Before (host-based access):
POSTGRES_HOST=localhost

# After (container-based access):
POSTGRES_HOST=postgres
```

![Architecture Evolution](/images/posts/2026-04-10-dockerizing-data-pipeline/3-system-architecture.png "Transitioning from hybrid to fully containerized architecture")

This simple change, from `localhost` to `postgres`, allows containers to find each other by name within the shared network `postgres_network`.

---

## Containerizing the Data Ingestion Script

### Creating the Dockerfile

Our `Dockerfile` uses a multi-stage build pattern for efficiency and reproducibility:

```bash
# Base Docker image that we will build on. 
# Start with slim Python 3.13 image for smaller size
FROM python:3.13.11-slim

# Copy uv binary from official uv image(multi-stage build pattern)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/

# Set working directory inside container
WORKDIR /app

# Add virtual environment in the working directory to PATH
ENV PATH="/app/.venv/bin:$PATH"

# Copy dependency files first for better layer caching to the container current working directory (/app)
COPY "pyproject.toml" "uv.lock" ".python-version" ./

# Install all dependencies by synchironizing with uv.locked to ensure reproducible dependency installation
RUN uv sync --locked

# Copy the application code
COPY data-ingestion.py . 

# Set entry point to run the application
ENTRYPOINT [ "python", "data-ingestion.py" ]
```

### Building the Docker Image

```bash
# Build the image with a name and tag
docker build -t nyc-taxi-data-ingest:v001 .

# Verify the created image 
docker images | grep nyc-taxi-data-ingest
```

### Running the Data Ingestion Container

To automatically load data from the internet into our Postgres database, we run our data ingestion container with a custom configuration for connectivity and data processing. The command aims to:

1. Start a temporary container that connects to the same network as our PostgreSQL database.
2. Load database credentials securely from an environment file.
3. Download and process NYC taxi trip data from a remote repository.
4. Ingest the data into PostgreSQL in manageable chunks to prevent memory issues.
5. Clean up after itself by removing the container when finished.

```bash
docker run -it --rm --env-file=.env-docker \
--network=postgres-pgadmin-codespaces_postgres_network \
nyc-taxi-data-ingest:v001 \
--url "https://github.com/tantikristanti/Datasets/releases/download/v1.0.0-yellow-alpha/yellow_tripdata_2026-02.csv.gz" \
--target_table yellow_taxi_02_2026 \
--chunksize=50000
```

**Container lifecycle and interaction**

- `docker run -it --rm`:

  Creates and runs a container interactively (`-it`) and automatically removes it when it stops (`--rm`), keeping our environment clean.

**Configuration and networking**

- `--env-file=.env-docker`:

  Loads environment variables from `.env-docker` can be accessed [here](https://github.com/tantikristanti/postgres-pgadmin-codespaces/blob/main/.env-docker), which contains database credentials and connection settings.
- `--network=postgres-pgadmin-codespaces_postgres_network`:

  Connects the container to the existing Docker network where PostgreSQL and pgAdmin are running, enabling communication between services.

**Container image and application**

- `nyc-taxi-data-ingest:v001`:

  Uses our custom Docker image tagged `v001`, which contains the data ingestion [Python](https://www.python.org/) script and its dependencies.

**Data processing parameters**

- `--url "https://github.com/.../yellow_tripdata_2026-02.csv.gz"`:

  Specifies the source data file, a compressed CSV of NYC yellow taxi trips from February 2026.
- `--target_table yellow_taxi_02_2026`:

  Defines the PostgreSQL table where the data will be loaded.
- `--chunksize=50000`:
  Processes the data in batches of 50,000 rows at a time, optimizing memory usage during ingestion.

### Verifying the Built Table

Verify our table generated by the data pipeline container with pgAdmin. Run a simple SQL command, for example, to check how many rows are stored in the table.

![alt text](/images/posts/2026-04-10-dockerizing-data-pipeline/4-pgadmin-yellow_taxi_02_2026.png "yellow_taxi_02_2026 Table")

---

## Push to GitHub Repository

Push our work to the GitHub repository.

```bash
git add .
git commit -m "dockerizing data ingestion scripts"
git push
```

---

## Conclusion

We've completed a transformative journey from local script to portable container, successfully dockerizing a complete data ingestion pipeline.

Our exploration covered:

- **Understanding architectural evolution** from hybrid to fully containerized systems.
- **Configuring container networking** for seamless inter-service communication.
- **Building optimized Docker images** with multi-stage builds and dependency management.
- **Orchestrating containerized data ingestion** with proper environment configuration.
- **Implementing efficient data processing** with chunked loading for large datasets.

---

## GitHub Repository

Docker configuration and codes are available in my [GitHub repository](https://github.com/tantikristanti/postgres-pgadmin-codespaces).

---

## References

1. Docker Inc. (2026). **Docker Docs**.
2. The PostgreSQL Global Development Group. (2026). **PostgreSQL: The World&#39;s Most Advanced Open Source Relational Database**.
3. pgAdmin Development Team. (2026). **pgAdmin**.
4. DataTalksClub. (2026). **DataTalksClub Data Engineering Zoomcamp Repository: Docker-SQL**.

---

[1]: https://docs.docker.com/
[2]: https://www.postgresql.org/
[3]: https://www.pgadmin.org/
[4]: https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/01-docker-terraform/docker-sql