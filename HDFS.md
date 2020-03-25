# HDFS

## SecondaryNameNode

并非NameNode的热备，当NameNode挂掉的时候，它并不能马上替换NameNode提供服务。

辅助NameNode，分担其工作量，比如定期合并Fsimage和Edits，并推送给NameNode

紧急情况下，可辅助回复NameNode

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

SecondaryNameNode的作用就是帮助NameNode进行Edits和Fsimage的合并工作。SecondaryNameNode首先会询问NameNode是否需要CheckPoint（**触发CheckPoint需要满足两个条件中的任意一个，定时时间到和Edits中数据写满了**）。直接带回NameNode是否检查结果。SecondaryNameNode执行CheckPoint操作，首先会让NameNode滚动Edits并生成一个空的edits.inprogress，滚动Edits的目的是给Edits打个标记，以后所有新的操作都写入edits.inprogress，其他未合并的Edits和Fsimage会拷贝到SecondaryNameNode的本地，然后将拷贝的Edits和Fsimage加载到内存中进行合并，生成fsimage.chkpoint，然后将fsimage.chkpoint拷贝给NameNode，重命名为Fsimage后替换掉原来的Fsimage。NameNode在启动时就只需要加载之前未合并的Edits和Fsimage即可，因为合并过的Edits中的元数据信息已经被记录在Fsimage中。