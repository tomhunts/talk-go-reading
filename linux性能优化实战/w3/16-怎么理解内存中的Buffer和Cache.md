## 16 | 基础篇：怎么理解内存中的Buffer和Cache？

### 查定义
从字面来说，Buffer是缓冲区，Cache是缓存，两者都是数据在内存中的临时存储

通过man free手册查到的说明：
+ Buffer：内核缓冲区用到的内存，对应的是/proc/meminfo中的Buffers值
+ Cache： 内核页缓存和Slab用到的内存，对应的是/proc/meminfo中的Cached与SReclaimable之和

在linux中一切皆文件，那么缓存的相关就可以通过查找相应的proc文件进行解析

通过man proc 查到的说明：

+ Buffers：是对原始磁盘块的临时存储，也就是用来缓存磁盘的数据，通常不会特别大（20MB左右），这样内核就可将分散的写几种起来
+ Cached：是从磁盘读取文件的页缓存，也就是用来缓存从文件读取的数据，这样下次访问这些文件数据时，就可以直接从内存中快速获取
+ SReclaimable：是Slab的一部分，其中Slab包括两部分，可回收部分用SReclaimable记录，不可回收部分用SUnreclaim记录


### 用案例演示

#### 步骤1 通过命令 vmstat 1 可看到相应内容

```sh

# 每隔1秒输出1组数据
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
0  0      0 7743608   1112  92168    0    0     0     0   52  152  0  1 100  0  0
 0  0      0 7743608   1112  92168    0    0     0     0   36   92  0  0 100  0  0
```

+ buff和cache就是我们之前看到的Buffers和Cache单位是KB
+ bi和bo则分别表示块设备读取和写入的大小，单位为块/秒，因为Linux中块的大小是1KB 所以这个单位也就等价于KB/s

####  写文件的案例

```sh
$ dd if=/dev/urandom of=/tmp/file bs=1M count=500
```
+ Cache刚开始增长时，块设备的I/O很少，bi只出现了一次488kb/sbo则只有一次4kb，过了一段时间才会出现大量的块设备写
+ 当命令结束时候，Cache不在增长，但块设备写还会持续一段时间

问题：发现写文件也有Cache的参与

#### 写磁盘的案例

由于命令会损坏相应磁盘的分区，不建议实验
```sh
# 首先清理缓存
$ echo 3 > /proc/sys/vm/drop_caches# 
然后运行dd命令向磁盘分区/dev/sdb1写入2G数据
$ dd if=/dev/urandom of=/dev/sdb1 bs=1M count=2048
```
+ 写磁盘时，Buffer和Cache都在增长，但显然Buffer的增长快的多，说明写磁盘用到了大量的Buffer


#### 读文件

```sh

# 首先清理缓存
$ echo 3 > /proc/sys/vm/drop_caches
# 运行dd命令读取文件数据
$ dd if=/tmp/file of=/dev/null
```
+ 读文件时候（bi大于0时），Buffer保持不变，而Cache则在不停则增长

#### 读磁盘

```sh

# 首先清理缓存
$ echo 3 > /proc/sys/vm/drop_caches
# 运行dd命令读取文件
$ dd if=/dev/sda1 of=/dev/null bs=1M count=1024
```
+ Buffer和Cache都在增长，但显然Buffer的增长快得多，说明数据缓存用到了Buffer

### 结论

Buffer是对磁盘数据的缓存，而Cache是文件数据的缓存，它们即会用在读中也会用在写中

### 磁盘和文件的区别

+ 磁盘：是一个块设备，是一个块文件 可以划分为不同的分区，在分区之上再创建文件系统，挂载到某个目录，之后才可以在这个目录中读写文件
+ 文件：普通文件

在读写普通文件时候，会经过文件系统，由文件系统负责与磁盘交互，而读写磁盘或者分区时，会跳过文件系统


