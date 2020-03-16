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
