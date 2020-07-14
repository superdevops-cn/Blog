---
title: Mysql 8 高级-18 MySql日志
date: 2020-03-29 19:36:26
tags: 
    - Mysql
    - CentOS
typora-root-url: ..
---

​       在任何一种数据库中，都会有各种各样的日志，记录着数据库工作的方方面面，以帮助数据库管理员追踪数据库曾经发生过的各种事件。MySQL 也不例外，在 MySQL 中，有 4 种不同的日志，分别是错误日志、二进制日志（BINLOG 日志）、查询日志和慢查询日志，这些日志记录着数据库在不同方面的踪迹。

### 1. 错误日志

   错误日志是 MySQL 中最重要的日志之一，它记录了当 mysqld 启动和停止时，以及服务器在运行过程中发生任何严重错误时的相关信息。当数据库出现任何故障导致无法正常使用时，可以首先查看此日志。

​       该日志是默认开启的 ， 默认存放目录为 mysql 的数据目录（var/lib/mysql）, 默认的日志文件名为hostname.err（hostname是主机名）。

<!--more-->

查看日志位置指令 ： 

```mysql
show variables like 'log_error%';
```

![image-20200419220620677](/image/mysql/18/180001.png)



查看日志内容 ： 

```mysql
tail -f /var/log/mysqld.log
```



### 2. 二进制日志

#### 2.1 概述

​        二进制日志（BINLOG）记录了所有的 DDL（数据定义语言）语句和 DML（数据操纵语言）语句，但是不包括数据查询语句。此日志对于灾难时的数据恢复起着极其重要的作用，MySQL的主从复制， 就是通过该binlog实现的。

​        二进制日志，默认情况下是没有开启的，需要到MySQL的配置文件中开启，并配置MySQL日志的格式。 

配置文件位置 :

```mysql
 vi  /etc/my.cnf
```



  日志存放位置 :

​      配置时，给定了文件名但是没有指定路径，日志默认写入Mysql的数据目录。

```mysql
#配置开启binlog日志， 日志的文件前缀为 mysqlbin -----> 生成的文件名如 : mysqlbin.000001,mysqlbin.000002
log_bin=mysqlbin

#配置二进制日志的格式
binlog_format=STATEMENT
```

重启mysql:

```mysql
service mysqld restart
```



#### 2.2 日志格式

##### 2.2.1 STATEMENT

​       该日志格式在日志文件中记录的都是SQL语句（statement），每一条对数据进行修改的SQL都会记录在日志文件中，通过Mysql提供的mysqlbinlog工具，可以清晰的查看到每条语句的文本。主从复制的时候，从库（slave）会将日志解析为原文本，并在从库重新执行一次。

##### 2.2.2 ROW

​    该日志格式在日志文件中记录的是每一行的数据变更，而不是记录SQL语句。比如，执行SQL语句 ： update tb_book set status='1' , 如果是STATEMENT 日志格式，在日志中会记录一行SQL文件； 如果是ROW，由于是对全表进行更新，也就是每一行记录都会发生变更，ROW 格式的日志中会记录每一行的数据变更。

##### 2.2.3  MIXED

​       这是目前MySQL默认的日志格式，即混合了STATEMENT 和 ROW两种格式。默认情况下采用STATEMENT，但是在一些特殊情况下采用ROW来进行记录。MIXED 格式能尽量利用两种模式的优点，而避开他们的缺点。

#### 2.3  日志读取

​      由于日志以二进制方式存储，不能直接读取，需要用mysqlbinlog工具来查看，语法如下 ：

```mysql
mysqlbinlog log-file;
```

##### 2.3.1  查看STATEMENT格式日志

查看日志格式：

```mysql
show variables like 'binlog_format'
```

![image-20200419222433511](/image/mysql/18/180002.png)

执行插入语句 ：

```mysql
insert into tb_book values(null,'Lucene','2088-05-01','0');
```

查看二进制文件文件名：

```mysql
show binary logs;
```

![image-20200419223139492](/image/mysql/18/180003.png)

查找日志所在位置：

```mysql
 find / -name 'mysqlbin.*'
```

![image-20200419224405527](/image/mysql/18/180004.png)

查看日志文件 ：

```mysql
mysqlbinlog /var/lib/mysql/mysqlbin.000001;
```

![image-20200419224504600](/image/mysql/18/180005.png)

##### 2.3.2 查看ROW格式日志 

配置 :

```mysql
#配置开启binlog日志， 日志的文件前缀为 mysqlbin -----> 生成的文件名如 : mysqlbin.000001,mysqlbin.000002
log_bin=mysqlbin

#配置二进制日志的格式
binlog_format=ROW
```

重启mysql

```mysq
service mysqld restart
```

查看日志格式：

```mysql
show variables like 'binlog_format'
```



![image-20200419224026345](/image/mysql/18/180006.png)

插入数据：

```mysql
insert into tb_book values(null,'SpringCloud实战','2088-05-05','0');
```

查看二进制文件文件名：

```mysql
#在mysql客户端执行
show binary logs;
```

![image-20200419224111885](/image/mysql/18/180007.png)

查找日志所在位置：

```mysql
 find / -name 'mysqlbin.*'
```

![image-20200419224405527](/image/mysql/18/180008.png)

如果日志格式是 ROW , 直接查看数据 , 是查看不懂的 ; 可以在mysqlbinlog 后面加上参数 -vv

```mysql
mysqlbinlog -vv /var/lib/mysql/mysqlbin.000002
```

![image-20200419224720357](/image/mysql/18/180009.png)

#### 2.4 日志删除

​     对于比较繁忙的系统，由于每天生成日志量大 ，这些日志如果长时间不清楚，将会占用大量的磁盘空间。下面我们将会讲解几种删除日志的常见方法 ：

##### 2.4.1   方式一

通过 Reset Master 指令删除全部 binlog 日志，删除之后，日志编号，将从 xxxx.000001重新开始 。

查询之前 ，先查询下日志文件 ： 

```mysql
 find / -name 'mysqlbin.*'
```

![image-20200419230721346](/image/mysql/18/180010.png)

执行删除日志指令： 

```mysql
#在mysql客户端执行
Reset Master
```

执行之后， 查看日志文件 ：

```mysql
 find / -name 'mysqlbin.*'
```

![image-20200419230921065](/image/mysql/18/180011.png)

##### 2.4.1 方式二

执行指令:

```mysql
purge  master logs to 'mysqlbin.******'
```

该命令将删除  ``` ******``` 编号之前的所有日志。 

##### 2.4.3   方式三

执行指令 :

```mysql
 purge master logs before 'yyyy-mm-dd hh24:mi:ss'
```

该命令将删除日志为 "yyyy-mm-dd hh24:mi:ss" 之前产生的所有日志 。

##### 2.4.4  方式四

设置参数：

```mysql
#在/etc/my.cnf中配置
--expire_logs_days=N
```

此参数的含义是设置日志的过期天数， 过了指定的天数后日志将会被自动删除，这样将有利于减少DBA 管理日志的工作量。

### 3. 查询日志

​     查询日志中记录了客户端的所有操作语句，而二进制日志不包含查询数据的SQL语句。

​      默认情况下， 查询日志是未开启的。如果需要开启查询日志，可以设置以下配置 ：

```mysql
#该选项用来开启查询日志 ， 可选值 ： 0 或者 1 ； 0 代表关闭， 1 代表开启 
general_log=1

#设置日志的文件名 ， 如果没有指定， 默认的文件名为 host_name.log 
general_log_file=file_name
```

在 mysql 的配置文件 /etc/my.cnf 中配置如下内容 ： 

```mysql
#开启查询日志
general_log=1

#设置日志的文件名,默认的文件名为 host_name.log 
general_log_file=mysql_query.log
```

重启mysql：

```mysql
service mysqld restart
```

配置完毕之后，在数据库执行以下操作 ：

```mysql
select * from tb_book;
select * from tb_book where id = 1;
update tb_book set name = 'lucene入门指南' where id = 5;
select * from tb_book where id < 8;
```

查找文件位置：

```mysql
find / -name  'mysql_query*'
```

![image-20200419231829586](/image/mysql/18/180012.png)

执行完毕之后， 再次来查询日志文件 ： 

```mysql
cat  /var/lib/mysql/mysql_query.log
```

![image-20200419231914887](/image/mysql/18/180013.png)

### 4. 慢查询日志

​      慢查询日志记录了所有执行时间超过参数 long_query_time 设置值并且扫描记录数不小于 min_examined_row_limit 的所有的SQL语句的日志。long_query_time 默认为 10 秒，最小为 0， 精度可以到微秒。

#### 4.1 文件位置和格式

慢查询日志默认是关闭的 。可以通过两个参数来控制慢查询日志 ：

```mysql
# 该参数用来控制慢查询日志是否开启， 可取值： 1 和 0 ， 1 代表开启， 0 代表关闭
slow_query_log=1 

# 该参数用来指定慢查询日志的文件名
slow_query_log_file=slow_query.log

# 该选项用来配置查询的时间限制， 超过这个时间将认为值慢查询， 将需要进行日志记录， 默认10s
long_query_time=10

```

#### 4.2 日志的读取

和错误日志、查询日志一样，慢查询日志记录的格式也是纯文本，可以被直接读取。

##### 4.2.1 查询long_query_time 的值。

```mysql
show variables like 'long_query_time';
```

![image-20200419232039135](/image/mysql/18/180014.png)

##### 4.2.2 执行查询操作

```sql
select * from tb_book;
```

![image-20200419232531237](/image/mysql/18/180015.png)

由于该语句执行时间很短，为0s ， 所以不会记录在慢查询日志中。

##### 4.2.3查看慢查询日志文件

查看慢查询日志路径：

```mysql
show variables like 'slow_%';
```

![image-20200419232855770](/image/mysql/18/180016.png)

直接通过cat 指令查询该日志文件 ： 

 ```mysql
cat /var/lib/mysql/localhost-slow.log
 ```

如果慢查询日志内容很多， 直接查看文件，比较麻烦， 这个时候可以借助于mysql自带的 mysqldumpslow 工具， 来对慢查询日志进行分类汇总。

```mysql
mysqldumpslow file_name;  
```

### 666. 彩蛋

请大家持续关注公众号：Java橙序猿

 ![](/image/common/superdevops.jpg) 





关注博客：

```
 http://superdevops.cn
```


