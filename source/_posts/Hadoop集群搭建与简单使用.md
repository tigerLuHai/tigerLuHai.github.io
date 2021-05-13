---
title: Hadoop集群搭建与简单使用
top: false
cover: false
toc: true
mathjax: true
date: 2019-12-01 14:13:08
password:
summary:
tags:
categories: 大数据
---

# Hadoop集群搭建与简单使用

首先需要搭建一个linux的集群,可以参见我的博客[linux集群搭建](https://tigerluhai.github.io/2019/12/29/shi-yong-vmware-an-zhuang-linux-xu-ni-ji/)

> Hadoop的运行是基于java的,所以需要先安装JDK,而且JDK版本必须高于1.7

### 安装JDK

（1）查询是否安装Java软件：

```shell
 rpm -qa | grep java
```

（2）如果安装的版本低于1.7，卸载该JDK：

 ```shell
sudo rpm -e 软件包
 ```

（3）查看JDK安装路径：

```shell
 which(or whereis) java
```

（4）解压JDK：

```shell
tar -zxvf 安装包名 -C 目标路径
```

（5）配置JDK环境：

```shell
vi /etc/profile

在文件末尾加上
#JAVA_HOME
export JAVA_HOME=/opt/module/jdk1.8.0_144
export PATH=$PATH:$JAVA_HOME/bin
```

**修改后需要立即生效需要运行如下命令:**

```shell
source /etc/profile
```

（6）测试JDK安装是否成功：

```shell
java -version
```

### 安装Hadoop

（1）解压hadoop安装包到指定位置

```shell
tar -zxvf 安装包 -C 指定目录
```

（2）添加环境变量

```shell
vi /etc/profile

在文件末尾加上
#HADOOP_HOME
export HADOOP_HOME=/opt/module/hadoop-3.1.2
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```

(3)修改配置文件

- 集群部署规划

|      | master        | slave1                  | slave2       | slave3                 |
| ---- | ---------------- | -------------------------- | ------------------------- | ------------------------- |
| HDFS | NameNodeDataNode | SecondaryNameNode DataNode               | DataNode |DataNode|
| YARN | NodeManager      | ResourceManager | NodeManager               |NodeManager|

- 核心配置文件

配置core-site.xml

```xml
vi core-site.xml

# 在该文件中编写如下配置
<!-- 指定HDFS中NameNode的地址 -->
<property>
	<name>fs.defaultFS</name>
    <value>hdfs://master:9000</value>
</property>
<!-- 指定Hadoop运行时产生文件的存储目录 -->
<property>
	<name>hadoop.tmp.dir</name>
	<value>/opt/module/hadoop-3.1.2/data/tmp</value>
</property>
```

- HDFS配置文件

配置hadoop-env.sh

```shell
 vi hadoop-env.sh

export JAVA_HOME=/opt/module/jdk1.8.0_144
```

配置hdfs-site.xml

```xml
vi hdfs-site.xml

<!--配置副本数量-->
<property>
		<name>dfs.replication</name>
		<value>3</value>
</property>
<!-- 指定Hadoop辅助名称节点主机配置 -->
<property>
      <name>dfs.namenode.secondary.http-address</name>
      <value>slave1:50090</value>
</property>

```

注意:文件副本数量不只是dfs.replication决定,而是min(datanode节点数,dfs.replication)

- YARN配置文件

配置yarn-env.sh

```shell
 vi yarn-env.sh

export JAVA_HOME=/opt/module/jdk1.8.0_144
```

- 配置yarn-site.xml

```xml
 vi yarn-site.xml

<!-- Reducer获取数据的方式 -->
<property>
	<name>yarn.nodemanager.aux-services</name>
	<value>mapreduce_shuffle</value>
</property>
<!-- 指定YARN的ResourceManager的地址 -->
<property>
	<name>yarn.resourcemanager.hostname</name>
	<value>slave2</value>
</property>
```

- MapReduce配置文件

```shell
vi mapred-env.sh

export JAVA_HOME=/opt/module/jdk1.8.0_144
```

**注意：hadoop3之前的版本,mapreduce会继承hadoop的配置,所以可以不用配置这一项,但是3以后的版本必须配置,否则运行mapreduce时会报环境出错。**

- 配置mapred-site.xml

```xml
cp mapred-site.xml.template mapred-site.xml
vi mapred-site.xml

<!-- 指定MR运行在Yarn上 -->
<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
</property>
```

（4）在集群上分发配置好的Hadoop配置文件

编写分发脚本

```shell
vi /usr/local/bin/xsync

#!/bin/bash
pcount=$#
if((pcount=0));then
echo no args;
exit;
fi
p1=$1
fname=`basename $p1`
echo fname=$fname
pdir=`cd -P $(dirname $p1);pwd`
echo pdir=$pdir

user=`whoami`
echo user=$user
for((host=1;host<4;host++));do
echo ------------salve$host-------------
rsync -rvl $pdir/$fname $user@slave$host:$pdir
done
```

 ```shell
xsync /opt/module/hadoop-3.1.2/
 ```

### 集群单节点启动

（1）如果集群是第一次启动，需要**格式化NameNode**

```shell
hadoop namenode -format
```

（2）在master上启动NameNode

```shell
hadoop-daemon.sh start namenode
```

查看节点启动状态

```shell
 jps
```

**注意:jps用于查看java进程,namenode和datanode都是java进程**

（3）在master,slave1,slave2以及slave3上分别启动DataNode

```shell
hadoop-daemon.sh start datanode
```

```shell
 jps
 
3461 NameNode
3608 Jps
3561 DataNode
```

思考：每次都一个一个节点启动，如果节点数太多怎么办？

### 群起集群

1.	配置slaves

```shell
vi /opt/module/hadoop-3.1.2/etc/hadoop/slaves

在该文件中增加如下内容：
master
slave1
slave2
slave3
```

**注意：该文件中添加的内容结尾不允许有空格，文件中不允许有空行。**

同步所有节点配置文件

```shell
 xsync slaves
```

2.	启动集群

（1）如果集群是第一次启动，需要格式化NameNode（注意格式化之前，一定要先停止上次启动的所有namenode和datanode进程，然后再删除data和log数据）

```shell
hdfs namenode -format
```

（2）启动HDFS

```shell
start-dfs.sh
```

（3）启动YARN

```shell
start-yarn.sh
```

**注意：NameNode和ResourceManger如果不是同一台机器，不能在NameNode上启动 YARN，应该在ResouceManager所在的机器上启动YARN。**

### 集群基本操作

- -ls: 显示目录信息

```shell
hadoop fs -ls /
```

- mkdir：在HDFS上创建目录

```shell
 hadoop fs -mkdir -p /parent/test
```

- moveFromLocal：从本地剪切粘贴到HDFS

```shell
hadoop fs  -moveFromLocal  ./kongming.txt  /parent/test
```

- appendToFile：追加一个文件到已经存在的文件末尾

```shell
 hadoop fs -appendToFile liubei.txt /parent/test/kongming.txt
```

- cat：显示文件内容

```shell
hadoop fs -cat /parent/test/kongming.txt
```

- chgrp 、-chmod、-chown：Linux文件系统中的用法一样，修改文件所属权限
- copyFromLocal：从本地文件系统中拷贝文件到HDFS路径去

```shell
 hadoop fs -copyFromLocal README.txt /
```

- copyToLocal：从HDFS拷贝到本地

```shell
 hadoop fs -copyToLocal /parent/test/kongming.txt ./
```

- cp ：从HDFS的一个路径拷贝到HDFS的另一个路径

```shell
hadoop fs -cp /parent/test/kongming.txt  /zhuge.txt
```

- mv：在HDFS目录中移动文件

```shell
hadoop fs -mv /zhuge.txt /parent/test/
```

- get：等同于copyToLocal，就是从HDFS下载文件到本地

```shell
hadoop fs -get /parent/test/kongming.txt ./
```

- getmerge：合并下载多个文件，比如HDFS的目录 /parent/test//test下有多个文件:log.1, log.2,log.3,...

```shell
hadoop fs -getmerge /parent/test/test/* ./zaiyiqi.txt
```

- put：等同于copyFromLocal

```shell
 hadoop fs -put ./zaiyiqi.txt /
```

- tail：显示一个文件的末尾


- rm：删除文件或文件夹


- rmdir：删除空目录


- du统计文件夹的大小信息

### JAVA代码操作HDFS

```java
Logger logger = LoggerFactory.getLogger(HdfsClient.class);

FileSystem fileSystem;

Configuration configuration;

@Test
/**
     * 测试环境正常
     */
public void HDFS_ENV() throws IOException {
    Logger logger = LoggerFactory.getLogger(HdfsClient.class);
    //1.获取hdfs客户端对象
    Configuration configuration = new Configuration();
    configuration.set("fs.defaultFS","hdfs://master:9000");
    FileSystem fileSystem = FileSystem.get(configuration);
    //2.在hdfs上执行相关操作
    boolean mkdirs = fileSystem.mkdirs(new Path("/client_test_environment"));
    //3.关闭资源
    fileSystem.close();

    System.out.println(mkdirs);
}

@Before
/**
     * 创建fileSystem对象
     */
public void createFileSystem() throws URISyntaxException, IOException, InterruptedException {
    //1.获取fs对象
    configuration = new Configuration();
    fileSystem = FileSystem.get(new URI("hdfs://master:9000"), configuration, "root");
}

@Test
public void copyFromLocalFile() throws IOException{
    //执行上传操作
    fileSystem.copyFromLocalFile(new Path("D:\\logs\\xc.2019-04-29.log"),new Path("/client_test_environment/"));
    //关闭资源
    fileSystem.close();
}


@Test
public void copyToLocalFile() throws IOException{
    //执行上传操作
    fileSystem.copyToLocalFile(true,new Path("/client_test_environment/xc.2019-04-29.log"),new Path("D:\\logs\\xc.2019-04-29-back.log"),false);
    //关闭资源
    fileSystem.close();
}

@Test
public void listFiles() throws IOException {
    //查看文件信息
    RemoteIterator<LocatedFileStatus> iterator = fileSystem.listFiles(new Path("/"), true);
    while (iterator.hasNext()){
        LocatedFileStatus fileStatus = iterator.next();
        //获取文件名称，文件权限，文件长度，块信息
        logger.info(fileStatus.getPath().getName());
        logger.info(fileStatus.getLen()+"");
        logger.info(fileStatus.getPermission()+"");
        BlockLocation[] blockLocations = fileStatus.getBlockLocations();
        for (BlockLocation blockLocation : blockLocations) {
            logger.info(blockLocation.toString());
        }
        logger.info("----------------                    ----------------------");
    }
}
```

