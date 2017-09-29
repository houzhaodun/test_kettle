# 内容简介 #

&emsp;&emsp;Pentaho Kettle 是一款国外开源的 ETL 工具，纯 java 编写，可以在 Window、Linux、Unix 上运行，绿色无需安装，部署方式灵活，可按照实际项目需求构建ETL集群。本文主要描述 Kettle 7.1 与 TiDB 部署联调过程 ，并对测试过程问题及其处理方式进行说明。

&emsp;&emsp;最新版 Kettle 下载地址： http://kettle.pentaho.org

&emsp;&emsp;常见部署平台为 Windows 和 Linux，部署非常简单，解压到指定目录即可。注意：Kettle 纯java编写，所以程序运行依赖Java环境，需配置好JDK及环境变量。（[JDK下载地址](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html " JDK 下载地址")）

# 测试环境 #

**&emsp;&emsp;Kettle 测试环境**

| 名称    | Win   |  Linux  |
| ------- | -----:| :----: |
| 操作系统  | Windows 8 |  CentOS 7.3  |
| JDK   | 1.8    |   1.8   |
| TiDB  | Pre_GA |   Pre_GA   |
| mysql | 5.6    | 5.6 |


**&emsp;&emsp;TiDB 测试环境 ( Centos7.3  4台虚拟机 )**

|Name	|Host IP	|Services |
| --------  | -----:   | :----: |
|node1	|192.168.100.160|	PD, TiDB, Monitor|
|node2	|192.168.100.161|	TiKV1|
|node3	|192.168.100.162|	TiKV2|
|node4	|192.168.100.163|	TiKV3|


**&emsp;&emsp;Oracle 测试环境 ( xp )**

|Name	|Host IP	|Services |
| --------  | -----:   | :----: |
|orcl	|192.168.100.134|	Oracle |
&emsp;&emsp;


# 详细测试 #
&emsp;&emsp;由于虚拟机配置环境有限，本次测试数据量较小。
### 1、配置 TiDB 为资源库 [ JDBC ] ###
1. 解压 Kettle 到本地目录：D:\data-integration，从 MySQL 下载 mysql-connector-java 驱动包,解压后将对应的 jar 包 copy 到 D:\data-integration\lib 下;
2. 启动 Kettle(spoon.bat)，如下图所示，主要有2中任务，一个是作业，一个是转换。一般来说，转换是一系列具体的操作，比如：调度 SP，导出 Excel 等等；作业，就是按照一定流程来调度一系列转换。
![image-w150](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k01.jpg)

3. TiDB 资源库用户的创建
```
mysql -uroot -h 192.168.100.160 -P 4000
>> create database resource;
>> create user kettle identified by '111111';
>> grant all privileges on resource.* to 'kettle'@'localhost';
>> \q
mysql -ukettle -p111111 -h 192.168.100.160 -P 4000
>> show databases;
```
4. TiDB资源库配置，点击右上角 connect，进入 Pentaho Repository，选择 Other Repositories --> Database Repository --> Get Started

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k02.jpg)

--> Database Connection 配置如下

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/03.jpg)

点击测试，成功连接数据库。

&emsp;&emsp;注：ETL 过程各个模块数据库连接可按照此方式配置调用，Kettle 按钮需要读取存储过程信息，否则报错，解决方式参见第五章错误分析。

5. 查看资源库
```
mysql -ukettle -p111111 -h 192.168.100.160 -P 4000
>> use resource;
>> show tables;
MySQL [resource]> show tables;
+--------------------------+
| Tables_in_resource       |
+--------------------------+
| R_CLUSTER                |
| R_CLUSTER_SLAVE          |
| R_CONDITION              |
| R_DATABASE               |
| R_DATABASE_ATTRIBUTE     |
| R_DATABASE_CONTYPE       |
| R_DATABASE_TYPE          |
... ...
```
### 2、ODBC 配置测试 ###
1、官网下载 Mysql ODBC 并安装。

2、控制面板-> 管理工具 -> ODBC 数据源（64位）

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k06.jpg)

服务端对应 TiDB 数据库 UTF-8，所以选择 ODBC 驱动的的时候请注意，否则报错如下：

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k04.jpg)

ODBC 链接 TiDB 测试通过。

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k05.jpg)

### 3、典型场景测试 ###

**1. 场景一**
- SQL脚本创建数据表 -> 文本文件输入 -> 加载到TiDB表 -> Excel导出本地

选择脚本-> SQL脚本

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k07.jpg)

输入->文本文件输入

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k08.jpg)
![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k09.jpg)

输出->表输出

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k10.jpg)

输出 -> Excel输出 -> 设置各个流程转换关系 -> 执行调用 -> 检查数据库表数据及本地Excel文件

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k11.jpg)


**2. 场景二**

- Kettle 构建测试数据 -> 导 入Oracle 表 -> 根据条件导出到 csv -> 执行批量 Insert 到 TiDB
注：Oracle 需要提前配置客户端，且把相关 Jar 包 copy 到 Kettle lib 文件夹下

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k12.jpg)

构造测试数据  -->  数据导入Oracle表

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k13.jpg)

读取Oracle数据表 --> 生成文本文件

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k15.jpg)

文本文件  -->  insert to TiDB

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k17.jpg)

确认结果是否正确，查询 TiDB数据条数是否为5000条。

### 4、测试总结 ###
通过 MySQL JDBC/ODBC 可实现 Kettle ETL　过程，TiDB 针对各个模块的调用访问  与　MySQL　访问调用区别不大。

### 5、错误分析 ###
```
---------------------------------------错误信息------------------------------------
java.lang.reflect.InvocationTargetException: 从数据库中获取信息时发生错误: org.pentaho.di.core.exception.KettleDatabaseException: 
因为错误不能提取数据库信息

Unable to get list of procedures from database meta-data: 
Communications link failure due to underlying exception: 

** BEGIN NESTED EXCEPTION ** 

java.io.EOFException
MESSAGE: Can not read response from server. Expected to read 4 bytes, read 0 bytes before connection was unexpectedly lost.

STACKTRACE:

java.io.EOFException: Can not read response from server. Expected to read 4 bytes, read 0 bytes before connection was unexpectedly lost.
	at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:1997)
	at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2411)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:2916)
	at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:1631)
	at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:1723)
	at com.mysql.jdbc.Connection.execSQL(Connection.java:3283)
	at com.mysql.jdbc.PreparedStatement.executeInternal(PreparedStatement.java:1332)
	at com.mysql.jdbc.PreparedStatement.executeQuery(PreparedStatement.java:1467)
	at com.mysql.jdbc.DatabaseMetaData$8.forEach(DatabaseMetaData.java:4183)
	at com.mysql.jdbc.DatabaseMetaData$IterateBlock.doForAll(DatabaseMetaData.java:76)
	at com.mysql.jdbc.DatabaseMetaData.getProceduresAndOrFunctions(DatabaseMetaData.java:4120)
	at com.mysql.jdbc.DatabaseMetaData.getProcedures(DatabaseMetaData.java:4084)
	at org.pentaho.di.core.database.Database.getProcedures(Database.java:4052)
	at org.pentaho.di.core.database.DatabaseMetaInformation.getData(DatabaseMetaInformation.java:406)
	at org.pentaho.di.ui.core.database.dialog.GetDatabaseInfoProgressDialog$1.run(GetDatabaseInfoProgressDialog.java:65)
	at org.eclipse.jface.operation.ModalContext$ModalContextThread.run(ModalContext.java:113)


** END NESTED EXCEPTION **



Last packet sent to the server was 1 ms ago.


	at org.pentaho.di.ui.core.database.dialog.GetDatabaseInfoProgressDialog$1.run(GetDatabaseInfoProgressDialog.java:67)
	at org.eclipse.jface.operation.ModalContext$ModalContextThread.run(ModalContext.java:113)
Caused by: org.pentaho.di.core.exception.KettleDatabaseException: 
因为错误不能提取数据库信息
---------------------------------------错误原因------------------------------------
通过Ketlle界面按钮获取数据库配置需要读取Database的Schema、表、视图、同义词、存储过程信息，默认通过MySQL JDBC 调用TiDB加载该类信息报错，直接填写表信息无问题。需要在 TiDB库内补建存储过程即可，存储过程语句如下：

 CREATE TABLE `proc` (
  `db` char(64) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL DEFAULT '',
  `name` char(64) NOT NULL DEFAULT '',
  `type` enum('FUNCTION','PROCEDURE') NOT NULL,
  `specific_name` char(64) NOT NULL DEFAULT '',
  `language` enum('SQL') NOT NULL DEFAULT 'SQL',
  `sql_data_access` enum('CONTAINS_SQL','NO_SQL','READS_SQL_DATA','MODIFIES_SQL_DATA') NOT NULL DEFAULT 'CONTAINS_SQL',
  `is_deterministic` enum('YES','NO') NOT NULL DEFAULT 'NO',
  `security_type` enum('INVOKER','DEFINER') NOT NULL DEFAULT 'DEFINER',
  `param_list` blob NOT NULL,
  `returns` longblob NOT NULL,
  `body` longblob NOT NULL,
  `definer` char(77) CHARACTER SET utf8 COLLATE utf8_bin NOT NULL DEFAULT '',
  `created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `modified` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `sql_mode` set('REAL_AS_FLOAT','PIPES_AS_CONCAT','ANSI_QUOTES','IGNORE_SPACE','NOT_USED','ONLY_FULL_GROUP_BY','NO_UNSIGNED_SUBTRACTION','NO_DIR_IN_CREATE','POSTGRESQL','ORACLE','MSSQL','DB2','MAXDB','NO_KEY_OPTIONS','NO_TABLE_OPTIONS','NO_FIELD_OPTIONS','MYSQL323','MYSQL40','ANSI','NO_AUTO_VALUE_ON_ZERO','NO_BACKSLASH_ESCAPES','STRICT_TRANS_TABLES','STRICT_ALL_TABLES','NO_ZERO_IN_DATE','NO_ZERO_DATE','INVALID_DATES','ERROR_FOR_DIVISION_BY_ZERO','TRADITIONAL','NO_AUTO_CREATE_USER','HIGH_NOT_PRECEDENCE','NO_ENGINE_SUBSTITUTION','PAD_CHAR_TO_FULL_LENGTH') NOT NULL DEFAULT '',
  `comment` text CHARACTER SET utf8 COLLATE utf8_bin NOT NULL,
  `character_set_client` char(32) CHARACTER SET utf8 COLLATE utf8_bin DEFAULT NULL,
  `collation_connection` char(32) CHARACTER SET utf8 COLLATE utf8_bin DEFAULT NULL,
  `db_collation` char(32) CHARACTER SET utf8 COLLATE utf8_bin DEFAULT NULL,
  `body_utf8` longblob,
  PRIMARY KEY (`db`,`name`,`type`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COMMENT='Stored Procedures'


```