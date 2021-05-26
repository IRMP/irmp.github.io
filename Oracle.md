# Oracle 笔记

## Oracle数据库架构

![截屏2021-01-11 下午2.52.29](/Users/shengli/IdeaProjects/irmp.github.io/images/截屏2021-01-11 下午2.52.29.png)

### 实例内存结构

系统全局区和程序全局区

- SGA:SGA 是一组共享的内存结构，包含一个数据库实例的数据和控制信息。 例如，SGA 组件包括数据块缓存和共享 SQL 区域

- PGA：PGA 是一个内存区域，其中包含某个服务器进程或后台进程的数据和控制信息。 PGA 只由专有的过程访问。 每个服务器进程和后台进程都有其自己的 PGA。



1. 物理存储结构

CREATE DATABASE 后会创建以下文件：

- Data files 表，索引

- Control files 元数据，数据库名称，文件位置
- Online redo log files

2. 逻辑存储结构

- Data blocks 数据块
- Extents 拓展区
- Segments 段 为用户对象（表或索引）、回滚数据、临时数据等分配的一组拓展区
- Tablespaces 表空间 段的逻辑容器，每个表空间至少包含一个数据文件。

## 表和表簇

1. 模式对象

   <img src="/Users/shengli/IdeaProjects/irmp.github.io/images/截屏2021-01-11 下午3.24.26.png" alt="截屏2021-01-11 下午3.24.26" style="zoom: 67%;" />

<img src="/Users/shengli/IdeaProjects/irmp.github.io/images/截屏2021-01-11 下午3.31.30.png" alt="截屏2021-01-11 下午3.31.30" style="zoom:67%;" />

这两个数据文件同属一个表空间，一个段不能跨多个表空间，但是可以属于多个数据文件。

查询模式是否有效：

```sql
SELECT OBJECT_NAME, STATUS FROM USER_OBJECTS WHERE OBJECT_NAME='TEST_PROC';
```

VARCHAR2和CHAR

前者可变长度存储，后者定长（空格填充），varchar2更节省空间

NUMBER(p,s)

p是精度，即数字总长度

s是小数位数yangling

