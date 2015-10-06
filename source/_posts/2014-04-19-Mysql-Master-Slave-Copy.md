---
layout: post
title: "Mysql主从复制原理"
date: 2014-04-19 22:21:35 +0800
comments: true
categories: 
  - Mysql
---
## 目录

1. [Mysql复制介绍](#Begin)
1. [复制常用框架](#Framework)
2. [Master服务器启动mysql](#Master)
3. [Slave服务器配置](#Slave)


## <a id="Begin">Mysql复制介绍</a>
分为同步复制和异步复制，实际复制架构中大部分为异步复制。

复制的基本过程如下：

1. Slave上面的IO进程连接上Master，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容；
1. Master接收到来自Slave的IO进程的请求后，通过负责复制的IO进程根据请求信息读取制定日志指定位置之后的日志信息，返回给Slave 的IO进程。返回信息中除了日志所包含的信息之外，还包括本次返回的信息已经到Master端的bin-log文件的名称以及bin-log的位置；
1. Slave的IO进程接收到信息后，将接收到的日志内容依次添加到Slave端的relay-log文件的最末端，并将读取到的Master端的 bin-log的文件名和位置记录到master-info文件中，以便在下一次读取的时候能够清楚的告诉Master“我需要从某个bin-log的哪个位置开始往后的日志内容，请发给我”；
1. Slave的Sql进程检测到relay-log中新增加了内容后，会马上解析relay-log的内容成为在Master端真实执行时候的那些可执行的内容，并在自身执行。

<!--more-->

Mysql为了解决这个风险并提高复制的性能，将Slave端的复制改为两个进程来完成。提出这个改进方案的人是Yahoo!的一位工程师“Jeremy Zawodny”。这样既解决了性能问题，又缩短了异步的延时时间，同时也减少了可能存在的数据丢失量。当然，即使是换成了现在这样两个线程处理以后，同样也还是存在slave数据延时以及数据丢失的可能性的，毕竟这个复制是异步的。只要数据的更改不是在一个事物中，这些问题都是会存在的。如果要完全避免这些问题，就只能用mysql的cluster来解决了。不过mysql的cluster是内存数据库的解决方案，需要将所有数据都load到内存中，这样就对内存的要求就非常大了，对于一般的应用来说可实施性不是太大。

## <a id="Framework">复制常用框架</a>
Mysql复制环境90%以上都是一个Master带一个或者多个Slave的架构模式，主要用于读压力比较大的应用的数据库端廉价扩展解决方案。因为只要master和slave的压力不是太大（尤其是slave端压力）的话，异步复制的延时一般都很少很少。尤其是自slave端的复制方式改成两个进程处理之后，更是减小了slave端的延时。而带来的效益是，对于数据实时性要求不是特别的敏感度的应用，只需要通过廉价的pc server来扩展slave的数量，将读压力分散到多台slave的机器上面，即可解决数据库端的读压力瓶颈。这在很大程度上解决了目前很多中小型网站的数据库压力瓶颈问题，甚至有些大型网站也在使用类似方案解决数据库瓶颈。

**Mysql主从复制配置过程：**  
环境：    
<code>
master: 192.168.0.3  
Slave: 192.168.0.4  
Mysql版本为5.0.67（编译安装）  
database: eric
</code>  

## <a id="Master">Master服务器启动mysql</a>
1. mysql –uroot –proot
2. 创建一个有复制权限的用户，只限slave远程连接访问.
mysql>grant replication slave on *.* to replication@192.168.0.4 identified by ‘password’;
mysql>flush privileges;
3. mysql>flush tables with read lock; #锁定master服务器所有表的写入。
4. 重新打开一终端，备份要复制的数据库。  
Var]#tar zcvf eric.tar.gz eric/    //eric所在路径/opt/mysql/var/eric/即一个库。  
]#scp eric.tar.gz 192.168.0.4:/opt/mysql/var/    //将主服务器的库传到slave相应路径下。
5. 返回上一终端。  
Mysql>show master status; 
 
	+------------------+----------+--------------+------------------+  
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |  
	+------------------+----------+--------------+------------------+  
	| mysql-bin.000014 |       98 | eric         |                  |  
	+------------------+----------+--------------+------------------+  
	
1 row in set (0.01 sec)  
其中mysql-bin.000014和98二个值将是slave与master的同步点。  
6. 给数据库解锁（当备份完成后） mysql>unlock tables;  
7. 编辑mysql的配置文件。Vim /etc/my.cnf 设置这三个参数，没有的添加，有的直接更改即可。  
log-bin=mysql-bin  
server-id  = 1  
binlog-do-db=eric  
保存退出。  

##<a id="Slave">Slave服务器配置</a>
1. 注意顺序，先重启master-à 然后是slave.
2. Slave服务器重启后，登录mysql
	mysql> stop slave;  
	
Query OK, 0 rows affected (0.00 sec)

	mysql> change master to  
    -> master_host='192.168.0.3',  
    -> master_user='replication',  
    -> master_password='password',  
    -> master_log_file='mysql-bin.000014',  
    -> master_log_pos=98;  
Query OK, 0 rows affected (0.02 sec)  

mysql> start slave;  

	Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G;  

	*************************** 1. row ***************************
	Slave_IO_State: Waiting for master to send event  
	Master_Host: 192.168.0.3  
	Master_User: replication  
	Master_Port: 3306  
	Connect_Retry: 60  
	Master_Log_File: mysql-bin.000014  
	Read_Master_Log_Pos: 98  
	Relay_Log_File: alan-relay-bin.000002  
	Relay_Log_Pos: 235  
	Relay_Master_Log_File: mysql-bin.000014  
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
	Exec_Master_Log_Pos: 98  
	Relay_Log_Space: 235  
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
	1 row in set (0.01 sec)  

ERROR: No query specified  
当这个参数都为yes时，证明主从复制成功  

	Slave_IO_Running: Yes      Slave_SQL_Running: Yes



