# Redis源码结构

## 客户端与服务器
```
server.c                 服务端                    
redis-cli.c              客户端                     
ae.c/ae_epoll.c/ae_evport.c/ae_kqueue.c/ae_select.c   事件处理模块                 
anet.c和networking.c     网络连接
scripting.c              lua脚本                       
slowlog.c                慢查询
```

## 数据结构
```
zmalloc.c                内存分配 
sds.c                    动态字符串
adlist.c                 双端链表
quicklist.c              快速列表
dict.c                   字典
server.h(zskiplist/zskiplistNode) 跳跃表
hyperloglog.c            基数统计
```
## 内存编码结构
```
intset.c                 整数集合数据结构
ziplist.c                压缩列表数据结构
```

## 数据类型
```
object.c                 对象系统 
t_string.c               字符串键 
t_list.c                 列表建
t_hash.c                 散列键
t_set.c                  集合键
t_zset.c                 有序集合键
hyperloglog.c            HyperLogLog键 
```

## 数据库
```
db.c                    数据库实现
notify.c                通知功能 
rdb.c                   RDB持久化 
aof.c                   AOF持久化 
```

## 独立功能模块
```
pubsub.c                发布和订阅
multi.c                 事务
```

## 多机部分
```
replication.c           复制功能
sentinel.c              Redis Sentinel 
cluster.c               集群 
```

## 测试
```
memtest.c                内存检测
redis_benchmark.c        用于redis性能测试的实现
redis_check_aof.c        用于AOF检查的实现
redis_check_rdb.c        用于RDB检查的实现
```

## 工具
```
bitops.c                 二进制位操作命令的实现
debug.c                  用于调试时使用
endianconv.c             高低位转换，不同系统，高低位顺序不同
help.h                   辅助于命令的提示信息
lzf_c.c                  压缩算法系列
lzf_d.c                  压缩算法系列
rand.c                   用于产生随机数
release.c                用于发布时使用
sha1.c                   sha算法的实现
util.c                   通用工具方法
crc64.c                  循环冗余校验
sort.c                   SORT命令的实现一些封装类的代码实现：
bio.c                    background I/O的意思，开启后台线程用的
latency.c                延迟类
pqsort.c                 排序算法类
rio.c                    redis定义的一个I/O类
syncio.c                 用于同步Socket和文件I/O操作
```
