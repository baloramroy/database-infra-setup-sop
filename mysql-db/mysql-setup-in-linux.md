# SOP: Install and Configure MySQL Server on Red Hat Linux

This Standard Operating Procedure (SOP) explains how to install, configure, secure, and verify a MySQL Server installation on Red Hat Enterprise Linux.

> **Note:** RHEL's default repositories often provide **MariaDB** instead of Oracle MySQL. To install the official MySQL Server, use the MySQL Community repository.

---

## Prerequisites

1. A RHEL 8 or RHEL 9 server
2. Root or sudo privileges
3. Internet connectivity (or access to an internal repository)
4. `dnf` package manager

---

## System Preparation

- Run below to check system OS:
  
  ```bash
  cat /etc/os-release
  ```

  or

  ```bash
  hostnamectl
  ```

- Update the System

  ```bash
  dnf update
  ```

  > (Optional reboot if required.)

---

## Download the MySQL Community Repository Package

- For RHEL 9:

  ```bash
  wget https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm
  ```

- For RHEL 8:

  ```bash
  wget https://dev.mysql.com/get/mysql84-community-release-el8-1.noarch.rpm
  ```

- Verify the file:

  ```bash
  ls
  ```

---

## Install the MySQL Repository

- For RHEL 9:

  ```bash
  rpm -ivh mysql84-community-release-el9-1.noarch.rpm
  ```

- For RHEL 8:

  ```bash
  rpm -ivh mysql84-community-release-el8-1.noarch.rpm
  ```

- Verify:

  ```bash
  dnf repolist | grep mysql
  ```

---

## Install MySQL Server

- Run:
 
  ```bash
  dnf install mysql-community-server
  ```

- Verify installation:

  ```bash
  mysql --version
  ```

- Example:

  ```
  mysql  Ver 8.4.x for Linux on x86_64
  ```

---

## Start and Enabled MySQL Service

### Start the MySQL Service

- Run:

  ```bash
  systemctl start mysqld
  ```

- Check status:

  ```bash
  systemctl status mysqld
  ```

  Expected:

  ```
  Active: active (running)
  ```

#

### Enable MySQL to Start Automatically

- Run:

  ```bash
  systemctl enable mysqld
  ```

- Verify:

  ```bash
  systemctl is-enabled mysqld
  ```

  Expected:

  ```
  enabled
  ```

---

## Obtain the Temporary Root Password

MySQL generates a temporary password during installation.

- Retrieve it:

  ```bash
  grep 'temporary password' /var/log/mysqld.log
  ```

- Example:

  ```
  temporary password is generated for root@localhost:  KlwRpd;pi4D7
  ```

Save this password securely.

---

## Run the MySQL Security Script

>[!IMPORTANT]
After completing **MySQL installation**, run the built-in **security script** to harden the server:

Execute:

```bash
mysql_secure_installation
```

The script will prompt you to:

1. Enter the temporary root password.
2. Set a new root password (The prompt may appear more than once).
3. Remove anonymous users.
4. Disable remote root login.
5. Remove the test database.
6. Reload privilege tables.

Recommended answers:

```
Remove anonymous users?           Y
Disallow root login remotely?     Y
Remove test database?             Y
Reload privilege tables?          Y
```

---

## MySQL Service Management

### Log In to MySQL

- Run:

  ```bash
  mysql -u root -p
  ```

  Enter the password you configured.

- Expected prompt:

  ```sql
  mysql>
  ```

#

### Verify the Server Version

- Inside MySQL:

  ```sql
  SELECT VERSION();
  ```

- Example output:

  ```
  +-----------+
  | VERSION() |
  +-----------+
  | 8.4.x     |
  +-----------+
  ```

#

### View Existing Databases

- Run:
  
  ```sql
  SHOW DATABASES;
  ```

- Expected output:

  ```
  information_schema
  mysql
  performance_schema
  sys
  ```

#

### Create a New Database

- Run:

  ```sql
  CREATE DATABASE myappdb;
  ```

- Verify:

  ```sql
  SHOW DATABASES;
  ```

#

### Create a New User

- Run:

  ```sql
  CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'Myuser@123';
  ```

#

### Grant Privileges

- Grant all privileges on the new database:

  ```sql
  GRANT ALL PRIVILEGES ON myappdb.* TO 'myuser'@'localhost';
  ```

- Apply changes:

  ```sql
  FLUSH PRIVILEGES;
  ```

- Exit:

  ```sql
  EXIT;
  ```

---

## Test the New User

- Log in with the new account:

  ```bash
  mysql -u myuser -p
  ```

- After entering the password, verify access:

  ```sql
  SHOW DATABASES;
  ```

  > You should see `myappdb` among the available databases.

---

## Check That MySQL Is Listening

- Run:

  ```bash
  ss -tulnp | grep 3306
  ```

- Example output:

  ```
  LISTEN 0 70 127.0.0.1:3306
  ```

---

## Check Service Status

- Run:

  ```bash
  systemctl status mysqld
  ```

  Or:

  ```bash
  systemctl is-active mysqld
  ```

- Expected:

  ```
  active
  ```

---

## Open the Firewall (Optional, for Remote Access)

- If clients need to connect from other hosts:

  ```bash
  firewall-cmd --permanent --add-service=mysql
  firewall-cmd --reload
  ```

- Verify:

  ```bash
  firewall-cmd --list-services
  ```


---

## Common MySQL Service Commands

- Start:

  ```bash
  systemctl start mysqld
  ```

- Stop:

  ```bash
  systemctl stop mysqld
  ```

- Restart:

  ```bash
  systemctl restart mysqld
  ```

- Reload:

  ```bash
  systemctl reload mysqld
  ```

- Check status:

  ```bash
  systemctl status mysqld
  ```

- Enable on boot:

  ```bash
  systemctl enable mysqld
  ```

- Disable on boot:

  ```bash
  systemctl disable mysqld
  ```

---

## Useful MySQL Commands

- Show databases:

  ```sql
  SHOW DATABASES;
  ```

- Select a database:

  ```sql
  USE myappdb;
  ```

- List tables:

  ```sql
  SHOW TABLES;
  ```

- Show users:

  ```sql
  SELECT user, host FROM mysql.user;
  ```

- Show grants for a user:

  ```sql
  SHOW GRANTS FOR 'myuser'@'localhost';
  ```

- Exit the MySQL shell:

  ```sql
  EXIT;
  ```

---

# Directory and File Locations

| Purpose                                        | Location              |
| ---------------------------------------------- | --------------------- |
| Configuration                                  | `/etc/my.cnf`         |
| Data directory                                 | `/var/lib/mysql/`     |
| Log file (temporary password on first install) | `/var/log/mysqld.log` |
| MySQL binary                                   | `/usr/bin/mysql`      |
| Service name                                   | `mysqld`              |
| Default TCP port                               | `3306`                |

---

# Troubleshooting

- **Check service status:**

  ```bash
  systemctl status mysqld
  ```

- **View recent logs:**

  ```bash
  journalctl -u mysqld -n 100
  ```

- **Follow logs in real time:**

  ```bash
  journalctl -u mysqld -f
  ```

- **Verify the listening port:**

  ```bash
  ss -ltnp | grep 3306
  ```

- **Test a local connection:**

  ```bash
  mysql -u root -p
  ```

> This procedure provides a production-oriented baseline for installing and securing MySQL on RHEL.

---