#  MySQL Master-Slave Replication in Docker

Here is my step-by-step guide and documentation for setting up MySQL Master-Slave replication using Docker containers. I have included all the commands I ran and the corresponding console outputs.

---

## Architecture
* **Master (`replication-master`)**: My primary database server (`server_id = 1`) that handles data writes and generates binary logs (`log_bin`).
* **Slave (`replication-slave`)**: My replica database server (`server_id = 2`) that reads binary logs from the master and synchronizes data (`read_only = 1`).

---

##  Step-by-Step Deployment

### 1. Launching MySQL Containers

First, I run two MySQL 8.3 containers with identical root passwords (`123456`):

```bash
docker run --name replication-master -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.3
Bash
docker run --name replication-slave -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.3
Then I check the running container status using docker ps:

Bash
docker ps
Console Output:

Plaintext
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                  NAMES
b3ebb389078d   mysql:8.3  "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   3306/tcp, 33060/tcp    replication-slave
226e9fb5bbf0   mysql:8.3  "docker-entrypoint.s…"   5 minutes ago   Up 5 minutes   3306/tcp, 33060/tcp    replication-master
2. Creating a Network and Connecting Containers
I create an isolated Docker network so the containers can communicate by their names:

Bash
docker network create replication
Then I connect both containers to this network:

Bash
docker network connect replication replication-master
Bash
docker network connect replication replication-slave
3. Configuring the Master Server
I access the Master container shell:

Bash
docker exec -it replication-master mysql -u root -p
(I enter root password: 123456)

I create the replication user with the standard mysql_native_password plugin to prevent authentication errors in Docker, and I grant replication privileges across all databases (*.*):

SQL
DROP USER IF EXISTS 'replication'@'%';
SQL
CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
SQL
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
SQL
FLUSH PRIVILEGES;
I verify the granted privileges:

SQL
SHOW GRANTS FOR 'replication'@'%';
Console Output:

Plaintext
+-----------------------------------------------------+
| Grants for replication@%                            |
+-----------------------------------------------------+
| GRANT REPLICATION SLAVE ON *.* TO `replication`@`%` |
+-----------------------------------------------------+
1 row in set (0.00 sec)
I check the Master's current binary log status to get the file name and position:

SQL
SHOW MASTER STATUS;
Console Output:

Plaintext
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     1008 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set, 1 warning (0.00 sec)
(I exit the Master shell using exit)

4. Configuring the Slave Server
I access the Slave container shell:

Bash
docker exec -it replication-slave mysql -u root -p
(I enter root password: 123456)

I configure connection settings to point to my Master server using the log file and position from the previous step:

SQL
CHANGE MASTER TO 
  MASTER_HOST='replication-master', 
  MASTER_USER='replication', 
  MASTER_PASSWORD='123456', 
  MASTER_LOG_FILE='mysql-bin.000001', 
  MASTER_LOG_POS=1008;
Console Output:

Plaintext
Query OK, 0 rows affected, 8 warnings (0.01 sec)
I start the replication process:

SQL
START REPLICA;
Console Output:

Plaintext
Query OK, 0 rows affected (0.00 sec)
I check the replication status:

SQL
SHOW SLAVE STATUS\G
Console Output:

Plaintext
*************************** 1. row ***************************
             Slave_IO_State: Waiting for source to send event
                Master_Host: replication-master
                Master_User: replication
                Master_Port: 3306
              Connect_Retry: 60
            Master_Log_File: mysql-bin.000001
        Read_Master_Log_Pos: 1889
             Relay_Log_File: mysql-relay-bin.000003
              Relay_Log_Pos: 328
        Relay_Master_Log_File: mysql-bin.000001
           Slave_IO_Running: Yes
          Slave_SQL_Running: Yes
       Seconds_Behind_Master: 0
                     Last_IO_Errno: 0
                     Last_IO_Error: 
                    Last_SQL_Errno: 0
                    Last_SQL_Error: 
(Note: Slave_IO_Running: Yes, Slave_SQL_Running: Yes, and Seconds_Behind_Master: 0 mean my replication is fully operational).

5. Testing Data Replication
I create a test database named sakila on my Master:

SQL
CREATE DATABASE sakila;
SQL
SHOW DATABASES;
Result on Master:

Plaintext
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sakila             |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
I check the database list on my Slave to verify automatic synchronization:

SQL
SHOW DATABASES;
Result on Slave:

Plaintext
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sakila             |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
