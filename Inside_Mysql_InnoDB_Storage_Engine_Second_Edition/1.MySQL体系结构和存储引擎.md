# 第一章 MySQL体系结构和存储引擎

## 1.1数据据库和实例

* 数据库
  * 物理系统文件
  * 是文件的集合
  * 是按照某种数据模型组织起来的并存放与二级存储器中的数据集合。
* 数据库实例
  * 程序
  * MySQL数据库由后台线程和一个共享内存组成
  * 是位于OS和用户之间的数据管理软件
  * 其他应用程序只有通过数据库实例和数据库打交道
* MySQL被设计为单线程多进程的程序，MySQL的实例在OS上的表现就是一个进程。
* 可以通过ps  -ef | grep mysqld来查看这个实例进程
* 可以通过mysql --help | grep my.cnf来查看MySQL的配置文件。一般会查询到很多个，具体的配置会按照最后一个列出来的配置文件为准。
* 可以用show variabiles like "%datadir%" 来查看mysql的数据目录
* 在mysql的命令终端可以用 system cmd 来执行系统命令

## 1.2MySQL体系结构

* C/S模式
* 组成
  * 连接池组件：可以用C API、JDBC、ODBC……
  * 服务管理和工具：backup、replication……
  * SQL接口组件
  * 查询分析组件
  * 优化器组件
  * 缓冲（cache）组件
  * 插件式的存储引擎
  * 文件系统（物理文件）



## 1.3存储引擎

### 1.3.1InnoDB

* 支持事务、行锁、支持外键，非锁定读
* 通过MVCC（对版本并发控制）来获取高并发
* 实现4中隔离级别，默认为REPEATABLE
* 使用next-key locking算法避免幻读（ohantom）
* 还提供插入缓冲、二次写、自适应哈希index、预读等高性能特性
* 数据采用聚集（clustered）的方式，每张表的存储都安主键顺序，如果没有显示的定义，默认指定6字节的ROWID作为主键

## 1.4引擎的比较

* 文件系统和数据库的最大区别就是：数据库支持事务

## 1.5连接数据库

*  连接数据库是：一个连接进程和MySQL数据库实例进行通行。本质上是进程通行。常见的进程通信方式有：管道、命名管道、socker等。

### 1.5.1TCP/IP

* 也就是我们常说的用mysql客户端连接 mysql -h -u -P -p

* 这个连接的过程中：MySQL数据库会先检查一张权限视图，校验连接者的权限

* 这个视图：

  ```linux
  select host,user,passward from mysql.user;
  ```


### 1.5.2管道和共享内存

### 1.5.3UNIX域套接字

* unix域套接字不是网络协议，只能用于MySQL数据库实例和客户端在同一台服务器上的情况
* 当数据库启动后可以用 show variables like "%socket%"来查看unix域套接字文件
* 连接：mysql -uwalter -S  /tmp/mysql.sock

