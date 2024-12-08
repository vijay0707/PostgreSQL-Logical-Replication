# Custom PostgreSQL Docker Image with `wal2json` Extension

This repository provides a `Dockerfile` and instructions to create a custom PostgreSQL Docker image that includes the `wal2json` extension. The `wal2json` extension is a widely used output plugin for PostgreSQL's logical replication, which allows you to capture changes in the database in a JSON format. This is particularly useful for Change Data Capture (CDC) or replicating changes to external systems.

You can also directly use the pre-built image available in the Docker registry from the link below:

- [Docker Hub Repository: `vijaysaran07/postgres-with-wal2json`](https://hub.docker.com/repository/docker/vijaysaran07/postgres-with-wal2json)

---

## **Table of Contents**

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [How to Build the Docker Image](#how-to-build-the-docker-image)
- [Dockerfile Explanation](#dockerfile-explanation)
- [How to Push the Image to a Container Registry](#how-to-push-the-image-to-a-container-registry)
- [Conclusion](#conclusion)

---

## **Overview**

In PostgreSQL, logical replication allows you to stream changes to other systems. The `wal2json` extension is an output plugin that converts changes in the write-ahead log (WAL) to a JSON format, making it easier to work with other systems that consume JSON data.

This project provides a Dockerfile to build a custom PostgreSQL image that includes the `wal2json` extension. The custom image can be used to enable logical replication in PostgreSQL.


---

## **Prerequisites**

Before you start, ensure you have the following tools installed:

- **Docker**: You need Docker to build and run containers.
- **Access to a Container Registry**: A registry like Docker Hub, AWS ECR, or any other container registry where you can push your custom image.
- **PostgreSQL Version 12 or Later**: The `wal2json` extension requires PostgreSQL 12 or higher.

---

## **How to Build the Docker Image**

### Step 1: Clone this Repository

Clone the repository to your local machine:

```bash
git clone https://github.com/vijay0707/PostgreSQL-Logical-Replication.git
cd postgres-with-wal2json
```

### Step 2: Build the Docker Image

Make sure you're in the same directory as the `Dockerfile`. Then, build the Docker image using the following command:

```bash
docker build -t <your_registry>/postgres-with-wal2json:<tag> .
```

Replace:
- `<your_registry>`: The name of your container registry (e.g., `docker.io/username`).
- `<tag>`: A version tag for your image (e.g., `v1`).

This command will download the PostgreSQL base image, install the necessary dependencies, build the `wal2json` extension from source, and create the custom image.

---

## **Dockerfile Explanation**

The `Dockerfile` used to build the custom PostgreSQL image is as follows:

```Dockerfile
FROM postgres:<Version>  # Replace with your PostgreSQL version (e.g., 13)

# Install build dependencies
RUN apt-get update && apt-get install -y \
    postgresql-server-dev-<Version> \
    build-essential \
    git \
    && rm -rf /var/lib/apt/lists/*

# Clone and install wal2json
RUN git clone https://github.com/eulerto/wal2json.git /tmp/wal2json \
    && cd /tmp/wal2json \
    && make && make install \
    && rm -rf /tmp/wal2json

# Copy the custom postgresql.conf to configure PostgreSQL
COPY postgresql.conf /etc/postgresql/postgresql.conf.sample

# Ensure PostgreSQL loads the wal2json extension
RUN echo "shared_preload_libraries = 'wal2json'" >> /etc/postgresql/postgresql.conf.sample

```

- **Base Image**: The `FROM postgres:<your_version>` line specifies the base PostgreSQL image. Replace `<your_version>` with the desired PostgreSQL version (e.g., `13`).
  
- **Install Dependencies**: The `RUN apt-get` command installs required packages like `postgresql-server-dev`, `build-essential`, and `git` to build the `wal2json` extension from source.

- **Clone and Install `wal2json`**: The `RUN git clone` command fetches the `wal2json` repository, compiles it, and installs it into the PostgreSQL container.

- **PostgreSQL Configuration**: The `COPY postgresql.conf` command ensures that PostgreSQL is configured to load the `wal2json` extension by adding it to the `shared_preload_libraries` setting.

---

---

## **Custom `postgresql.conf`**

This project uses a custom `postgresql.conf` file to enable logical replication and load the `wal2json` extension. The key settings in the `postgresql.conf` are:

- **`wal_level = logical`**: Enables logical replication, which is required for `wal2json` to function.
- **`shared_preload_libraries`**: Preloads `pg_stat_statements` and `wal2json` so that PostgreSQL loads the extension at startup.

### Example `postgresql.conf`

```ini
#------------------------------------------------------------------------------
# PostgreSQL configuration file
#------------------------------------------------------------------------------

#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# Listen on all addresses (useful if running in Docker)
listen_addresses = '*'

# Maximum number of concurrent connections
max_connections = 100

#------------------------------------------------------------------------------
# RESOURCE USAGE
#------------------------------------------------------------------------------

# Shared memory buffers (typically 25% of available memory)
shared_buffers = 128MB

# Work memory for operations (adjust based on available memory)
work_mem = 4MB

# Maintenance work memory (used for maintenance operations)
maintenance_work_mem = 64MB

#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

# Write-Ahead Log (WAL) settings
wal_level = logical            # Set WAL level to 'logical' for logical replication
max_wal_senders = 4         # Maximum number of replication connections
max_replication_slots = 4      # Number of replication slots for logical replication

#------------------------------------------------------------------------------
# EXTENSIONS
#------------------------------------------------------------------------------

# Preload necessary extensions
shared_preload_libraries = 'pg_stat_statements, wal2json'  # Load the wal2json extension

#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------

# Enable detailed logging for debugging
log_statement = 'all'          # Log all queries (change as needed for production)
log_duration = on              # Log the duration of queries

#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------

# Enable autovacuum to prevent table bloat
autovacuum = on
autovacuum_vacuum_scale_factor = 0.2
autovacuum_analyze_scale_factor = 0.1

#------------------------------------------------------------------------------
# Other PostgreSQL settings
#------------------------------------------------------------------------------

# Default time zone (you can adjust this to your region)
timezone = 'UTC'
```

### How to Use This `postgresql.conf`:

1. **Save this file** as `postgresql.conf` in the same directory as your `Dockerfile`.
2. When you build the Docker image, the `COPY` command will copy this configuration into the container’s PostgreSQL configuration directory.

---

## **How to Push the Image to a Container Registry**

Once you've built the image, you can push it to your container registry. First, log in to your container registry (if required):

```bash
docker login <your_registry>
```

Then, push the image:

```bash
docker push <your_registry>/postgres-with-wal2json:<tag>
```

Replace `<your_registry>` and `<tag>` with the appropriate values for your setup.

---

## **Conclusion**

By following these steps, you will have a custom PostgreSQL Docker image with the `wal2json` extension installed. This image is ready for use in logical replication scenarios, including Change Data Capture (CDC) or replicating changes to external systems.

If you prefer not to build the image yourself, you can directly use the pre-built image available in the Docker Hub repository:

- [Docker Hub Repository: `vijaysaran07/postgres-with-wal2json`](https://hub.docker.com/repository/docker/vijaysaran07/postgres-with-wal2json)

For further configuration or deployment instructions, refer to the PostgreSQL and Docker documentation.

---

### License

This repository is licensed under the MIT License. See [LICENSE](LICENSE) for more details.
