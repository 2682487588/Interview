# *二*、简单动态字符串

~~~mysql
Set msg "hello world"
~~~

> 键值对的键是字符串对象，底层保存"msg"的SDS
>
> 键值对的值是字符串对象，底层实现"hello world"的SDS

~~~go
REPUSH fruits "apple" "banana" "cherry"
~~~

> 键值对的键是一个字符串对象，对象底层保存字符串"fruits"的SDS
>
> 键值对的值是列表对象，由三个SDS实现 "apple","banana","cherry"

## 1.SDS定义

~~~c
struct sdshdr{
    //记录Buf数组已使用字节的数量
    //等于SDS保存字符串的长度
    int len;
    //记录buf未使用字节的数量
    int free;
    //字节数组用于保存字符串
    char buf[];
}
~~~

- `free`的属性为0，表示SDS没有分配空间
- `len`的属性为5，表示SDS保存了五字节长的字符串
- `buf`是一个char类型的数组， 数组前五个字节分别保存了'R','e','d','i','s',最后一个字节保存'\0'

SDS遵循C字符串以空字符结尾，

![image-20220706145434193](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220706145434193.png)

## 2 SDS和C字符串的区别

根据传统，C语言使用长度为N+1的字符串数组保存长度为N的字符串，并且字符数组的最后一个元素总是空字符'\0'

![image-20220706150001597](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220706150001597.png)

> C语言这样表达字符串并不能满足，安全性，效率，功能方面的要求

> SDS的字符串的优点如下

### 1. 常数复杂度获取字符串长度

> C语言不记录本身长度信息，所以C语言的需要遍历整个字符串，复杂度O(n)

> 但是SDS直接在len记录了长度，获取SDS长度的复杂度仅为0（1）

![image-20220706150553818](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220706150553818.png)

### 2. 杜绝缓冲区溢出

> C（语言字符串的拼接）容易造成缓冲区溢出，因为C字符串不记录本身的长度

> SDS的空间分配策略杜绝了发生缓冲区的可能，SDS的API需要对SDS进行修改，会先检查SDS的空间是否满足修改的需求，不满足API会将SDS空间拓展到满足SDS修改的所需的大小

### 3.减少字符串修改代码内存重分配次数

> C字符串每次增长或者缩短一个C字符串，程序都要对这个C字符串进行一次内存重分配
>
> - 程序如果是增长字符串，那么执行操作之前，程序需要先通过内存重分配来拓展底层大小——否则缓冲区溢出
> - 缩短字符串(trim)，程序需要释放字符串你不在使用的那个部分——否则就会内存泄漏

对于未使用的空间SDS做了 **空间预分配**和 **惰性空间**两种优化策略

#### 1.空间预分配

> SDS的长度小于1MB,程序本身修改以后len变为13，那么也会分配13字节的free空间

![image-20220706155030047](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220706155030047.png)

#### 2.惰性空间释放

>   SDS的API需要缩短SDS的保存字符串,程序并不理解使用内存重分配来回收缩短后的字节

> 举个例子,sdstrim函数接收一个SDS和一个C字符串为参数，移除SDS所有在C字符串出现过的数字

![image-20220706155455785](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220706155455785.png)

![image-20220706155501273](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220706155501273.png)

### 4.二进制安全

C字符串里面不能有空字符

![image-20220707153457503](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220707153457503.png)

但是对Redis里面字符串SDS的API都会处理二进制存放在Buf数组中的数据，程序不会对其中做限制，过滤或者假设

![image-20220707153551169](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220707153551169.png)

### 5.兼容C字符串<string.h>部分函数

通过遵循C字符串以空字符串结尾的管理，SDS可以在有需要重用<string.h>

### 6.总结

![image-20220707154330979](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220707154330979.png)

## 3.SDS API

![image-20220707154537862](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220707154537862.png)

![image-20220707154547535](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220707154547535.png)

## 4.回顾

> Redis只会使用C字符串作为字面量，大多情况下Redis使用SDS(Simple Dynamic String 简单动态字符串)

> SDS对比C字符串有如下优点

- 常数复杂度获取字符串长度
- 杜绝缓冲区溢出
- 减少字符串长度需要重分配次数
- 二进制安全
- 兼容C字符串函数

# 三、链表

## 3.1链表和链表节点的实现

~~~c
typedef struct listNode{
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    //节点的值
    void *value;
}listNode;
~~~

多个ListNode和prev和next指针组成双端链表

![image-20220712211050372](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220712211050372.png)

**使用adlist.h/list**持有链表，操作起来会更为方面

~~~c
typedef struct list{
    //链表头结点
    listNode *head;
    //表尾节点
    listNode *tail;
    //链表包含节点数量
    unsigned long len;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void (*free)(void *ptr)
    //节点值对比函数
    int (*match)(void *ptr,void *key);
}list;
~~~

> list结构给链表提供了表头指针head,表尾指针tail,长度计数器Len
>
> dup,free,match用多态链表实现特定函数

- `dup`函数用于赋值链表节点保存的值
- `free`释放链表节点保存的值
- `match`匹配链表节点保存的值和另外一个链表是否相同

![image-20220712214422588](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220712214422588.png)

- 双端： 链表节点带有prev和next指针，获取某个节点的前置节点和后置节点复杂度为O(1)
- **无环**: 表头的prev和表尾的next指向NULL
- **带表头指针和表尾指针**：通过list结构的head指针和tail指针，程序获取表头指针和表尾指针的复杂度都为O(1)
- **带链表长度的计数器**:len属性获取链表节点长度，获取链表长度时间复杂度为O(1)
- **多态**：void*指针保存节点的值，可以通过list的dup,free,match设置类型函数，链表保存不同类型的值

## 3.2 链表和链表节点API

![image-20220712221610411](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220712221610411.png)

![image-20220712221624142](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220712221624142.png)

![image-20220712221636345](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220712221636345.png)

## 3.3 重点回顾

> 1.链表被广泛用于Redis的各种功能
>
> - 列表键
> - 发布和订阅
> - 慢查询
> - 监视器
>
> 2.每个链表节点是listNode组成的，每个链表节点都有一个指向前置节点和后置节点的指针，所以Redis链表是双端链表
>
> 3.每个链表的结构用list表示，这个结构有表头指针，表尾节点指针和链表长度信息
>
> 4.因为链表表头的前置节点和链表表尾的后置节点指向为null，所以Redis链表是无环链表
>
> 5.链表设置不同类型的指定函数，所以Redis链表可以保存不同类型的值

# 四、字典

字典，又称为符号表(symbol table),关联数组，或者映射，是一种保存键值对(key-value pair)的抽象数据结构

字典中每个键是独一无二的，通道根据键更新值或者根据键删除键值对

## 1字典的实现

### 1哈希表

Redis的字典使用哈希表的底层实现，一个哈希表可以都有多个哈希表节点，每个哈希表节点就是一个键值对

~~~c
typedef struct dictht{
    //哈希表数组
	dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，计算索引值
    //总是size-1
    unsigned long sizemask;
    //哈希表已经有的节点的数量
    unsigned long used;
}dictht;
~~~

> table是一个数组，数组元素之下你个dict.h/dictEntry结构的指针,
>
> 每个dictEntry结构保存一个键值对。`size`记录了哈希表的大小，
>
> 即为`table`数组的大小，`used`记录哈希表当前节点(键值对)的数量

![image-20220713111551426](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220713111551426.png)

### 2哈希表的节点

~~~go
typedef struct dictEntry{
    //键
    void *key;
    //值
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    }
    //指向下个哈希表节点，形成链表
    struct dictEntry *next;
}dictEntry;
~~~

键值对的`value`可以是指针，可以使uint64,可以是int64

`next`是指向另外一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起![image-20220714140841226](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220714140841226.png)

### 3 字典

~~~c
typedef struct dict{
    //类型特定函数
    dictType *type;
    //私有数据
    void *pridata;
    //哈希表
    dictht ht[2] 
    //rehash索引
    //rehash不进行的时候，值为-1
    int trehashidx;
}dict;
~~~

`type`属性和`privdata`属性针对不同类型的键值对，为创建动态字典设定的

- `type`指向`dictType`的指针，每个`dictType`保存一簇用于操作特定类型键值对的函数
- `privdata`属性保存需要传给特定函数的可选参数

![image-20220714144931951](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220714144931951.png)

> ht尚需经是包含两个项的数组，每个项都是一个`dictht`哈希表，字典只使用ht[0]哈希表,ht[1]会对ht[0]在rehash使用， `rehashidx`记录rehash目前的进度，没有rehash这个idx会变为-1

![image-20220714145200448](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220714145200448.png)

## 2 哈希算法

> 需要将一个键值对添加到字典里面的时候，程序需要现根据键值对计算哈希值和索引值，然后根据索引值，将哈希表节点放到对应索引上面

> Redis计算哈希值和索引值的方法
>
>  字典设置哈希函数，计算key的索引值
>
> hash = key -> type -> hashFunction(key)
>
> **哈希表的sizemask属性和哈希值，计算哈希索引**
>
> ht[x]根据情况可以使ht[0]和ht[1]
>
> index = hash & dict->ht[x].sizemask

![image-20220714150307647](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220714150307647.png)

## 3解决键冲突

> Redis的哈希表采用链地址法解决键冲突（程序总是将新节点添加到链表的表头位置，排在其他已经有的节点前面）

![image-20220714152924475](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220714152924475.png)

## 4 rehash

### 1.rehash的过程

随着操作执行，哈希表键值对会逐渐增加或者减少，为了让哈希表进行扩容和缩容

扩展和收缩哈希表的工作可以通过执行`rehash`(重新散列)

1.给ht[1]分配空间，取决于要进行的是扩容还是缩容

- 扩容操作,ht[1]大小第一个大于等于ht[0].used*2的2的n次方
- 缩容操作,ht[1]大小为第一个大于等于ht[0].used的2的n次方

2.保存在ht[0]所有的键值对通过rehash放到ht[1]上

3.当将ht[0]上面包含的所有键值对搬迁到ht[1] (ht[0])变为空表,我们将ht[0]释放掉，然后将ht[1]设置为ht[0]，在ht[1]创建空白的rehash表，为下一次rehash做准备

> 如果要对ht[0]进行拓展，那么我们会进行以下

![image-20220724193917732](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220724193917732.png)

> 1.ht[0].used的空间为4，4*2 =8，8(2的3次方)恰好等于第一个大于等于4的2的n次方，程序会把ht[1]设置为8

![image-20220724194532046](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220724194532046.png)

> 2.将ht[0]的键值对rehash到ht[1]中(由于这里的sizemask变了，索引值不变),所以index = hash & dict[0].sizemake

![image-20220724195803592](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220724195803592.png)

> 3.释放ht[0],将ht[1]设置为ht[0]，然后给ht[1]设置一个空白哈希表

![image-20220724200223057](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220724200223057.png)

### 2.哈希表的扩展与收缩

#### 拓展操作

> 1.服务器没有执行`BGSAVE`和`BGREWRITEROF`命令，并且负载因子>=1
>
> 2.服务器执行`BGSAVE`和`BGREWRITEROF`命令，并且负载因子>=5

**负载因子计算公式**

> loadFactor = ht[0].used/ht[0].size    --> 已经使用的/总容量

**为什么我们需要根据`BGSAVE`和`BGREWRITEROF`来调整loadFactor**

> 为什么我们需要根据`BGSAVE`和`BGREWRITEROF`命令执行与否来调整扩容策略，因为我们在执行这些命令需要创建子进程，Redis中通过写时复制来优化子进程的使用效率，**我们通过拓展负载因子来避免进行子进程的`BGSAVE`和`BGREWRITEROF`的不必要的内存写入**,节省内存
>
> 当loadFactor<0.1,程序会对hash表进行收缩操作

## 5.渐进式rehash

> 我们进行rehash不是通过`一次性`,`集中式`完成的，而是通过`多次` `渐进式` 完成的
>
> **原因：**
>
> 如果ht[0]存的是4个`key,value`，我们可以一瞬间搬迁到ht[1]，但是如果是四百万，四千万，四亿，那我们一瞬间搬迁，庞大的计算量会让服务器停止运算，而且还可能瘫痪

### 渐进式搬迁步骤

哈希表渐进式的`rehash`的详细步骤：

![image-20220728220207099](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220728220207099.png)

### 渐进式Hash执行期间的哈希表操作

渐进式hash会使用ht[0]和ht[1]两个表，所以渐进式rehash，字典会先找ht[0],然后去找ht[1]

## 7.重点回顾

- 字典被用于哈希表和哈希键
- Redis字典底层由两个哈希表构成，一个平时使用，一个rehash使用
- Redis用链地址法解决冲突，被分配到一个索引值上的会形成一个单链表
- 对哈希表进行收缩或者扩容，这就要求哈希表包含的键值对rehash到新的哈希表中，这个rehash不是一次完成的，是渐进式完成



# 五、跳跃表

跳跃表(skiplist)是一种有序数据结构，通过每个节点维持多个其他节点指针，来达到访问相对应的节点的目的

**时间复杂度:**最好OlogN，最坏O(n)

**使用场景**:

- 实现zset
- 集群做内部数据结构

## 1.实现

**两部分**:

- `redis.h/zskipListNode`:表示跳跃表节点
- `redis.h/zskipList`：跳跃表节点相关信息

![image-20220728223115460](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220728223115460.png)

### zskiplist

- **header**: 跳跃表表头节点
- **tail**：跳跃表表尾节点
- **level**：记录目前跳跃表，层数最大节点层数(不包括表头)
- **length**：长度，包含节点的数量(不包括表头)

#### 结构：

![image-20220728225244192](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220728225244192.png)

> head和tail直接指向表头和表尾，通过这两个定位表头表尾时间复杂度为0(1)
>
> length记录节点数量，获取长度复杂度也为O(1)
>
> level表示层高最大的节点层数(不包括表头层高)

### zskiplistNode

![image-20220728223844666](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220728223844666.png)

#### 1.层

> level数组可以包括多个元素，通过层加快访问其他节点速度，一般来说，层越多，访问节点速度越快

#### 2.前进指针

> 每个层都有个指向表尾的前进指针，下图表示遍历表尾的路径

![image-20220728224454580](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220728224454580.png)

#### 3.跨度

> 层的跨度（level[i].span）记录节点之间的距离

- 节点跨度越大，距离就越远
- 指向null的前进指针的跨度为0

#### 4.后退指针

>  从表尾向表头方向访问节点，每个节点只有一个后退指针，只能后退到前一个节点

#### 5.分值和成员

> 分值是double类型的浮点数，从小到大排列
>
> 成员是一个指针，指向一个字符串对象，字符串对象保存一个SDS的值
>
> ps:同一个跳跃表中， **字符串对象**必须唯一， **分值**可以相同，分值相同的节点按照字典序大小排序，小的靠近表头，大的靠近表尾

## 2.API

![image-20220728225613398](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220728225613398.png)



## 3.回顾

- zskiplist保存跳跃表信息，zskiplistNode保存跳跃表节点
- 跳跃表节点层高是1~32的随机数
- 同一个跳跃表中，多个节点分值可以相同，但是节点成员对象必须唯一
- 跳跃表节点按照分值排序，分值相同，按照字符串对象字典序大小进行排序

# 六、整数集合

整数集合(intset)是集合键的底层实现之一

## 1.实现

~~~c
typedef struct intset{
    //编码方式
    uint32_t encoding;
    //集合元素数量
    uint32_t length;
    //保存元素数组
    int8_t contents[];
}intset;
~~~

![image-20220903192138645](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903192138645.png)

## 6.2升级

> 当我们需要添加一个新元素到整数集合里面，并且新元素的类型比整数集合现有元素的长度都要唱，整数集合需要先升级

升级过程

1) 根据新元素类型计算空间
2) 将底层数组转换为和新元素相同类型，将类型转换后元素放到正确位置
3) 将新元素添加到底层数组

**升级好处**

> 提升灵活性

> 节约内存

## 4 降级

> 整数集合不支持降级操作

## 5.整数集合API

![image-20220903193959680](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903193959680.png)

## 6.重点回顾

- 整数集合是集合键的底层实现之一
- 整数集合底层为数组，数组以有序，无重复的形式保存元素，需要的时候程序会添加元素的类型，改变数组类型
- 升级为整数集合提升灵活性，节约内存的操作
- 整数集合只支持升级不支持降级

# 七、压缩链表

**列表键和哈希表底层实现之一**

### 场景：

列表键包含少量列表性/列表项是小整型值，比如少量整形或者短字符串

## 1.组成

特殊编码组成的顺序型结构，每个节点可以**保存一个字节数组或一个整数值**，一个压缩链表可以包含多个节点

![image-20220903200216325](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903200216325.png)

![image-20220903200623165](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903200623165.png)

## 2.压缩链表(ziplist)节点

### 1.previous_entry_length

> 记录压缩链表前一个字节长度，可以是1字节，也可以是5字节

### 2.encoding

> content属性已经保存的数据的类型和长度

### 3.content

> 节点content负责保存节点的值，节点值可以是字节数组或整数

## 5.回顾

1. 压缩链表是为了节约内存开发的顺序性数据结构
2. 压缩列表被用于列表键和哈希键的底层
3. 压缩列表包含多个节点，每个节点保存一个字节数组或整数值
4. 添加新节点到压缩列表或删除节点，可能会引起连锁更新

# 八.对象

## 1.对象类型和编码

> Redis使用对象表示数据库的k,v
>
> Redis对象是一个`redisObject`表示，分别是type属性,encoding属性,ptr属性

~~~c
typedef struct redisObject{
    //类型
    unsigned type:4;
    //编码
    unsigned encoding:4;
    //指向底层数组的指针
    void *ptr;
}robj;
~~~

### 1.类型

![image-20220903213217285](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903213217285.png)

### 2.编码实现

![image-20220903213323639](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903213323639.png)



> 不同编码数据和对象

![image-20220903213542345](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903213542345.png)

## 2.字符串对象

> 编码 int ,raw ,embstr

区别：

> int :整数
>
> raw:字符串,len>32字节
>
> embstr:字符串,len<=32字节

**embstr保存字符串的优点**

- `embstr`将创建字符串对象所需内存分配次数从raw的2次降低为1次
- 释放`embstr`编码字符串对象为1次,`raw`编码的字符串对象为2次
- `embstr`对象保存在一块内存里

![image-20220903220049005](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220903220049005.png)

> **embstr是只读的**,所以我们在修改embstr的时候会先变为raw编码的对象

![image-20220905170342712](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220905170342712.png)

## 3.列表对象

### 编码

- ziplist
- linkedlist

> **Ziplist**编码的numbers列表对象

![image-20220905171021895](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220905171021895.png)

> `linkedList`编码的numbers列表对象

![image-20220905171310694](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220905171310694.png)

> StringObject代表的结构如下

![image-20220905171830522](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220905171830522.png)

### 1.编码转换

> ziplist使用场景（不能满足随意一个都要转成linedlist）

- 列表对象保存所有字符串元素长度<64B
- 列表对象保存的元素数量<512个

> 边界可以修改 ，详见`list-max-ziplist-value`和`list-max-ziplist-entries`

### 2.命令实现

![image-20220905173801715](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220905173801715.png)

## 4. 哈希对象

### 编码

- ziplist
- hashtable

`ziplist` 哈希对象使用压缩链表作为底层实现，程序会把保存键的压缩链表节点推到队尾，再把值推到队尾

#### ziplist形式

![image-20220906165126333](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220906165126333.png)

#### hashtable形式

![image-20220906165143781](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220906165143781.png)

### 编码转换

哈希对象使用`ziplist`编码的情况如下(不能满足下列两个条件将转为hashtable)

- 哈希对象保存的键值应该<64B
- 哈希对象保存的键值的数量都应该小于512个

![image-20220906173808206](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220906173808206.png)

## 5.集合对象

### 编码

- `intset`
- `hashtable`

#### `intset`编码

![image-20220906174849641](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220906174849641.png)

#### `hashtable`编码

![image-20220906174917053](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220906174917053.png)

### 编码转换

`intset`编码（不同时满足以下条件）

- 集合保存的对象都是整数值
- 集合保存对象的个数小于512个（上限可以通过`set-max-intset-entries`修改)

### 2.命令实现

![image-20220912170825941](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220912170825941.png)

## 6.有序集合对象

### 编码

- `ziplist`
- `skiplist`

`ziplist`编码的压缩队列对象使用压缩链表作为底层实现，每个集合元素用两个紧挨着的压缩链表节点表示，第一个是key，第二个是value

当使用`skiplist`作为`zset`的底层实现，一个zset结构包含一个`zskiplist`和一个`dict`

~~~c
typedef struct zset{
    zskiplist *zsl;
    dict *dict;	//元素分值是一个double类型的浮点数
}zset;
~~~

![image-20220913092329536](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220913092329536.png)

### 为什么有序集合需要使用跳跃表和字典实现

> 理论上，使用跳跃表或者字典其中一种数据结构实现就可以，但是 **单独使用比同时使用的效率更低**
>
> 如果单独使用`dict`，我们进行成员访问的时间复杂度是O（1），但是进行范围查找的时候时间复杂度是O(NlogN)，并且还有O（N）的内存开销
>
> 单独使用`zskiplist`，我们进行范围查找是O(NlogN)，但是进行成员访问的实践就可能是O(logN)

![image-20220913105224911](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220913105224911.png)

### 编码转换

对于下列情况使用`ziplist`，其中有一个不满足就会转为`skiplist`

- 1.存储的元素数量<128个
- 2.每个元素的长度都<64B

### `Zset`的实现

![image-20220913105847182](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220913105847182.png)

## 7. 类型检查和命令多态

进行键的执行 `DEL`命令,`EXPIRE`命令,`RENAME`命令,`TYPE`命令,`OBJECT`命令

> 但是有的命令只能有特殊的键执行

- **SDS**——    `SET`,`GET`,`APPEND`,`STRLEN`
- **Hash** —— `HGET`,`HSET`,`HDEL`,`HLEN`
- **LIST**—— `RPUSH`  `RPOP` `LINSERT` `LLEN`
- **SET**—— `SADD` `SINSERT` `SPOP` `SCARD`
- **ZSET** —— `ZADD` `ZCARD` `ZRANK` `ZSCORE`

## 8.内存回收

`C`没有内存回收的功能,`Redis`采用引用计数的方法实现内存的回收机制

~~~c
typedef struct redisObject{
    //引用计数
    int refCount;
}robj;
~~~

- 创建新对象，引用计数被初始化为1
- 对象被新程序使用，count++
- 对象被程序不再使用count --
- refCount = 0 的时候对象占用的内存会进行释放

## 9.对象共享

如果A存储了一个整数值为100的字符串对象，键B也要创建一个整数值为100的字符串对象，此时会进行键A和B的共享

![image-20220913152126616](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220913152126616.png)

> 共享对象不止有字符串键可以使用，数据结构镶嵌字符串对象(`linkedlist`编码的列表对象，`hashtable`编码的hash对象，`hashtable`编码的集合对象,`zset`的有序集合对象)

## 10.空转时长

> RedisObject还有一个LRU属性（空转时间=当前时间-lru定义的时间）

![image-20220913153904306](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220913153904306.png)

> 如果`volatile-lru`活着`allkeys-lru`，服务器会回收空转时间高的一部分

## 11.重点回顾

- Redis数据库每一个键和值都是一个对象
- 服务器在执行某些命令之前，会先检查这个键的类型，检查这个键的类型就是检查键的值对象类型
- Redis会共享0~9999的字符串对象
- Redis引入`引用计数`对象回收，当对象不被引用就被回收了
- Redis会存储最后一次对象被访问的时间，用于计算对象的空转时间

# 九、数据库

Redis的服务器都会保存在`redis.h/redisServer`

![image-20220913162512219](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220913162512219.png)

## 2.切换数据库

> 通过 SELECT + 数据库名 实现数据库的切换

### 数据库Client结构

~~~c
typedef struct redisClient{
    //记录客户端当前正在使用的数据库
    redisDb *db
}redisClient;
~~~

## 3.数据库键空间

Redis是一个键值对(key-value pair服务器)数据库服务器，服务器每个数据库都由一个`redis.h/redisDb`表示，`redisDb`中的`dict`保存数据库的所有键值对

~~~c
typedef struct redisDb{
    //数据库键空间，保存所有的键值对
    dict *dict;
}redisDb;
~~~

> 我们有如下场景

![image-20220914144330191](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220914144330191.png)

![image-20220914144341048](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220914144341048.png)



## 4.设置过期时间

1. `EXPIRE`<key><ttl>将生存时间设置为秒
2. `PEXPIRE`<key><ttl>将生存时间设置为毫秒
3. `EXPIREAT`将生存时间设置为秒时间戳
4. `PEXPIREAT`将生存时间设置为毫秒时间戳

## 5.过期键的删除策略

**定期删除：** 

> 设置一个定时器`Timer`，让定时器在过期时间来临，执行对键的删除操作

**惰性删除:**

> 放任键过期不管，每次取这个键的时候都判断这个键是否过期，如果过期就删除这个键，没有过期就返回这个键

**定性删除**：

> 隔一段时间，程序就对数据库进行检查，删除里面的过期键，如何删除通过算法决定

### 1.定期删除

**特点** ： 对CPU不友好，对内存友好 （如果CPU过于紧张，可能会影响服务器的吞吐量和响应时间）

> 除此之外，设置定时器的实现方式是无序链表， 查找一个东西的时间复杂度是 `O(n)`, 并不能处理大量事件，所以用定期删除不现实

### 2.惰性删除

**特点**： 对CPU友好，只有取出来键的时候才会进行过期检查，但是对内存不友好，内存里面有过多的过期键来占用内存

### 3.定期删除

- 定时删除占用太多CPU时间，影响服务器反应时间
- 惰性删除浪费内存，有内存泄漏的风险

定期删除是策略的整合和折中：

- **定期删除策略**是每隔一段时间执行一次删除操作，  并通过限制删除操作执行的时长和频率来减少删除操作对CPU的影响

定期删除的难点是确定删除操作执行的时长和频率

- 执行太频繁或者执行时间太长，定期删除就会退化为定时删除，以至于CPU时间过多小号在删除过期键
- 执行时间太短，又会退化为惰性删除

## 6.过期键的实现

>  Redis服务器实际使用的是惰性删除和定期删除

### 1.惰性删除的实现

过期键的惰性删除策略是 `db.c/expireIfNeeded`实现

- 输入键已经过期，那么`expireIfNeeder`函数将输入键从数据库删除
- 输入键未过期,`expireIfNeeded`不做处理

### 2. 定期删除策略实现

`redis.c/activeExpireCycle`函数实现, 每当`redis服务`周期性操作`redis.c/serverCron` ，`activeExpireCycle`就被调用

> 运行过程大致概括为如下：

- 函数运行时候，都从一定数量数据库检查并且删除过期键
- `current_db`会记录当前`activeExpireCycle`函数检查的进度，并且在下一次函数调用，按照上一次进度处理
- 随着`activeExpireCycle`函数的不断执行，服务器数据库被检查一遍，此时`current_db`会被重置为0，再进行下一次的工作

## 7.AOF,RDB对过期键的处理

### 1.生成RDB文件

`SAVE`和`BGSAVE`创建新的RDB文件，程序会对数据库的键进行检查，如果k1,k2,k3中k2过期，那么程序只会保存k1,k3到RDB中

### 2.AOF文件写入

当过期键被惰性删除或定期删除，程序会给AOF文件追加一个`DEL`命令，来显示记录键被删除

- 数据删除国期间
- AOF日志追加DEL `message`命令到AOF文件
- 向执行GET命令的客户端返回空回复

## 9. 重点回顾

- `Redis`服务器所有数据库都保存在`redisServer.db`上,数据库数量是`redisServer.dbnum`保存
- 客户端通过修改目标数据库的指针，让它指向`redisServer.db`数组的不同元素
- 数据库主要是`dict`和`expires`组成的，其中`dict`字典负责保存键值对，`expires`负责保存过期时间
- `SAVE`和`BGSAVE`生成的`RDB`文件不会包含过期键
- `BGREWRITEAOF`生成的`AOF`文件不会包含过期键

# 十、RDB持久化

## 1.RDB文件的写入

分为两种`SAVE`和`BGSAVE`

**SAVE**

> 服务器被阻塞，当`SAVE`命令执行，客户端发送的命令被拒绝

**BGSAVE**

> fork一个子进程执行保存，`Redis`服务器仍然可以处理客户端的命令

**SAVE/BGSAVE/BGREWRITEROF** 之间的竞争关系

- `BGSAVE`执行，`SAVE`被拒绝
- `BGSAVE`执行，有`BGSAVE`请求也会被拒绝
- `BGSAVE`执行,`BGREWRITERAOF`会到`BGSAVE`执行完毕之后执行
- `BGWRITERAOF`正在执行，客户端发送的`BGSAVE`会被服务端拒绝

## 3.RDB的文件结构(3 文件结构和4 分析未看)

|       |            |           |      |           |
| ----- | ---------- | --------- | ---- | --------- |
| REDIS | db_version | databases | EOF  | check_sum |

## 5.重点回顾

- `RDB`文件用于保存和还原`Redis`服务器所有数据库的所有键值对
- `SAVE`执行保存操作，会阻塞服务器
- `BGSAVE`由子进程执行保存操作，该命令不会阻塞服务器
- 服务器状态会保存`save`选项的保存条件，当任意一个条件被满足，服务器会自动执行`BGSAVE`
- `RDB`是执行压缩的二进制文件，通过多个部分组成

# 十一、AOF持久化



## 1.AOF 持久化的实现

- 命令追加
- 文件写入
- 文件同步

### 1.命令追加

> 服务器执行完一个写命令之后，会通过协议的形式写命令追加到服务器的 `aof_buf`末尾

~~~c
struct redisServer{
    
    //AOF缓冲区
    sds aof_buf;
}
~~~

## 2.AOF文件的写入和同步

`Redis`服务器进程是一个事件循环(loop),时间事件负责执行像`serverCron`函数，服务器会执行写命令，使内容被追加到`aof_buf`缓冲区，所以每一次都会调用`flusnAppendOnlyFile`

```python
def eventLoop(): 
    while True:
    # 处理文件命令,接收命令
    # 处理命令可能会被新内容加入到 aof_buf 缓冲区
    processFileEvents()
    # 处理时间事件
    processTimeEvents()
    #考虑是否将`aof_buf`内容写道和保存到AOF文件中
    flushAppendOnlyFile()
```

| `appendfsync`值 | flushappednonlyfile函数行为                                  |
| --------------- | ------------------------------------------------------------ |
| always          | `aof_buf`的所有内容同步到AOF文件                             |
| everysec        | `aof_buf`所有内容写道AOF文件，时间超过1s,再次进行同步        |
| no              | `aof_buf`缓冲区内容写到AOF文件，但是不进行同步，如何同步由操作系统决定 |

## 3.AOF重写

> `AOF` 通过保存执行的写命令记录数据库状态，AOF文件内容越来越多，如果不加以控制，AOF文件就会过大，对宿主机产生影响

![image-20221026172659036](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221026172659036.png)

> 如果服务器想用尽量少的命令记录`list`键的命令，最简单高效的是直接从数据库读取键`list`，然后用 `RPUSH list "C" "D" "E" "F" "G"` 代替AOF文件中的6个命令

### 1.AOF文件重写的实现

![image-20221026174033667](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221026174033667.png)

### 2.AOF后台重写

AOF重写`aof_rewrite`完成创建`AOF`文件，Redis不希望AOF重写造成服务器无法处理请求，所以 **AOF重写放入到子程序执行**  

![image-20221027092121723](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221027092121723.png)

> 这种情况我们重写容易出现重写前后显示不一致的情况，所以我们引入`AOF重写缓冲区`

1. 执行客户端发来的命令
2. 将执行后的命令写入AOF缓冲区
3. 将执行后的命令写入AOF重写缓冲区

## 4. 重点回顾

- AOF文件保存所有修改数据库的命令来记录服务器的数据库状态
- AOF文件都以Redis命令协议形式保存
- AOF重写可以产生一些新的AOF文件
- 执行`BGREWRITERAOF`时候，Redis服务器会维护一个AOF重写缓冲区，这个命令会在子进程创建AOF文件的时候，记录服务器执行的写命令，当子进程完成AOF文件的创建之后，新旧的两个AOF的文件读写一致

# 十二、文件事件

**大致有两类事件：**

**文件事件(file event)**: `Redis服务器`通过套接字和客户端进行连接，文件事件通过服务器对套接字进行操作的抽象

**时间事件(time event)**:`Redis服务器`的一些操作，需要给定时间点执行，时间事件就是服务器对这类定时操作

## 1.文件事件

`Redis`基于`Reactor`模式开发了自己的网络事件处理器: 这个处理器被称为文件时间处理器

- 文件时间根据`I/O多路复用`监听多个套接字，并根据套接字执行任务
- 被监听套接字准备好执行连接应答(accept),读取(read),写入(write)

> 文件通过 `单线程运行的方式`, 通过`I/O多路复用`来监听多个套接字
>
> - 既实现高性能的网络通信模型
> - 也很好的和Redis的其他模块进行对接

### 1.文件时间处理器的构成

**构成：**

- 套接字
- I/O多路复用程序
- 文件事件分排期
- 事件处理器

![image-20221104220413620](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221104220413620.png)



> 我们通过IO复用是套接字，一个服务器可能会连接多个套接字，所以文件系统可能并发出现
>
> 尽管多个文件可能并发出现，但是I/O多路复用总是将产生的事件套接字放到一个队列里面， `当一个套接字处理完毕，I/O多路复用程序才会继续向文件分配套接字`

![image-20221104220911114](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221104220911114.png)

### 2.I/O多路复用的实现

> Redis的`IO多路复用程序`都是包装常见的`select`, `epoll`, `evport`,`kqueue`这些`I/O多路函数库`实现的，**不过这些函数公用一套API**，所以这些的底层实现是可以互换的



> I/O程序会根据`#include`宏定义决定挑选那个库函数

![image-20221104221828967](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221104221828967.png)

## 2.时间事件

### 分为两类

**定时事件：**让一段程序`在指定的时间后执行一次`

**周期性时间**: 让一段程序`每隔指定时间执行一次`

### 三大属性

- `id`: 服务器为时间事件创建全局`唯一ID`， ID从小到大，新事件ID一定大于旧事件ID
- `when`: `毫秒`级别的unix时间戳,记录时间事件到达的时间
- `timeProc`：时间事件处理器，一个函数，当时间事件到达就会调用相应的函数

### 如何判断是什么函数

> 根据事件处理器返回值判断函数

- 返回的是`AE_NOMORE`，就是定时时间
- 返回的是非`AE_NOMORE`,就是周期性事件

### 1.实现

> 服务器将时间事件放在一个`无序链表(时间无需)`,每次执行器执行，就遍历整个链表，新的时间事件总是插入到链表表头

![image-20221114090446001](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221114090446001.png)

## 3.时间时间:`serverCron函数`

- 更新服务器的各类统计信息： 时间，内存，数据库占用
- 清理数据库过期键值对
- 关闭无效的`Redis客户端`
- 进行`AOF`和`RDB`操作
- 如果是主服务器会对`从服务器`定期同步
- 如果是`集群`会做`定期同步`和`连接测试`

## 4.重点回顾

- `Redis`服务器是一个事件驱动程序，服务器处理的事件分为`时间事件`和`文件事件`两类
- 文件时间处理器是基于`Reactor模式`的网络通信程序
- 文件时间对套接字，每次是`acceptable` ,`readable`,`writable`可以实现
- 时间事件： 定时事件和周期性时间，定时事件会等一段时间后只执行一次，周期性事件会反复执行
- `Server`一般`周期性`执行`ServerCron`这个函数
- `时间事件`和`文件时间`是协同关系，每个事件不会独占
- `时间事件`的实际处理时间一半比预期时间要来的晚一些



















