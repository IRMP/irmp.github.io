# 编译hadoop(centOS 64位)

## 编译工具准备

### 安装jdk

```shell
[root@hadoop01 software] # tar -zxf jdk-8u144-linux-x64.tar.gz -C /opt/module/
[root@hadoop01 software]# vi /etc/profile
#JAVA_HOME
export JAVA_HOME=/opt/module/jdk1.8.0_144
export PATH=$PATH:$JAVA_HOME/bin
[root@hadoop01 software]#source /etc/profile
```

### 安装maven

```shell
[root@hadoop01 software]# tar -zxvf apache-maven-3.0.5-bin.tar.gz -C /opt/module/
[root@hadoop01 apache-maven-3.0.5]# vi conf/settings.xml 
```

```xml
 <mirror>
     <id>nexus-aliyun</id>
     <mirrorOf>central</mirrorOf>
     <name>Nexus aliyun</name>
     <url>http://maven.aliyun.com/nexus/content/groups/public</url>
</mirror>
```

```shell
[root@hadoop01 apache-maven-3.0.5]# vi /etc/profile
#MAVEN_HOME
export MAVEN_HOME=/opt/module/apache-maven-3.0.5
export PATH=$PATH:$MAVEN_HOME/bin
[root@hadoop01 apache-maven-3.0.5]# source /etc/profile
[root@hadoop01 apache-maven-3.0.5]# mvn -version
Apache Maven 3.0.5 (r01de14724cdef164cd33c7c8c2fe155faf9602da; 2013-02-19 21:51:28+0800)
Maven home: /opt/module/apache-maven-3.0.5
Java version: 1.8.0_144, vendor: Oracle Corporation
Java home: /opt/module/jdk1.8.0_144/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "2.6.32-754.27.1.el6.x86_64", arch: "amd64", family: "unix"
```

### 安装ANT

```shell
[root@hadoop01 software]# tar -zxvf apache-ant-1.9.9-bin.tar.gz -C /opt/module/
[root@hadoop01 apache-ant-1.9.9]# vi /etc/profile
#ANT_HOME
export ANT_HOME=/opt/module/apache-ant-1.9.9
export PATH=$PATH:$ANT_HOME/bin
[root@hadoop01 apache-ant-1.9.9]# source /etc/profile
[root@hadoop01 apache-ant-1.9.9]# ant -version
Apache Ant(TM) version 1.9.9 compiled on February 2 2017
```

### 安装glibc-headers , g++,make,cmake

```shel
[root@hadoop01 apache-ant-1.9.9]# yum install glibc-headers gcc-c++ make cmake
已加载插件：fastestmirror, refresh-packagekit, security
设置安装进程
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirror.bit.edu.cn
 * updates: mirrors.163.com
包 glibc-headers-2.12-1.212.el6_10.3.x86_64 已安装并且是最新版本
包 gcc-c++-4.4.7-23.el6.x86_64 已安装并且是最新版本
包 1:make-3.81-23.el6.x86_64 已安装并且是最新版本
包 cmake-2.8.12.2-4.el6.x86_64 已安装并且是最新版本
无须任何处理
```

### 安装protobuf

```shell
[root@hadoop01 software]# tar -zxvf protobuf-2.5.0.tar.gz -C /opt/module/
[root@hadoop01 opt]# cd /opt/module/protobuf-2.5.0/

[root@hadoop01 protobuf-2.5.0]#./configure 
[root@hadoop01 protobuf-2.5.0]# make
[root@hadoop01 protobuf-2.5.0]# make check
[root@hadoop01 protobuf-2.5.0]# make install
[root@hadoop01 protobuf-2.5.0]# ldconfig
[root@hadoop01 hadoop-dist]# vi /etc/profile
#LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/opt/module/protobuf-2.5.0
export PATH=$PATH:$LD_LIBRARY_PATH
[root@hadoop01 software]#source /etc/profile
[root@hadoop01 protobuf-2.5.0]# protoc --version
libprotoc 2.5.0
```

### 安装openssl库,ncurses-devel库

```shell
[root@hadoop01 module]# yum install openssl-devel ncurses-devel
```

## 编译源码

```shell
[root@hadoop01 software]# tar -zxvf hadoop-2.7.2-src.tar.gz -C /opt/
[root@hadoop01 software]# cd /opt/hadoop-2.7.2-src/
[root@hadoop01 hadoop-2.7.2-src]# mvn package -Pdist,native -DskipTests -Dtar

```

