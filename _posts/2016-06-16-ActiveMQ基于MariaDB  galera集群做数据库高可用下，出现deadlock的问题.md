---
layout: post
title: "ActiveMQ基于MariaDB  galera集群做数据库高可用下，出现deadlock的问题"
date: 2016-06-16
tags:
    - java
    - ActiveMQ
    - Work
author: "huangliangliang"
---

问题描述：数据库采用MaraDB galera做多主节点高可用。
activeMQ基于数据库集群做了主备节点，共享同样的数据库节点。
出现activeMQ节点一启动后，再次启动activeMQ节点二，节点二抢到了表锁，导致一和二节点产生死锁，程序错误。

问题日志：
```
2016-06-16 00:06:50,078 | ERROR | Failed to update database lock: com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction | org.apache.activemq.store.jdbc.DefaultDatabaseLocker | ActiveMQ JDBC PA Scheduled Task
com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
```

抛出该问题日志的地点：

```
at org.apache.activemq.store.jdbc.DefaultDatabaseLocker.keepAlive(DefaultDatabaseLocker.java:164)[activemq-jdbc-store-5.11.0.jar:5.11.0]
at org.apache.activemq.broker.LockableServiceSupport.keepLockAlive(LockableServiceSupport.java:126)[activemq-broker-5.11.0.jar:5.11.0]
at org.apache.activemq.broker.LockableServiceSupport$1.run(LockableServiceSupport.java:98)[activemq-broker-5.11.0.jar:5.11.0]
```

查看源代码：
DefaultDatabaseLocker

有两个方法值得关注：

```
public void doStart() throws Exception;
public boolean keepAlive() throws IOException；
```

首先说下日志问题的显示：

```
2016-06-16 00:06:20,068 | INFO  | Using a separate dataSource for locking: org.apache.commons.dbcp.BasicDataSource@21d5754 | org.apache.activemq.store.jdbc.JDBCPersistenceAdapter | main
2016-06-16 00:06:20,069 | INFO  | Attempting to acquire the exclusive lock to become the Master broker | org.apache.activemq.store.jdbc.DefaultDatabaseLocker | main
2016-06-16 00:06:20,073 | INFO  | Becoming the master on dataSource: org.apache.commons.dbcp.BasicDataSource@21d5754 | org.apache.activemq.store.jdbc.DefaultDatabaseLocker | main
```

在出现死锁之前，程序执行了DefaultDatabaseLocker 的doStart()方法，并且成功拿到了锁。
起初怀疑锁表语句在MariaDB节点下不支持。于是手写了测试程序,
运行TestJDBC程序，发现进程一成功拿到锁，进程二拿不到该锁，出现等待。符合表锁的结果预期。是正确的。
测试代码摘出了ActiveMQ锁的核心代码，按预期应该是和TestJDBC程序保持一致。但现在出现差异。考虑前置步骤是否有差异。

方向：
1. DataSource原因引起
2. MQ 锁表前的前置操作
3. 锁表后的处理

首先排除了DataSource，在测试程序中，后来把它改为了和MQ同样的DataSource配置。测试程序仍然运行正常。

MQ锁表前的前置操作：

```
2016-06-16 00:45:42,887 | DEBUG | Executing SQL: CREATE TABLE ACTIVEMQ_LOCK( ID BIGINT NOT NULL, TIME BIGINT, BROKER_NAME VARCHAR(250), PRIMARY KEY (ID) ) | org.apache.activemq.store.jdbc.adapter.DefaultJDBCAdapter | main
2016-06-16 00:45:42,890 | DEBUG | Could not create JDBC tables; The message table already existed. Failure was: CREATE TABLE ACTIVEMQ_LOCK( ID BIGINT NOT NULL, TIME BIGINT, BROKER_NAME VARCHAR(250), PRIMARY KEY (ID) ) Message: Table 'activemq_lock' already exists SQLState: 42S01 Vendor code: 1050 | org.apache.activemq.store.jdbc.adapter.DefaultJDBCAdapter | main
2016-06-16 00:45:42,890 | DEBUG | Executing SQL: INSERT INTO ACTIVEMQ_LOCK(ID) VALUES (1) | org.apache.activemq.store.jdbc.adapter.DefaultJDBCAdapter | main
2016-06-16 00:45:42,892 | DEBUG | Could not create JDBC tables; The message table already existed. Failure was: INSERT INTO ACTIVEMQ_LOCK(ID) VALUES (1) Message: Duplicate entry '1' for key 'PRIMARY' SQLState: 23000 Vendor code: 1062 | org.apache.activemq.store.jdbc.adapter.DefaultJDBCAdapter | main
2016-06-16 00:45:42,897 | INFO  | Database lock driver override not found for : [mysql_connector_java].  Will use default implementation. | org.apache.activemq.store.jdbc.JDBCPersistenceAdapter | main
2016-06-16 00:45:42,898 | DEBUG | Using default JDBC Locker: org.apache.activemq.store.jdbc.DefaultDatabaseLocker@45de5dc6 | org.apache.activemq.store.jdbc.JDBCPersistenceAdapter | main
2016-06-16 00:45:42,899 | INFO  | Using a separate dataSource for locking: org.apache.commons.dbcp.BasicDataSource@498b30a3 | org.apache.activemq.store.jdbc.JDBCPersistenceAdapter | main
2016-06-16 00:45:42,899 | INFO  | Attempting to acquire the exclusive lock to become the Master broker | org.apache.activemq.store.jdbc.DefaultDatabaseLocker | main
2016-06-16 00:45:42,900 | DEBUG | Locking Query is SELECT * FROM ACTIVEMQ_LOCK FOR UPDATE | org.apache.activemq.store.jdbc.DefaultDatabaseLocker | main
2016-06-16 00:45:42,903 | INFO  | Becoming the master on dataSource: org.apache.commons.dbcp.BasicDataSource@498b30a3 | org.apache.activemq.store.jdbc.DefaultDatabaseLocker | main
```

 查看日志，发现在锁表前，做了表插入操作，但是因为数据库存在这些表了，没有插入成功。怀疑这个地方问题？
在ServiceSupport.java 中，start方法中执行了预处理preStart();
该方法的实现，在AbstractJDBCLocker类中，执行了Create 表方法。

```
if (createTablesOnStartup) {

            String[] createStatements = getStatements().getCreateLockSchemaStatements();

            Connection connection = null;
```
createTablesOnStartup是个可配置的参数，查看activemq文档后，在activemq.xml中可以配置：

```
<jdbcPersistenceAdapter dataSource="#mysql-ds" createTablesOnStartup="false"/>
```
重启MQ后，死锁的问题就解决了。

 
