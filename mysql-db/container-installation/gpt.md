# SOP: Run MySQL in a Docker Container Using Docker Compose (Production-Ready)

This SOP explains how to deploy a MySQL server using Docker Compose with persistent storage, environment variables, health checks, and automatic restart policies.

---

# Prerequisites

Ensure the following are installed:

* Docker Engine
* Docker Compose Plugin

Verify installation:

```bash
docker --version
docker compose version
```

---

# Step 1: Create a Project Directory

```bash
mkdir -p ~/mysql-docker
cd ~/mysql-docker
```

Example structure:

```
mysql-docker/
├── compose.yaml
├── .env
├── mysql_data/
└── initdb/
```

---

# Step 2: Create an Environment File

Create `.env`:

```bash
vim .env
```

Example:

```env
MYSQL_ROOT_PASSWORD=StrongRootPassword@123
MYSQL_DATABASE=myappdb
MYSQL_USER=myuser
MYSQL_PASSWORD=StrongUserPassword@123

TZ=Asia/Dhaka
```

Using a separate `.env` file keeps secrets out of the Compose file.

---

# Step 3: Create the Docker Compose File

Create `compose.yaml`:

```yaml
services:
  mysql:
    image: mysql:8.4

    container_name: mysql-server

    restart: unless-stopped

    ports:
      - "3306:3306"

    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      TZ: ${TZ}

    volumes:
      - ./mysql_data:/var/lib/mysql
      - ./initdb:/docker-entrypoint-initdb.d

    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 10
```

> **Note:** The `healthcheck` shown above uses a placeholder password (`rootpassword`). In production, update it to match your actual root password or use a safer mechanism such as a dedicated health-check user.

---

# Step 4: (Optional) Add Initialization SQL

Create the directory:

```bash
mkdir initdb
```

Create a SQL file:

```bash
vim initdb/init.sql
```

Example:

```sql
CREATE TABLE employee (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100)
);

INSERT INTO employee(name)
VALUES ('Alice');
```

This script runs only when the database is initialized for the first time (before persistent data exists).

---

# Step 5: Create the Persistent Data Directory

```bash
mkdir mysql_data
```

This stores MySQL data outside the container so it survives restarts and container recreation.

---

# Step 6: Start the MySQL Container

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Expected output:

```
NAME            STATUS
mysql-server    Up
```

---

# Step 7: View Container Logs

```bash
docker compose logs
```

Or continuously:

```bash
docker compose logs -f
```

Wait until you see messages indicating the server is ready for connections.

---

# Step 8: Verify the Container Is Running

```bash
docker ps
```

Example:

```
CONTAINER ID   IMAGE       STATUS
xxxxxx         mysql:8.4   Up
```

---

# Step 9: Connect to MySQL from Inside the Container

```bash
docker exec -it mysql-server mysql -u root -p
```

Enter:

```
StrongRootPassword@123
```

Then verify:

```sql
SHOW DATABASES;
```

Expected:

```sql
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| myappdb            |
| performance_schema |
+--------------------+
```

---

# Step 10: Verify the Created User

```sql
SELECT user, host
FROM mysql.user;
```

You should see entries similar to:

```
root
myuser
```

---

# Step 11: Connect from the Host Machine

```bash
mysql -h 127.0.0.1 -P 3306 -u myuser -p
```

Password:

```
StrongUserPassword@123
```

---

# Step 12: Test Database Operations

```sql
USE myappdb;

CREATE TABLE test (
    id INT PRIMARY KEY AUTO_INCREMENT,
    message VARCHAR(100)
);

INSERT INTO test(message)
VALUES ('Docker MySQL Works');

SELECT * FROM test;
```

Example output:

```
+----+----------------------+
| id | message              |
+----+----------------------+
| 1  | Docker MySQL Works   |
+----+----------------------+
```

---

# Step 13: Stop the Container

```bash
docker compose stop
```

Start it again:

```bash
docker compose start
```

The data remains intact because it is stored in `./mysql_data`.

---

# Step 14: Restart the Entire Stack

```bash
docker compose restart
```

---

# Step 15: Shut Down and Remove Containers

```bash
docker compose down
```

The container and network are removed, but the data in `./mysql_data` remains.

---

# Step 16: Remove Everything (Including Database Data)

```bash
docker compose down -v
```

Or manually delete the host data directory:

```bash
rm -rf mysql_data
```

**Warning:** This permanently deletes all database data.

---

# Directory Layout

```
mysql-docker/
├── compose.yaml
├── .env
├── initdb/
│   └── init.sql
└── mysql_data/
```

---

# Useful Docker Commands

| Task                          | Command                                         |
| ----------------------------- | ----------------------------------------------- |
| Start in background           | `docker compose up -d`                          |
| View logs                     | `docker compose logs -f`                        |
| Show containers               | `docker compose ps`                             |
| Stop services                 | `docker compose stop`                           |
| Start stopped services        | `docker compose start`                          |
| Restart services              | `docker compose restart`                        |
| Remove containers             | `docker compose down`                           |
| Remove containers and volumes | `docker compose down -v`                        |
| Open a shell in the container | `docker exec -it mysql-server bash`             |
| Open the MySQL client         | `docker exec -it mysql-server mysql -u root -p` |

## Best Practices

1. Use a named volume or bind mount (such as `./mysql_data`) for persistent storage.
2. Store credentials in a `.env` file or a secrets management solution rather than hard-coding them.
3. Use strong, unique passwords for both the root account and application users.
4. Restrict exposure of port `3306` with firewall rules or by binding it only where necessary.
5. Back up the database regularly and test restoration procedures.
6. Use a specific MySQL image tag (for example, `mysql:8.4`) instead of `latest` to avoid unexpected upgrades.
7. Create a dedicated application user with only the privileges your application requires instead of using the `root` account.
