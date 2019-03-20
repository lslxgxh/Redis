# Redis设计与实现笔记

## 数据结构

### SDS

- 简单动态字符串（Simple Dynamic String）
- 字段：`len, free, []char`, `len+free+1(空字符) = len([]char)`
- 保留空字符是为了重用部分C函数。
- 修改字符串时减少内存重分配次数。增长时使用free，缩短时惰性回收,暂时不回收内存，记入free，提供API在需要的时候真正释放。如果free不够用重新申请内存，`free = newLen < 1MB ? newLen : 1MB`(free最多1MB)
- 二进制安全：SDS的buf属性被称为字节数组，保存二进制数据


![1552969355676](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552969355676.png)

![1552969373913](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552969373913.png)

![1552969382757](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552969382757.png)

#### 总结

Redis 只会使用c字符串作为字面量，在大多数的情况下Redis使用SDS作为字符串表示

**SDS优点**

1. 常数复杂度获取字符串长度
2. 杜绝缓冲区溢出
3. 减少修改字符串时所需的内存重分配次数
4. 二进制安全
5. 兼容部分c字符串函数

### 链表

- 双向链表。{前置节点、后置节点、节点值}

- list{表头节点、表尾节点、链表所包含的节点数量、dup节点值复制函数、free节点值释放函数、match节点值对比函数}

  #### 总结

  1. 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后直节点的复杂度都为O（1）
  2. 无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点
  3. 长度有计数器
  4. 多态：保存各种不同类型的值

  ![1552982071082](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552982071082.png)

  ![1552982085848](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552982085848.png)

  ![1552982114614](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552982114614.png)

### 字典

- 结构{哈希表数组、哈希表大小、哈希表大小掩码（用于计算索引值 = size-1）、该哈希表已有的节点数量}
- 通过链表解决键冲突。
- 两个数组，其中一个作为rehash使用。
- 新节点添加到链表的表头位置

#### rehash

过程

1. 扩张后的数组大小：大于等于（2*当前节点数）的最小的2^n。

​          收缩后的数组大小：大于等于（当前节点数）的最小的2^n。

2. 将ht[0]复制到ht[1]
3. 释放ht[0],把ht[1]设置成ht[0]然后给ht[1]分配空白的哈希表

- 渐进式rehash: 用`rehashidx`表示当前正在rehash的索引，rehash时对字典的操作会同时在两个数组上进行。

  1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
  2. rehashidx设置为0，开始rehash，期间每次CRUD讲ht[0]在rehashidx索引上的所有键值对rehsah到ht[1],rehashidx++
  3. 复制完成设为-1，完成
  4. 注：渐进式过程中，在两个表中同时进行先0后1

- 负载因子：哈希表以保存的节点数/哈希表大小

- 发生的条件：

  扩展时：负载因子（节点数/数组大小）> 1, 如果在执行`BGSAVE | BGREWRITEAOF`则 > 5

  收缩时：负载因子<0.1

  ![1552984254103](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552984254103.png)

  ![1552984264853](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552984264853.png)

### 跳跃表

支持平均O(logN)、最坏O(N)复杂度节点查找，有序集合键的底层实现之一。（元素数量多、有序集合中的元素成员为较长字符串）

**应用：**有序集合键、集群节点中用作内部数据结构

结构：{指向表头节点、指向表尾节点、level层数最大节点的层数，length：跳跃表长度}

![1552984728068](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552984728068.png)

![1552984794491](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552984794491.png)

![1552984819491](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1552984819491.png)

### 整数集合intset

结构{	

​	编码方式

​	集合中包含的元素数量

​	保存元素的数组（从小到大排序，不重复，类型取决于编码方式）

}

- 不重复。
- encoding: `int16（初始）, int32, int64`
- 升级。比如当前是int16，当添加一个int32类型的整数时就需要升级，新添加的整数要么大于所有旧元素，要么小于所有旧元素？？？
- 搜索使用二分查找。
- 添加和删除的最坏时间复杂度为`O(n)`。

### 压缩列表ziplist

列表键和哈希键底层实现之一。

当储存少量、小整数值、短字符串使用。

![1553062683485](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1553062683485.png)

每个压缩列表节点可以保存一个字节数组或者一个整数值，其中，字节数组三种长度

1. 长度小于等于63字节的字节数组
2. 长度小于等于16383字节的字节数组
3. 长度小于等于4294967295字节的字节数组

整数值：

- 4位长，介于0-12之间的无符号整数
- 1字节长的有符号整数
- 3字节长的有符号整数
- int16_t类型整数
- int32_t类型整数
- int64_t类型整数

组成{

​	前一个节点长度  <254? 1:5 

​	encoding保存所保存属性类型以及长度：可以保存1，2，5字节长的字节数组或者1字节长的整型。

​	content节点的值

}

![1553063436910](C:\Users\l84122200\AppData\Roaming\Typora\typora-user-images\1553063436910.png)

- 特殊编码的连续内存块。
- 主要是为了省内存。
- 连锁更新：在表头插入了新节点（>=254字节），导致后面所有的节点的“前节点长度”都从1字节变为5字节了。删除也可能会导致连锁更新。最坏复杂度是`O(n^2)`。
- 添加的复杂度是`O(n)|O(n^2)`， 按索引查找的复杂度`O(n)`, 按值查找的复杂度`O(n^m)`（m是字节数组的长度，因为要比较值）.删除的复杂度`O(n)|O(n^2)`。（`O(n^2）`都是因为可能会连锁更新）
