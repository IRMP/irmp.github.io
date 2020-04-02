# HDFS

## HDFS写数据流程

![截屏2020-03-27下午2.42.58](https://irmp.github.io/images/截屏2020-03-27下午2.42.58.png)

1. 客户端通过Distributed FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在。
2. NameNode返回是否可以上传。
3. 客户端请求第一个 Block上传到哪几个DataNode服务器上。
4. NameNode返回3个DataNode节点，分别为dn1、dn2、dn3。
5. 客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。
6. dn1、dn2、dn3逐级应答客户端。
7. 客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet为单位，dn1收到一个Packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。
8. 当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。（重复执行3-7步）



## HDFS读数据流程

![截屏2020-03-27下午2.49.43](https://irmp.github.io/images/截屏2020-03-27下午2.49.43.png)

1. 客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址。
2. 挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。
3. DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet为单位来做校验）。
4. 客户端以Packet为单位接收，先在本地缓存，然后写入目标文件。

## HDFS块的大小

HDFS块的大小设置主要取决于磁盘传输速率

寻址时间为传输时间的1%最佳

## HDFS副本数设置

```shell
hdfs dfs -setrep 副本数 文件或目录
```

如果设置的副本数大于datanode的个数，新增datanode后会自动同步数据过去。

## 客户端配置
windows里配置HADOOP_HOME环境变量，添加%HADOOP_HOME%\bin到PATH中
如果java安装在Program Files目录里需要改变JAVA_HOME，保证没有空格，改成C:\Progra~1\Java\jdk1.8.0_231即可
IDEA如果一直运行报找不到winutils的错误，清缓存重启几次即可
log4j配置
```properties
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n
log4j.appender.logfile=org.apache.log4j.FileAppender
log4j.appender.logfile.File=target/spring.log
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] - %m%n
```
## HDFS副本数如果设置成3，三个副本存放位置是怎样的？
第一个存放在客户端节点，如果客户端不在集群里，随机选择一个节点
第二个副本存放在第一个节点相同机架里的不同节点
第三个副本存放在与前两个节点不同的机架里

## 网络拓扑距离，datenode的选择
两个节点到共同祖先的距离之和
namenode在选择datenode的时候会选择拓扑距离最近的节点

## NN和2NN工作机制
namenode的元数据存在哪里？
存在内存中，同时会在磁盘里存储镜像文件fsimage
namenode会同时滚动记录操作日志，2nn会拿到操作日志对
fsimage进行操作，创造fsimage.checkpoint给nn

<img src="https://irmp.github.io/images/wpsZAdegC.png" alt="namenode工作机制" style="zoom:80%;" />

**NN和2NN工作机制详解：**

Fsimage：NameNode内存中元数据序列化后形成的文件。

Edits：记录客户端更新元数据信息的每一步操作（可通过Edits运算出元数据）。

NameNode启动时，先滚动Edits并生成一个空的edits.inprogress，然后加载Edits和Fsimage到内存中，此时NameNode内存就持有最新的元数据信息。Client开始对NameNode发送元数据的增删改的请求，这些请求的操作首先会被记录到edits.inprogress中（查询元数据的操作不会被记录在Edits中，因为查询操作不会更改元数据信息），如果此时NameNode挂掉，重启后会从Edits中读取元数据的信息。然后，NameNode会在内存中执行元数据的增删改的操作。由于Edits中记录的操作会越来越多，Edits文件会越来越大，导致NameNode在启动加载Edits时会很慢，所以需要对Edits和Fsimage进行合并（所谓合并，就是将Edits和Fsimage加载到内存中，照着Edits中的操作一步步执行，最终形成新的Fsimage）。

SecondaryNameNode的作用就是帮助NameNode进行Edits和Fsimage的合并工作。SecondaryNameNode首先会询问NameNode是否需要CheckPoint（**触发CheckPoint需要满足两个条件中的任意一个，定时时间到和Edits中数据写满了**）。直接带回NameNode是否检查结果。SecondaryNameNode执行CheckPoint操作，首先会让NameNode滚动Edits并生成一个空的edits.inprogress，滚动Edits的目的是给Edits打个标记，以后所有新的操作都写入edits.inprogress，其他未合并的Edits和Fsimage会拷贝到SecondaryNameNode的本地，然后将拷贝的Edits和Fsimage加载到内存中进行合并，生成fsimage.chkpoint，然后将fsimage.chkpoint拷贝给NameNode，重命名为Fsimage后替换掉原来的Fsimage。
NameNode在启动时就只需要加载之前未合并的Edits和Fsimage即可，因为合并过的Edits中的元数据信息已经被记录在Fsimage中。

**FSImage和Edit**

![截屏2020-03-25下午10.00.05](https://irmp.github.io/images/截屏2020-03-25下午10.00.05.png)

**oiv查看fsimage文件**

```shell
#oiv            apply the offline fsimage viewer to an fsimage
#hdfs oiv -p 文件类型 -i镜像文件 -o 转换后文件输出路径
[mr@hadoop01 current]$ pwd
/opt/module/hadoop-2.7.2/data/tmp/dfs/name/current

[mr@hadoop01 current]$ hdfs oiv -p XML -i fsimage_0000000000000000025 -o /opt/module/hadoop-2.7.2/fsimage.xml

[mr@hadoop01 current]$ cat /opt/module/hadoop-2.7.2/fsimage.xml
```

- Fsimage中没有记录块所对应DataNode，为什么？

```text
在集群启动后，要求DataNode上报数据块信息，并间隔一段时间后再次上报，保证集群数据可靠性。
```



**oev查看edit文件**

```shell
#oev            apply the offline edits viewer to an edits file
#hdfs oev -p 文件类型 -i编辑日志 -o 转换后文件输出路径
[mr@hadoop01 current]$ hdfs oev -p XML -i edits_0000000000000000012-0000000000000000013 -o /opt/module/hadoop-2.7.2/edits.xml

[mr@hadoop01 current]$ cat /opt/module/hadoop-2.7.2/edits.xml
```

- NameNode如何确定下次开机启动的时候合并哪些Edits?

```text
根据seen_txid保存的信息确定。
```

## checkpoint设置

- 通常情况下，SecondaryNameNode每隔一小时执行一次。

​	[hdfs-default.xml]

```xml
<property>

 <name>dfs.namenode.checkpoint.period</name>

 <value>3600</value>

</property>
```

- 一分钟检查一次操作次数，当操作次数达到1百万时，SecondaryNameNode执行一次。

```xml
<property>

 <name>dfs.namenode.checkpoint.txns</name>

 <value>1000000</value>

<description>操作动作次数</description>

</property>
 
<property>

 <name>dfs.namenode.checkpoint.check.period</name>

 <value>60</value>

<description> 1分钟检查一次操作次数</description>

</property >
```

## NameNode故障处理

NameNode故障后，可以采用如下两种方法恢复数据。

### 方法一：将SecondaryNameNode中数据拷贝到NameNode存储数据的目录；

1. kill -9 NameNode进程

2. 删除NameNode存储的数据（/opt/module/hadoop-2.7.2/data/tmp/dfs/name）

  ```shell
  [mr@hadoop01 hadoop-2.7.2]$ rm -rf /opt/module/hadoop-2.7.2/data/tmp/dfs/name/*
  ```

3. 拷贝SecondaryNameNode中数据到原NameNode存储数据目录

   ```shell
   [mr@hadoop01 dfs]$ scp -r atguigu@hadoop104:/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary/* ./name/
   ```

4. 重新启动NameNode

  ```shell
  [mr@hadoop01 hadoop-2.7.2]$ sbin/hadoop-daemon.sh start namenode
  ```



### 方法二：使用-importCheckpoint选项启动NameNode守护进程，从而将SecondaryNameNode中数据拷贝到NameNode目录中。

1. 修改hdfs-site.xml

```xml
<property>
  <name>dfs.namenode.checkpoint.period</name>
  <value>120</value>
</property>
<property>
  <name>dfs.namenode.name.dir</name>
  <value>/opt/module/hadoop-2.7.2/data/tmp/dfs/name</value>
</property>
```

2. kill -9 NameNode进程

3. 删除NameNode存储的数据（/opt/module/hadoop-2.7.2/data/tmp/dfs/name）

   ```shell
   [mr@hadoop01 hadoop-2.7.2]$ rm -rf /opt/module/hadoop-2.7.2/data/tmp/dfs/name/*
   ```

4. 如果SecondaryNameNode不和NameNode在一个主机节点上，需要将SecondaryNameNode存储数据的目录拷贝到NameNode存储数据的平级目录，并删除in_use.lock文件

   ```shell
   [mr@hadoop01 dfs]$ scp -r atguigu@hadoop104:/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary ./
   [mr@hadoop01 namesecondary]$ rm -rf in_use.lock
   [mr@hadoop01 dfs]$ pwd
   /opt/module/hadoop-2.7.2/data/tmp/dfs
   [mr@hadoop01 dfs]$ ls
   data  name  namesecondary
   ```

5. 导入检查点数据（等待一会ctrl+c结束掉）

   ```shell
   [mr@hadoop01 hadoop-2.7.2]$ bin/hdfs namenode -importCheckpoint
   ```

6. 启动NameNode

   ```shell
   [mr@hadoop01 hadoop-2.7.2]$ sbin/hadoop-daemon.sh start namenode
   ```

## 集群安全模式

![截屏2020-03-27下午2.40.08](https://irmp.github.io/images/截屏2020-03-27下午2.40.08.png)

集群处于安全模式，不能执行重要操作（写操作）。集群启动完成后，自动退出安全模式。

- bin/hdfs dfsadmin -safemode get		（功能描述：查看安全模式状态）
- bin/hdfs dfsadmin -safemode enter  （功能描述：进入安全模式状态）
- bin/hdfs dfsadmin -safemode leave	（功能描述：离开安全模式状态）
- bin/hdfs dfsadmin -safemode wait	（功能描述：等待安全模式状态）

