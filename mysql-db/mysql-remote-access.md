# SOP: MySQL Remote Access Configuration & Hardening

## Objective

To securely enable remote access to MySQL while ensuring:

* Controlled network exposure
* Strong authentication
* Firewall protection
* Least privilege access

---

## Prerequisites

* MySQL Server installed (`mysqld`)
* Root access on Linux server
* Firewall service running (`firewalld`)
* Basic MySQL security already applied (`mysql_secure_installation` completed)

---

## Default Security State

By default MySQL:

* Listens on: `127.0.0.1:3306`
* Allows only local connections
* Blocks all remote access

Run this query:

```sql
SELECT user, host FROM mysql.user ORDER BY user, host;
```
Expected Output:

```
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| myuser           | localhost |
| root             | localhost |
+------------------+-----------+

```
> Thats mean only local connection is allowed.

---

## Configure MySQL for Remote Access

### First Check 

- Run:

  ```bash
  ss -tulnp | grep 3306
  ```

- Output:

  ```
  127.0.0.1:3306
  ```

  > If you find **mysql port** is bind to **localhost** then follow below step otherwise skip to Configure Firewall part.

### Create Custom Configuration File

Do Not modify `/etc/my.cnf` directly.

- Create:

  ```bash
  vim /etc/my.cnf.d/mysql-server.cnf
  ```

- Add:

  ```ini
  [mysqld]
  bind-address = 0.0.0.0
  ```

- Explanation:

  * `127.0.0.1` → local only
  * `0.0.0.0` → listens on all interfaces

#

### Restart MySQL Service

- Run:

  ```bash
  systemctl restart mysqld
  ```

- Verify:

  ```bash
  systemctl status mysqld
  ```

#

### Confirm MySQL is Listening on Network

```bash
ss -tulnp | grep 3306
```

Expected:

```
0.0.0.0:3306
or
*:3306

```

---

## Configure Firewall

### Open MySQL Port

- Run
  
  ```bash
  firewall-cmd --permanent --add-port=3306/tcp
  firewall-cmd --reload
  ```

- Verify:

  ```bash
  firewall-cmd --list-ports
  ```

- Expected:

  ```
  3306/tcp
  ```

#

### Production Best Practice

- Instead of opening to all networks, restrict access:

  ```bash
  firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="3306" protocol="tcp" accept'
  firewall-cmd --reload
  ```

---

## Create Remote MySQL User (Secure Method)

- Login to MySQL:

  ```bash
  mysql -u root -p
  ```

- Create Database (if needed)

  ```sql
  CREATE DATABASE myappdb;
  ```

- Create Remote User (Restrict by IP Range)

  ```sql
  CREATE USER 'remoteuser'@'192.168.1.%' IDENTIFIED BY 'StrongPassword@123!';
  ```

- Grant Least Privileges (Best Practice)

  ```sql
  GRANT SELECT, INSERT, UPDATE, DELETE ON myappdb.* TO 'remoteuser'@'192.168.1.%';
  ```
  > Avoid full access unless required

- OR (only if required):

  ```sql
  GRANT ALL PRIVILEGES ON myappdb.* TO 'remoteuser'@'192.168.1.%';
  ```

- Provide acces to all databases with grant access (Not recommanded):

  ```sql
  GRANT ALL PRIVILEGES ON *.* TO 'remoteuser'@'%' WITH GRANT OPTION;
  ```

- ### Apply Changes

  ```sql
  FLUSH PRIVILEGES;
  ```

---

## Verify Remote Access

- From client machine:

  ```bash
  mysql -u remoteuser -p -h <MYSQL_SERVER_IP> myappdb
  ```

- Or for all database access:

  ```bash
  mysql -u remoteuser -p -h <MYSQL_SERVER_IP>
  ```

---

## SELinux (Important in Production)

- Check SELinux:

  ```bash
  getenforce
  ```

- If `Enforcing`, allow MySQL network connection:

  ```bash
  setsebool -P mysql_connect_any 1
  ```

---

## Security Hardening Checklist

* ✔ Do NOT use `bind-address = 0.0.0.0` in public internet environments
* ✔ Restrict firewall to trusted subnet
* ✔ Use strong password policy
* ✔ Avoid `'%'` wildcard user unless necessary
* ✔ Use least privilege grants
* ✔ Disable root remote login (already done via mysql_secure_installation)

---

## Common Errors & Fixes

### Error 1: Connection refused

**Cause:**\
MySQL not listening on network

**Fix:**

```bash
bind-address = 0.0.0.0
systemctl restart mysqld
```

#

### Error 2: Host not allowed (ERROR 1130)

**Cause:**\
User not permitted for remote host

**Fix:**

```sql
CREATE USER 'myuser'@'client_ip' IDENTIFIED BY 'password';
```

#

### Error 3: Access denied (ERROR 1045)

**Cause:**\
Wrong password or missing privileges

**Fix:**

```sql
GRANT ALL PRIVILEGES ON db.* TO 'user'@'host';
FLUSH PRIVILEGES;
```

#

### Error 4: Firewall blocking connection

**Fix:**

```bash
firewall-cmd --add-port=3306/tcp --permanent
firewall-cmd --reload
```

---

## Final Production Architecture View

```
Client Machine
      |
      | TCP 3306
      v
Firewall (restricted subnet only)
      |
      v
MySQL Server (bind-address = 0.0.0.0)
      |
      v
Database (myappdb)
```

---

## Summary

To enable secure MySQL remote access:

1. Configure `bind-address`
2. Open firewall safely
3. Create restricted user
4. Use least privilege grants
5. Validate connection

---

