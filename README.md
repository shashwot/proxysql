# What is ProxySql ?
ProxySql serves as an intermediary between a MySQL server and the applications that access its databases. ProxySQL can improve performance by distributing traffic among a pool of multiple database servers and also improve availability by automatically failing over to a standby if one or more of the database servers fail.

# What is Mysql Replication?
MySQL database replication provides the facility to make replicas of databases. The ability to make exact copies of databases and keep them in real-time sync as changes are made at the “master” provides a number of advantages. In summary:
- Scaling out a database application
- Reducing database backup impact
- Facilitate reporting without affecting production load
- Failover/High Availability

<br>

# Configuration of Mysql Replication
**IN MASTER NODE**

Uncomment or add the line in the file **/etc/mysql/mysql.conf.d/mysqld.conf**

```
server-id = 1
```
Then restart the mysql-server.
```
$ systemctl restart mysql
```

### Creating a Replication User
```
$ mysql -u root -p

mysql(master) > create user 'replica_user'@'replica_server_ip' IDENTIFIED WITH mysql_native_password BY 'password';

mysql(master) > grant replication slave on *.* to 'replica_user'@'replica_server_ip';

mysql(master) > flush privileges;
```
<br>

### Retrieve Binary Log Co-ordinates from Source
When using MySQL’s binary log file position-based replication, you must provide the replica with a set of coordinates that detail the name of the source’s binary log file and a specific position within that file. The replica then uses these coordinates to determine the point in the log file from which it should begin copying database events and track which events it has already processed.
```
mysql(master) > flush tables with read lock;
mysql(master) > show master status

Output:
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      899 | db           |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

### Migrate Data From Master To Slave
```
$ mysqldump -u root -p –all-databases > data.sql
$ scp data.sql root@<slave-ip-address>:/tmp/
$ unlock tables;
```

**IN SLAVE NODE**

Uncomment or add the line in the file **/etc/mysql/mysql.conf.d/mysqld.conf**

```
server-id = 2
```
Then restart the mysql-server.

```
$ systemctl restart mysql
```

```
$ cd /tmp/
$ mysql -u root -p <data.sql

$ mysql -u root -p

mysql(slave) > select uuid();

Output:
+--------------------------------------+
| uuid()                               |
+--------------------------------------+
| 45j52962b-l236-14tg-au71-0244ac123093 |
+--------------------------------------+
1 row in set (0.00 sec)

Copy the content from the uuid and change it to the file location:
$ vi /var/lib/mysql/auto.cnf

[auto]
server-uuid=45j52962b-l236-14tg-au71-0244ac123093
:wq

$ systemctl restart mysql

mysql(slave) > change master to 
> master_host='<master-host-ip>',
> master_user='replica_user',
> master_password='password',
> master_log_file='mysql-bin.000001',
> master_log_pos=899;

mysql(slave) > START REPLICA;

mysql(slave) > SHOW REPLICA STATUS \G;
Output
*************************** 1. row ***************************
             Replica_IO_State: Waiting for master to send event
                  Source_Host: x.x.x.x
                  Source_User: replica_user
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000001
          Read_Source_Log_Pos: 1111
               Relay_Log_File: mysql-relay-bin.000003
                Relay_Log_Pos: 111
        Relay_Source_Log_File: mysql-bin.000001
```

<br>

# Installation of Proxy Sql

Assuming that the Operating Sytem being used is ubuntu:
```
$ wget https://github.com/sysown/proxysql/releases/download/v2.2.0/proxysql_2.2.0-ubuntu20_amd64.deb

$ dpkg -i proxysql_2.2.0-ubuntu20_amd64.deb

$ systemctl start proxtsql
```

By default, ProxySql interface listens on port 6032. We can connect to proxysql using:

```
$ mysql -u admin -padmin -h 127.0.0.1 -P6032

mysql(proxy) > show databases;
```

### Define HostGroups
Proxysql uses a concept of hostgroups - a group of backend which serve the same purpose or handle similar type of traffic.When it comes to replication, at a minimum two types of backends come to mind - a master, which handles writes (and also reads, if needed) and slave (or slaves), which handles read-only traffic.
<br>
```
mysql(proxy) > INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_replication_lag) VALUES (0, '172.17.0.2', 3306, 20);

mysql(proxy) > INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_replication_lag) VALUES (1, '172.17.0.3', 3306, 20);
```
What we did here is create 2 backend with ip 172.17.0.2 and 172.17.0.3. The first was assigned hostgroup '0' acts as a master whereas the hostgroup with id '1' acts as slave. 

<br>

### Define Application User
We create a user in proxysql, it authenticates against it and then makes a suitable connection to the backend server. 
```
mysql(proxy) > INSERT INTO mysql_users (username, password, active, default_hostgroup, max_connections) VALUES ('proxyuser', 'proxypassword', 1, 0, 200);
```

### Define Query Rule
ProxySQL uses query rules to route traffic. Those rules can distribute traffic across multiple hostgroups. In our case we define two rules, one is to read and another is to write.
#### WRITE
We are checking if the query is not by chance a select ... for update. If it is, we will route it to hostgroup '0' which is write.
```
mysql(proxy) > INSERT INTO mysql_query_rules (active, match_pattern, destination_hostgroup, cache_ttl) VALUES (1, '^SELECT .* FOR UPDATE', 0, NULL);
```

#### READ
Here we are checking if the query is a regular select. If so, it will be routed to the hostgroup '1' which is the read hostgroup.
```
mysql(proxy) > INSERT INTO mysql_query_rules (active, match_pattern, destination_hostgroup, cache_ttl) VALUES (1, '^SELECT .*', 1, NULL);
```
<br>

## Setup Monitoring Module And HealthCheck Timeout

Lets create a user for monitoring in the **master** mysql server instance.

```
mysql(master) > create user monuser@'ip' identified by 'monuser';
mysql(master) > grant replication client on *.* to monuser@'ip';
```
<br>

Now lets assign the correct username and password to the monitoring module.
```
mysql(proxy) > SET mysql-monitor_username='monuser';

mysql(proxy) > SET mysql-monitor_password='monuser';

mysql(proxy) > SET mysql-connect_timeout_server_max=20000;
```

## Define read, write hostgroups and load the configs

The final step is to let ProxySql know which hostgroup or slave and master should belong to by adding the entries. ProxySql also doesn't update runtime configuration when you make changes, so we make them persistance by saving them on the disk.

```
mysql(proxy) > INSERT INTO mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup) VALUES (0, 1);

mysql(proxy) > LOAD MYSQL USERS TO RUNTIME;
mysql(proxy) > SAVE MYSQL USERS TO DISK;

mysql(proxy) > LOAD MYSQL QUERY RULES TO RUNTIME;
mysql(proxy) > SAVE MYSQL QUERY RULES TO DISK;

mysql(proxy) > LOAD MYSQL VARIABLES TO RUNTIME;
mysql(proxy) > SAVE MYSQL VARIABLES TO DISK;

mysql(proxy) > LOAD MYSQL SERVERS TO RUNTIME;
mysql(proxy) > SAVE MYSQL SERVERS TO DISK;
```
<br>

# CONCLUSION:
In this way we have setup MySql Replication as well as ProxySql. The procedure outlined in this guide represents only one way of configuring the replication and proxy sql in MySql.
