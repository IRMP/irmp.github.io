redis 
端口6379
单线程+多路IO复用
# String
最大512M
## 常用命令
```shell script
set k1 v1
append k1 abc
get k1
strlen k1
setnx k1 v1
set k2 100
incr k2
decr k2
incrby/decrby k2 10
setex age 20 val20
ttl age
getset key val
mset k1 v1 k2 v2
```
# List
```shell script
lpush/rpush key v1 v2 v3
lrange k1 0 -1
lpop/rpop key
rpoplpush k1 k2
lindex k2 0
llen k2

linsert key before/after v1 v2
lrem key n val 
lset key index val
```
# Set
```shell script
sadd key v1 v2 v3
smembers key
sismember key value
scard key #返回个数
spop key
srandmember key
srem key v1 v2 v3
smove k1 k2 v1
sinter k1 k2
sunion k1 k2 
sdiff k1 k2 # k1中有，k2里没有的元素
```

# hset
```shell script
hset user:1001 name zhangsan age 30
hget user:1001 name
```