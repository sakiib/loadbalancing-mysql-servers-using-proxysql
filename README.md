# DIY Doc - Setup & Configure Load-Balancing, Read-Write Query Splitting for MySQL Servers using ProxySQL 

## Setup MySQL Servers (Master-Slave replication)

### Create a Docker network

```bash
$ docker network create proxysql

```

### Create 3 MySQL containers (1 master, 2 slaves)

```bash
# create the master node

$ docker run -d --rm --name=master --net=proxysql --hostname=master \
  -e MYSQL_ROOT_PASSWORD=mypass \
  --health-cmd='mysqladmin ping -u root -p$${MYSQL_ROOT_PASSWORD}' \
  --health-start-period=10s \
  mysql:5.7.26 \
  --server-id=3 \
  --log-bin='mysql-bin-1.log' \
  --relay_log_info_repository=TABLE \
  --master-info-repository=TABLE \
  --gtid-mode=ON \
  --log-slave-updates=ON \
  --enforce-gtid-consistency

```

```bash
# create 2 slave nodes

$ for N in 1 2
    do docker run -d --rm --name=slave$N --net=proxysql --hostname=slave$N \
        -e MYSQL_ROOT_PASSWORD=mypass \
        --health-cmd='mysqladmin ping -u root -p$${MYSQL_ROOT_PASSWORD}' \
        --health-start-period=10s \
        mysql:5.7.26 \
        --server-id=$N \
        --relay_log_info_repository=TABLE \
        --master-info-repository=TABLE \
        --gtid-mode=ON \
        --enforce-gtid-consistency \
        --skip-log-slave-updates \
        --skip-log-bin \
        --read_only=TRUE
    done

```

### Wait until the MySQL container are in `healthy`:

```bash
$ docker ps 

CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS                        PORTS                       NAMES
ffb1272eb7e1   mysql:5.7.26           "docker-entrypoint.s…"   About a minute ago   Up About a minute (healthy)   3306/tcp, 33060/tcp         slave2
94e31a11b9ea   mysql:5.7.26           "docker-entrypoint.s…"   About a minute ago   Up About a minute (healthy)   3306/tcp, 33060/tcp         slave1
b773496576e5   mysql:5.7.26           "docker-entrypoint.s…"   About a minute ago   Up About a minute (healthy)   3306/tcp, 33060/tcp         master

```


### Configure master slaves replication

```bash
$ docker exec -it master mysql -uroot -pmypass \
    -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepass';" \
    -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
    -e "SHOW MASTER STATUS;"

+--------------------+----------+--------------+------------------+------------------------------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+--------------------+----------+--------------+------------------+------------------------------------------+
| mysql-bin-1.000003 |      635 |              |                  | 1cd70a23-b5f3-11ed-935a-0242ac140002:1-7 |
+--------------------+----------+--------------+------------------+------------------------------------------+

```

```bash
for N in 1 2
  do docker exec -it slave$N mysql -uroot -pmypass \
    -e "CHANGE MASTER TO MASTER_HOST='master', MASTER_USER='repl', \
    MASTER_PASSWORD='slavepass', MASTER_AUTO_POSITION = 1;"

docker exec -it slave$N mysql -uroot -pmypass -e "START SLAVE;"
done

```

### 4. Check slaves replication status

```bash
$ docker exec -it slave1 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"

$ docker exec -it slave2 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"

```
---

### 5. Grant access to proxySQL in MySQL

Run the command below to grant access to `monitor` user:

```bash
# this user is presnt in proxysql mysql_servers table
# just create it in the mysql db & give necessary permissions
# change the monitor username & password if it's something different than monitor:monitor

$ docker exec -it master mysql -uroot -pmypass \
  -e "CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';" \
  -e "GRANT ALL PRIVILEGES on *.* TO 'monitor'@'%';" \
  -e "FLUSH PRIVILEGES;"

```

### Launch ProxySQL container

```bash
# using the proxysql.cnf file

$ docker run -d --rm -p 16032:6032 -p 16033:6033 \
  --name=proxysql --net=proxysql \
  -v $PWD/proxysql.cnf:/etc/proxysql.cnf proxysql/proxysql

```

At this point, we have MySQL server & ProxySQL ready.

---

## Configure ProxySQL

### Insert MySQL Servers info in ProxySQL

```bash
# to exec into proxysql
$ docker exec -it proxysql bash

$ mysql -uadmin1 -padmin1 -h127.0.0.1 -P6032 --prompt='ProxySQL Admin> '

# default weight is 1
# default port is 3306, change if needed
# hostgroup_id can be any value (note that: you must set read/write hostgroup id based on this one)

# we're using Writer hostgroup = 1
$ insert into mysql_servers (hostgroup_id, hostname, status, max_connections, comment) values (1, 'master', 'ONLINE', 100, 'read/write');

# we're using ReadOnly hostgroup = 2
$ insert into mysql_servers (hostgroup_id, hostname, status, max_connections, comment) values (2, 'slave1', 'ONLINE', 100, 'readonly');

$ insert into mysql_servers (hostgroup_id, hostname, status, max_connections, comment) values (2, 'slave2', 'ONLINE', 100, 'readonly');

``` 

### Verify the insertion

```bash
$ select * from mysql_servers;

+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------+
| hostgroup_id | hostname | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment    |
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------+
| 1            | master   | 3306 | 0         | ONLINE | 1      | 0           | 100             | 0                   | 0       | 0              | read/write |
| 2            | slave1   | 3306 | 0         | ONLINE | 1      | 0           | 100             | 0                   | 0       | 0              | readonly   |
| 2            | slave2   | 3306 | 0         | ONLINE | 1      | 0           | 100             | 0                   | 0       | 0              | readonly   |
+--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+------------+

```
### Define the Writer/Reader hostgroup

```bash
$ insert into mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup, check_type) values (1, 2, 'read_only');

```

### Create the App User who has privilege to connecto to MySQL via ProxySQL

```bash
# we're creating an user `app` with password `1234`

$ docker exec -it master mysql -uroot -pmypass \
  -e "CREATE USER 'app'@'%' IDENTIFIED BY '1234';" \
  -e "GRANT ALL PRIVILEGES on *.* TO 'app'@'%';" \
  -e "FLUSH PRIVILEGES;"

```

### Whitelist this created user to ProxySQL 

```bash
# default hostgroup is set to 1 (read/write)
$ insert into mysql_users (username, password, active, default_hostgroup, max_connections) values ('app', '1234', 1, 1, 100);

```

### Backend Health Check

```bash

$ SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC;

+----------+------+------------------+-------------------------+---------------+
| hostname | port | time_start_us    | connect_success_time_us | connect_error |
+----------+------+------------------+-------------------------+---------------+
| master   | 3306 | 1677430997095253 | 1631                    | NULL          |
| slave1   | 3306 | 1677430997078143 | 1181                    | NULL          |
| slave2   | 3306 | 1677430997061091 | 1414                    | NULL          |
+----------+------+------------------+-------------------------+---------------+

$ SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC;

+----------+------+------------------+----------------------+------------+
| hostname | port | time_start_us    | ping_success_time_us | ping_error |
+----------+------+------------------+----------------------+------------+
| slave2   | 3306 | 1677430997091975 | 403                  | NULL       |
| master   | 3306 | 1677430997076445 | 342                  | NULL       |
| slave1   | 3306 | 1677430997060876 | 405                  | NULL       |
+----------+------+------------------+----------------------+------------+

```

### Check MySQL Connection Pool

```bash
$ select hostgroup, srv_host, srv_port, status, ConnOK, ConnERR, MaxConnUsed, Queries, Bytes_data_sent, Bytes_data_recv, Latency_us from stats_mysql_connection_pool;

+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+
| hostgroup | srv_host | srv_port | status | ConnOK | ConnERR | MaxConnUsed | Queries | Bytes_data_sent | Bytes_data_recv | Latency_us |
+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+
| 1         | master   | 3306     | ONLINE | 0      | 0       | 0           | 0       | 0               | 0               | 353        |
| 2         | slave1   | 3306     | ONLINE | 0      | 0       | 0           | 0       | 0               | 0               | 363        |
| 2         | slave2   | 3306     | ONLINE | 0      | 0       | 0           | 0       | 0               | 0               | 430        |
+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+

```

At this point, we're ready to connecto to MySQL via ProxySQL & test the configuration.

---

## Connect to MySQL via ProxySQL & Test

### Create a DB, Table & run some queries

```bash
# note that: we're using the port 6033
# we're using the user app that we gave permission
$ docker exec -it proxysql bash
# check that you can see the mysql dbs

$ show databases;
$ mysql -u app -p1234 -h 127.0.0.1 -P6033 --prompt='MySQL via ProxySQL> '

# create a db, make some queries
$ create database test;
$ use test;
$ create table sre(id int);

$ insert into sre values (1);
$ insert into sre values (1);
$ insert into sre values (1);

$ select * from test.sre;

```

### Check the query distribution

```bash
# log back to ProxySQL & check the query distribution
# currently, all the queries are executed on the master node
select hostgroup, srv_host, srv_port, status, ConnOK, ConnERR, MaxConnUsed, Queries, Bytes_data_sent, Bytes_data_recv, Latency_us from stats_mysql_connection_pool;

+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+
| hostgroup | srv_host | srv_port | status | ConnOK | ConnERR | MaxConnUsed | Queries | Bytes_data_sent | Bytes_data_recv | Latency_us |
+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+
| 1         | master   | 3306     | ONLINE | 1      | 0       | 1           | 15      | 296             | 267             | 224        |
| 2         | slave1   | 3306     | ONLINE | 0      | 0       | 0           | 0       | 0               | 0               | 334        |
| 2         | slave2   | 3306     | ONLINE | 0      | 0       | 0           | 0       | 0               | 0               | 350        |
+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+


```

We can see that, all the queries are run on the master node.

---

### Configure a Query Rule to Route the SELECT queries to Read replicas

```bash
# notice that, we're adding a new rule to route SELECT queries to hostgroup 2, which is readonly
$ insert into mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply ) values (1, 1, '^SELECT .*', 2, 1);


$ SELECT rule_id, match_digest, match_pattern, replace_pattern, cache_ttl, apply FROM mysql_query_rules ORDER BY rule_id;

```

### Connect to MySQL via ProxySQL & run some read queries 

```bash
$ docker exec -it proxysql bash
$ mysql -u app -p1234 -h 127.0.0.1 -P6033 --prompt='MySQL via ProxySQL> '

$ select * from test.sre;

```

### Connect to ProxySQL & Check the Query Distribution

```bash
$ docker exec -it proxysql bash
$ mysql -uadmin1 -padmin1 -h127.0.0.1 -P6032 --prompt='ProxySQL Admin> '

$ select hostgroup, srv_host, srv_port, status, ConnOK, ConnERR, MaxConnUsed, Queries, Bytes_data_sent, Bytes_data_recv, Latency_us from stats_mysql_connection_pool;

+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+
| hostgroup | srv_host | srv_port | status | ConnOK | ConnERR | MaxConnUsed | Queries | Bytes_data_sent | Bytes_data_recv | Latency_us |
+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+
| 1         | master   | 3306     | ONLINE | 1      | 0       | 1           | 15      | 296             | 267             | 250        |
| 2         | slave1   | 3306     | ONLINE | 1      | 0       | 1           | 3       | 66              | 54              | 269        |
| 2         | slave2   | 3306     | ONLINE | 1      | 0       | 1           | 4       | 88              | 72              | 292        |
+-----------+----------+----------+--------+--------+---------+-------------+---------+-----------------+-----------------+------------+


```

We can see that the read queries have been distributed fairly among only the slave nodes. No select query was redirected to the master node.

---

### Configure Load between two replicas by setting the weight

```bash
# we can also configure load among instances by setting weights

$ update mysql_servers set weight = 100 where hostname='slave1';

```

---

## Notes to self:

- Run sysbench tests
- Do analysis & Find expensive queries
- Generic Read/Write split using regex
- Intelligent Read/Write split using regex and digest


### Important tables to configure

```bash
# add the mysql servers here
- mysql_servers
# users who are allowed to use proxysql to connect to mysql
- mysql_users
# we'll have reader, writer hostgroups
- mysql_replication_hostgroups
# define the query rules
- mysql_query_rules

```


### Persist the changes

```bash
$ load mysql servers to runtime;
$ save mysql servers to disk;
$ load mysql users to runtime;
$ save mysql users to disk;
$ load mysql variables to runtime;
$ save mysql variables to disk;

```

## Resources:
- https://proxysql.com/
- https://github.com/akopytov/sysbench