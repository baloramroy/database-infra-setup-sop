**STANDARD OPERATING PROCEDURE**
# Running MySQL on Docker Using Docker Compose


|**Document Title** |Running MySQL on Docker Using Docker Compose                 |
|-------------------|-------------------------------------------------------------|
| **Version**       | 1.0                                                         |
| **Effective Date**| June 24, 2026                                               |
| **Scope**         | Development, Staging, and Production environments           |
| **Applies To**    | DevOps Engineers, Backend Developers, System Administrators |

---

## Purpose

This Standard Operating Procedure (SOP) provides step-by-step instructions for **deploying** and managing a **MySQL database** server inside a **Docker** container using **Docker Compose**.

It ensures **persistent storage**, **environment variables**, **health checks**, and **automatic restart policies** setups across development, staging, and production environments.

---

## Prerequisites

### Software Requirements

| Software | Min Version | Notes |
|----------|-------------|-------|
| Docker Engine | 20.10+ | Docker Desktop includes Docker Compose V2 |
| Docker Compose | 2.0+ | Use `docker compose` (not docker-compose) |
| OS | Linux / macOS / Windows | WSL2 recommended on Windows |

#

### Verify Installation

Run the following commands to confirm Docker and Docker Compose are installed:

- **Check Docker version**

  ```bash
  docker --version
  ```

- **Check Docker Compose version**

  ```bash
  docker compose version
  ```

- **Verify Docker daemon is running**

  ```bash
  docker info
  ```

---

## Project Directory Structure

- **Create the following directory structure for the project:**

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

- **Create the project directory:**

  ```bash
  mkdir mysql-docker && cd mysql-docker
  mkdir -p config initdb mysql_data
  ```

---

## Set Necessary Permission

- **Verify user and ID information by running temporary container:**

  ```bash
  docker run --rm -it --entrypoint /bin/sh mysql:8.4
  ```
- **Then run `whoami` and `id` command inside container**
  ```
  /bash-5.1 $ whoami
  Output: root
  
  /bash-5.1 $ id root
  uid=0(root) gid=0(root) groups=0(root)
  ```

### Now change the directory permission

- The MySQL container starts as the **root** user to perform initialization tasks such as creating the **data** directory, adjusting file **ownership**, and executing initialization **scripts**. 
- After initialization, the **MySQL server** runs as the **mysql** user inside the **container**.
- Create the required directories and assign temporary ownership to root:

  ```bash
  chown -R 0:0 config initdb mysql_data
  ```

>[!NOTE]
◼️ The **ownership** of the **mysql_data** directory may automatically change after the **container starts.** \
◼️ During initialization, the MySQL entrypoint script changes the **ownership** of the data directory to the internal **mysql** user (currently **UID 999/GID 999** in the official **mysql:8.4** image). \
◼️ On the host, these **numeric IDs** may appear as different **usernames** (for example, **systemd-coredump** or another local account) because Linux displays the **host's local username** associated with the same **UID/GID**. \
◼️ This behavior is **normal** and does not indicate that the **ownership** has been **assigned** to the **host user**.


---

## Create Configuration Files

### Create the .env File


◾ **Create the `.env` file in mysql-docker directory:**

```bash
vim .env
```

◾ **Insert below lines:**

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
>[!TIP]
> **Note:** Store all sensitive credentials in a `.env` file. Never commit this file to version control. \
> **Warning:** Add `.env` to your `.gitignore` file to prevent credentials from being exposed in version control.


#

### Create the Compose File

◾ **Create `compose.yml` file in mysql-docker directory:**

```yaml
# compose.yml
services:
  mysql:
    image: mysql:8.4                    # Use official MySQL 8.0 image
    container_name: mysql_db
    restart: unless-stopped             # Auto-restart on failure
    ports:
      - '${MYSQL_PORT}:3306'            # host:container
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      TZ: ${TZ}
    volumes:
      - ./mysql_data:/var/lib/mysql       # Bind volume for persistence
      - ./config/my.cnf:/etc/mysql/conf.d/my.cnf:ro   # Custom config
      - ./initdb:/docker-entrypoint-initdb.d:ro          # Init scripts
    networks:
      - db_network
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "mysqladmin ping -h localhost -uroot -p$MYSQL_ROOT_PASSWORD"
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

networks:
  db_network:
    driver: bridge
    external: true
```

#

### Create Initialization SQL Script (Optional)

◾ **Create a SQL file:**

```bash
vim initdb/01_init.sql
```

◾ **Create `init/01_init.sql` to run on first startup:**

```sql
-- init/01_init.sql
-- This script runs once when the container is first created

CREATE DATABASE IF NOT EXISTS myappdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE myappdb;

CREATE TABLE IF NOT EXISTS users (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  username   VARCHAR(50) NOT NULL UNIQUE,
  email      VARCHAR(100) NOT NULL UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Grant privileges to app user
GRANT ALL PRIVILEGES ON myappdb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

---

## Deploy MySQL Container

### Create Docker Network First

- Run:
  
  ```bash
  docker network create db_network
  ```

#

### Docker Compose Validation

- Before deployment
  
  ```bash
  docker compose config
  ```

- This validates

  - YAML
  - variables
  - syntax

#

### Start the Container Now

- Navigate to the project directory:

  ```bash
  cd mysql-docker
  ```

- Start the services in detached mode:

  ```bash
  docker compose up -d
  ```

- Verify the container is running:

  ```bash
  docker compose ps
  ```

  Expected output:
  ```
  NAME        IMAGE       COMMAND                  STATUS          PORTS
  mysql_db    mysql:8.4   'docker-entrypoint.s...'  Up (healthy)    0.0.0.0:3306->3306/tcp
  ```

#

### Inspect Logs

- Check container logs for successfull startup:

  ```bash
  docker compose logs -f mysql
  ```

  > Look for: `[System] [MY-010931] ... ready for connections.`

#

### Verify Health Check

- Verify the Container Is Running

  ```bash
  docker ps
  ```

  Example:

  ```
  CONTAINER ID   IMAGE       STATUS
  xxxxxx         mysql:8.4   Up
  ```

- Check health status

  ```bash
  docker inspect mysql_db --format='{{.State.Health.Status}}'
  ```

  >Expected: `healthy`

- Network Inspection

  ```bash
  # List network 
  docker network ls

  # Inspect network
  docker network inspect db_network
  ```

---

## Connect to MySQL

### Connect via Docker Exec (CLI)

Open a **MySQL shell** inside the **container**:

- **Connect as `root`**
  ```bash
  docker exec -it mysql_db mysql -u root -p
  ```

- **Connect as `application user`**

  ```bash
  docker exec -it mysql_db mysql -u appuser -p myappdb
  ```

#

### Connect from Host Machine

- **Using the **mysql client** installed on host:**

  ```bash
  mysql -h 127.0.0.1 -P 3306 -u appuser -p myappdb
  ```

### Connect from Another Docker Container

- **Other services in the same `Compose` file can connect using the `service name` as `hostname`:**

  ```yaml
  # In another service's environment variables:
  DB_HOST=mysql
  DB_PORT=3306
  DB_NAME=myappdb
  DB_USER=appuser
  ```

---

## Common Operations

### Container Lifecycle

| Action | Command |
|--------|---------|
| Start containers | `docker compose up -d` |
| Stop containers | `docker compose stop` |
| Restart containers | `docker compose restart` |
| Stop & remove containers | `docker compose down` |
| Remove + delete volumes | `docker compose down -v` |
| View running containers | `docker compose ps` |
| View logs (live) | `docker compose logs -f mysql` |
| Pull latest image | `docker compose pull` |

#

### Backup & Restore

◾ **Backup Database**

```bash
# Dump a single database
docker exec mysql_db mysqldump -u root -p${MYSQL_ROOT_PASSWORD} myappdb \
  > ./backups/myappdb_$(date +%Y%m%d_%H%M%S).sql

# Dump all databases
docker exec mysql_db mysqldump -u root -p${MYSQL_ROOT_PASSWORD} --all-databases \
  > ./backups/all_databases_$(date +%Y%m%d_%H%M%S).sql
```

◾ **Restore Database**

```bash
# Restore from SQL dump
docker exec -i mysql_db mysql -u root -p${MYSQL_ROOT_PASSWORD} myappdb \
  < ./backups/myappdb_20240101_120000.sql
```

#

### Upgrade MySQL Version

◾ **Update the image tag in `docker-compose.yml`:**

```yaml
image: mysql:8.4   # Update version tag
```

◾ **Pull the new image and recreate the container:**

```bash
docker compose pull
docker compose up -d --force-recreate
```

> **📝 Note:** Always back up your data before upgrading MySQL versions. Downgrades between major versions are not supported.

---

## Quick Reference Checklist

Use this checklist each time you set up MySQL on Docker:

- Create project directory structure (`mysql-docker/`, `config/`, `initdb/`, `mysql_data/`)
- Create `.env` file with secure credentials
- Add `.env` to `.gitignore`
- Create `compose.yml` with service, volumes, networks, and healthcheck
- (Optional) Create `config/my.cnf` for custom MySQL settings
- (Optional) Create `initdb/*.sql` for initialization scripts
- Run: `docker compose up -d`
- Verify: `docker compose ps` — status should show healthy
- Test connection: `docker exec -it mysql_db mysql -u appuser -p`
- Set up backup schedule for production environments

---


## Best Practices

1. Use a named volume or bind mount (such as `./mysql_data`) for persistent storage.
2. Store credentials in a `.env` file or a secrets management solution rather than hard-coding them.
3. Use strong, unique passwords for both the root account and application users.
4. Restrict port exposure — bind to `127.0.0.1` for local-only access: `'127.0.0.1:3306:3306'`.
5. Back up the database regularly and test restoration procedures.
6. Use a specific MySQL image tag (for example, `mysql:8.4`) instead of `latest` to avoid unexpected upgrades.
7. Create a dedicated application user with only the privileges your application requires instead of using the `root` account.

---
