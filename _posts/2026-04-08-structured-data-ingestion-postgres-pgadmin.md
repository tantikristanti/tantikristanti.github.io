---
title: 'Mastering Data Pipelines with Postgres & pgAdmin in GitHub Codespaces'
date: 2026-04-08
collection: posts
permalink: /posts/2026/04/structured-data-ingestion-postgres-pgadmin/
toc: true
tags:
  - data-ingestion
  - postgres
  - docker
  - github-codespaces
---
Building a reliable data pipeline shouldn't start with complex environment setups, dependency conflicts, and inconsistencies. What if we could eliminate these obstacles and focus solely on what matters: designing efficient data flows and extracting value?

In this guide, we’ll construct a powerful, reproducible data ingestion environment using *PostgreSQL*, managed through the intuitive *pgAdmin* interface, all running seamlessly inside a *GitHub Codespace*. This approach eliminates the **"it works on my machine"** problem and turns our data project setup from a chore into a one-click process. We’ll walk through how to structure the data, provision a dedicated database, and leverage the cloud-native Codespaces environment to create a portable, shareable, and efficient foundation for any data-driven application.

---

## Prerequisites

Before starting this data ingestion guide, ensure you have:

### 1. GitHub Account & Codespace Access

- An active **GitHub account**.
- Basic familiarity with **VS Code** (either Desktop or browser version).
- Understanding that **GitHub Codespaces** provides cloud-hosted development environments.

### 2. Environment Setup (Already Configured)

This guide assumes you've already completed the foundational setup from our companion tutorial. Otherwise, please complete it first using these setup steps: **[Spin Up PostgreSQL and pgAdmin in GitHub Codespaces](https://tantikristanti.github.io/posts/2026/04/postgres-pgadmin-codespace/)**.

**What the setup provides:**

- A running GitHub Codespace [[1]].
- Docker [[2]] containers for PostgreSQL [[3]] and pgAdmin [[4]].
- pgAdmin configured with server connection.
- Configured `.env` file with credentials. We will update the file slightly to change the database name as follows:

```
# PostgreSQL environment variables for Docker Compose
POSTGRES_USER=root
POSTGRES_PASSWORD=root
POSTGRES_DB=ny_taxi_db
POSTGRES_HOST=localhost # Postgres service running on host machine
POSTGRES_PORT=5432 # Change this port if the port `5432` is occupied, such as `5431`.

# pgAdmin environment variables for Docker Compose
PGADMIN_DEFAULT_EMAIL=admin@admin.com
PGADMIN_DEFAULT_PASSWORD=root
```

### 3. Verification Steps

Open your existing Codespace and verify everything is ready:

**Step A: Check Core Tools**

In your Codespace terminal:

```bash
python3 --version
docker --version
docker compose version
```

**Step B: Verify Running Services**

```bash
docker ps
```

You should see both `postgres` and `pgadmin` containers with status **"Up"**.

![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/1-postgres-pgadmin-up.png "Running infrastructures")

**Step C: Confirm pgAdmin Access**

- Open `http://localhost:8085` in your Codespace browser.
- Login with credentials from your `.env` file.

![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/2-pgadmin-server.png "pgAdmin")

If any step fails, **return to [the companion tutorial](https://tantikristanti.github.io/posts/2026/04/postgres-pgadmin-codespace/)** to fix your setup before proceeding.

## About This Guide's Structure

Now that your environment is ready, this guide will shift focus to **building the data pipeline**. The following architecture outlines the general flow we will implement [[5]]:

![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/3-system-architecture.png "System Architecture")

The system architecture consists of three core services running in containers:

* **Postgres:** The primary relational database for structured data storage.
* **pgAdmin:** A web-based interface for database administration and analysis, enabling direct SQL query execution and exploration of the ingested data.
* **Data Ingestion Service:** A custom application that automatically retrieves data from a specified GitHub repository and incrementally loads it into the Postgres database.

---

## Datasets

For this demonstration, we will use the public [NYC Taxi &amp; Limousine Commission (TLC) Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page), a rich, real-world dataset published monthly. It contains millions of trip records from yellow and green taxis, with details on timestamps, geographic coordinates, trip distances, fares, and passenger demographics.

Specifically, we will use certain data releases:

* **Yellow trip data (2024-2026):** [Download Link](https://github.com/tantikristanti/Datasets/releases/tag/taxi-yellow)
* **Green trip data (2024-2026):** [Download Link](https://github.com/tantikristanti/Datasets/releases/tag/taxi-green)
* **Taxi Zone Lookup Data:** [Download Link](https://github.com/tantikristanti/Datasets/releases/tag/taxi-zones)

While the original data is stored in PARQUET format, for this project it has been converted to CSV and GZ-compressed. Details on the data dictionary, taxi zones, and schedule are available on the official TLC website.

---

## Create a Virtual Environment with UV

Virtual environments create isolated, project-specific spaces for Python [[6]] dependencies. This ensures each project has precisely the packages and versions it needs, preventing conflicts. Traditional package managers can be slow, especially when resolving complex dependencies. That's where uv [[7]], an extremely fast Python package manager written in Rust, comes in.

While our initial PostgreSQL and pgAdmin setup might not require many Python packages, establishing a virtual environment from the start creates a **clean, reproducible foundation**. This prepares a conflict-free space for any future Python tools or applications that interact with our database.

### Step-by-Step: Creating a Virtual Environment

Follow these steps to set up an isolated environment with UV.

**Install UV**

If `uv` is not already installed, you can install it using one of the following methods:

* *Via the install script:*

```bash
  # Download and install uv
  curl -LsSf https://astral.sh/uv/install.sh | sh
```

* *Via pip:*

```bash
pip install uv
```

After installation, verify uv on our device by running the following command:

```bash
  uv --version
```

**Initialize a Project**

Initialize a project with `uv` to generate the essential configuration files: `.python-version`, `pyproject.toml`,`uv.lock` and `main.py`.

```bash
  uv init --python=3.13
```

- *`.python-version`* → The Python Guarantee

  This file specifies exactly which Python version our project requires. Tools such as `uv` and version managers such as `pyenv` read this file to automatically switch to the correct Python version when working on our project.
- *`pyproject.toml`* → The Project Blueprint

  This is the standard for Python project configuration replacing `requirements.txt` and `setup.py`.
- *`uv.lock`* → The Dependency Snapshot

  This is an automatically generated lock file, similar to `package-lock.json` in Node.js or `Pipfile.lock` in Pipenv. It records the exact versions of every package and sub-dependency installed in our project at a specific point in time.
- *`main.py`* → The Starting Point

  For this guide, we won't use it.

**Verify Python Versions**

   Let's run the following command to display the Python version on both the host and virtual environment. We can see that both environments have different Python versions.

```bash
  which python 
  python -V
  uv run which python
  uv run python -V
```

The `uv run python -V` command will also create a `.venv` folder containing a copy of the Python interpreter.

![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/4-python-versions.png "Python versions")

### Activate the Virtual Environment

Activate the environment:

```bash
   source .venv/bin/activate
```

### Add Dependencies

With our virtual environment active, we can install the Python packages needed for this project. Run the following command to install several dependencies, including pandas, sqlalchemy, psycopg2-binary, and python-dotenv:

```bash
uv add pandas requests sqlalchemy psycopg2-binary python-dotenv click tqdm
```

After installation, we can verify the installed packages with:

```bash
uv pip list
```

### Quick Environment Management Cheatsheet

| Task               | Command                       |
| ------------------ | ----------------------------- |
| Create environment | `uv venv .venv`             |
| Activate           | `source .venv/bin/activate` |
| Install package    | `uv add package-name`       |
| View installed     | `uv pip list`               |
| Deactivate         | `deactivate`                |
| Remove environment | `rm -rf .venv`              |

> ⚠️ `rm -rf .venv` action is irreversible.

---

## Building the Data Ingestion Pipeline

Data ingestion is the process of moving data from a source system into a target storage, often transforming it along the way. In this project, our goal is to ingest the NYC Taxi dataset into our PostgreSQL database.

We'll implement this using a Python script run within our isolated virtual environment. The process will involve:

1. **Downloading** a sample CSV file from a GitHub repository hosting the NYC Taxi data.
2. **Reading and Processing** the file efficiently in chunks using `pandas`.
3. **Transforming** data types, such as converting string timestamps to proper datetime objects.
4. **Loading** the clean data into the corresponding tables in our running PostgreSQL container using **SQLAlchemy**.

This ingestion service runs alongside our database (`Postgres`) and admin interface (`pgAdmin`), enabling us to automatically populate the database with real-world data for analysis.

This section will break down the code used to ingest the data.

### Run Jupyter Notebook

To understand our dataset before building the full pipeline, we can explore it interactively using a Jupyter Notebook.
Since this is for development and exploration only, we'll first install Jupyter in development mode.

```bash
uv add --dev jupyter
```

Then, launch the Jupyter notebook server:

```bash
uv run jupyter notebook
```

![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/5-jupyter-notebook.png "Jupyter Notebook")

Once running, the server will provide a local URL (`http://localhost:8888`) and a security token. Open this URL in your browser and enter the token when prompted to access the notebook interface.

### Step by Step Data Ingestion Process

Let's start building the data ingestion pipeline.

**Step 1: Create and Set Up the Notebook**

We begin by creating a new Jupyter Notebook file, for example, `data-ingestion.ipynb`. Inside this notebook, we first import the necessary Python libraries, including `pandas` for data manipulation and `sqlalchemy` for database interaction.

![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/6-import-libraries.png "Import libraries")

**Step 2: Acquire and Load the Source Data**

We'll source our data from this [`yellow_tripdata_2026-01.csv.gz`](https://github.com/tantikristanti/Datasets/releases/download/taxi-yellow/yellow_tripdata_2026-01.csv.gz). You can select any file you want by browsing the list of files in these [releases](https://github.com/tantikristanti/Datasets/releases/tag/taxi-yellow).

When loading the CSV with `pandas`, we may encounter a `DtypeWarning`. This is a common issue where pandas infers inconsistent data types.
![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/7-download-data-1.png "Load Data 1")

**Step 3: Define the Data Schema**

To resolve the warnings and ensure data integrity, we explicitly define the data type for each column when reading the file. This is especially important for datetime columns such as `tpep_pickup_datetime`.
![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/8-data-type-conversion.png "Data Type Conversion")

**Step 4: Reload data with specified schema**

Reloading the data now confirms we are working with a substantial dataset, over 3.7 million yellow taxi trips for January 2026.
![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/7-download-data-2.png "Load Data 2")

Let's examine the first few rows to understand the dataset's structure and the nature of its columns.
![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/9-first-rows-data.png "First rows data")

**Step 5: Implement Batch Processing**

Ingesting millions of rows at once can strain memory and cause process failures. The recommended approach is to use **batch processing**. By reading the CSV in manageable chunks (e.g., 100,000 rows at a time), we control memory usage and make the process more resilient.
![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/10-data-iteration.png "Data in batches")

**Step 6: Establish the Database Connection**

We use SQLAlchemy to create a connection engine to our running PostgreSQL container. This engine handles connection pooling and translates our Python operations into SQL.
![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/11-database-connection.png "Database connection")

**Step 7: Ingest Data into PostgreSQL**

For each batch of data, we use the `to_sql` method to efficiently insert the rows into a new table (`yellow_taxi_data_01_2026`). We use the `if_exists='append'` parameter to add each chunk to the same table.

![alt text](/images/posts/2026-04-08-structured-data-ingestion-postgres-pgadmin/12-data-ingestion.png "Data ingestion")

> You can find the complete notebook for this exercise in the [project repository](https://github.com/tantikristanti/postgres-pgadmin-codespaces/blob/main/data-ingestion.ipynb).

**Step 8: Verify the Ingestion in pgAdmin**

Finally, we verify the success of our pipeline by querying the database. Connecting to the PostgreSQL instance via pgAdmin and running a simple count query confirms that our data has been transferred correctly.

```sql
select count(*) from yellow_taxi_data_01_2026
```

## From Notebook to Production Script

While Jupyter Notebooks are excellent for exploration and prototyping, a robust data pipeline requires reproducible, executable scripts. We'll now convert our interactive notebook into a production-ready Python script and enhance it with command-line flexibility.

**Step 1: Convert the Notebook to a Python Script**

We'll use Jupyter's `nbconvert` tool to extract the executable Python code from our notebook:

```bash
uv run jupyter nbconvert --to=script data-ingestion.ipynb
```

This generates a `data-ingestion.py` file containing all the code from our notebook, ready to be run as a standalone script.

**Step 2: Add some dependencies**

Before enhancing our script, we need to install additional Python packages that will provide command-line interface functionality, progress tracking, and environment management, respectively:

```bash
uv add click tqdm python-dotenv
```

**Step 3: Add Command-Line Interface with Click**

To make our ingestion script more flexible and reusable, we'll adapt it to accept user-defined parameters using Click [[8]], a Python package for creating command-line interfaces. We'll modify the script to accept three key arguments:

1. **Source URL:** The URL where the CSVs reside.
2. **Chunk Size:** The number of rows to process in each batch.
3. **Table Name:** The target PostgreSQL table name for the ingested data.

```python
@click.command()
@click.option('--target_table', default='yellow_taxi_data_01_2026', help='Target database table name')
@click.option('--url', required=True, help='URL or path to the data file (CSV or Parquet)')
@click.option('--chunksize', default=100000, type=int, help='Chunk size (rows per batch)')
def main(target_table, url, chunksize):
    pg_user = os.getenv('POSTGRES_USER')
    pg_pass = os.getenv('POSTGRES_PASSWORD')
    pg_host = os.getenv('POSTGRES_HOST')
    pg_port = os.getenv('POSTGRES_PORT')
    pg_db = os.getenv('POSTGRES_DB')

    logging.info('Starting taxi trip data ingestion...')
    # Create Database Connection with fast executemany enabled
    connection_engine = create_engine(f'postgresql://{pg_user}:{pg_pass}@{pg_host}:{pg_port}/ny_taxi_db')
  
    # Read the data in batches
    ingest_data(
        url=url,
        engine=connection_engine,
        target_table=target_table,
        chunksize=chunksize
    )

if __name__ == "__main__":
    main()
```

**Step 4: Verify the Script Interface**

To view the available command-line options and their descriptions, use the built-in `--help` flag:

```bash
uv run data-ingestion.py --help
```

**Step 5: Run the Script with Custom Parameters**

Now you can execute the script with your own parameters. For example, to ingest a different dataset with a smaller chunk size:

```bash
uv run data-ingestion.py \
    --url "https://github.com/tantikristanti/Datasets/releases/download/taxi-yellow/yellow_tripdata_2026-01.csv.gz" \
    --target_table yellow_taxi_01_2026 \
    --chunksize 50000
```

This command will download the specified file, process it in batches of 50,000 rows, and store the results in a table named `yellow_taxi_01_2026`.

> **💡 Complete Code Access**
> The full, production-ready script—including the `ingest_data()` function, error handling, and logging, is available in the [project repository](https://github.com/tantikristanti/postgres-pgadmin-codespaces).

---

## Push to GitHub Repository

Push our work to the GitHub repository.

```bash
git add .
git commit -m "add data ingestion scripts"
git push
```

---

## Conclusion

We've successfully built a complete data ingestion pipeline using modern, reproducible practices. Our journey covered:

- **Environment Setup:** Leveraged GitHub Codespaces to eliminate "works on my machine" issues.
- **Containerized Infrastructure:** Used Docker Compose to orchestrate PostgreSQL and pgAdmin.
- **Isolated Development:** Created a virtual environment with `uv` for fast, conflict-free dependency management.
- **Data Exploration:** Used Jupyter Notebooks to understand our NYC Taxi dataset.
- **Batch Processing:** Implemented efficient chunk-based ingestion to handle 3.7+ million records.
- **Automated Pipeline:** Connected Python scripts to PostgreSQL using SQLAlchemy for seamless data loading.

---

## References

1. GitHub Codespaces. (2026). **GitHub Codespaces Documentation**.
2. Docker Inc. (2026). **Docker Documentation**.
3. The PostgreSQL Global Development Group. (2026). **PostgreSQL: The World&#39;s Most Advanced Open Source Relational Database**.
4. pgAdmin Development Team. (2026). **pgAdmin**.
5. DataTalksClub. (2026). **DataTalksClub Data Engineering Zoomcamp Repository**.
6. Python Software Foundation. (2026). **Python**.
7. UV. (2026). **UV**.
8. Pallets. (2014). **Click documentation**.

---

[1]: https://docs.github.com/en/codespaces/
[2]: https://docs.docker.com/
[3]: https://www.postgresql.org/
[4]: https://www.pgadmin.org/
[5]: https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main
[6]: https://www.python.org/
[7]: https://docs.astral.sh/uv/pip/environments/
[8]: https://click.palletsprojects.com/en/stable/


