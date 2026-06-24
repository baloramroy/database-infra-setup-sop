I've converted the SOP document into a clean Markdown format. Here it is:

---

# STANDARD OPERATING PROCEDURE
## Running MySQL on Docker Using Docker Compose

| **Document Title** | SOP: Running MySQL on Docker Using Docker Compose |
|-------------------|---------------------------------------------------|
| **Version**       | 1.0                                               |
| **Effective Date**| June 24, 2026                                     |
| **Scope**         | Development, Staging, and Production environments |
| **Applies To**    | DevOps Engineers, Backend Developers, System Administrators |

---

## 1. Purpose

This Standard Operating Procedure (SOP) provides step-by-step instructions for deploying and managing a MySQL database server inside a Docker container using Docker Compose. It ensures consistent, reproducible, and environment-agnostic database setups across development, staging, and production environments.

---

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

## 3. Project Directory Structure

Create the following directory structure for the project:

```
mysql-docker/
├── docker-compose.yml       # Main Compose configuration
├── .env                     # Environment variables (secrets)
├── config/
│   └── my.cnf               # Custom MySQL configuration (optional)
├── init/
│   └── 01_init.sql          # Initialization SQL scripts (optional)
└── data/                    # Persistent volume mount (auto-created)
```

1. Create the project directory:

```bash
mkdir mysql-docker && cd mysql-docker
mkdir -p config init data
```

---

## 4. Create Configuration Files

### 4.1 Create the .env File

> Store all sensitive credentials in a `.env` file. Never commit this file to version control.

> **⚠️ Warning:** Add `.env` to your `.gitignore` file to prevent credentials from being exposed in version control.

2. Create the `.env` file:

```env
# .env

# MySQL Root Password
MYSQL_ROOT_PASSWORD=StrongR00tP@ssword!

# Application database
MYSQL_DATABASE=myappdb

# Application user credentials
MYSQL_USER=appuser
MYSQL_PASSWORD=AppUs3rP@ss!

# Port to expose on host
MYSQL_PORT=3306
```

### 4.2 Create the docker-compose.yml File

3. Create the main Compose file:

```yaml
# docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0                    # Use official MySQL 8.0 image
    container_name: mysql_db
    restart: unless-stopped             # Auto-restart on failure
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE:      ${MYSQL_DATABASE}
      MYSQL_USER:          ${MYSQL_USER}
      MYSQL_PASSWORD:      ${MYSQL_PASSWORD}
    ports:
      - '${MYSQL_PORT}:3306'            # host:container
    volumes:
      - mysql_data:/var/lib/mysql       # Named volume for persistence
      - ./config/my.cnf:/etc/mysql/conf.d/my.cnf:ro   # Custom config
      - ./init:/docker-entrypoint-initdb.d:ro          # Init scripts
    networks:
      - db_network
    healthcheck:
      test: ['CMD', 'mysqladmin', 'ping', '-h', 'localhost',
             '-u', 'root', '-p${MYSQL_ROOT_PASSWORD}']
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

volumes:
  mysql_data:                           # Named volume managed by Docker
    driver: local

networks:
  db_network:
    driver: bridge
```

### 4.3 Create Custom MySQL Configuration (Optional)

4. Create `config/my.cnf` for performance tuning:

```ini
# config/my.cnf
[mysqld]
# Character set
character-set-server    = utf8mb4
collation-server        = utf8mb4_unicode_ci

# Connection settings
max_connections         = 200
connect_timeout         = 10

# InnoDB settings
innodb_buffer_pool_size = 256M
innodb_log_file_size    = 64M

# Logging
slow_query_log          = 1
slow_query_log_file     = /var/log/mysql/slow.log
long_query_time         = 2

[client]
default-character-set   = utf8mb4
```

### 4.4 Create Initialization SQL Script (Optional)

5. Create `init/01_init.sql` to run on first startup:

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

## 5. Deploy MySQL Container

### 5.1 Start the Container

6. Navigate to the project directory:

```bash
cd mysql-docker
```

7. Start the services in detached mode:

```bash
docker compose up -d
```

8. Verify the container is running:

```bash
docker compose ps
```

Expected output:
```
NAME        IMAGE       COMMAND                  STATUS          PORTS
mysql_db    mysql:8.0   'docker-entrypoint.s...'  Up (healthy)    0.0.0.0:3306->3306/tcp
```

9. Check container logs:

```bash
docker compose logs -f mysql
```

Look for: `[System] [MY-010931] ... ready for connections.`

### 5.2 Verify Health Check

```bash
# Check health status
docker inspect mysql_db --format='{{.State.Health.Status}}'
```

Expected: `healthy`

---

## 6. Connect to MySQL

### 6.1 Connect via Docker Exec (CLI)

10. Open a MySQL shell inside the container:

```bash
# Connect as root
docker exec -it mysql_db mysql -u root -p

# Connect as application user
docker exec -it mysql_db mysql -u appuser -p myappdb
```

### 6.2 Connect from Host Machine

11. Using the mysql client installed on host:

```bash
mysql -h 127.0.0.1 -P 3306 -u appuser -p myappdb
```

### 6.3 Connect from Another Docker Container

Other services in the same Compose file can connect using the service name as hostname:

```yaml
# In another service's environment variables:
DB_HOST=mysql
DB_PORT=3306
DB_NAME=myappdb
DB_USER=appuser
```

### 6.4 Connection String Examples

```text
# Generic JDBC
jdbc:mysql://127.0.0.1:3306/myappdb

# Python (SQLAlchemy)
mysql+pymysql://appuser:AppUs3rP@ss!@127.0.0.1:3306/myappdb

# Node.js
mysql://appuser:AppUs3rP@ss!@127.0.0.1:3306/myappdb
```

---

## 7. Common Operations

### 7.1 Container Lifecycle

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

### 7.2 Backup & Restore

**Backup Database**

```bash
# Dump a single database
docker exec mysql_db mysqldump -u root -p${MYSQL_ROOT_PASSWORD} myappdb \
  > ./backups/myappdb_$(date +%Y%m%d_%H%M%S).sql

# Dump all databases
docker exec mysql_db mysqldump -u root -p${MYSQL_ROOT_PASSWORD} --all-databases \
  > ./backups/all_databases_$(date +%Y%m%d_%H%M%S).sql
```

**Restore Database**

```bash
# Restore from SQL dump
docker exec -i mysql_db mysql -u root -p${MYSQL_ROOT_PASSWORD} myappdb \
  < ./backups/myappdb_20240101_120000.sql
```

### 7.3 Upgrade MySQL Version

12. Update the image tag in `docker-compose.yml`:

```yaml
image: mysql:8.4   # Update version tag
```

13. Pull the new image and recreate the container:

```bash
docker compose pull
docker compose up -d --force-recreate
```

> **📝 Note:** Always back up your data before upgrading MySQL versions. Downgrades between major versions are not supported.

---

## 8. Security Best Practices

- Never hardcode credentials in `docker-compose.yml` — always use `.env` or Docker secrets.
- Add `.env` to `.gitignore` to prevent accidental credential exposure.
- Use a dedicated non-root MySQL user for application connections (never use root in apps).
- Restrict port exposure — bind to `127.0.0.1` for local-only access: `'127.0.0.1:3306:3306'`.
- In production, remove the `ports:` section entirely and use internal Docker networks.
- Enable MySQL SSL/TLS for encrypted connections in production.
- Regularly rotate passwords and audit user privileges.
- Keep the MySQL Docker image up-to-date to receive security patches.

---

## 9. Troubleshooting

| Issue | Likely Cause | Resolution |
|-------|--------------|------------|
| Container exits immediately | Wrong env variables or port conflict | Run: `docker compose logs mysql` |
| Port 3306 already in use | Another MySQL is running on host | Change `MYSQL_PORT` in .env to 3307 |
| Access denied for user | Wrong credentials or missing GRANT | Verify .env values; reconnect after fix |
| Data not persisting after restart | Volume not configured correctly | Ensure named volume in compose file |
| Container unhealthy | MySQL not ready yet | Wait 30s; check logs for errors |
| Permission denied on ./data | Wrong file ownership | Run: `sudo chown -R 999:999 ./data` |

---

## 10. Quick Reference Checklist

Use this checklist each time you set up MySQL on Docker:

14. Create project directory structure (`mysql-docker/`, `config/`, `init/`, `data/`)
15. Create `.env` file with secure credentials
16. Add `.env` to `.gitignore`
17. Create `docker-compose.yml` with service, volumes, networks, and healthcheck
18. (Optional) Create `config/my.cnf` for custom MySQL settings
19. (Optional) Create `init/*.sql` for initialization scripts
20. Run: `docker compose up -d`
21. Verify: `docker compose ps` — status should show healthy
22. Test connection: `docker exec -it mysql_db mysql -u appuser -p`
23. Set up backup schedule for production environments

---

## Appendix: Useful MySQL Commands

```sql
-- Inside MySQL shell

-- List all databases
SHOW DATABASES;

-- Use a database
USE myappdb;

-- List all tables
SHOW TABLES;

-- List all users
SELECT User, Host FROM mysql.user;

-- Show current user
SELECT CURRENT_USER();

-- Show MySQL version
SELECT VERSION();

-- Show active processes
SHOW PROCESSLIST;

-- Exit
EXIT;
```

---

*End of Document*
```

---

This Markdown version preserves all the original content while adding proper formatting with:

- Headers and subheaders using `#`, `##`, `###`
- Tables for structured data
- Code blocks with appropriate language tags
- Blockquotes for warnings and notes
- Lists for steps and checklists
- Horizontal rules for section separation

You can copy this directly into any Markdown editor or platform (GitHub, GitLab, Notion, etc.).