---
title: HBase建立二级索引的几种方式
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-16 21:35:01
password:
summary:
tags:
img: /medias/featureimages/hbase.png
categories: 大数据
---

### **为什么需要HBse二级索引**

**HBase里面只有rowkey作为一级索引，** 如果要对库里的非rowkey字段进行数据检索和查询， 往往要通过MapReduce/Spark等分布式计算框架进行，硬件资源消耗和时间延迟都会比较高。

**只依赖rowkey作为索引,最高可以支持数据量为十万级的数据快速响应.**为了HBase的数据查询更高效、适应更多的场景， **诸如使用非rowkey字段检索也能做到秒级响应，或者支持各个字段进行模糊查询和多字段组合查询等**， 因此需要在HBase上面构建二级索引， 以满足现实中更复杂多样的业务需求。

#### 1.MapReduce方案

自己编写MapReduce实现二级索引,相对成本较高,但是可以将MapReduce的并发特性运用起来,相对开发成本较高,而且以来开发人员代码水平.

#### 2.Phoenix

**Apache Phoenix：** 功能围绕着SQL on hbase，支持和兼容多个hbase版本， 二级索引只是其中一块功能。 二级索引的创建和管理直接有SQL语法支持，使用起来很简便， 该项目目前社区活跃度和版本更新迭代情况都比较好。

**ApachePhoenix在目前开源的方案中，是一个比较优的选择。主打SQL on HBase ， 基于SQL能完成HBase的CRUD操作，支持JDBC协议**。 Apache Phoenix在Hadoop生态里面位置：

**Phoenix二级索引特点：**

Covered Indexes(覆盖索引) ：把关注的数据字段也附在索引表上，只需要通过索引表就能返回所要查询的数据（列）， 所以索引的列必须包含所需查询的列(SELECT的列和WHRER的列)。

Functional indexes(函数索引)： 索引不局限于列，支持任意的表达式来创建索引。

Global indexes(全局索引)：适用于读多写少场景。通过维护全局索引表，所有的更新和写操作都会引起索引的更新，写入性能受到影响。 在读数据时，Phoenix SQL会基于索引字段，执行快速查询。

Local indexes(本地索引)：适用于写多读少场景。 在数据写入时，索引数据和表数据都会存储在本地。在数据读取时， 由于无法预先确定region的位置，所以在读取数据时需要检查每个region（以找到索引数据），会带来一定性能（网络）开销。

**该方案可适用于数量级百万甚至千万.**

#### 3.ElasticSearch

总所周知,ElasticSearch是一个基于Lucene开发的分布式搜索引擎,可以做到对亿级数据实现快速查询,我们只需要将需要建立二级索引的字段与rowkey放入elasticsearch中,需要使用时,先在elasticsearch中通过字段查询到对应的rowkey,再利用hbase自身的rowkey索引搜索到对应的数据,**该方案可以实现亿级数据的快速响应**,是一种冷热分离思想的运用,