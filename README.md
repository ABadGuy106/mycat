# MYCAT教程

## MySQL主从复制安装

### MySQL安装

1.  检测下系统有没有自带的mysql：yum list installed | grep mysql , 如果已经有的话执行命令yum -y remove mysql-libs.x86_64卸载已经安装的mysql。 
2.  先到mysql官网下载5.7.11的安装包，download-yum选择Red Hat Enterprise Linux 7 / Oracle Linux 7 (Architecture Independent), RPM Package 。 

```shell
wget http://repo.mysql.com//mysql57-community-release-el7-7.noarch.rpm 
```

3.  添加选择yum源： 

```shell
yum localinstall mysql57-community-release-el7-7.noarch.rpm
yum repolist all | grep mysql
```

4.  安装mysql：

```shell
yum install mysql-community-server
```

5.  启动mysql： 

```shell
service mysqld start
```

6.  查看密码： 

```shell
cat /var/log/mysqld.log | grep password
```

7. 登录mysql,修改密码

```shell
mysql -u root -p
>ALTER USER 'root'@'localhost' IDENTIFIED BY 'QAZwsx123!' PASSWORD EXPIRE NEVER;
>flush privileges;
```

8. 授权root用户远程登录

```shell
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'QAZwsx123!';
flush privileges;
```

### 配置主从复制

#### binlog日志三种格式

##### STATEMENT(默认格式):

把所有写操作日志写到binlog日志中，然后复制到从机去运行；

缺点：

update xxx set xx time=now(); 此时由于主机和从机的时间不一样，会造成数据不一致

##### ROW:

不记录写SQL,记录每一行的改变

如果单个表更新数量比较大，日志会记录大量的变化日志，再将这些变化，一行一行的在从机执行

##### MAIED:

自动切换STATEMENT模式和ROW模式

缺点：

无法识别 @@host name,无法识别系统变量



#### master主机配置

1.  开启binlog ,vim /etc/my.cnf

```properties
#主服务器唯一ID
server-id=1
#开启二进制日志
log-bin=/var/lib/mysql/mysql-bin
#设置不要复制的数据库(可选，可以设置多个)
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
#设置需要复制的数据库（可选）
binlog-do-db=需要复制的数据库名字
#设置logbin格式
binlog_format=STATEMENT
```

2.  重启mysql服务

```shell
service mysqld restart
```

3.  进入mysql,查看binlog是否开启成功，输入cd /var/lib/mysql，看到如下文件代表开启成功 

```shell
mysql -u root -p
>show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
```

4.  创建用户并授权 slave

```shell
>GRANT REPLICATION SLAVE ON *.* TO 'backup'@'192.168.1.18' IDENTIFIED BY 'QAZwsx123!';
```

####  **配置slave 从库机器** 

1.  开启binlog
   输入vi /etc/my.cnf 进入配置文件，按Insert键进入编辑模式，添加如下参数 , ( ( server-id机器的唯一标识),后面两步是用来打开slave的relaylog功能的) 

```properties
server-id=2
relay-log-index=slave-relay-bin.index
#启用中继日志
relay-log=slave-relay-bin
```

2.  重启 mysql服务

```shell
service mysqld restart
```

3.  建立主从关系， 打开主库机器登录mysql(如果已经有了master关系配置，可以使用命令：stop slave;reset master重置master)

```shell
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      695 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

4.  打开从库机器登录mysql 

   a.  输入 show slave status\G; 查看状态，状态未开启时进行如下设置： 

```shell
#主节点
>change master to master_host='192.168.1.18';
#主节点的端口号
>change master to master_port=3306;
#账号
>change master to master_user='backup';
#密码
>change master to master_password='QAZwsx123!';
#在主库show master status 对应的的日志
>change master to master_log_file='mysql-bin.000004';
#show master status 对应的
>change master to master_log_pos=154;
```

5.  开启从节点 

```shell
start slave;
```

6.  查看状态 

```shell
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.18
                  Master_User: backup
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 154
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 527
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 2b51d4e8-66b1-11ea-b4b1-08002740832a
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.01 sec)
```

#### 遇到问题：

1. 没有给用户远程授权登录

2. Last_IO_Errno: 1236
                   Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file'

   解决办法：

   ```
   1.由于uuid相同，而导致触发此异常；我安装时发现主库居然没有uuid
   2.查询命令找此auto.cnf修改uuid即可:find -name auto.cnf[没有就创建]重新启动mysql
   3.登录mysql，重启slave，再次验证
   ```

   















## 安装MYCAT

1)mycat下载地址： http://dl.mycat.io/ 

### mycat的三个配置文件

schema.xml:定义逻辑库，表，分片节点等内容

rule.xml:定义分片规则

server.xml:定义用户以及系统相关变量，如端口等

修改server.xml

```xml
	<user name="mycat">
                <property name="password">123456</property>
                <property name="schemas">TESTDB</property>

                <!-- 表级 DML 权限设置 -->
                <!--            
                <privileges check="false">
                        <schema name="TESTDB" dml="0110" >
                                <table name="tb01" dml="0000"></table>
                                <table name="tb02" dml="1111"></table>
                        </schema>
                </privileges>           
                 -->
        </user>

```

修改schema.xml配置文件

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
        </schema>
        <dataNode name="dn1" dataHost="localhost1" database="db1" />
        <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="localhost:3306" user="root"
                                   password="123456">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS2" url="192.168.1.200:3306" user="root" password="xxx" />
                </writeHost>
        </dataHost>
</mycat:schema>
```

### 启动程序

控制台启动：去mycat/bin 目录下执行 ./mycat console

后台启动：去mycat/bin 目录下执行 ./mycat start

### 登录

#### 登录后台管理窗口

此登录方式用于管理维护MyCAT

```shell
mysql -umycat -p123456 -P 9066 -h 192.168.1.18
```

mycat相关操作

```shell
mysql> show databases;
+----------+
| DATABASE |
+----------+
| TESTDB   |
+----------+
1 row in set (0.03 sec)

mysql> show @@help;
+------------------------------------------+--------------------------------------------+
| STATEMENT                                | DESCRIPTION                                |
+------------------------------------------+--------------------------------------------+
| show @@time.current                      | Report current timestamp                   |
| show @@time.startup                      | Report startup timestamp                   |
| show @@version                           | Report Mycat Server version                |
| show @@server                            | Report server status                       |
| show @@threadpool                        | Report threadPool status                   |
| show @@database                          | Report databases                           |
| show @@datanode                          | Report dataNodes                           |
| show @@datanode where schema = ?         | Report dataNodes                           |
| show @@datasource                        | Report dataSources                         |
| show @@datasource where dataNode = ?     | Report dataSources                         |
| show @@datasource.synstatus              | Report datasource data synchronous         |
| show @@datasource.syndetail where name=? | Report datasource data synchronous detail  |
| show @@datasource.cluster                | Report datasource galary cluster variables |
| show @@processor                         | Report processor status                    |
| show @@command                           | Report commands status                     |
| show @@connection                        | Report connection status                   |
| show @@cache                             | Report system cache usage                  |
| show @@backend                           | Report backend connection status           |
| show @@session                           | Report front session details               |
| show @@connection.sql                    | Report connection sql                      |
| show @@sql.execute                       | Report execute status                      |
| show @@sql.detail where id = ?           | Report execute detail status               |
| show @@sql                               | Report SQL list                            |
| show @@sql.high                          | Report Hight Frequency SQL                 |
| show @@sql.slow                          | Report slow SQL                            |
| show @@sql.resultset                     | Report BIG RESULTSET SQL                   |
| show @@sql.sum                           | Report  User RW Stat                       |
| show @@sql.sum.user                      | Report  User RW Stat                       |
| show @@sql.sum.table                     | Report  Table RW Stat                      |
| show @@parser                            | Report parser status                       |
| show @@router                            | Report router status                       |
| show @@heartbeat                         | Report heartbeat status                    |
| show @@heartbeat.detail where name=?     | Report heartbeat current detail            |
| show @@slow where schema = ?             | Report schema slow sql                     |
| show @@slow where datanode = ?           | Report datanode slow sql                   |
| show @@sysparam                          | Report system param                        |
| show @@syslog limit=?                    | Report system mycat.log                    |
| show @@white                             | show mycat white host                      |
| show @@white.set=?,?                     | set mycat white host,[ip,user]             |
| show @@directmemory=1 or 2               | show mycat direct memory usage             |
| switch @@datasource name:index           | Switch dataSource                          |
| kill @@connection id1,id2,...            | Kill the specified connections             |
| stop @@heartbeat name:time               | Pause dataNode heartbeat                   |
| reload @@config                          | Reload basic config from file              |
| reload @@config_all                      | Reload all config from file                |
| reload @@route                           | Reload route config from file              |
| reload @@user                            | Reload user config from file               |
| reload @@sqlslow=                        | Set Slow SQL Time(ms)                      |
| reload @@user_stat                       | Reset show @@sql  @@sql.sum @@sql.slow     |
| rollback @@config                        | Rollback all config from memory            |
| rollback @@route                         | Rollback route config from memory          |
| rollback @@user                          | Rollback user config from memory           |
| reload @@sqlstat=open                    | Open real-time sql stat analyzer           |
| reload @@sqlstat=close                   | Close real-time sql stat analyzer          |
| offline                                  | Change MyCat status to OFF                 |
| online                                   | Change MyCat status to ON                  |
| clear @@slow where schema = ?            | Clear slow sql by schema                   |
| clear @@slow where datanode = ?          | Clear slow sql by datanode                 |
+------------------------------------------+--------------------------------------------+
58 rows in set (0.02 sec)

mysql> 

```

#### 登录数据窗口

此登录方式用于MYCAT查询数据

```shell
mysql -umycat -p123456 -P 8066 -h 192.168.1.18
```

## 搭建读写分离

修改schema.xml配置文件

修改<dataHost>的balance属性，通过此属性配置设置读写分离的类型

```
负载均衡类型，目前的取值有4种：
1、balance="0",不开启读写分离机制，所有读操作都发送到当前可用的writeHost上
2、balance="1",全部的readHost与stand by writeHost参与select语句的负载均衡，简单说，当双主双从模式(M1->S1,M2->S2,并且M1与M2互为主备)，正常情况下，M2、S1、S2都参与select语句的负载均衡。
3、balance="2",所有读操作都随机在writHost、readhost上发
4、balance="3",所有读请求随机的分发到readHost执行，writHost不负担读压力
```

两台读写分离使用3

四台双珠双从使用1

## 双主双从配置

| 编号 | 角色    | IP地址       | 机器名称        |
| ---- | ------- | ------------ | --------------- |
| 1    | Master1 | 192.168.1.18 | find02          |
| 2    | Slave1  | 192.168.1.19 | bogon           |
| 3    | Master2 | 192.168.1.17 | find01          |
| 4    | Slave2  | 192.168.1.6  | DESKTOP-33DIQJQ |

### Master1配置

```properties
#主服务器唯一ID
server-id=1
#开启二进制日志
log-bin=mysql-bin
#设置不要复制的数据库(可设置多个)
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
#设置需要复制的数据库
binglog-do-db=需要复制的主数据库名称
#设置logbin格式
binlog_format=STATEMENT
#作为从库的时候，有写入操作也要更新二进制日志文件
log-slave-updates
#表示自增长字段每次递增的量，指自增字段的起始值，其默认值是1，取值范围是1..65535
auto-increment-increment=2
#表示自增长字段从哪个数字开始，指字段第一次递增多少，它的取值范围1..65535
auto-increment-offset=1
```

### Master2配置

```properties
#主服务器唯一ID
server-id=3
#开启二进制日志
log-bin=mysql-bin
#设置不要复制的数据库(可设置多个)
binlog-ignore-db=mysql
binlog-ignore-db=information_schema
#设置需要复制的数据库
binglog-do-db=需要复制的主数据库名称
#设置logbin格式
binlog_format=STATEMENT
#作为从库的时候，有写入操作也要更新二进制日志文件
log-slave-updates
#表示自增长字段每次递增的量，指自增字段的起始值，其默认值是1，取值范围是1..65535
auto-increment-increment=2
#表示自增长字段从哪个数字开始，指字段第一次递增多少，它的取值范围1..65535
auto-increment-offset=2

```

### Slave1配置

```properties
#从服务器唯一ID
server-id=2
#启用中继日志
relay-log=mysql-relay
```

### Slave2配置

```properties
#从服务器唯一ID
server-id=4
#启用中继日志
relay-log=mysql-relay
```

### 重启每台服务器上的mysql服务,关闭防火墙

```shell
systemctl restart mysqld
```

### 在两个master上创建slave用户并授权

```sql
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%' IDENTIFIED BY 'QAZwsx123!';
select host,user from mysql.user;
```

### 查询master状态

```sql
show master status;
```

master1的状态

```sql
+------------------+----------+--------------+--------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB         | Executed_Gtid_Set |
+------------------+----------+--------------+--------------------------+-------------------+
| mysql-bin.000005 |      443 |              | mysql,information_schema |                   |
+------------------+----------+--------------+--------------------------+-------------------+
```

master2的状态

```sql
+------------------+----------+--------------+--------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB         | Executed_Gtid_Set |
+------------------+----------+--------------+--------------------------+-------------------+
| mysql-bin.000001 |      443 |              | mysql,information_schema |                   |
+------------------+----------+--------------+--------------------------+-------------------+
```

### 配置slave1为master1的从机

Slave1执行

```sql
CHANGE MASTER TO
MASTER_HOST='192.168.1.18',
MASTER_USER='slave',
MASTER_PASSWORD='QAZwsx123!',
MASTER_LOG_FILE='mysql-bin.000005',
MASTER_LOG_POS=443;
start slave;
show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.18
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 443
               Relay_Log_File: mysql-relay.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 443
              Relay_Log_Space: 523
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 2b51d4e8-66b1-11ea-b4b1-08002740832a
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

### 配置slave2为master2的从机

Slave2执行

```sql
CHANGE MASTER TO
MASTER_HOST='192.168.1.17',
MASTER_USER='slave',
MASTER_PASSWORD='QAZwsx123!',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=443;
start slave;
show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.17
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 443
               Relay_Log_File: mysql-relay.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 443
              Relay_Log_Space: 523
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 3
                  Master_UUID: 27467a09-66b1-11ea-99b8-0800276fc31b
             Master_Info_File: d:\mysql-5.7.28-winx64\data\master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

### 配置Master1为Master2的从机

Master1执行

```sql
CHANGE MASTER TO
MASTER_HOST='192.168.1.17',
MASTER_USER='slave',
MASTER_PASSWORD='QAZwsx123!',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=443;
start slave;
show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.17
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 443
               Relay_Log_File: find02-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 443
              Relay_Log_Space: 528
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 3
                  Master_UUID: 27467a09-66b1-11ea-99b8-0800276fc31b
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

### 配置Master1为Master2的从机

Master2执行

```sql
CHANGE MASTER TO
MASTER_HOST='192.168.1.18',
MASTER_USER='slave',
MASTER_PASSWORD='QAZwsx123!',
MASTER_LOG_FILE='mysql-bin.000005',
MASTER_LOG_POS=443;
start slave;
show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.18
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 443
               Relay_Log_File: find01-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 443
              Relay_Log_Space: 528
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 2b51d4e8-66b1-11ea-b4b1-08002740832a
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

### 创建数据库

```sql
create database testdb;
```

### 创建表,插入数据

```sql
use testdb;
create table mytable(id int,name varchar(20));
insert into mytable values(1,'zhangsan');
select * from mytable;
```

### 修改MYCAT的配置文件

#### 修改schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
        </schema>
        <dataNode name="dn1" dataHost="host1" database="test_db" />
        <dataHost name="host1" maxCon="1000" minCon="10" balance="1"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.1.18:3306" user="root"
                                   password="QAZwsx123!">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS1" url="192.168.1.19:3306" user="root" password="QAZwsx123!" />
                </writeHost>
                <writeHost host="hostM2" url="192.168.1.17:3306" user="root"
                                   password="QAZwsx123!">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS2" url="192.168.1.6:3306" user="root" password="QAZwsx123!" />
                </writeHost>

        </dataHost>
</mycat:schema>
```

balance="1":全部的readHost与stand by writeHost参与select 语句的负载均衡

writeType="0"：所有写操作发送到配置的第一个writeHost,第一个挂了切换到生存的第二个

writeType="1"：所有写操作都随机发送到配置的writeHost，1.5以后废弃不推介使用

writeHost,重新启动后以切换的为准，切换记录在配置文件中：dnindex.properties

switchType="1": 1 默认值，自动切换

​							-1 表示不自动切换

​							2 基于MySQL主从同步的状态决定是否切换

### 启动MYCAT

```shell
 ./mycat start
```

### 验证读写分离

```shell
#登录mycat
mysql -umycat -p123456 -P 8066 -h 192.168.1.18
#插入数据
insert into mytable values(2,@@hostname);
```

## 垂直拆分-分库

```sql
#客户表 rows:20万
CREATE TABLE customer(
	id INT AUTO_INCREMENT,
	name VARCHAR(200),
	PRIMARY KEY(id)
);
#订单表 rows:600万
CREATE TABLE orders(
	id INT AUTO_INCREMENT,
	order_type INT,
	customer_id INT,
	amount DECIMAL(10,2),
	PRIMARY KEY(id)
);
#订单详情表 rows:600万
CREATE TABLE orders_detail(
	id INT AUTO_INCREMENT,
	detail VARCHAR(200),
	order_id INT,
	PRIMARY KEY(id)
);
#订单状态字典表  rows:20
CREATE TABLE dict_order_type(
	id INT AUTO_INCREMENT,
    order_type VARCHAR(200),
    PRIMARY KEY(id)
);
```

#### 修改schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
                <table name="customer" dataNode="dn2"></table>
        </schema>
        <dataNode name="dn1" dataHost="host1" database="orders" />
        <dataNode name="dn2" dataHost="host2" database="orders" />
        <dataHost name="host1" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.1.18:3306" user="root"
                                   password="QAZwsx123!">
                        <!-- can have multi read hosts -->
                </writeHost>
        </dataHost>
        <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.1.17:3306" user="root"
                                   password="QAZwsx123!">
                        <!-- can have multi read hosts -->
                </writeHost>
        </dataHost>
</mycat:schema>
```

#### 新增两个空白库

```sql
#在数据节点dn1、dn2上分别创建数据库orders
CREATE DATABASE orders;
```

#### 启动mycat

```shell
./mycat console
```

#### 访问mycat

```shell
mysql -umycat -p123456 -P 8066 -h 192.168.1.18
```

#### 创建用户表

```sql
#客户表 rows:20万
CREATE TABLE customer(
	id INT AUTO_INCREMENT,
	name VARCHAR(200),
	PRIMARY KEY(id)
);
```

#### 去两个数据节点dn1和dn2查看数据创建情况

#### 创建订单相关的表

```sql
#订单表 rows:600万
CREATE TABLE orders(
	id INT AUTO_INCREMENT,
	order_type INT,
	customer_id INT,
	amount DECIMAL(10,2),
	PRIMARY KEY(id)
);
#订单详情表 rows:600万
CREATE TABLE orders_detail(
	id INT AUTO_INCREMENT,
	detail VARCHAR(200),
	order_id INT,
	PRIMARY KEY(id)
);
#订单状态字典表  rows:20
CREATE TABLE dict_order_type(
	id INT AUTO_INCREMENT,
    order_type VARCHAR(200),
    PRIMARY KEY(id)
);
```

#### 去两个数据节点dn1和dn2查看数据创建情况

## 水平拆分-分表

### 实现分表

#### 修改schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
                <table name="customer" dataNode="dn2"></table>
                <table name="orders" dataNode="dn1,dn2" rule="mod_rule"></table>
        </schema>
        <dataNode name="dn1" dataHost="host1" database="orders" />
        <dataNode name="dn2" dataHost="host2" database="orders" />
        <dataHost name="host1" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.1.18:3306" user="root"
                                   password="QAZwsx123!">
                        <!-- can have multi read hosts -->
                </writeHost>
        </dataHost>
        <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.1.17:3306" user="root"
                                   password="QAZwsx123!">
                        <!-- can have multi read hosts -->
                </writeHost>
        </dataHost>
</mycat:schema>
```

#### 修改rule.xml

在标签<mycat:rule xmlns:mycat="http://io.mycat/"></mycat>中添加如下内容：

```xml
<tableRule name="mod_rule">
      <rule>
           <columns>customer_id</columns>
           <algorithm>mod-long</algorithm>
       </rule>
 </tableRule>
```

修改如下节点内容

```xml
		<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
                <!-- how many data nodes -->
                <property name="count">2</property>
        </function>
```

#### 在两个节点上都创建订单表

```
#订单表 rows:600万
CREATE TABLE orders(
	id INT AUTO_INCREMENT,
	order_type INT,
	customer_id INT,
	amount DECIMAL(10,2),
	PRIMARY KEY(id)
);
```

#### 启动mycat

```shell
./mycat console
```

#### 访问mycat

```shell
mysql -umycat -p123456 -P 8066 -h 192.168.1.18
```

#### 往mycat的orders表中插入数据

注意：在mycat中向orders中插入数据时，INSERT字段不能省略

```sql
INSERT INTO orders(id,order_type,customer_id,amount) values(1,101,100,100100);
INSERT INTO orders(id,order_type,customer_id,amount) values(2,101,100,100300);
INSERT INTO orders(id,order_type,customer_id,amount) values(3,101,101,120000);
INSERT INTO orders(id,order_type,customer_id,amount) values(4,101,101,100300);
INSERT INTO orders(id,order_type,customer_id,amount) values(5,102,101,100400);
INSERT INTO orders(id,order_type,customer_id,amount) values(6,102,100,100020);
```

### MyCAT分片的“join”

##### 修改schema.xml

```xml
	<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
                <table name="customer" dataNode="dn2"></table>
                <table name="orders" dataNode="dn1,dn2" rule="mod_rule">
                        <childTable name="orders_detail" primaryKey="id" joinKey="order_id" parentKey="id"/>
                </table>
        </schema>
```

##### 在两个节点上都创建订单表

```
#订单详情表 rows:600万
CREATE TABLE orders_detail(
	id INT AUTO_INCREMENT,
	detail VARCHAR(200),
	order_id INT,
	PRIMARY KEY(id)
);
```

##### 启动mycat

```shell
./mycat console
```

##### 访问mycat

```shell
mysql -umycat -p123456 -P 8066 -h 192.168.1.18
```

##### 在mycat中向orders_detail表插入数据

```sql
INSERT INTO orders_detail(id,tetail,order_id) values(1,'detail1',1);
INSERT INTO orders_detail(id,tetail,order_id) values(2,'detail2',2);
INSERT INTO orders_detail(id,tetail,order_id) values(3,'detail3',3);
INSERT INTO orders_detail(id,tetail,order_id) values(4,'detail4',4);
INSERT INTO orders_detail(id,tetail,order_id) values(5,'detail5',5);
INSERT INTO orders_detail(id,tetail,order_id) values(6,'detail6',6);
```

##### 在mycat、dn1、dn2中运行两个表join语句

```sql
SELECT o.*,od.detail  FROM orders o inner join orders_detail od on o.id=od.order_id;
```

#### 全局表

在分片的情况下，当业务表因为规模而进行分片以后，业务表就与这些附属字典表之间的关系，就成了比较棘手的问题，考虑到字典表具有一下几个特性：

1. 变动不频繁
2. 数据总体变化不大
3. 数据规模不大，很少有超过数十万条记录

鉴于此，MyCAT定义了一种特殊的表，称之为“全局表”，全局表有以下特点：

全局表的插入、更新操作会实时在所有节点上执行，保持各分片数据的一致性

全局表的查询操作，只从一个节点获取

全局表可以跟任何一个表进行join操作

​	将字典表或者服务字典表特性的一些表定义为全局表，则从另一个方面，很好的解决了JOIN的难题。通过全局表+基于E-R关系的分片策略，MyCAT可以满足80%以上的企业应用

##### 修改schema.xml配置文件

```xml
        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
                <table name="customer" dataNode="dn2"></table>
                <table name="orders" dataNode="dn1,dn2" rule="mod_rule">
                        <childTable name="orders_detail" primaryKey="id" joinKey="order_id" parentKey="id"/>
                </table>
                <table name="dict_order_type" dataNode="dn1,dn2" type="global"></table>
        </schema>
```

##### 在两个节点上都创建字典表

```
#订单状态字典表  rows:20
CREATE TABLE dict_order_type(
	id INT AUTO_INCREMENT,
    order_type VARCHAR(200),
    PRIMARY KEY(id)
);
```

##### 启动mycat

```shell
./mycat console
```

##### 访问mycat

```shell
mysql -umycat -p123456 -P 8066 -h 192.168.1.18
```

##### 在mycat中向dict_order_type表插入数据

```sql
INSERT INTO dict_order_type(id,order_type) VALUES(101,'type1');
INSERT INTO dict_order_type(id,order_type) VALUES(102,'type2');
```

##### 在mycat、dn1、dn2中查询全局表

```sql
select * from dict_order_type;
```

### 常用分片规则

#### 1、取模

​	此规则为对分片字段求模运算。页数水平分表常用最为常用的规则。

#### 2、分片枚举

​	通过在配置文件中配置可能的枚举id,自己配置分片，本规则适用于特定的场景，比如有些业务需要按照省份或区县来保存，而全国的省份区县是固定的，这类业务使用本条规则

(1)修改schema.xml
```xml
<table name="orders_ware_info" dataNode="dn1,dn2" rule="sharding_by_intfile"></table>
```

(2)修改rule.xml

```xml
<tableRule name="sharding_by_intfile">
    <rule>
      <columns>areacode</columns>
      <algorithm>hash-int</algorithm>
     </rule>
</tableRule>
...
<function name="hash-int"
        class="io.mycat.route.function.PartitionByFileMap">
        <property name="mapFile">partition-hash-int.txt</property>
        <property name="type">1</property>
        <property name="defaultNode">0</property>
</function>
```

#columns:分片字段，algorithm:分片函数
#mapFile:标识配置文件名称，type: 0为int型，非0为String
#defaultNode:默认节点：小于0表示不设置默认节点，大于0表示设置默认节点。设置默认节点如果碰到不识别的枚举值，就让它路由到默认节点，如果不设置不识别就报错
(3)修改partition-hash-int.txt

```properties
110=0
120=1
```

(4)启动MyCAT
(5)连接MyCAT创建表

```sql
#订单归属区域信息表
CREATE TABLE orders_ware_info(
	id INT AUTO_INCREMENT comment '编号',
	order_id INT comment '订单编号',
	address VARCHAR(200) comment '地址',
	areacode VARCHAR(20) comment '区域编号',
	PRIMARY KEY(id)
);
```

(6)插入数据

```sql
INSERT INTO orders_ware_info(id,order_id,address,areacode) VALUES(1,1,'beijing','110');
INSERT INTO orders_ware_info(id,order_id,address,areacode) VALUES(2,2,'tianjin','120');
```

(7)查看数据保存位置

#### 3、范围约定

修改schema.xml

```xml
<table name="payment_info" dataNode="dn1,dn2" rule="auto_sharding_long"></table>
```

修改rule.xml

```xml
<tableRule name="auto_sharding_long">
        <rule>
                <columns>order_id</columns>
                <algorithm>rang-long</algorithm>
        </rule>
</tableRule>
...
<function name="rang-long"
        class="io.mycat.route.function.AutoPartitionByLong">
        <property name="mapFile">autopartition-long.txt</property>
        <property name="defaultNode">0</property>
</function>
```

修改autopartition-long.txt 

```properties
0-102=0
103-200=1
```

启动MyCAT
连接MyCAT创建表

```sql
#支付信息表
CREATE TABLE payment_info(
	id INT AUTO_INCREMENT comment '编号',
	order_id INT comment '订单编号',
	payment_status INT comment '支付状态',
	PRIMARY KEY(id)
);
```

插入数据

```sql
INSERT INTO payment_info(id,order_id,payment_status) VALUES(1,101,0);
INSERT INTO payment_info(id,order_id,payment_status) VALUES(2,102,1);
INSERT INTO payment_info(id,order_id,payment_status) VALUES(3,103,0);
INSERT INTO payment_info(id,order_id,payment_status) VALUES(4,104,1);
```

查看数据保存位置

#### 4、按日期(天)分片

修改schema.xml

```xml
<table name="login_info" dataNode="dn1,dn2" rule="sharding_by_date"></table>
```

修改rule.xml

```xml
<tableRule name="sharding_by_date">
        <rule>
                <columns>login_date</columns>
                <algorithm>shardingByDate</algorithm>
        </rule>
</tableRule>
...
<function name="shardingByDate" class="io.mycat.route.function.PartitionByDate">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2019-01-01</property>
	<property name="sEndDate">2019-01-04</property>
	<property name="sPartionDay">2</property>
</function>
```

```
columns:分片字段，
algorithm:分片函数
dataFormat:日期格式
sBeginDate:开始日期
sEndDate:结束日期，则达标数据达到了这个日期的分片后循环从开始分片插入
sPartionDay:分区天数，即默认从开始日期算起，分隔2天一个分区
```

启动MyCAT
连接MyCAT创建表

```sql
CREATE TABLE login_info(
	id INT AUTO_INCREMENT comment '编号',
	user_id INT comment '用户编号',
	login_date date comment '登录日期',
	PRIMARY KEY(id)
);
```

插入数据

```sql
INSERT INTO login_info(id,user_id,login_date) VALUES(1,101,'2019-01-01');
INSERT INTO login_info(id,user_id,login_date) VALUES(2,102,'2019-01-02');
INSERT INTO login_info(id,user_id,login_date) VALUES(3,103,'2019-01-03');
INSERT INTO login_info(id,user_id,login_date) VALUES(4,104,'2019-01-04');
INSERT INTO login_info(id,user_id,login_date) VALUES(5,103,'2019-01-05');
INSERT INTO login_info(id,user_id,login_date) VALUES(6,104,'2019-01-06');
```

查询MyCAT、dn1、dn2可以看到数据分片效果

## 全局序列

### 本地文件

此方式MyCAT将sequence配置到文件中，当使用到sequence中的配置后，MyCAT会更新classpath中的sequence_conf.properties文件中sequence当前的值。

优点：本地加载，读取速度较快

缺点：抗风险能力差，MyCAT所在主机宕机后，无法读取本地文件

### 数据库方式（一般使用这种情况）

利用数据库一个表来进行数据累加。但是每次生成序列都读写数据库，这样效率太低。MyCAT会预加载一部分号段到MyCAT的内存中，这样大部分读写序列都是在内存中完成的。如果内存中的号段用完了MyCAT会再向数据库要一次。

### 时间戳方式（需要保持主机和备机时间一致）

全局序列 ID=64位二进制(42(毫秒)+5(机器ID)+5(业务编码)+12(重复累加))换算成十进制为18位long类型，每毫秒可以并发12位二进制的累加

优点：配置简单

缺点：18位ID过长

### 自主生成全局序列

可在java项目里自己生成全局序列，如下：

根据业务逻辑组合

## 基于HA机制的MyCAT高可用

### 安装配置HAProxy

下载地址： https://src.fedoraproject.org/repo/pkgs/haproxy/ 

上传安装包haproxy-1.5.18.tar.gz到/opt目录下

将软件解压到指定目录

```shell
tar -xzvf haproxy-1.5.18.tar.gz -C /usr/local/src
```

进入解压后的emulate，查看内核版本，进行编译

```shell
cd /usr/local/src/haproxy-1.5.18/
uname -r
3.10.0-1062.el7.x86_64
make TARGET=linux310 PREFIX=/usr/local/haproxy ARCH=x86_64
```

TARGET=linux310 内核版本，使用uname -r查看内核，如3.10.0-1062.el7.x86_64，此时参数就是linux310

ARCH=x86_64,系统位数

PREFIX=/usr/local/haproxy  ，haproxy安装位置

编译完成后，进行安装

```shell
make install PREFIX=/usr/local/haproxy
```

创建目录和HAProxy配置文件

```shell
mkdir -p /usr/data/haproxy/
vim /usr/local/haproxy/haproxy.conf
```

haproxy.conf内容

```
global
		log 127.0.0.1	local0
		#log 127.0.0.1	local1 notice
		#log loghost	local0 info
		maxconn 4096
		chroot /usr/local/haproxy
		pidfile /usr/data/haproxy/haproxy.pid
		uid 99
		gid 99
		daemon
		#debug
		#quiet
defaults
		log		global
		mode	tcp
		option	abortonclose
		option redispatch
		retries	3
		maxconn 2000
		timeout connect 5000
		timeout	client 50000
		timeout	server 50000
listen proxy_status
	bind:48066
		mode tcp
		balance roundrobin
		server mycat_1 192.168.1.18
		server mycat_2 192.168.1.17
frontend admin_stats
	bind:7777
		mode http
		stats enable
		option httplog
		maxconn	10
		stats refresh 30s
		stats uri/admin
		stats auth admin:1231213
		stats hide-version
		stats admin if TRUE
```

启动两个mycat

启动HAProxy

```
/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.conf
```

## MyCAT安全权限配置

server.xml配置

```xml
<user name="mycat">
        <property name="password">123456</property>
        <property name="schemas">TESTDB</property>

        <!-- 表级 DML 权限设置 -->
        <!--            
        <privileges check="false">
                <schema name="TESTDB" dml="0110" >
                        <table name="tb01" dml="0000"></table>
                        <table name="tb02" dml="1111"></table>
                </schema>
        </privileges>           
         -->
</user>

<user name="user">
        <property name="password">user</property>
        <property name="schemas">TESTDB</property>
        <property name="readOnly">true</property>  //开启只读权限
</user>
```

| 标签属性 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| name     | 应用连接中间件逻辑库的用户名                                 |
| password | 应用连接中间件逻辑库的用户密码                               |
| TESTDB   | 应用当前连接的逻辑库中所对应的逻辑表。schemas中可以配置一个或者多个 |
| readOnly | 应用连接中间件逻辑库所具有的权限。true为只读，false是读写都有。默认为false |

privileges标签权限控制

​	在user标签下privileges标签可以对逻辑库(schema)、表(tables)进行精细化的DML权限控制

​	privileges标签下的check属性，如果为true开启权限检查，为false不开启，默认为false

​	由于MyCAT一个用户的schemas属性可配置多个逻辑库（schema）,所以privileges的下级节点schema节点同样可配置多个，对多库多表进行细粒度的DML权限控制

server.xml配置文件privileges部分

​	配置orders表没有增删改查权限

```xml
<user name="mycat">
        <property name="password">123456</property>
        <property name="schemas">TESTDB</property>

        <!-- 表级 DML 权限设置-->
        <privileges check="true">
                <schema name="TESTDB" dml="1111" >
                        <table name="orders" dml="0000"></table>
                </schema>
        </privileges>
</user>

<user name="user">
        <property name="password">user</property>
        <property name="schemas">TESTDB</property>
        <property name="readOnly">true</property>
</user>

```





| DML权限 | 增加(insert) | 更新(update) | 查询(select) | 删除(delete) |
| ------- | ------------ | ------------ | ------------ | ------------ |
| 0000    | 禁止         | 禁止         | 禁止         | 禁止         |
| 0010    | 禁止         | 禁止         | 可以         | 禁止         |
| 1110    | 可以         | 禁止         | 禁止         | 禁止         |
| 1111    | 可以         | 可以         | 可以         | 可以         |

## SQL拦截

​	firewall标签用来定义防火墙；firewall下whitehost标签用来定义ip白名单，blacklist用来定义SQL黑名单

### 白名单

​	可以通过设置白名单，实现某主机可以访问MyCAT,而其他主机用户禁止访问

设置白名单

server.xml配置

```xml
        <!-- 全局SQL防火墙设置 -->
        <firewall>
           <whitehost>
              <host host="192.168.1.17" user="mycat"/>
              <host host="127.0.0.2" user="mycat"/>
           </whitehost>
       <blacklist check="false">
       </blacklist>
        </firewall>
```

### 黑名单

​	可以通过设置黑名单，实现MyCAT对具体的SQL操作拦截，如果增删改查等操作拦截

设置黑名单

```xml
        <!-- 全局SQL防火墙设置 -->
        <firewall>
           <whitehost>
              <host host="192.168.1.17" user="mycat"/>
              <host host="127.0.0.2" user="mycat"/>
           </whitehost>
       <blacklist check="true">
                <property name="deleteAllow">false</property>
       </blacklist>
        </firewall>
```

## MyCAT监控工具

### 安装MyCAT-web

安装zookeeper

mycat-web安装

下载安装包： http://dl.mycat.io/mycat-web-1.0/ 

解压安装包

进入bin目录启动

cd ${MYCAT_WEB_HOME}/

./start.sh &

访问地址： http://ip:8082/mycat/ 