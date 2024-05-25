# Mysql Configuration: Master - Slave

![alt text](images/master-slave.png)!

### references:
- https://phoenixnap.com/kb/mysql-master-slave-replication

- https://www.digitalocean.com/community/tutorials/how-to-set-up-mysql-master-master-replication

- https://hevodata.com/learn/mysql-master-slave-replication/

- https://www.digitalocean.com/community/tutorials/how-to-set-up-replication-in-mysql

## 1. MASTER

#### 1.0 install mysql:

```
sudo apt-get update

sudo apt-get install mysql-server mysql-client

sudo mysql
mysql>  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'root';
```

#### 1.1 check and edit configuration file MASTER:

```
- files/master-mysql.cnf

- sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

#### 1.2 create user replicator user:

```
# create replicator user:
mysql>  create user 'replicator'@'%' identified by '123456';

# grant permissions to replicator:
mysql>  grant replication slave on *.* to 'replicator'@'%';

mysql>  flush privileges;
```
#### 1.3 check status:

```
mysql>  show master status;

# Result
+------------------+----------+------------------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB           | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+------------------------+------------------+-------------------+
| mysql-bin.000001 |   154271 | critical_db,testing_db |                  |                   |
+------------------+----------+------------------------+------------------+-------------------+

```

#### 1.4  take database backup and passing to slave server

```
mysqldump -u root -p critical_db > critical_db.sql

scp critical_db.sql deploy@192.168.56.32:/home/deploy
```

#### 1.5 Add slave to server (not run when using join command on SLAVE):
```
# step1:
# SOURCE_LOG_FILE: from SLAVE
# SOURCE_LOG_POS: from SLAVE
mysql>  CHANGE REPLICATION SOURCE TO
        SOURCE_HOST='192.168.56.32',
        SOURCE_USER='replicator',
        SOURCE_PASSWORD='123456',
        SOURCE_LOG_FILE='mysql-bin.000001',
        SOURCE_LOG_POS=154;

# step2:
mysql>  START REPLICA;

# step3:
mysql>  SHOW SLAVE HOSTS | SHOW REPLICAS;

# Result
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|         2 |      | 3306 |         1 | 4785f23d-1aa8-11ef-9224-023b7bb73b2d |
+-----------+------+------+-----------+--------------------------------------+
```
#### 1.6 check SLAVE status:
```
mysql>  SHOW SLAVE HOSTS | SHOW REPLICAS;

# Result
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|         2 |      | 3306 |         1 | 4785f23d-1aa8-11ef-9224-023b7bb73b2d |
+-----------+------+------+-----------+--------------------------------------+
```

## 2. SLAVE

#### 2.0 install mysql:

```
sudo apt-get update

sudo apt-get install mysql-server mysql-client

sudo mysql

mysql>  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'root';

# block read in master:
mysql>  FLUSH TABLES WITH READ LOCK;
```
#### 2.1 check master status before edit config:

```
mysql>  SHOW MASTER STATUS;

# result:
Empty set (0.00 sec)
```

#### 2.2 check and edit configuration file for SLAVE :

```
- files/slave01-mysql.cnf

- sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf

- sudo systemctl restart mysql
```

#### 2.3 Recheck master status before edit config:

```
mysql>  SHOW MASTER STATUS;

# result:
+------------------+----------+------------------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB           | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+------------------------+------------------+-------------------+
| mysql-bin.000001 |      154 | critical_db,testing_db |                  |                   |
+------------------+----------+------------------------+------------------+-------------------+
```

#### 2.4 import database;

```
# inside mysql:
mysql>  source critical_db.sql

# outside mysql:
mysql -u root -p critical_db < /opt/critical_db.sql

```

#### 2.5 joining to Master (not run when add slave on MASTER):
```
# 1:
mysql>  STOP SLAVE;

# 2:
# MASTER_LOG_FILE: From the Master Server
# MASTER_LOG_POS: From the Master Server
mysql>  CHANGE MASTER TO
        MASTER_HOST='192.168.56.31',
        MASTER_USER='replicator',
        MASTER_PASSWORD='123456',
        MASTER_LOG_FILE='mysql-bin.000001',
        MASTER_LOG_POS=155477;

# 3:
mysql>  START SLAVE;
```

#### 2.6 check MASTER status:
```
mysql>  SHOW MASTER STATUS\G

# Result:
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 154
     Binlog_Do_DB: critical_db,testing_db
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```

## 3. checking status replication:

#### On MASTER create new tables and then check result on SLAVE:

#### 3.1: On MASTER create new tables
```
# MASTER:
mysql>  use critical_db;

mysql>  create table test(job varchar(20), salary int(10));

mysql>  show tables;
```

#### 3.2 On SLAVE check result:
```
# SLAVE
mysql>  use critical_db;

mysql>  show tables;
```

[alt text](images/master-master.png)