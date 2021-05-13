---
title: MapReduce中的两表join方案
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-05 18:18:44
password:
summary:
tags:
categories: 大数据
---

# MapReduce中两表join方案

### 概述

随着互联网行业的发展,数据量变得越来越大,随之而来的就是hadoop生态圈的兴起,其中MapReduce作为较原始的分布式计算框架,在当时也是解决了很多分布式计算的问题,其中包括两张表的join操作,当然其中是包含了一些特殊技巧的,本文主要介绍MapReduce常用的join实现方法,并展示一个简单的demo.

### 1.reduce side join

reduce side join是一种最简单的join方式，其主要思想如下：

在map阶段，map函数同时读取两个文件File1和File2，为了区分两种来源的key/value数据对，对每条数据打一个标签.

在reduce阶段，reduce函数获取key相同的来自File1和File2文件的value list， 然后对于同一个key，根据map阶段打上的标签分别处理实现对File1和File2中的数据进行join.

##### 合并订单表与商品表案例

```java
public class TableBean implements Writable {

    private String order_id; //订单id
    private String p_id; //产品id
    private int amount; //产品数量
    private String pname;   //产品名称
    private String flag;    //表的标记

    @Override
    public String toString() {
        return order_id + "\t" + pname + "\t" + amount;
    }

    public TableBean(String order_id, String p_id, int amount, String pname, String flag) {
        this.order_id = order_id;
        this.p_id = p_id;
        this.amount = amount;
        this.pname = pname;
        this.flag = flag;
    }

    public TableBean() {
    }

    public String getOrder_id() {
        return order_id;
    }

    public void setOrder_id(String order_id) {
        this.order_id = order_id;
    }

    public String getP_id() {
        return p_id;
    }

    public void setP_id(String p_id) {
        this.p_id = p_id;
    }

    public int getAmount() {
        return amount;
    }

    public void setAmount(int amount) {
        this.amount = amount;
    }

    public String getPname() {
        return pname;
    }

    public void setPname(String pname) {
        this.pname = pname;
    }

    public String getFlag() {
        return flag;
    }

    public void setFlag(String flag) {
        this.flag = flag;
    }

    @Override
    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeUTF(order_id);
        dataOutput.writeUTF(p_id);
        dataOutput.writeInt(amount);
        dataOutput.writeUTF(pname);
        dataOutput.writeUTF(flag);
    }

    @Override
    public void readFields(DataInput dataInput) throws IOException {
        order_id = dataInput.readUTF();
        p_id = dataInput.readUTF();
        amount = dataInput.readInt();
        pname = dataInput.readUTF();
        flag = dataInput.readUTF();
    }
}


public class TableMapper extends Mapper<LongWritable, Text,Text,TableBean> {

    private String name;

    private TableBean tableBean = new TableBean();

    private Text text = new Text();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

        String[] split = value.toString().split("\t");

        if (name.startsWith("order")){

            tableBean.setOrder_id(split[0]);
            tableBean.setP_id(split[1]);
            tableBean.setAmount(Integer.parseInt(split[2]));
            tableBean.setPname("");
            tableBean.setFlag("order");

            text.set(split[1]);
        }else {
            tableBean.setP_id(split[0]);
            tableBean.setPname(split[1]);
            tableBean.setAmount(0);
            tableBean.setOrder_id("");
            tableBean.setFlag("pd");

            text.set(split[0]);
        }
        context.write(text,tableBean);
    }
    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        //获取输入文件切片
        FileSplit fileSplit = (FileSplit) context.getInputSplit();

        //获取输入文件名字
        name = fileSplit.getPath().getName();
    }
}


public class TableReducer extends Reducer<Text,TableBean,TableBean, NullWritable> {
    @Override
    protected void reduce(Text key, Iterable<TableBean> values, Context context) throws IOException, InterruptedException {
        ArrayList<TableBean> tableBeans = new ArrayList<>();

        TableBean tableBean = new TableBean();

        for (TableBean value : values) {

            if (value.getFlag().equals("order")) {
                TableBean tableBean1 = new TableBean();
                try {
                    BeanUtils.copyProperties(tableBean1,value);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
                tableBeans.add(tableBean1);
            }else {
                try {
                    BeanUtils.copyProperties(tableBean,value);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
        for (TableBean bean : tableBeans) {
            System.out.println(tableBean.getPname());
            bean.setPname(tableBean.getPname());
            //写出数据
            context.write(bean,NullWritable.get());
        }
    }
}
package com.university.MapReduce.join.reduce_join;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class TableDriver {
    public static void main(String[] args) throws Exception {

        // 1 获取配置信息，或者job对象实例
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);

        // 2 指定本程序的jar包所在的本地路径
        job.setJarByClass(TableDriver.class);

        // 3 指定本业务job要使用的Mapper/Reducer业务类
        job.setMapperClass(TableMapper.class);
        job.setReducerClass(TableReducer.class);

        // 4 指定Mapper输出数据的kv类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(TableBean.class);

        // 5 指定最终输出的数据的kv类型
        job.setOutputKeyClass(TableBean.class);
        job.setOutputValueClass(NullWritable.class);

        // 6 指定job的输入原始文件所在目录
        FileInputFormat.setInputPaths(job, new Path("C:\\Users\\tiger\\Desktop\\code1_count\\input1"));
        FileOutputFormat.setOutputPath(job, new Path("C:\\Users\\tiger\\Desktop\\code1_count\\flow_count9"));

        // 7 将job中配置的相关参数，以及job所用的java类所在的jar包， 提交给yarn去运行
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

### 2.map side join

reduce的阶段是为了解决map阶段不能获取所有需要的join字段，即：同一个key对应的字段可能位于不同map中。但reduce side join是非常低效的，因为shuffle阶段要进行大量的数据传输。

Map side join是针对以下场景进行的优化：两个待连接表中，有一个表非常大，而另一个表非常小，以至于小表可以直接存放到内存中。这样，我们可以将小表缓存起来放入每一个map任务中，然后只扫描大表：对于大表中的每一条记录key/value，在缓存中查找是否有相同的key的记录。

##### 合并订单表与商品表案例

```java
public class DistributedCacheMapper extends Mapper<LongWritable, Text,Text, NullWritable> {

    private Map<String,String> pdMap = new HashMap<>();

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String[] split = value.toString().split("\t");

        String pId = split[1];

        String pdName = pdMap.get(pId);
        System.out.println(pdName);

        split[1] = pdName;

        StringBuffer buffer = new StringBuffer();
        for (String s : split) {
            buffer.append(s);
        }
        context.write(new Text(buffer.toString()),NullWritable.get());
    }

    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        URI[] cacheFiles = context.getCacheFiles();
        String path = cacheFiles[0].getPath();

        BufferedReader reader = new BufferedReader(new InputStreamReader(new FileInputStream(path)));

        String line;
        while(StringUtils.isNotEmpty(line = reader.readLine())){
            //切割
            String[] split = line.split("\t");
            //缓存数据到集合
            pdMap.put(split[0],split[1]);
        }
        //关闭资源
        reader.close();
    }
}


public class DistributedCacheDriver {

    public static void main(String[] args) throws IOException, URISyntaxException, ClassNotFoundException, InterruptedException {
        // 1 获取job信息
        Configuration configuration = new Configuration();
        //开启map端输出压缩
        configuration.setBoolean("mapreduce.map.output.compress",true);
        //设置map端输出压缩
        configuration.setClass("mapreduce.map.output.compress.codec", BZip2Codec.class, CompressionCodec.class);

        Job job = Job.getInstance(configuration);

        // 2 设置加载jar包路径
        job.setJarByClass(DistributedCacheDriver.class);

        // 3 关联map
        job.setMapperClass(DistributedCacheMapper.class);

        // 4 设置最终输出数据类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(NullWritable.class);

        // 5 设置输入输出路径
        FileInputFormat.setInputPaths(job, new Path("C:\\Users\\tiger\\Desktop\\code1_count\\input1\\order.txt"));
        FileOutputFormat.setOutputPath(job, new Path("C:\\Users\\tiger\\Desktop\\code1_count\\flow_count10"));

        //设置reduce端输出压缩开启
        FileOutputFormat.setCompressOutput(job,true);
        //设置压缩方式
        FileOutputFormat.setOutputCompressorClass(job,BZip2Codec.class);

        // 6 加载缓存数据
        job.addCacheFile(new URI("file:///C:/Users/tiger/Desktop/code1_count/input1/pd.txt"));

        // 7 Map端Join的逻辑不需要Reduce阶段，设置reduceTask数量为0
        job.setNumReduceTasks(0);

        // 8 提交
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

### 2.3 Semi Join

Semi Join，也叫半连接，是从分布式数据库中借鉴过来的方法。它的产生动机是：对于reduce side join，跨机器的数据传输量非常大，这成了join操作的一个瓶颈，如果能够在map端过滤掉不会参加join操作的数据，则可以大大节省网络IO。

实现方法很简单：选取一个小表，假设是File1，将其参与join的key抽取出来，保存到文件File3中，File3文件一般很小，可以放到内存中。在map阶段，使用DistributedCache将File3复制到各个TaskTracker上，然后将File2中不在File3中的key对应的记录过滤掉，剩下的reduce阶段的工作与reduce side join相同。

### 2.4 reduce side join + BloomFilter

在某些情况下，SemiJoin抽取出来的小表的key集合在内存中仍然存放不下，这时候可以使用BloomFiler以节省空间。

BloomFilter最常见的作用是：判断某个元素是否在一个集合里面。它最重要的两个方法是：add() 和contains()。最大的特点是不会存在 false negative，即：如果contains()返回false，则该元素一定不在集合中，但会存在一定的 false positive，即：如果contains()返回true，则该元素一定可能在集合中。

因而可将小表中的key保存到BloomFilter中，在map阶段过滤大表，可能有一些不在小表中的记录没有过滤掉（但是在小表中的记录一定不会过滤掉），这没关系，只不过增加了少量的网络IO而已。