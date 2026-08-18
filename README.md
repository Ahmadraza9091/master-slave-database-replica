# MySQL Master-Slave Replication with Orchestrator

A practical High Availability MySQL project implementing **GTID-based Master-Slave (Primary-Replica) replication**, **Orchestrator monitoring and automatic failover**, **automatic rejoin of the failed Primary as a Replica**, and **Nginx reverse proxy/load balancing**.

> **Note:** Modern MySQL documentation uses the terms **Source/Replica** instead of Master/Slave. This project uses both terms where necessary because "Master-Slave" is commonly used when describing MySQL replication.

---

## 1. Project Objective

The goal of this project is to build a MySQL High Availability environment where:

1. MySQL Primary handles normal write operations.
2. MySQL Replica continuously replicates data from the Primary.
3. GTID is used for reliable replication positioning.
4. Orchestrator monitors the MySQL topology.
5. If the Primary fails, the Replica is promoted to Primary.
6. The new Primary continues accepting writes.
7. When the old Primary comes back, it does **not** become Primary again.
8. The old Primary is reconfigured and rejoins as a Replica.
9. The recovered Replica synchronizes all missing transactions from the new Primary.
10. Nginx can provide a reverse-proxy/load-balancing layer for application traffic.

---

# 2. Architecture

## Normal State

```text
                         Application
                             |
                             v
                     Nginx Reverse Proxy
                             |
                             v
                       MySQL Primary
                          :3306
                             |
                       GTID Replication
                             |
                             v
                       MySQL Replica
                          :3306
                             ^
                             |
                       Orchestrator
                         :3000
```

## After Primary Failure

```text
                         Application
                             |
                             v
                     Nginx / Proxy
                             |
                             v
                    MySQL Replica
                    NEW PRIMARY
                         :3306
```

Orchestrator detects the failure and promotes the Replica.

## After Old Primary Recovers

```text
                         Application
                             |
                             v
                     MySQL Replica
                     NEW PRIMARY
                          :3306
                             |
                       GTID Replication
                             |
                             v
                     Old MySQL Primary
                      Rejoined Replica
                          :3306
```

The old Primary is **rejoined as Replica**, not promoted back to Primary.

---

# 3. Technologies Used

| Technology     | Purpose                                |
| -------------- | -------------------------------------- |
| Ubuntu / Linux | Server operating system                |
| Docker         | Containerization                       |
| Docker Compose | Multi-container management             |
| MySQL          | Database                               |
| GTID           | Replication positioning                |
| Orchestrator   | MySQL topology monitoring and failover |
| Nginx          | Reverse proxy and load balancing       |
| Bash           | Automation                             |
| curl           | API testing                            |
| jq             | JSON response processing               |

---

# 4. Project Structure

The main project directory:

```bash
~/docker_project
```

Create it:

```bash
mkdir -p ~/docker_project
cd ~/docker_project
```

Recommended structure:

```text
docker_project/
│
├── docker-compose.yml
├── Dockerfile.rejoin
├── replica.cnf
├── orchestrator.conf.json
│
└── scripts/
    └── rejoin-primary.sh
```

Check the structure:

```bash
tree ~/docker_project
```

If `tree` is not installed:

```bash
sudo apt update
sudo apt install tree -y
```

---

# 5. Prerequisites

Install Docker:

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin -y
```

Start Docker:

```bash
sudo systemctl enable --now docker
```

Check:

```bash
docker --version
docker compose version
```

Test Docker:

```bash
sudo docker run hello-world
```

Optional: allow the current user to run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Then restart the shell/session.

Verify:

```bash
docker ps
```

---

# 6. Create Docker Network

The containers need to communicate with each other.

Create a dedicated network:

```bash
docker network create mysql-ha
```

Verify:

```bash
docker network ls
```

The network should contain:

```text
mysql-ha
```

---

# 7. MySQL Primary Configuration

Create the project directory:

```bash
mkdir -p ~/docker_project
cd ~/docker_project
```

Create the Docker Compose file:

```bash
nano docker-compose.yml
```

The Primary MySQL service should have a unique `server-id`.

Important MySQL settings include:

```text
server-id
log-bin
binlog_format
gtid_mode
enforce_gtid_consistency
log_replica_updates
```

A simplified Primary configuration looks like:

```ini
[mysqld]
server-id=1
log-bin=mysql-bin
binlog_format=ROW

gtid_mode=ON
enforce_gtid_consistency=ON

log_replica_updates=ON
```

The most important settings are:

```text
server-id=1
```

and:

```text
gtid_mode=ON
```

---

# 8. MySQL Replica Configuration

Create the Replica configuration:

```bash
nano replica.cnf
```

Example:

```ini
[mysqld]
server-id=2

log-bin=mysql-bin
binlog_format=ROW

gtid_mode=ON
enforce_gtid_consistency=ON

log_replica_updates=ON
read_only=ON
super_read_only=ON
```

The Replica must have a **different server ID**.

Primary:

```text
server-id=1
```

Replica:

```text
server-id=2
```

---

# 9. Start MySQL Containers

From the project directory:

```bash
cd ~/docker_project
```

Build the environment:

```bash
docker compose build
```

Start the containers:

```bash
docker compose up -d
```

Check:

```bash
docker ps
```

Expected containers:

```text
mysql-primary
mysql-replica
```

Check logs:

```bash
docker logs mysql-primary
```

Replica logs:

```bash
docker logs mysql-replica
```

---

# 10. Enter MySQL Primary

Open the MySQL shell:

```bash
docker exec -it mysql-primary mysql -uroot -p
```

Enter the MySQL root password.

You can also execute a command without entering the shell:

```bash
docker exec mysql-primary mysql -uroot -p'Root@123' \
-e "SELECT VERSION();"
```

> Avoid putting passwords directly on the command line in production because they can appear in shell history/process information.

---

# 11. Verify Primary Configuration

Inside MySQL:

```sql
SHOW VARIABLES LIKE 'server_id';
```

Expected:

```text
server_id = 1
```

Check GTID:

```sql
SHOW VARIABLES LIKE 'gtid_mode';
```

Expected:

```text
ON
```

Check GTID consistency:

```sql
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
```

Expected:

```text
ON
```

Check binary logging:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

Expected:

```text
ON
```

Check binlog format:

```sql
SHOW VARIABLES LIKE 'binlog_format';
```

Expected:

```text
ROW
```

Check:

```sql
SHOW MASTER STATUS;
```

---

# 12. Create Replication User

Enter the Primary:

```bash
docker exec -it mysql-primary mysql -uroot -p
```

Create the replication user:

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'ReplicationPassword';
```

Grant replication permissions:

```sql
GRANT REPLICATION SLAVE, REPLICATION CLIENT
ON *.*
TO 'repl'@'%';
```

Apply:

```sql
FLUSH PRIVILEGES;
```

Verify:

```sql
SELECT user, host
FROM mysql.user
WHERE user='repl';
```

Expected:

```text
repl    %
```

---

# 13. Verify Replica Configuration

Enter the Replica:

```bash
docker exec -it mysql-replica mysql -uroot -p
```

Check server ID:

```sql
SHOW VARIABLES LIKE 'server_id';
```

Expected:

```text
server_id = 2
```

Check GTID:

```sql
SHOW VARIABLES LIKE 'gtid_mode';
```

Expected:

```text
ON
```

Check read-only:

```sql
SELECT @@read_only, @@super_read_only;
```

Expected during normal operation:

```text
read_only = 1
super_read_only = 1
```

This is correct because the Replica should not normally accept application writes.

---

# 14. Configure GTID Replication

On the Replica:

```bash
docker exec -it mysql-replica mysql -uroot -p
```

If an old replication configuration exists, stop replication first:

```sql
STOP REPLICA;
```

Reset replication metadata if this is a fresh/reconfigured Replica:

```sql
RESET REPLICA ALL;
```

Configure the Primary:

```sql
CHANGE REPLICATION SOURCE TO
SOURCE_HOST='mysql-primary',
SOURCE_PORT=3306,
SOURCE_USER='repl',
SOURCE_PASSWORD='ReplicationPassword',
SOURCE_AUTO_POSITION=1;
```

The important option is:

```text
SOURCE_AUTO_POSITION=1
```

This enables GTID automatic positioning.

Start replication:

```sql
START REPLICA;
```

---

# 15. Verify Replication

Run:

```sql
SHOW REPLICA STATUS\G
```

Important fields:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

Both should be:

```text
Yes
```

Also check:

```text
Seconds_Behind_Source
```

Ideally:

```text
0
```

Check replication errors:

```sql
SHOW REPLICA STATUS\G
```

Look at:

```text
Last_IO_Error
Last_SQL_Error
```

They should be empty.

---

# 16. Verify GTID Replication

On Primary:

```sql
SELECT @@gtid_executed;
```

On Replica:

```sql
SELECT @@gtid_executed;
```

The Replica should eventually contain the transactions executed by the Primary.

This confirms that GTID transactions are being replicated.

---

# 17. Create Test Database

On Primary:

```bash
docker exec -it mysql-primary mysql -uroot -p
```

Create database:

```sql
CREATE DATABASE ha_test;
```

Select it:

```sql
USE ha_test;
```

Create table:

```sql
CREATE TABLE test_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(255)
);
```

Insert test data:

```sql
INSERT INTO test_data (message)
VALUES ('Replication test');
```

Insert another test row:

```sql
INSERT INTO test_data (message)
VALUES ('BEFORE FAILOVER');
```

Verify:

```sql
SELECT * FROM test_data;
```

Expected:

```text
+----+-----------------------+
| id | message               |
+----+-----------------------+
|  1 | Replication test      |
|  2 | BEFORE FAILOVER       |
+----+-----------------------+
```

---

# 18. Verify Data on Replica

Enter Replica:

```bash
docker exec -it mysql-replica mysql -uroot -p
```

Run:

```sql
USE ha_test;
```

Then:

```sql
SELECT * FROM test_data;
```

The same rows should appear.

This confirms:

```text
Primary
   |
   | Replication
   v
Replica
```

is working.

---

# 19. Install Orchestrator

Orchestrator is responsible for monitoring the MySQL replication topology and handling failover.

The Orchestrator service was configured to listen on:

```text
3000
```

Check whether port 3000 is listening:

```bash
sudo ss -lntp | grep 3000
```

Expected:

```text
LISTEN ... *:3000
```

---

# 20. Create Orchestrator Database/User

Orchestrator requires a backend database for its metadata.

Enter the MySQL environment:

```bash
mysql -uroot -p
```

Create database:

```sql
CREATE DATABASE orchestrator;
```

Create user:

```sql
CREATE USER 'orchestrator'@'%' IDENTIFIED BY 'OrchestratorPassword';
```

Grant permissions:

```sql
GRANT ALL PRIVILEGES
ON orchestrator.*
TO 'orchestrator'@'%';
```

Apply:

```sql
FLUSH PRIVILEGES;
```

---

# 21. Orchestrator Configuration

The Orchestrator configuration file contains settings for:

* Backend database
* MySQL topology
* HTTP port
* Replication recovery
* Authentication
* Discovery

Example configuration location:

```text
/etc/orchestrator.conf.json
```

or inside the Docker project:

```text
orchestrator.conf.json
```

Check configuration:

```bash
cat orchestrator.conf.json
```

The configuration must point Orchestrator to the correct MySQL instances and backend database.

---

# 22. Start Orchestrator

If Orchestrator is running through Docker:

```bash
docker compose up -d orchestrator
```

Check:

```bash
docker ps
```

Check logs:

```bash
docker logs orchestrator
```

Follow logs:

```bash
docker logs -f orchestrator
```

Stop following with:

```text
Ctrl+C
```

The container itself remains running.

---

# 23. Verify Orchestrator Port

Run:

```bash
sudo ss -lntp | grep 3000
```

Expected:

```text
LISTEN 0 4096 *:3000
```

You can also test:

```bash
curl -I http://localhost:3000
```

---

# 24. Open Orchestrator Web UI

Open:

```text
http://localhost:3000
```

The Orchestrator dashboard should display the discovered MySQL topology.

Expected topology:

```text
mysql-primary:3306
        |
        v
mysql-replica:3306
```

---

# 25. Discover MySQL Instances

Orchestrator must discover the MySQL servers before it can monitor them.

An API request can be used to query an instance:

```bash
curl -s http://localhost:3000/api/instance/mysql-primary:3306
```

Another example:

```bash
curl -s http://localhost:3000/api/instance/mysql-replica:3306
```

If the response is JSON, format it with:

```bash
curl -s http://localhost:3000/api/instance/mysql-replica:3306 | jq
```

---

# 26. Important Orchestrator API Issue — 404

During testing, this command returned:

```bash
curl -s http://localhost:3000/api/instance/mysql-replica:3306 | jq
```

Result:

```text
404
jq: parse error: Invalid numeric literal
```

The problem was **not jq**.

The API returned:

```text
404
```

which is plain text rather than JSON.

Therefore:

```text
API returned 404
       |
       v
jq received "404"
       |
       v
jq failed to parse it as JSON
```

Correct troubleshooting:

```bash
curl -i http://localhost:3000/api/instance/mysql-replica:3306
```

The `-i` option displays the HTTP status.

Then verify:

```bash
curl -s http://localhost:3000/api/replication-analysis
```

Also check:

```bash
docker logs orchestrator --tail 150
```

The important thing is to verify that the instance is actually discovered by Orchestrator before using an instance-specific API endpoint.

---

# 27. Check Orchestrator Replication Analysis

Run:

```bash
curl -s http://localhost:3000/api/replication-analysis
```

If the response is JSON:

```bash
curl -s http://localhost:3000/api/replication-analysis | jq
```

This helps verify what Orchestrator believes about the replication topology.

---

# 28. Check Orchestrator Problems

Run:

```bash
curl -s http://localhost:3000/api/problems
```

Format JSON:

```bash
curl -s http://localhost:3000/api/problems | jq
```

This is useful when troubleshooting:

* Primary failure
* Replica failure
* Replication stopped
* Connectivity problems
* Topology problems

---

# 29. Monitor Orchestrator Logs

During failover testing:

```bash
docker logs -f orchestrator
```

Or:

```bash
docker logs orchestrator --tail 150
```

This is one of the most important troubleshooting commands in the project.

It helps determine whether Orchestrator:

1. Detected the Primary failure.
2. Analyzed the topology.
3. Selected a Replica.
4. Promoted the Replica.
5. Updated the topology.

---

# 30. Test Failover

Before stopping the Primary, insert a test row:

```bash
docker exec mysql-primary mysql -uroot -p'Root@123' \
-e "USE ha_test; INSERT INTO test_data(message) VALUES ('BEFORE FAILOVER');"
```

Verify:

```bash
docker exec mysql-primary mysql -uroot -p'Root@123' \
-e "USE ha_test; SELECT * FROM test_data;"
```

Verify Replica has the same data:

```bash
docker exec mysql-replica mysql -uroot -p'Root@123' \
-e "USE ha_test; SELECT * FROM test_data;"
```

---

# 31. Simulate Primary Failure

Stop the Primary:

```bash
docker stop mysql-primary
```

Check:

```bash
docker ps
```

The Primary should no longer appear as running.

Monitor Orchestrator:

```bash
docker logs orchestrator --tail 150
```

Also open:

```text
http://localhost:3000
```

---

# 32. Orchestrator Failover

After detecting the Primary failure, Orchestrator should promote the Replica.

Before:

```text
mysql-primary
     |
     v
mysql-replica
```

After:

```text
mysql-replica
NEW PRIMARY
```

The old Primary is unavailable.

---

# 33. Verify New Primary

Enter the promoted Replica:

```bash
docker exec -it mysql-replica mysql -uroot -p
```

Check:

```sql
SELECT @@read_only, @@super_read_only;
```

Expected:

```text
@@read_only = 0
@@super_read_only = 0
```

The promoted Replica must be writable.

Check:

```sql
SELECT @@global.gtid_executed;
```

---

# 34. Write Data to New Primary

On the new Primary:

```sql
USE ha_test;
```

Insert:

```sql
INSERT INTO test_data(message)
VALUES ('DATA FROM NEW PRIMARY');
```

Verify:

```sql
SELECT * FROM test_data;
```

Expected:

```text
+----+------------------------+
| id | message                |
+----+------------------------+
|  1 | Replication test       |
|  2 | BEFORE FAILOVER        |
|  3 | DATA FROM NEW PRIMARY  |
+----+------------------------+
```

This confirms that the promoted Replica is now functioning as the Primary.

---

# 35. Important Issue — Replica Still Read-Only

One issue encountered during failover testing was that the Replica could remain:

```text
read_only = 1
```

or:

```text
super_read_only = 1
```

Check:

```bash
docker exec mysql-replica mysql -uroot -p -e \
"SELECT @@read_only, @@super_read_only;"
```

If the server is supposed to be the new Primary, these values must be changed by the failover/promotion mechanism.

For troubleshooting, inspect Orchestrator:

```bash
docker logs orchestrator --tail 200
```

Also check the topology:

```bash
curl -s http://localhost:3000/api/replication-analysis | jq
```

Do not simply use:

```sql
SET GLOBAL read_only=0;
```

as the normal failover mechanism. The objective of this project is for Orchestrator to perform the promotion correctly.

---

# 36. Why the Old Primary Must Not Immediately Become Primary

Suppose:

```text
OLD PRIMARY
A
B
C
```

The Replica receives:

```text
A
B
C
```

After failover, the new Primary writes:

```text
D
E
```

Now:

```text
NEW PRIMARY
A
B
C
D
E
```

while the old Primary has:

```text
OLD PRIMARY
A
B
C
```

If the old Primary is simply started as writable, there can be two different sources of writes.

This can result in:

```text
                 Split Brain
                    /    \
                   /      \
          Old Primary    New Primary
               |              |
             Writes         Writes
```

Therefore the old Primary must be rejoined as a Replica.

---

# 37. Recover the Old Primary

Start the old Primary again:

```bash
docker start mysql-primary
```

Check:

```bash
docker ps
```

Check logs:

```bash
docker logs mysql-primary
```

At this stage, **do not immediately send application writes to it**.

It is the old Primary and must be reconfigured.

---

# 38. Rejoin Script

The custom recovery script is:

```text
scripts/rejoin-primary.sh
```

Check:

```bash
ls -lah scripts/rejoin-primary.sh
```

Expected:

```text
-rwxr-xr-x ... scripts/rejoin-primary.sh
```

If it is not executable:

```bash
chmod +x scripts/rejoin-primary.sh
```

View the script:

```bash
cat scripts/rejoin-primary.sh
```

The script automates the process of converting the recovered old Primary into a Replica of the current Primary.

---

# 39. Dockerfile for Rejoin Process

The project contains:

```text
Dockerfile.rejoin
```

View it:

```bash
cat Dockerfile.rejoin
```

Build the rejoin image:

```bash
docker build -f Dockerfile.rejoin -t mysql-rejoin .
```

Verify:

```bash
docker images | grep mysql-rejoin
```

---

# 40. Rejoin Logic

The recovery process follows this architecture:

```text
Current Primary
mysql-replica
      |
      | GTID replication
      v
Recovered Server
mysql-primary
```

The recovered server must:

1. Stop any old replication configuration.
2. Reset inappropriate old replication metadata if required.
3. Enable Replica mode.
4. Point replication to the current Primary.
5. Enable GTID auto-positioning.
6. Start replication.
7. Verify replication status.

---

# 41. Manual Rejoin Process

If manual troubleshooting is required, enter the recovered server:

```bash
docker exec -it mysql-primary mysql -uroot -p
```

Stop replication if it exists:

```sql
STOP REPLICA;
```

Reset old replication configuration:

```sql
RESET REPLICA ALL;
```

Enable read-only mode:

```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;
```

Configure the new Primary:

```sql
CHANGE REPLICATION SOURCE TO
SOURCE_HOST='mysql-replica',
SOURCE_PORT=3306,
SOURCE_USER='repl',
SOURCE_PASSWORD='ReplicationPassword',
SOURCE_AUTO_POSITION=1;
```

Start replication:

```sql
START REPLICA;
```

---

# 42. Verify Rejoined Replica

Run:

```sql
SHOW REPLICA STATUS\G
```

The expected values:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

Check:

```text
Last_IO_Error
Last_SQL_Error
```

Both should be empty.

Check:

```sql
SELECT @@read_only, @@super_read_only;
```

Expected:

```text
read_only = 1
super_read_only = 1
```

The old Primary is now a Replica.

---

# 43. Verify Rejoined Data

On the current Primary:

```bash
docker exec mysql-replica mysql -uroot -p
```

Run:

```sql
USE ha_test;

SELECT * FROM test_data;
```

Expected:

```text
Replication test
BEFORE FAILOVER
DATA FROM NEW PRIMARY
```

Now check the rejoined Replica:

```bash
docker exec mysql-primary mysql -uroot -p
```

Run:

```sql
USE ha_test;

SELECT * FROM test_data;
```

The same data should exist.

---

# 44. Verify New Transactions Are Replicated

On the current Primary:

```sql
INSERT INTO test_data(message)
VALUES ('REJOIN TEST');
```

Check:

```sql
SELECT * FROM test_data;
```

Wait for replication and check the old Primary:

```bash
docker exec mysql-primary mysql -uroot -p \
-e "USE ha_test; SELECT * FROM test_data;"
```

The new row should appear.

This proves that the recovered server is correctly following the new Primary.

---

# 45. Complete Final Topology

After recovery:

```text
                 Application
                      |
                      v
                  Nginx
                      |
                      v
              CURRENT PRIMARY
              mysql-replica
                   :3306
                      |
                  GTID
                      |
                      v
               OLD PRIMARY
              mysql-primary
                  :3306
                  Replica
                      ^
                      |
                Orchestrator
                  :3000
```

---

# 46. Nginx Reverse Proxy

Nginx can be used as a reverse proxy between the application and backend services.

Install Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

Check:

```bash
sudo systemctl status nginx
```

Enable:

```bash
sudo systemctl enable nginx
```

Start:

```bash
sudo systemctl start nginx
```

Check:

```bash
curl http://localhost
```

---

# 47. Nginx Reverse Proxy Configuration

Create a configuration:

```bash
sudo nano /etc/nginx/sites-available/mysql-ha
```

Example:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/mysql-ha \
/etc/nginx/sites-enabled/mysql-ha
```

Test configuration:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

---

# 48. Nginx Load Balancing

For HTTP application backends, Nginx can distribute requests across multiple servers.

Example:

```nginx
upstream backend {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Test:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

> Nginx HTTP load balancing should be used for application traffic. MySQL failover itself is handled by the database topology/Orchestrator rather than by ordinary HTTP load balancing.

---

# 49. Important Nginx Configuration Commands

Check Nginx status:

```bash
sudo systemctl status nginx
```

Test configuration:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

Restart:

```bash
sudo systemctl restart nginx
```

View logs:

```bash
sudo tail -f /var/log/nginx/access.log
```

Error logs:

```bash
sudo tail -f /var/log/nginx/error.log
```

---

# 50. Troubleshooting — Docker

Check containers:

```bash
docker ps -a
```

Check logs:

```bash
docker logs <container>
```

Example:

```bash
docker logs mysql-primary
```

Restart:

```bash
docker restart mysql-primary
```

Enter container:

```bash
docker exec -it mysql-primary bash
```

Check network:

```bash
docker network inspect mysql-ha
```

---

# 51. Troubleshooting — MySQL

Check MySQL:

```bash
docker exec mysql-primary mysqladmin -uroot -p ping
```

Expected:

```text
mysqld is alive
```

Check Replica:

```bash
docker exec -it mysql-replica mysql -uroot -p
```

Then:

```sql
SHOW REPLICA STATUS\G
```

Important fields:

```text
Replica_IO_Running
Replica_SQL_Running
Last_IO_Error
Last_SQL_Error
Seconds_Behind_Source
```

---

# 52. Troubleshooting — Replica IO Thread

If:

```text
Replica_IO_Running: No
```

check:

```sql
SHOW REPLICA STATUS\G
```

Look at:

```text
Last_IO_Error
```

Common causes:

```text
Primary unavailable
Wrong hostname
Wrong port
Wrong replication username
Wrong password
Network connectivity
Firewall
MySQL not listening on correct interface
```

Test connectivity:

```bash
docker exec mysql-replica getent hosts mysql-primary
```

Test port:

```bash
docker exec mysql-replica bash -c \
"cat < /dev/null > /dev/tcp/mysql-primary/3306"
```

---

# 53. Troubleshooting — Replica SQL Thread

If:

```text
Replica_SQL_Running: No
```

check:

```sql
SHOW REPLICA STATUS\G
```

Look at:

```text
Last_SQL_Error
```

The SQL thread may stop because a replicated transaction cannot be applied.

Do not blindly skip transactions.

First understand the error and correct the underlying data/configuration problem.

---

# 54. Troubleshooting — GTID

Check:

```sql
SHOW VARIABLES LIKE 'gtid%';
```

Important:

```text
gtid_mode
enforce_gtid_consistency
```

Check executed GTIDs:

```sql
SELECT @@gtid_executed;
```

On both servers compare the transaction sets.

---

# 55. Troubleshooting — Orchestrator

Check process:

```bash
ps aux | grep orchestrator
```

Check port:

```bash
sudo ss -lntp | grep 3000
```

Check logs:

```bash
docker logs orchestrator --tail 200
```

Check API:

```bash
curl -i http://localhost:3000
```

Check replication analysis:

```bash
curl -s http://localhost:3000/api/replication-analysis | jq
```

Check problems:

```bash
curl -s http://localhost:3000/api/problems | jq
```

---

# 56. Troubleshooting — Orchestrator Web UI

If:

```text
http://localhost:3000
```

does not work:

### Step 1

Check port:

```bash
sudo ss -lntp | grep 3000
```

### Step 2

Check process:

```bash
ps aux | grep orchestrator
```

### Step 3

Check logs:

```bash
docker logs orchestrator --tail 200
```

### Step 4

Test locally:

```bash
curl -I http://localhost:3000
```

This determines whether the problem is:

```text
Orchestrator service
        OR
Network/access
        OR
Web UI
```

---

# 57. Troubleshooting — Terminal Appears Stuck

If Orchestrator is running in the foreground and the terminal appears stuck, do not immediately assume that the service has failed.

Open another terminal and run:

```bash
sudo ss -lntp | grep 3000
```

If you see:

```text
*:3000
```

the service is listening.

Also check:

```bash
docker ps
```

and:

```bash
docker logs orchestrator --tail 100
```

Use:

```text
Ctrl+C
```

only when you intentionally want to stop a foreground process.

For long-running services, prefer:

```bash
docker compose up -d
```

---

# 58. Troubleshooting — `jq` Parse Error

Example:

```bash
curl -s http://localhost:3000/api/instance/mysql-replica:3306 | jq
```

If output is:

```text
404
jq: parse error
```

do not troubleshoot `jq` first.

Test:

```bash
curl -i http://localhost:3000/api/instance/mysql-replica:3306
```

If HTTP status is:

```text
404
```

the API endpoint or requested instance is not valid.

The correct sequence is:

```text
curl
  |
  v
Check HTTP status
  |
  v
Verify API endpoint
  |
  v
Verify instance discovery
  |
  v
Then use jq
```

---

# 59. Troubleshooting — Primary Does Not Fail Over

If the Primary is stopped:

```bash
docker stop mysql-primary
```

but Replica is not promoted, check:

```bash
docker logs orchestrator --tail 200
```

Then:

```bash
curl -s http://localhost:3000/api/replication-analysis | jq
```

Check Replica:

```bash
docker exec mysql-replica mysql -uroot -p \
-e "SHOW REPLICA STATUS\G"
```

Before failure, the Replica should have been healthy.

Also verify that Orchestrator can connect to the MySQL nodes.

---

# 60. Troubleshooting — Old Primary Cannot Rejoin

Check the recovered server:

```bash
docker exec -it mysql-primary mysql -uroot -p
```

Check:

```sql
SHOW REPLICA STATUS\G
```

If old replication metadata exists:

```sql
STOP REPLICA;
RESET REPLICA ALL;
```

Then configure the current Primary:

```sql
CHANGE REPLICATION SOURCE TO
SOURCE_HOST='mysql-replica',
SOURCE_PORT=3306,
SOURCE_USER='repl',
SOURCE_PASSWORD='ReplicationPassword',
SOURCE_AUTO_POSITION=1;
```

Start:

```sql
START REPLICA;
```

Verify:

```sql
SHOW REPLICA STATUS\G
```

---

# 61. Important Safety Rule During Failover

Never allow both servers to accept application writes at the same time.

Bad:

```text
             Application
              /       \
             v         v
       Old Primary   New Primary
          WRITE         WRITE
```

Correct:

```text
             Application
                  |
                  v
             New Primary
                  |
               Replica
                  |
             Old Primary
```

Only one server should be the authoritative writable Primary.

---

# 62. Complete Testing Procedure

## Test 1 — Replication

Insert on Primary:

```sql
INSERT INTO test_data(message)
VALUES ('Replication test');
```

Check Replica:

```sql
SELECT * FROM test_data;
```

Expected: data appears.

---

## Test 2 — Primary Failure

Stop Primary:

```bash
docker stop mysql-primary
```

Check Orchestrator:

```bash
docker logs orchestrator --tail 200
```

Check topology:

```text
Replica -> New Primary
```

---

## Test 3 — New Primary Write

On new Primary:

```sql
INSERT INTO test_data(message)
VALUES ('DATA FROM NEW PRIMARY');
```

Check:

```sql
SELECT * FROM test_data;
```

---

## Test 4 — Old Primary Recovery

Start:

```bash
docker start mysql-primary
```

Do not use it for application writes.

Rejoin it as Replica.

---

## Test 5 — Rejoin Verification

On old Primary:

```sql
SHOW REPLICA STATUS\G
```

Verify:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

---

## Test 6 — Replication After Rejoin

On current Primary:

```sql
INSERT INTO test_data(message)
VALUES ('REJOIN TEST');
```

Then on old Primary:

```sql
SELECT * FROM test_data;
```

The new row should appear.

---

# 63. Useful Commands Summary

## Docker

```bash
docker ps
docker ps -a
docker compose up -d
docker compose down
docker compose restart
docker logs <container>
docker exec -it <container> bash
```

## MySQL

```bash
docker exec -it mysql-primary mysql -uroot -p
docker exec -it mysql-replica mysql -uroot -p
```

## Replication

```sql
SHOW REPLICA STATUS\G;
START REPLICA;
STOP REPLICA;
RESET REPLICA ALL;
SELECT @@gtid_executed;
SELECT @@read_only, @@super_read_only;
```

## Orchestrator

```bash
sudo ss -lntp | grep 3000

docker logs orchestrator --tail 200

curl -i http://localhost:3000

curl -s http://localhost:3000/api/replication-analysis | jq

curl -s http://localhost:3000/api/problems | jq
```

## Nginx

```bash
sudo systemctl status nginx
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl restart nginx
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

---

# 64. Final Architecture

The completed system works as follows:

```text
                         CLIENT
                           |
                           v
                    NGINX / PROXY
                           |
                           v
                  +----------------+
                  | CURRENT PRIMARY |
                  |    MySQL        |
                  +----------------+
                           |
                     GTID Replication
                           |
                           v
                  +----------------+
                  |     REPLICA    |
                  |    MySQL       |
                  +----------------+
                           ^
                           |
                     ORCHESTRATOR
                      Port 3000
```

When Primary fails:

```text
                  PRIMARY
                     X
                     |
                     |
               Failure Detected
                     |
                     v
               ORCHESTRATOR
                     |
                     v
                  REPLICA
                     |
                     v
               NEW PRIMARY
```

When the old Primary recovers:

```text
                NEW PRIMARY
                mysql-replica
                     |
                 GTID Sync
                     |
                     v
                OLD PRIMARY
                mysql-primary
                   Replica
```

---

# 65. Project Outcome

The project successfully demonstrates:

* MySQL Primary-Replica replication
* GTID-based replication
* Automatic replication positioning
* Docker-based MySQL infrastructure
* Orchestrator topology monitoring
* Primary failure detection
* Replica promotion
* New Primary write operations
* Old Primary recovery
* Old Primary rejoin as Replica
* Post-failover data synchronization
* Nginx reverse proxy
* Nginx load balancing
* Bash automation
* Database High Availability concepts
* Split-brain prevention
* Failover troubleshooting

The final HA workflow is:

```text
             NORMAL OPERATION
                    |
                    v
          Primary -> Replica
                    |
                    v
             Primary Failure
                    |
                    v
          Orchestrator Detection
                    |
                    v
            Replica Promotion
                    |
                    v
              New Primary
                    |
                    v
             Application Works
                    |
                    v
           Old Primary Recovers
                    |
                    v
             Rejoin as Replica
                    |
                    v
              GTID Catch-up
                    |
                    v
             Healthy HA Setup
```

---

# 66. Key Lessons Learned

### 1. Replication alone is not High Availability

Having:

```text
Primary -> Replica
```

does not automatically provide complete HA.

Failover and recovery mechanisms are also required.

### 2. GTID simplifies failover

GTID allows the Replica to identify transactions automatically and makes rejoining easier.

### 3. Orchestrator manages topology

Orchestrator provides monitoring and automated recovery capabilities instead of requiring manual promotion every time.

### 4. Old Primary must be rejoined safely

A recovered Primary should not simply be made writable again.

It must first synchronize with the current Primary.

### 5. Split brain must be avoided

Only one MySQL server should act as the authoritative writable Primary.

### 6. Logs are essential

The most useful troubleshooting commands were:

```bash
docker logs mysql-primary
docker logs mysql-replica
docker logs orchestrator
```

along with:

```sql
SHOW REPLICA STATUS\G;
```

### 7. API errors should be debugged from the HTTP response first

If:

```text
curl -> 404
jq -> parse error
```

the real problem is the API returning `404`, not `jq`.

---

# 67. Final Verification

Before considering the project complete, run:

```bash
docker ps
```

Then verify replication:

```bash
docker exec mysql-primary mysql -uroot -p \
-e "SELECT @@read_only, @@super_read_only;"
```

Verify Replica:

```bash
docker exec mysql-primary mysql -uroot -p \
-e "SHOW REPLICA STATUS\G"
```

Verify Orchestrator:

```bash
sudo ss -lntp | grep 3000
```

Verify API:

```bash
curl -s http://localhost:3000/api/replication-analysis | jq
```

Verify database:

```bash
docker exec mysql-primary mysql -uroot -p \
-e "USE ha_test; SELECT * FROM test_data;"
```

If the topology is:

```text
CURRENT PRIMARY
      |
      v
REJOINED REPLICA
```

and:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

then the MySQL HA replication environment is healthy.

---

# Conclusion

This project implements a practical **MySQL High Availability architecture using Docker, GTID replication, Orchestrator, automated failover/recovery, and Nginx**.

The most important workflow is:

```text
Primary
   |
   | GTID Replication
   v
Replica
   |
   | Primary Failure
   v
Orchestrator
   |
   | Promotion
   v
Replica becomes Primary
   |
   | Old Primary Recovery
   v
Old Primary rejoins as Replica
   |
   | GTID Synchronization
   v
Healthy Replication Topology
```

This provides a complete hands-on demonstration of **database replication, automatic failover, recovery, topology management, reverse proxying, load balancing, and High Availability principles**.
