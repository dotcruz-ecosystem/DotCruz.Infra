# DotCruz.Infra

This repository contains the database infrastructure definitions and deployment processes for the **DotCruz** project service network. The infrastructure is Docker-based and consists of isolated instances of **PostgreSQL 18** and **MongoDB 8**.

---

## 🛡️ Architecture & Security Decisions

The design of this infrastructure was planned with a focus on data security and hardware efficiency (optimized to run on entry-level VPS servers):

### 1. Secure Access and Isolation (`127.0.0.1`)
*   **Decision**: The database ports in [docker-compose.yml](./docker-compose.yml) (`5432` for PostgreSQL and `27017` for MongoDB) are bound exclusively to the host's loopback address (`127.0.0.1`).
*   **Reason**: This ensures that the databases **are not exposed to the public internet**, blocking automated port scanning attacks.
*   **How to connect**: To access the databases using visual tools (such as DBeaver, pgAdmin, or MongoDB Compass) installed on your local machine, you must use an **SSH Tunnel** pointing to the VPS IP and the database's internal port.

### 2. Isolated Private Network (`dotcruz_net`)
*   **Decision**: The databases belong to an external Docker network named `dotcruz_net`.
*   **Reason**: This allows APIs or other microservices running on the same server to communicate with the databases internally using Docker service names, keeping the traffic isolated from other applications on the VPS.

### 3. Strict Resource Control (CPU/RAM)
*   **Decision**: A strict limit of **512MB RAM** per container was set, and MongoDB was configured with `--wiredTigerCacheSizeGB 0.25` (256MB cache limit).
*   **Reason**: This prevents sudden RAM depletion (*Out-Of-Memory*) on the production server, ensuring overall system stability without one database degrading the performance of other applications.

### 4. Secret Leak Protection
*   **Decision**: The `.env` file containing production credentials is never versioned (blocked via [.gitignore](./.gitignore)). In production, the deployment pipeline creates the file with strict permissions (`chmod 600`) so that only the process owner can read or edit it.

---

## 🛠️ Technologies Used

*   **Docker & Docker Compose**: Service execution in containers.
*   **PostgreSQL 18**: Main relational database.
*   **MongoDB 8**: Document-oriented NoSQL database.
*   **GitHub Actions**: Continuous deployment automation on the VPS.

---

## 🚀 Local Configuration & Setup

### Prerequisites
*   Docker and Docker Compose installed locally.

### Step-by-Step Guide

1.  **Create the External Docker Network**:
    Since the network is marked as external in [docker-compose.yml](./docker-compose.yml), create it manually before running the services:
    ```bash
    docker network create dotcruz_net
    ```

2.  **Configure Environment Variables**:
    Copy the [.env.example](./.env.example) file to `.env` and fill in the environment variables:
    ```bash
    cp .env.example .env
    ```
    Insert local development credentials into the generated `.env` file.

3.  **Run the Services**:
    Start the containers in the background (detached mode):
    ```bash
    docker compose up -d
    ```

4.  **Monitor Status**:
    ```bash
    docker compose ps
    docker compose logs -f
    ```

---

## 🔄 Continuous Deployment (CI/CD)

Deployment is automated via the [deploy.yml](./.github/workflows/deploy.yml) workflow. Whenever there is a push to the `main` branch with changes to [docker-compose.yml](./docker-compose.yml), the pipeline:
1.  Verifies VPS reachability.
2.  Copies the new [docker-compose.yml](./docker-compose.yml) to the remote server via SCP.
3.  Creates the `dotcruz_net` network on the VPS if it does not exist.
4.  Injects the encrypted secrets stored in GitHub into a `.env` file secured with restricted permissions (`chmod 600`).
5.  Recreates/restarts the necessary containers with `docker compose up -d`.
