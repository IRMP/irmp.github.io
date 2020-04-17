# hbase
## hbase安装
1. 先启动zk
```shell script
zk.sh start
```
zk.sh
```shell script
#!/bin/bash
for i in hadoop01 hadoop02 hadoop03
do
	echo "**********************$i zk $1....************"
	ssh $i "/opt/module/zookeeper-3.4.10/bin/zkServer.sh $1"
done
```
2. 启动hdfs和yarn
```shell script
hadoop.sh start
```
hadoop.sh
```shell script
#!/bin/bash
case $1 in
	"start" )
/opt/module/hadoop-2.7.2/sbin/start-dfs.sh
ssh hadoop02 "/opt/module/hadoop-2.7.2/sbin/start-yarn.sh"
		;;
	"stop")
ssh hadoop02 "/opt/module/hadoop-2.7.2/sbin/stop-yarn.sh"
/opt/module/hadoop-2.7.2/sbin/stop-dfs.sh
;;
esac
```
3. 解压hbase
```shell script
tar -zxvf hbase-1.3.1-bin.tar.gz -C /opt/module
```
4. 修改hbase配置文件
修改hbase-env.sh
```shell script
export JAVA_HOME=/opt/module/jdk1.8.0_144
# Tell HBase whether it should manage it's own instance of Zookeeper or not.
export HBASE_MANAGES_ZK=false
```
修改hbase-site.xml
```xml
<configuration>
	<!-- 每个regionServer的共享目录,用来持久化Hbase,默认情况下在/tmp/hbase下面 -->  
    <property>  
        <name>hbase.rootdir</name>  
        <value>hdfs://hadoop01:9000/HBase</value>  
    </property>
    <!-- hbase集群模式,false表示hbase的单机，true表示是分布式模式 -->  
    <property>  
        <name>hbase.cluster.distributed</name>  
        <value>true</value> 
    </property>
    <!-- hbase master节点的端口 -->  
    <property>  
        <name>hbase.master.port</name>  
        <value>16000</value>  
        <description>The port the HBase Master should bind to.</description>  
    </property>  
    <!-- hbase master的web ui页面的端口 -->  
    <property>  
        <name>hbase.master.info.port</name>  
        <value>16010</value>    
    </property>
    <!-- hbase依赖的zk地址 -->  
    <property>  
        <name>hbase.zookeeper.quorum</name>  
        <value>hadoop01,hadoop02,hadoop03</value>  
    </property>
	<property>  
        <name>hbase.zookeeper.property.dataDir</name>  
        <value>/opt/module/zookeeper-3.4.10/zkData</value>  
        <description>Property from ZooKeeper's config zoo.cfg.  
            The directory where the snapshot is stored.  
        </description>  
    </property>
</configuration>
```
regionservers
```text
hadoop01
hadoop02
hadoop03
```
软连接hadoop 配置文件到 HBase：
```shell script
ln -s /opt/module/hadoop-2.7.2/etc/hadoop/core-site.xml /opt/module/hbase-1.3.1/conf/core-site.xml
ln -s /opt/module/hadoop-2.7.2/etc/hadoop/hdfs-site.xml /opt/module/hbase-1.3.1/conf/hdfs-site.xml
```
同步到其他机器
```shell script
xsync hbase-1.3.1
```
xsync.sh
```shell script
#!/bin/bash
#1 获取输入参数个数，如果没有参数，直接退出
pcount=$#
if((pcount==0)); then
echo no args;
exit;
fi

#2 获取文件名称
p1=$1
fname=`basename $p1`
echo fname=$fname

#3 获取上级目录到绝对路径
pdir=`cd -P $(dirname $p1); pwd`
echo pdir=$pdir

#4 获取当前用户名称
user=`whoami`

#5 循环
for((host=2; host<4; host++)); do
        echo ------------------- hadoop$host --------------
        rsync -rvl $pdir/$fname $user@hadoop0$host:$pdir
done
```
5. 启动hbase
```shell script
/opt/module/hbase-1.3.1/bin/start-hbase.sh
```
http://hadoop01:16010/
即可查看hbase服务状态

## hbase查看hfile命令
```shell script
bin/hbase org.apache.hadoop.hbase.io.hfile.HFile -p <HFile路径>
```
## hbase什么时候删除数据
flush和major compact

## hbase的master挂了，可以正常读写，但不能建表

## hbase的region split 自动切分会造成数据热点问题，解决办法-预分区