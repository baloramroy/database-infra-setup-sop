**STANDARD OPERATING PROCEDURE**
# Running MySQL on Docker Using Docker Compose


| **Document Title** | Running MySQL on Docker Using Docker Compose     |
|-------------------|---------------------------------------------------|
| **Version**       | 1.0                                               |
| **Effective Date**| June 24, 2026                                     |
| **Scope**         | Development, Staging, and Production environments |
| **Applies To**    | DevOps Engineers, Backend Developers, System Administrators |


## Purpose

This Standard Operating Procedure (SOP) provides step-by-step instructions for deploying and managing a MySQL database server inside a Docker container using Docker Compose.

It ensures persistent storage, environment variables, health checks, and automatic restart policies setups across development, staging, and production environments.

## 2. Prerequisites

### 2.1 Software Requirements

| Software | Min Version | Notes |
|----------|-------------|-------|
| Docker Engine | 20.10+ | Docker Desktop includes Docker Compose V2 |
| Docker Compose | 2.0+ | Use `docker compose` (not docker-compose) |
| OS | Linux / macOS / Windows | WSL2 recommended on Windows |

### 2.2 Verify Installation

Run the following commands to confirm Docker and Docker Compose are installed:

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Verify Docker daemon is running
docker info
```

---

## Project Directory Structure

Create the following directory structure for the project:

```
mysql-docker/
├── compose.yml       # Main Compose configuration
├── .env                     # Environment variables (secrets)
├── config/
│   └── my.cnf               # Custom MySQL configuration (optional)
├── initdb/
│   └── 01_init.sql          # Initialization SQL scripts (optional)
└── mysql_data/              # Persistent volume mount (auto-created)
```

Create the project directory:

```bash
mkdir mysql-docker && cd mysql-docker
mkdir -p config initdb mysql_data
```

---

## Create Configuration Files

### Create the .env File

>[!TIP]
> **Note:** Store all sensitive credentials in a `.env` file. Never commit this file to version control. \
> **Warning:** Add `.env` to your `.gitignore` file to prevent credentials from being exposed in version control.


- Create the `.env` file:

  ```bash
  vim .env
  ```

- Example:

  ```env
  # .env

  # MySQL Root Password
  MYSQL_ROOT_PASSWORD=Admin@123

  # Application database
  MYSQL_DATABASE=myappdb

  # Application user credentials
  MYSQL_USER=appuser
  MYSQL_PASSWORD=Appuser@123

  # Port to expose on host
  MYSQL_PORT=3306

  TZ=Asia/Dhaka
  ```


### Create the compose.yml File