# 内容简介 #

&emsp;&emsp;Pentaho Kettle是一款国外开源的ETL工具，纯java编写，可以在Window、Linux、Unix上运行，绿色无需安装，部署方式灵活，可按照实际项目需求构建ETL集群。本文详细描 Kettle 7.1 与 TiDB 部署联调过程，并对测试过程进行详细记录，并对测试过程问题及其处理方式进行说明。

&emsp;&emsp;最新版 Kettle 可以在官网下载： http://kettle.pentaho.org

&emsp;&emsp;常见部署平台为 Windows 和 Linux，部署非常简单，解压到指定目录即可。注意：Kettle 纯java编写，所以程序运行依赖Java环境，需配置好JDK及环境变量。（[JDK下载](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html "JDK下载地址")，win and linux java环境变量配置自行百度）

# 测试环境 #


&emsp;&emsp;Kettle 部署环境

| 名称    | Win   |  Linux  |
| ------- | -----:| :----: |
| 操作系统  | Windows 8 |  CentOS 7.3  |
| JDK   | 1.8    |   1.8   |
| TiDB  | Pre_GA |   Pre_GA   |
| mysql | 5.6    | 5.6 |


&emsp;&emsp;TiDB本地测试环境(Centos 7.3  4台虚拟机)

|Name	|Host IP	|Services |
| --------  | -----:   | :----: |
|node1	|172.168.100.160|	PD, TiDB, Monitor|
|node2	|172.168.100.161|	TiKV1|
|node3	|172.168.100.162|	TiKV2|
|node4	|172.168.100.163|	TiKV3|

&emsp;&emsp;


# 详细测试 #
### 1、配置TiDB为资源库 [JDBC] ###
1. 解压Kettle到本地目录：D:\data-integration，从MySQL下载mysql-connector-java驱动包,解压后将对应的jar包copy到D:\data-integration\lib下;
2. 启动Kettle(spoon.bat)，如下图所示，主要有2中任务，一个是作业，一个是转换。一般来说，转换是一系列具体的操作，比如：调度SP，导出Excel等等；作业，就是按照一定流程来调度一系列转换。
![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k01.jpg)

3. 资源库用户的创建
```
mysql -uroot -h 192.168.100.160 -P 4000
>> create database resource;
>> create user kettle identified by '111111';
>> grant all privileges on resource.* to 'kettle'@'localhost';
>> \q
mysql -ukettle -p111111 -h 192.168.100.160 -P 4000
>> show databases;
```
4. 资源库配置，点击右上角connect，进入Pentaho Repository，选择Other Repositories --> Database Repository --> Get Started
![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k02.jpg)
--> Database Connection 配置如下
![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/03.jpg)
点击测试，成功连接数据库。

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
### 2、ODBC配置测试 ###
1、官网下载Mysql ODBC并安装。

2、控制面板-> 管理工具 -> ODBC 数据源（64位）
![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k06.jpg)

服务端对应TiDB数据库为UTF-8，所以选择ODBC驱动的的时候请注意，否则报错如下：

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k04.jpg)

ODBC链接TiDB测试通过。

![image](https://github.com/houzhaodun/test_kettle/raw/master/kettle-test/k05.jpg)

### 3、功能测试 ###
1. 准备测试数据表，测试数据;
2. 数据加载测试【Excel，文本文件】
3. 