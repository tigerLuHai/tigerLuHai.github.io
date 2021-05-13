---
title: Sqoop-HDFS与外界交互数据的工具
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-16 21:34:23
password:
summary:
tags:
categories: 大数据
---

# Sqoop简介

Sqoop是一款开源的工具，主要用于在Hadoop(Hive)与传统的数据库(mysql、postgresql...)间进行数据的传递，可以将一个关系型数据库（例如 ： MySQL ,Oracle ,Postgres等）中的数据导进到Hadoop的HDFS中，也可以将HDFS的数据导进到关系型数据库中。

Sqoop项目开始于2009年，最早是作为Hadoop的一个第三方模块存在，后来为了让使用者能够快速部署，也为了让开发人员能够更快速的迭代开发，Sqoop独立成为一个[Apache](https://baike.baidu.com/item/Apache/6265)项目。

Sqoop2的最新版本是1.99.7。请注意，2与1不兼容，且特征不完整，它并不打算用于生产部署。

# Sqoop原理

将导入或导出命令翻译成mapreduce程序来实现。

在翻译出的mapreduce中主要是对inputformat和outputformat进行定制。

# Sqoop安装

安装Sqoop的前提是已经具备Java和Hadoop的环境。

## 3.1 下载并解压

1) 下载地址：<http://mirrors.hust.edu.cn/apache/sqoop/1.4.6/>

2) 上传安装包sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz到虚拟机中

3) 解压sqoop安装包到指定目录，如：

```shell
 tar -zxf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz -C /opt/module/
```

## 3.2 修改配置文件

Sqoop的配置文件与大多数大数据框架类似，在sqoop根目录下的conf目录中。

1) 重命名配置文件

```shell
 mv sqoop-env-template.sh sqoop-env.sh
```

2) 修改配置文件

```html
# sqoop-env.sh
export HADOOP_COMMON_HOME=/opt/module/hadoop-2.7.2
export HADOOP_MAPRED_HOME=/opt/module/hadoop-2.7.2
export HIVE_HOME=/opt/module/hive
export ZOOKEEPER_HOME=/opt/module/zookeeper-3.4.10
export ZOOCFGDIR=/opt/module/zookeeper-3.4.10
export HBASE_HOME=/opt/module/hbase
```

## 3.3 拷贝JDBC驱动

拷贝jdbc驱动到sqoop的lib目录下，如：

```shell
cp mysql-connector-java-5.1.27-bin.jar /opt/module/sqoop-1.4.6.bin__hadoop-2.0.4-alpha/lib/
```

## 3.4 验证Sqoop

我们可以通过某一个command来验证sqoop配置是否正确：

$ bin/sqoop help

出现一些Warning警告（警告信息已省略），并伴随着帮助命令的输出：

```html
Available commands:
  codegen            Generate code to interact with database records
  create-hive-table     Import a table definition into Hive
  eval               Evaluate a SQL statement and display the results
  export             Export an HDFS directory to a database table
  help               List available commands
  import             Import a table from a database to HDFS
  import-all-tables     Import tables from a database to HDFS
  import-mainframe    Import datasets from a mainframe server to HDFS
  job                Work with saved jobs
  list-databases        List available databases on a server
  list-tables           List available tables in a database
  merge              Merge results of incremental imports
  metastore           Run a standalone Sqoop metastore
  version            Display version information
```

## 3.5 测试Sqoop是否能够成功连接数据库

```shell
 bin/sqoop list-databases --connect jdbc:mysql://slave2:3306/ --username root --password 000000
```

出现如下输出：

```html
information_schema
metastore
mysql
oozie
performance_schema
```

# 第4章 Sqoop的简单使用案例

## 4.1 导入数据

在Sqoop中，“导入”概念指：从非大数据集群（RDBMS）向大数据集群（HDFS，HIVE，HBASE）中传输数据，叫做：导入，即使用import关键字。

### 4.1.1 RDBMS到HDFS

1) 确定Mysql服务开启正常

2) 在Mysql中新建一张表并插入一些数据

```sql
mysql -uroot -p000000
mysql> create database company;
mysql> create table company.staff(id int(4) primary key not null auto_increment, name varchar(255), sex varchar(255));
mysql> insert into company.staff(name, sex) values('Thomas', 'Male');
mysql> insert into company.staff(name, sex) values('Catalina', 'FeMale');
```

3) 导入数据

```shell
 bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t"
```

## 4.2、导出数据

在Sqoop中，“导出”概念指：从大数据集群（HDFS，HIVE，HBASE）向非大数据集群（RDBMS）中传输数据，叫做：导出，即使用export关键字。

### 4.2.1 HIVE/HDFS到RDBMS

```shell
bin/sqoop export \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--export-dir /user/hive/warehouse/staff_hive \
--input-fields-terminated-by "\t"
```

提示：Mysql中如果表不存在，不会自动创建