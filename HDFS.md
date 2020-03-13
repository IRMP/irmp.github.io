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

