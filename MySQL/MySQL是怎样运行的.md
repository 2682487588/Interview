# 1.装作自己是个小白

## 1.3 启动MySQL服务

~~~bash
net start mysql  #启动mysql
net stop mysql  #停止mysql
~~~

## 1.4启动MySQL客户端

mysql -h 主机名（可以省略） -u 用户名 - p密码

## 1.5 客户端和服务器的链接过程

MySQL采用TCP协议进行通信，每个进程可以向操作系统申请一个端口号，范围0~65535,IP+端口号对进程进行连接

MySQL服务器在启动会默认申请3306端口号

~~~bash
mysql -P #通过mysql -P(大写)可以指定端口
~~~

### 1.5.2 共享内存通信

- 命名管道通信  -enable-named-pipe
- 共享内存  --shared-memory

### 1.5.3 UNIX域套接字

~~~bash
mysqld --socket=/tmp/a.txt
#客户端也想通过UNIX域套接字通信，也需要显示连接UNIX域
mysql -hlocalhost -uroot --socket=/tmp/a.txt -p
~~~

## 1.6 服务器处理客户端请求

客户端向服务端发文本(MySQL语句),服务端向客户端发送文本(处理结果),服务端做了什么处理（增删改查结果）

![image-20220613202425176](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613202425176.png) 

- 连接灌流
- 解析优化
- 存储引擎

### 1.6.1连接管理

客户端进程简历连接

- TCP/IP
- 命名管道
- 共享内存
- UNIX域套接字

和服务端程序建立连接

每次有一个客户端连接进行来时，服务器会分配一个进程给这个客户端，专门和这个客户端进行交互，**客户端退出会和这个服务器断开连接，但是线程不会销毁**，而是**缓存起来**，另外一个客户端开启的时候，这个被缓存的线程会分配给新的客户端，这样不会频繁创建和销毁线程

如果客户端和服务端不在一台机器上，可以用传输层安全协议(TLS)进行加密，保证传输的可靠性

建立连接后，服务器一直等待MySQL客户端发送过来的请求，MySQL服务器发送过来的请求只是一个文本消息，该文本消息还需要经过处理

### 1.6.2 解析和优化

接收到客户端传来的文本消息后，MySQL服务器可以获取文本，但是还需要下面

- 查询缓存
- 语法解析
- 查询优化

**1.查询缓存**

MySQL服务器会把之前的**查询请求和结果储存下来**，如果有同样请求，直接从缓存查就好了

但是,MySQL不会那么聪明，两个查询如果有请求不同(空格，注释，大小写)都会导致缓存不会命中

查询缓存可以不同的客户端共享，客户端A发送这个查询请求，客户端B发送同样查询请求，那么B就能使用这次缓存了

两次查询调用同一个函数结果可能不一样，比如NOW函数，第一次缓存了结果，那么第二次就是错的

**缓存失效：**如果对表的结构和数据修改这个缓存就会失效

- insert
- update
- delete
- truncate
- table
- alter table
- drop table
- drop database

都会让查询缓存变为无效缓存

**2.语法分析**

查询缓存没有命中，接下来就需要查询阶段，客户端发过来是文本，需要解析这个文本语法是否正确，然后把 **需要查询的表，语法**提取出来

**3.查询优化**

语法解析之后，服务器知道要获取哪些信息，但是这样是不够的，我们写MySQL的执行效率可能不高，MySQL优化程序会对我们语句优化，如外连接转为内连接，表达式简化，子查询转为连接

### 1.6.3 存储引擎

完成查询优化，服务器可能没有真正访问数据（或者反问一部分数据），MySQL服务器吧数据的存储和提取操作封装到了名为存储引擎的模块

物理上如何**表示记录**，从**表中处理数据**，将**数据写在物理存储器**上都是**存储引擎**负责的事情

MySQL服务器处理请求的过程分为Server层和存储引擎层

除了存储引擎，其他全是Server层，所以server做好查询优化，只需要按照生成计划调用底层存储引擎提供的接口就行

> server在判断记录符合要求，先是将其发送到缓冲区，然后缓冲区满，才向客户端发真正记录，缓冲区大小由系统变量 net_buffer_length控制

## 1.7存储引擎

![image-20220613214854690](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613214854690.png)

![image-20220613214959572](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613214959572.png)

## 1.8  存储引擎操作

![image-20220613215318193](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220613215318193.png)

Support:是否可用

Comment:描述

Transactions:事务

XA：分布式事务

Savepoints:事务是否支持回滚

### 1.8.2 设置表的存储引擎

显示指定存储引擎

~~~mysql
create table 表名(
	建表语句;
)engine = 存储引擎名字;
~~~

# 3. 字符集和比较规则

## 3.1 字符集和比较规则简介

### 3.1.1字符集简介

计算机存储的是二进制数据，怎么存储字符串，  **建立字符和二进制之间的关系**

- 字符映射二进制数据
- 将字符映射成二进制数据的过程叫编码，将二进制数据映射成过程的操作叫做解码

定义一个字符集  xiaohaizi4919

包括如下

![image-20220614144253107](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614144253107.png)

![image-20220614144259850](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614144259850.png)

### 3.1.2 比较规则简介

比较二进制的大小来比较字符大小

### 3.1.3 一些重要字符集

#### **ASCII字符集**：

收录128字符，包括空格，标点符号，数字，ASCII字符集总共128字符，所以可以用一个字节编码

#### **ISO 8859-1字符集**：

256字符，在ASCII字符基础扩充了 西欧常用字符

#### **GB2312字符集**

- 汉字
- 拉丁字符
- 希腊字母
- 拉丁评价自
- 俄语西里尔字母
- 兼容ASCII

“爱u”

爱占两个字节，0xB0AE  u占一个字节， 0x75  拼起来就是0xB0AE75

#### GBK

知识对GB2312字符集进行扩充

**UTF-8字符集**：收录了世界各个国家/地区常用的字符，现在还在继续扩充，字符集兼容ASCII字符集，采用变长方式，编码字符需要1~4字节

'L' ->01001100(1字节 0x4C)

'啊' -> 111001011001010110001010 (3字节，0xE5958A)

![image-20220614145648579](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614145648579.png)



## 3.2  MySQL支持的字符集和比较规则

UTF-8定义一个字符需要1~4字节，但是常用的只需要1~3字节

- ut8mb3:1~3字节定义字符
- utf8mb4:1~4字节定义字符

### 3.2.2字符集查看

~~~bash
SHOW (CHARACTER SET|CHARSET) [LIKE 匹配的模式];
~~~

![image-20220614151345477](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614151345477.png)



### 3.2.3 比较规则的查看

~~~go
SHOW COLLATION [LIKE 匹配的规则]
~~~



![image-20220614151731207](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614151731207.png)

- utf8_general_ci 指的是通用的比较规则

![image-20220614151953173](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614151953173.png)

## 3.3 字符集比较规则

![image-20220614152848458](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614152848458.png)

### 2.数据库级别

![image-20220614154114355](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614154114355.png)

#### 3.表级别

![image-20220614154709307](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614154709307.png)

#### 4.列级别

![image-20220614154727222](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614154727222.png)

#### 5. 修改字符集或修改比较规则

字符集和比较规则相连，修改其中任意一个，另外都会变化

- 修改字符集，比较规则会变为修改后字符集默认比较规则
- 修改比较规则，字符集变为修改后比较规则对应字符集

#### 6.小结

- 创建修改列没有显示指定比较规则，则默认使用表字符集规则
- 创建表没有显示比较规则，默认用数据库字符集规则
- 创建数据库没有显示指定字符集和比较规则，默认使用服务器比较规则



### 3.3.2 客户端和服务器通信使用字符集

**1.编码和解码不一致**

![image-20220614160641973](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614160641973.png)

### 2.字符串转换概念

0xE68891然后按照GBK字符集进行编码 ，编码后的字节序列就是0xCED2

### 3.MySQL字符集转换过程

**客户端发送请求**

- 在Windows操作系统

在Windows中，字符集称为代码页，一个代码页跟一个数字管理

![image-20220614162735326](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614162735326.png)

![image-20220614163732292](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614163732292.png)

## 3.4总结

![image-20220614164934063](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614164934063.png)

![image-20220614164945584](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220614164945584.png)

# 4. InnoDB记录存储结构

# 6.B+树索引

## 6.1 没有索引进行查找

~~~mysql
SELECT [查询列表] FROM 表明 WHERE 列名 = XXX
~~~

### 6.1.1 在页中查找

在查找的时候可以根据搜索条件的不同分为两种情况:

**主键为搜索条件:** 页目录采用二分法找到对应的槽，然后遍历该槽找到对应的记录

**其他列为搜索条件**：  非主键列没有对应的页目录，无法通过二分法查找，只能从Infimum开始遍历单向链表的记录



### 6.1.2  在很多页中查找

- 定位到记录所在的页
- 从所在的页查找相应的记录

在没有索引的情况，根据主键列还是其他列的值进行查找，我们不能快速定位记录所在页，只能从第一页沿着双向链表向下找

## 6.2 索引

~~~mysql
create table index_demo(
		c1 INT,
		c2 INT,
		c3 char(1),
		PRIMARY KEY(c1)
)ROW_FORMAT= COMPACT;
~~~

我们定义2个INT的值，1个CHAR(1)的值，我们定义c1为主键，使用COMPACT定义记录

![image-20220615142735142](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220615142735142.png)

**record_type：**记录头信息的一些属性，表示记录属性

- 0：普通记录
- 2：Infimum记录
- 3：Supremum记录
- 1：还没用过

**next_record**:记录头信息的属性，表示从当前记录的真实数据到下一条记录的真实数据的距离，**我们会用箭头表明下一条记录是谁**

**各个列的值**：c1,c2,c3

**其他信息**: 除了3种信息以外的所有信息， 包括其他隐藏列的值和记录的额外信息

![image-20220615144441589](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220615144441589.png)

### 6.2.1 简单索引案例

为什么我们根据搜索条件查找记录，会遍历所有的数据页，因为各个页记录之前没有规律，我们可以 **为快速定位记录的数据页面建立一个别的记录**， 建立目录的过程中必须完成两个事

   1.下一个数据页用户记录的主键值 必须大于 上一个页中用户记录的主键值

~~~mysql
INSERT INTO index_demo VALUES (1,4,'u'),(3,9,'d'),(5,3,'y');
~~~

这三条记录被串成单向链表了，假设每个数据页只能存放3条记录

![image-20220615145545068](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220615145545068.png)

分配的页号分配成28，因为分配的数据页本来就不是连续的，这些页在磁盘上并不挨着, 页10有一个主键为5，页28主键为4，5>4 这就不符合 1.下一个数据页用户记录的主键值 必须大于 上一个页中用户记录的主键值要求，

**我们需要把5向后移动，把4移动到页10**，如下图

![image-20220615150354970](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220615150354970.png)

该过程表明，我们需要通过 记录移动等操作保证  **1条件**城里， 这个过程可以被称为页分裂

2. 给所有页建立一个页目录项

数据页编号不是连续的，所以向index_demo插入多条记录可能形成这样的效果

![image-20220615150916176](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220615150916176.png)

这些大小16KB的页在磁盘并不挨着，为了根据主键快速锁定这些记录所在的页，就要给他们编制一个目录，**每个页对应一个目录项，每个目录项包括下面的两部分**

- 页的用户记录中最小的主键值：用key表示
- 页号:用page_no表示

![image-20220615151611538](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220615151611538.png)

我们只需要几个目录项在物理存储器连续存储，然后把他们放到一个数组中，就可以实现根据主键快速查找某条记录的功能

如果我们想要查找主键值为20需要分为两步，

1.在目录项中通过二分查找出20在目录3中，（12<20<209），对应的页是9

2.然后在页中查找方式定位页9定位具体的记录

这样简易目录就搞定了，这个目录有个别名，叫索引

### 6.2.2 InnoDB的索引方案

刚才为每个数据页指定目录项的过程是一个简易制作过程，  我们根据主键进行查找的时候，通过二分法快速定义到目录项

为了使用二分法进行定位目录项，假设目录项在物理存储器上连续存储，但是有以下几个问题

- InnoDB使用页管理空间的基本单位， 最多保证16KB的连续存储空间，随着记录越来越多，这些空间记录连续目录项是不现实的
- 我们时常会增删改查，假设我们把页28删除，这样目录项2也就没必要，我们就需要把后面的目录往前移，**这种牵一发动全身不是好建议**    或者我们将他作为冗余放在目录项列表

解决方案：我们发现这些记录和用户记录很像，只不过 目录项两个列一个代表主键，一个代表页号，为了区分我们将目录项形成的记录记为目录项记录

我们通过record_type区分该记录是目录项记录还是页记录

- 0 ：普通的用户记录
- 1 ： 目录项记录
- 2：Infimum
- 3:   Supremum

![image-20220618164123447](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220618164123447.png)

- 目录项记录只有两个主键列，普通用户记录只是用户自己定义的，可能包含很多列， 另外还有InnoDB自己的隐藏列
- min_rec_flag属性，只有目录项记录min_rec_flag才为1，普通用户为0

其他没啥区别 ，同样用的是数据页(页面类型是0x45BF,为主键值生成 Page Directory(页目录))

**有目录项记录之后查找方式**,以查找主键20为例

- 1.通过二分法查找对应的目录项记录(12<20<209) 找到页9
- 2.通过二分法查找页9锁定键

问题： 一个页的大小是固定的，如果一个页不足以放更多， 只能新建也来存储多余的目录项，我们假设一个页只能存放四个目录项记录，此时插入一个主键值为320的用户，需要新分配一个存储目录项记录的页

![image-20220618171117420](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220618171117420.png)

从6-11可以看出来，插入主键为320的用户记录之后，需要两个新的数据页

- 为存储用户记录产生31
- 因为页30不能存放更多目录项记录，所以只能开辟一个新的存储目录项记录的页32来存储31

我们想找用户记录大致分为3个步骤

- 确定存储目录项记录的页
- 通过目录项找到用户记录所在页
- 定位真正用户所在的记录

但是问题来的，我们要定位很多存储目录项记录的页，这些页不挨着，如果数据很多，我们需要生成一个更高级目录来管理 ， 如图 6-12所示

![image-20220618172252374](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220618172252374.png)

在6-12，我们生成了一个存储更高级目录的页33，这两条记录对应30和32，如果用户记录主键在[1,320)，就在页30，否则就在页32

**这个像是一个倒着过来的树——上面是树根，下面是树叶，这是一种数据结构，叫做B+树**

无论是存放用户记录的数据项，还是存放目录项记录的数据项，我们都把它存放到B+树，我们将这些数据页称为B+树的节点， **但是我们真正存放用户记录的节点被放在B+最底层的节点，称为叶子节点**

![image-20220618173600587](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220618173600587.png)



从图6-13可以看出，InnoDB把最底层设计为0层，如果存放用户记录叶子节点数据页100条用户记录，存放目录项的数据页可以存放1000条目录项数据，那么

![image-20220618195817559](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220618195817559.png)

![image-20220618195905574](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220618195905574.png)

如上图，一般情况下B+树不超过四层，

> Page Header有一个PAGE_LEVEL的属性，这就代表节点在B+中的层级

#### 1.聚簇索引

记录主键值进行记录和页的排序，包括以下三方面的含义

- 页（叶子节点和内节点）内的记录按照主键大小排成单向链表，每组主键值最大的记录在页内偏移量会当做槽放在页目录中（Supremum比任何记录都大），我们可以通过二分法快速定位主键列等于某个值的记录
- 存放用户记录的页也是根据主键的大小排列成一个双向链表
- 存储目录项的记录页分为不同的层级，同一层级的页是根据页中目录项记录主键大小排成双向链表

B+树存储是完整的用户记录，就是指这个记录存储所有列的值(包括隐藏列)

具有这两个特点的叫做聚簇索引，所有用户记录都放在聚簇索引的叶子节点，聚簇索引不需要我们在MySQL中创建，InnoDB引擎会自动帮我们创建聚簇索引

InnoDB存储引擎中，聚簇索引就是数据存储方式

#### 2.二级索引

聚簇索引只能在搜索条件是主键时候发挥作用，原因是B+数的数据都是按照主键排序，

**问题：** 以别的列为索引，只能从头到尾遍历链表进行排序吗

**解决：**我们可以多建立几个B+树，并且不同B+树使用不同的排序规则，我们以c2列大小为数据页

![image-20220618211425663](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220618211425663.png)



这个B+树和前文聚簇索引有些不同

- 记录c2列的大小进行记录和页的排序
  - 页（叶子节点和内节点）按照c2列大小形成单向链表，页内记录分为若干组，每组c2列最大记录在页的偏移量会被当做槽存放在也记录
  - 各个存放记录根据页内记录c2列大小排成双向链表
  - 存放目录项的页分为不同层级，同一层级根据c2列大小进行排序
- B+树的叶子及诶单不是完整的用户记录，而是c2列+主键两个列的值
- 目录项不是主键+页号搭配 而是c2列+页号

如果我们想查找c2=4,因为c2列没有唯一性约束，我们需要找到第一个满足c2=4的记录，然后一直往后搜索

另外，各个叶子节点组成了单向链表，我们搜索完本页面可以顺利跳转到下一个页面

**步骤1**：确定c2=4所在的页，根据页44找到页42(2<4<9)

**步骤2:** 确定符合条件的第一条记录在34页或者35页中(2<4<=4)

**步骤3:** 在真正的存储第一条符合c2=4的页中定位到具体的记录

**步骤4：**这个B+数的叶子节点记录只存储了c2和c1两个列

在B+树的叶子节点定位到第一个符合用户记录，我们需要根据这个主键信息到聚簇索引中定位到完整的用户记录 也称为 **回表**， 

然后定位到符合条件的用户记录，沿着记录组成的单向链表，搜偶所其他满足的记录  **每找到一条记录就进行一次回表操作**，重复这个过程，知道不满足c2=4为之

**问题：**为什么还需要进行回表，直接把完整的用户记录放在叶子节点就行，这样确实避免回表

**但是每次创建B+树都需要把用户记录遍历一遍，这样浪费内存空间**

因为这种用非主键列的大小排序简历的B+树,需要执行回表操作才能定位到完整用户记录，这种B+树称为二级索引或者辅助索引

我们以c2列的大小为B+树的排列郭泽，这种B+树索引被称为索引列。二级索引列和聚簇索引也是一样的记录行，但是耳机索引记录存储列不像聚簇索引那么完整

> 我们把聚簇索引和二级索引中叶子节点叫为用户记录，我们把聚簇索引的叶子节点叫做完整的用户记录，二级索引的叶子节点叫做不完整的用户记录
>
> c1,c2列都是数字，如果c3列(字符串)建立索引，又有哪些会不一样

#### 3.联合索引

我们可以以多个列的大小进行索引排序，比如 **我们要B+树给c2,c3建立索引**，这包含两个规则

- 先将各个记录和页以c2的大小进行排序
- 在c2相同的情况下，根据c3列的大小进行排序

根据c2列和c3列的大小进行排序如下

![image-20220620155929117](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220620155929117.png)

- 每条目录项记录都由 c2 列、 c3 列、页号这 部分组成.各条记录先按照 c2 列的值进行排序 如果记录的 c2 列相同，则按照 c3 列的值进行排序
- B+树的叶子节点处的用户记录是又c2,c3和主键c1组成

 以c2,c3列大小排序建立的B+树的索引称为二级索引，称为复合索引或多列索引，本质也是一个二级索引，索引列包括c2,c3

**注意：“以c1和c2列建立联合索引”和对“c1,c2列建立索引代表的意思是不一样的”**

- 建立联合索引，只会建立图6-15这个B+树
- 给c2,c3列建立索引，会分别给c2,c3列的大小为排序规则建立索引

### 6.2.3  InnoDB中B+树索引的建立规则

#### 1.根页面万年不动根

在B+树介绍的时候我们把存储用户记录的叶子节点画出来，然后画出存储目录项记录的内节点，实际情况是这样的

- 先给表创建B+索引(聚簇索引不是认为创建的，默认就存在)，都会为这个索引创建一个根节点。**最开始表中没有数据，每个B+树索引对应的根节点没有用户记录**
- 向表中插入用户记录，然后将用户记录存储到这个根节点
- 根节点可用空间用完继续插入揭露，会将根节点赋值到新分配的页，然后进行 **页分裂**操作，新插入的记录会根据键值(聚簇索引中主键的值，二级索引对应索引列的值)的大小分配到页a和页b

B+树索引的根节点从创建开始不会移动(页号不会改变)，**这样对某个表创建索引，这个根节点的页号会记录到某个地方**，后续凡是用到这个索引，都会从固定地方获取根节点的页号

> "存储某个索引的根节点的页面"就是 **数据字典中的一项信息**

#### 2.内节点的目录项记录的唯一性

B+树索引的内节点中， 目录项记录内容是 **索引列加页号**搭配

![image-20220620164505180](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220620164505180.png)

二级索引，目录项记录是 索引列+页号，为c2列建立索引后B+树, **如图6.16所示**

![image-20220620165742030](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220620165742030.png)

我们想插入一行记录，其中c1,c2,c3列值分别为9,1,'c',修改c2列简历，对于页三来说，页四和页五的主键范围都是1，**所以不知道应该放在页4还是页5中**

如果让新插入的记录找到自己在那一页，需要保证B+树同一层内节点的目录项除了页号这个字段是唯一的

- 索引列的值
- 主键值
- 页号

**也就是我们把主键值添加到二级索引内节点的目录项记录中**，这样就保证B+树的每一层节点各条目录项记录除了页号这个字段的记录是唯一的

![image-20220620171428684](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220620171428684.png)

这样c2列+主键的值肯定不一样，这样应该能插入到页5

> 对于二级索引记录来说，先按照二级索引列进行排序，二级索引列相同的情况下，根据主键值进行排序，**为c2列建立索引相当于为(c2,c1)列建立联合索引**

#### 3. 一个页面至少容纳2条记录

一个B+树只需要很小的层级，就可以存储很多的记录

**问题：** 如果一个大目录只能存放一个子目录是是什么效果 ？

目录层级会比较多，而且最后存放真正数据只是一条记录

> B+数的叶子节点只存储，让内节点存储多条记录，还是可以发挥B+树作用的，但是InnoDB为了防止B+树的层级过高，要求数据页可以容纳至少2条记录

### 6.2.4  MyISAM索引方案简介

### 6.2.5 MySQL中创建和删除索引的语句

InnoDB或MyISAM都会为**主键或带有UNION属性**的列创建索引，如果想为其他列创建索引，需要我们显示指明

**创建表添加索引**

~~~mysql
CREATE TABLE 表名(
	各个列的信息
    (KEY|INDEX) 索引名  （需要被索引的多个列或单个列）
)
~~~

**修改表添加索引**

~~~mysql
ALTER TABLE 表明 ADD （INDEX|KEY） 索引名 (需要被索引的单个列或多个列)
~~~

如果我们想为c2,c3列**加索引**

~~~mysql
CREATE TABLE index_demo(
  c1 INT,
  c2 INT,
  C3 CHAR(1),
  PRIMARY KEY c1,
  index index_c2_c3 (c2,c3)
);
~~~

**删除索引**

~~~mysql
ALTER TABLE index_demo DROP INDEX index_c2_c3;
~~~

## 6.3 总结

InnoDB索引引擎是B+树，完整的用户记录都存在B+树第0层的叶子节点

其他层次的节点都属于内节点，存储的是目录项记录

- **聚簇索引：**以主键值的大小和记录的排序规则，在叶子节点保存的是表中所有的列
- **二级索引**：以索引列的大小和记录的排序规则，在叶子节点中保存的是索引列和主键值

**InnoDB存储引擎的B+树根节点从创建时候就不再移动**

> 二级索引中，目录项由索引列的值，主键名和页号组成

> 一个数据页至少容纳两条记录



# 7 索引使用

## 索引详细使用参考这篇文章

https://blog.csdn.net/weixin_47162914/article/details/123793589

## 2索引适用条件

**我们建立索引有** 

~~~mysql
CREATE TABLE person_info(
 id INT NOT NULL auto_increment,
 name VARCHAR(100) NOT NULL,
 birthday DATE NOT NULL,
 phone_number CHAR(11) NOT NULL,
 country varchar(100) NOT NULL,
 PRIMARY KEY (id),
 KEY idx_name_birthday_phone_number (name, birthday, phone_number)
);
~~~

### 最左匹配原则

**最左优先，从最左边为起点的任何连续的索引都会匹配上**

### 1. 全值匹配

~~~mysql
SELECT * FROM person_info WHERE name = 'Ashburn' AND birthday = '1990-09-27' AND phone_num
ber = '15123983239';
~~~

> 如果我们搜索条件和索引一直就是全值匹配
>
> 这种情况，我们idx索引的3个列都在查询语句了

### 2.匹配左边的列

> 由于索引的匹配顺序是 `name` , `birthday`, `phone_number` , 所以我们 **尽量按照匹配最左边连续的列**

### 3.列前缀匹配

> 我们存储的字符串都是通过列前缀匹配的，所以我们需要匹配的时候比如用 `AS%` 开头的记录就可以用索引， 但是例如 `%AS`和`%AS%`就不能用索引

### 4. 匹配范围值

我们看 `idx_name_birthday_phone_number`索引, **所有记录都是按照索引值从小到达排序的**,  

~~~mysql
SELECT * FROM person_info WHERE name > 'Asa' AND name < 'Barlow' AND birthday > '1980-01-0
1';
~~~

这个查询可以分为两部分：

- 查询name
- 根据name的结果给birthday筛选

> 但是我们根据`name`查询后的结果可能对于`birthday`是无序的，所以birthday就走不了索引

### 5.精确匹配某一列并范围匹配另外一些

对于联合索引来说，如果是左边是 **精确查询**,那么右边可以是 **范围查找**

~~~mysql
SELECT * FROM person_info WHERE name = 'Ashburn' AND birthday > '1980-01-01' AND birthday
< '2000-12-31' AND phone_number > '15100000000';
~~~

这个查询可以分为三部分：

1.  name =  'Asburn' ，精确查找用B+树索引
2. birthday > '1980-01-01' AND birthday < '2000-12-31' , 由于name是精确查找，所以结果name 都是想通的，所以可以继续按照birthday排序
3. phone_number > '15100000000' 因为 经过 birthday排序后的 phone_number值可能不同，所以就无法用B+树索引了

### 6.排序

如果通过`ORDER BY` 排序，一般情况下我们都能把记录加载到内存，再用一些排序算法，一般排序如果数据量大的话就非常慢了，但是如果排序用到了索引字段，就等于说已经是排好序的，直接 `回表`找索引就可以了

```mysql
SELECT * FROM person_info ORDER BY name, birthday, phone_number LIMIT 10;
```

#### 1.联合索引排序注意事项

对于`联合索引`有个问题需要注意，`ORDER BY`**子句后面的顺序必须按照索引列的顺序给出**

####  2. 不可以使用索引排序的几种情况

1.**ASC,DESC混用**

2.**where字句出现field排序用到的索引列**

~~~mysql
-- 非排序使用到的索引列，排序是匹配不到索引的
SELECT * FROM person_info WHERE country = 'China' ORDER BY name LIMIT 10;
~~~

~~~mysql
SELECT * FROM person_info WHERE name = 'A' ORDER BY birthday, phone_number LIMIT 10;
-- 这种情况 name是索引，而且过滤后的记录还在索引中，是可以进行索引排序的
~~~

3.**排序列包含非一个索引的列**

~~~mysql
SELECT * FROM person_info ORDER BY name, country LIMIT 10;
~~~

4.**排序列采用复杂表达式**

要想使用索引列，就需要保证 **索引列是单独列的形式出现**，比如下面就不是

~~~mysql
SELECT * FROM person_info ORDER BY UPPER(name) LIMIT 10;
~~~

### 7. 用于分组

> 对于下列分组排序适用于索引

~~~mysql
SELECT name, birthday, phone_number, COUNT(*) FROM person_info GROUP BY name, birthday, ph
one_number;
~~~

## 3. 回表的代价

~~~mysql
SELECT * FROM person_info WHERE name > 'Asa' AND name < 'Barlow';
~~~

分为两个步骤：

1.从索引取出name的范围

2.根据范围进行回表查询(因为索引不是所有的字段，还需要找country)

第一步因为 name从  `Asa`到`Barlow`之间的记录在磁盘是相连的，是顺序`IO`

第2步 因为需要回表找聚簇索引这些是按照 id进行排序的，是不连续的是 随机`IO`

**需要回表的次数越多，二级索引的性能越低下**，如果需要的过多性能就不如全表扫描

##  4.挑选索引











## 7.5 更好使用和创建索引

### 7.5.1 为搜索，连接，排序，分组创建索引

**创建索引的条件**

- Where
- 连接
- Order by
- Group by

### 7.5.2  考虑索引列不重复数多不多

如果重复数很多，那么二级索引+回表的消耗就会很多，所以必须考虑不重复数在所有记录中占的比例

### 7.5.3  索引需要尽可能小

索引尽可能小是为了一个数据页可以加载更多的用户记录，数据类型越小，数据页加载的记录越多，磁盘的I/O消耗越小（一次I/O可以加载更多的用户记录到磁盘）

### 7.5.4 索引可以取列的前缀进行匹配

如果一个字符串特别长，按照这个叶子节点存储这个字符串是很浪费空间的，我们可以取这个字符串的前缀作为索引列

但是如果取10字符为前缀，那么该索引无法对10前缀以后的字符串

### 7.5.5  覆盖索引

为了告别索引带来的性能消耗，我们尽可能查询列表只包含索引列，

![image-20220622171430826](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220622171430826.png)

只需要我们查询Key1和id的值，不需要通过主键id进行回表查询

### 7.5.6  索引列以列名的形式出现

![image-20220622171747462](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220622171747462.png)

**索引列不以列名的形式出现是无效的**

### 7.5.7 新插入记录对主键的影响

如果一个页满了，我们新插入记录，就必须要进行页分裂，这意味着性能消耗

### 7.5.8 冗余和重复索引

定位并且删除表的冗余和重复索引

### 总结

#### 1.B+树在时间和空间上都有代价

#### 2.B+树适用于以下情况

1. 全值匹配
2. 匹配最左边的列
3. 匹配范围值
4. 精确匹配某一列并范围匹配另外一列
5. 用于排序
6. 用于分组

#### 3.使用索引需要注意以下事项

1. 只为用于搜索、排序或分组的列创建索引;
2. 当列中不重复值的个数在总记录条数中的占比很大时 才为列建立索引
3. 索引列的类型尽量小
4. 可以只为索引列前缀创建索引，以减小索引占用的存储空间
5. 尽量使用覆盖索引进行查询，以避免因表操作带来 性能损耗;
6. 在表达式中，让索引列以列名的形式单独出现在搜索条件中，
7. 为了尽可能少地让聚簇索引发生 **列分页** 的情况 建议让主键拥有 AUTO CREMENT属性，
8. 定位并删除表中的冗余和重复索引

# 10.单表的访问方法



## 1.访问方法概念

1.全表扫描查询

2.索引查询

- 主键或唯一二级索引的等值查询
- 针对普通二级索引的等值查询
- 针对索引列的范围查询
- 直接扫描整个索引

## 2.const

> 对于二级索引匹配常数，InnoDB把它定义为`const`

例如：

~~~mysql
SELECT * FROM single_table WHERE key2 = 3841;
~~~

![image-20221025112144233](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221025112144233.png)

## 3.ref

~~~mysql
SELECT * FROM single_table WHERE key1 = 'abc';
~~~

> 因为 key1我们加的是普通索引，所以回表的时候可能找出 **多列匹配的值**，所以我们定义为`ref`, 

~~~mysql
SELECT * FROM single_table WHERE key2 IS NULL;
~~~

> 上述的虽然key2是唯一索引,但是为空处理可以有多个NULL值，所以也是 `ref`

> ps: 注意 如果左侧连续的不是等值标胶的话，访问方法就不能称为`ref`了，比如

~~~mysql
 SELECT * FROM single_table WHERE key_part1 = 'god like' AND key_part2 > 'legendary';
~~~

## 4.ref_or_null

> 有时候我们不仅想找出某个二级索引列的值等于某个常数的记录，还想把值为`NULL`的记录

~~~mysql
SELECT * FROM single_demo WHERE key1 = 'abc' OR key1 IS NULL;
~~~

> 当使用二级索引而不是全表扫描的方式执行该记录，这种类型的访问方法就称为`req_of_null`

![image-20221025133740619](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221025133740619.png)

## 5.range

> 如果是对于范围，我们用`二级索引+回表`的方式执行，这中方法就叫做`range`

~~~mysql
SELECT * FROM single_table WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79);
~~~

## 6.index

~~~mysql
SELECT key_part1, key_part2, key_part3 FROM single_table WHERE key_part2 = 'abc';
~~~

> `key_part2`不是联合索引`idx_key_part`的最左索引列 ，
>
> 但是这个语句符合以下两个条件

- 查询列只有散列 `key_part1`,`key_part2`,`key_part3` ，索引`idx_key_part`包括这三列
- 搜索条件只有`key_part2`列，这个列包括于`index_key_part`

> 我们可以直接遍历`idx_key_part`进行比较，由于二级索引比聚簇索引小得多，而且这个过程也不需要回表，这种方式就叫做`index`

## 7. all

> 全表扫描，直接扫描全部聚簇索引

## 8.注意事项(没看)

# 11.连接的原理

## 1.简介

### 1.本质

**前置操作:**

~~~mysql
mysql> CREATE TABLE t1 (m1 int, n1 char(1));
Query OK, 0 rows affected (0.02 sec)
mysql> CREATE TABLE t2 (m2 int, n2 char(1));
Query OK, 0 rows affected (0.02 sec)
mysql> INSERT INTO t1 VALUES(1, 'a'), (2, 'b'), (3, 'c');
Query OK, 3 rows affected (0.00 sec)
Records: 3 Duplicates: 0 Warnings: 0
mysql> INSERT INTO t2 VALUES(2, 'b'), (3, 'c'), (4, 'd');
Query OK, 3 rows affected (0.00 sec)
Records: 3 Duplicates: 0 Warnings: 0
~~~

### 2.连接过程

![image-20221025141345355](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20221025141345355.png)

### 3.内连接和外连接







# 18 事务

## 18.1 事务起源

### ACID

- **原子性**（Atomicity）
- **隔离性**(Isolation)
- **一致性**(Consistency)
- **持久性**(Durability)

## 18.2 事务的概念

![image-20220623142123979](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220623142123979.png)

只有事务到**提交的状态**或者 **终止的状态**这个事务的生命周期才算是真正的完成了， 处于提交状态的事务会永久生效，处于中止状态的事务会回滚到事务开始之前的状态

### 开始

**BEGIN[WORK]**

BEGIN代表开启事务，后面的WORK可有可无

**START TRANSACTION**

和BEGIN相似，也代表开启一个事务，不过后面可以跟修饰符

- **READ ONLY**   只读
- **READ WRITER** 读写
- **WITH CONSISTENT SNAPSHOT**:一致性读

START TRANSACTION后面可以跟多个修饰符(用,分隔)

~~~mysql
START TRANSACTION READ ONLY,WITH CONSISTENT SNAPSHOT
~~~

一个事务不能既设置为**只读的**，也设置为**读写的**‘’

### 提交

~~~mysql
COMMIT[WORK]
~~~

### 终止事务

如果在写事务的过程中发现事务写错了，我们可以启动回滚

~~~mysql
ROLLBACK[WORK];
~~~



## 18.3 事务用法

### 5 自动提交

autocommit会把每一条语句看做一个独立的事务，如果不想autocommit这样有两种办法

- 显式的使用START TRANSACTION和COMMIT
-  设置autocommit为off  

~~~mysql
set autocommit = off
~~~

### 6 隐式提交

- **使用数据库操纵语句(DDL)** 比如ALTER TABLE 这些
- **隐式使用或者修改表的语句** 比如 ALTER TABLE RENAME TABLE
- **这个事务没有执行完，又开启了一些事务**，那么这些事务会自动提交
- **如果出现了数据方面的复制的语**句，START SLAVE
- **出现数据加载语句**   LOAD DATA
- **其他语句** ANALYZE TABLE LOAD TABLE RESET

### 7 保存点

如果写了很多代码，但是有一个出错了，就ROLLBACK 会出现一夜回到解放前的感觉

~~~mysql
SAVEPOINT 节点名称;
~~~

**回滚到保存点**

~~~mysql
ROLLBACK TO 保存点名称;
~~~

**删除保存点**

~~~mysql
RELEASE SAVEPOINT 节点名称;
~~~

# 21 事务隔离记录别和MVCC

## 1  事前准备

~~~mysql
create table hero(
	number int,
	name VARCHAR(100),
	country VARCHAR(100),
	PRIMARY KEY (number)
)
~~~

~~~mysql
insert into  hero values(1,"刘备","蜀");
~~~

## 2 事务隔离级别

我们要向保证事务的隔离性，需要用串行的操作，但是这样太耗费资源了，所以我们，**可以在事务访问某个数据的时候，对其他同样想要访问这个数据的事务进行限制**， 这种执行方式叫做可串行化执行

### 1 事务并发遇到一致性问题

#### 脏写(Dirty Write)

脏写可能影响到事物的原子性

![image-20220624140629504](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220624140629504.png)

现在的问题是如果T1终止需要对T2已经做完的事务进行回滚，这影响到事务的原子性，如果对T2全部回滚，那么影响到事务的持久性

#### 脏读（Dirty Read）

**如果一个事务读到了另一个未提交事务修改过的数据**，意味着脏读

Pl : wl [xJ.. . r2(xl. . . ((cl 0[" al) and (c2 or a2) in any order) 

w1[x=1]r2[x=1]r2[y=0]c2w1[y=1]c1

T2是只读事务，依次读取x和y的值，但是由于T2读取数据项x是未提交T1修改过的，所以T2最后x=1,y=0但是真正的数据最后是x=1,y=1

**脏读的另外一种解释**

T1先修改x的值，T2读取未提交事务的T1，然后T1中止，T2提交，T2读到了一个不存在的值

#### **不可重复读（Not-Repeatable Read）**

事务修改了另外一个未提交事务的数据， 就意味着发生不可重复读，或者模糊读现象

对P2的操作执行序列

**不可重复读的广义解释**

~~~bash
P2: r1[x]... w2[x]... ((a1 or c1) and (a2 or c2))
~~~

![image-20220624143944533](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220624143944533.png)

T1是一个只读事务，第一读x=0,然后x=1,y=1，最后T1读的时候y=1 虽然没有发生脏写，脏读，但是T1得到了不一致的状态



**不可重复度的严格解释**

~~~bash
A1:r1[x]...w2[x]...c2...r1[x]...c1
~~~

T1先是读了x，然后T2修改了未提交事务T1的值x，然后T1又读了x,发生前后不一致的现象

#### 幻读（Phantom）

**广义解释:** 如果一个事务先根据某些查询条件查询一些记录，该事务未提交时候，另外一个事务插入了一些符合查询条件的记录（INSERT,DELETE,UPDATE），这就出现了幻读现象

~~~bash
P3：r1[P]...w2[y in P]...((c1 or a1) and (c2 or a2))
~~~

**幻读现象也可能引发一致性问题**

~~~bash
r1[P]w2[insert y to P]r2[z=3]w2[z=4]c2r1[z=4]c1
~~~

**严格解释**：

~~~bash
A3:r1[P]...w2[y in P]...c2...r1[P]...c1
~~~

> 幻读强调的是在多次读取一条记录，在后读取却读取到了没有读到的记录，这些之前读取的时候不存在的记录被称为幻影记录

假设T1读取P了一些记录，然后T2删除了一些符合搜索条件P的记录，T1再次搜索获取不同的结果集，这种也称为不可重复读现象

### 2 SQL四种隔离级别

我们遇到一些现象，这些现象会对事务一致性产生不同影响

> 脏写>脏读>不可重复读>幻读

**"舍弃一部分隔离性来换取性能的体现"**：四个隔离级别的设定

- **READ UNCOMMITED**:读未提交
- **READ COMMITED**:读已提交
- **REPEATABLE READ**:可重复读
- **SERIALIZABLE**:可串行化

| 隔离界别        | 脏读   | 不可重复读 | 幻读   |
| --------------- | ------ | ---------- | ------ |
| READ UNCOMMITED | 可能   | 可能       | 可能   |
| READ COMMITED   | 不可能 | 可能       | 可能   |
| REPEATABLE READ | 不可能 | 不可能     | 可能   |
| SERIALIZABLE    | 不可能 | 不可能     | 不可能 |

**为什么没有脏写**,脏写对这个现象影响太严重了，无论是哪个隔离级别，都不允许脏写的存在

### 3 MySQL的隔离级别

~~~mysql
SET [GLOBAL|SESSION]TRANSACTION ISOLCATION LEVEL level;
~~~

使用GLOBAL关键字(全局范围产生影响)

- 只对执行完该语句的新会话产生作用
- 已经存在的会话无效

使用SESSION关键字(会话产生影响)

- 对当前会话所有的后续事务生效
- 可以在已经开始的事务运行，但是不会影响该事务的执行
- 如果在事务间执行，对后续的事务有效

查看会话默认隔离级别

~~~mysql
SHOW VARIABLES LIKE 'transaction_isolation'
#或者
SELECT @@transaction-isolation
~~~

![image-20220624161335716](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220624161335716.png)

## 3. MVCC原理

#### 1.MVCC版本链

对于InnoDB存储引擎来说，居村索引包含下面两个必要的隐藏列

聚簇索引包含下面两个必须的隐藏列(row_id不是必须的，创建的表含有主键时或有有不为空的UNIQUE时候,不包含row_id列)

- trx_id： 一个事务对聚簇索引列进行修改时，都会把这条事务的id赋值给trx_id隐藏列
- roll_pointer:  每次对聚簇索引进行改动的时候，都会把旧的版本写道undolog中，这个roll_pointer就相当于一个指针，找到修改前的记录

![image-20220624163405835](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220624163405835.png)

> 实际上insert undo 只在事务回滚的时候起作用，当事务提交之后,该类型的undo 日志就失去效果， 但是roll_pointer的值不会消除
>
> roll_pointer占7字节，第一个字节表示undo日志类型,如果为1，表示执行
>
> TRX_UNDO_INSERT,也是insert undo日志

![image-20220624164742196](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220624164742196.png)

> 为了阻止脏写现象(两个事务交叉更新同一条数据),InnoDB使用了锁

每次记录一条改动都会生成一个undo日志，每条undo日志都会有一个roll_pointer属性，Insert对应的undo日志也有该属性

![image-20220624165415370](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220624165415370.png)

> 每次更新记录后，都会把旧的记录放在undo日志中,随着更新次数变多，所有版本都会被roll_pointer连接成链表，这个链表称为版本链
>
> 每个版本中都有该版本的事务id
>
> 我们之后会用该版本链来控制并发事务访问相同记录的行为，**这种机制称为多版本并发控制(Multi-Version Concurrency Controller,MVCC)**

我们指导，在UPDATE操作产生的undo日志中，  **只会记录索引列和被更新的列的信息**，对于trx_id=80的列来说本来是没有country信息，

**问题：**我们怎么知道该版本的country的值， **没有更新该列说明和上一个版本的值相同**

#### 2. ReadView

- READ UNCOMMITED : 可以读到未提交的事务，直接读取
- SERIALIZABLE:InnoDB采用加锁的方式访问记录
- READ  COMMITED和REPEATABLE READ:保证已经提交事务修改的记录

**核心问题：**哪些版本链是当前事务可见,InnoDB提出了ReadView("一致性视图")

- m_ids:在生成ReadView,当前系统活跃的读写事务的id列表
- min_trx_id:生成ReadView，当前系统活跃的读写事务最小id，是m_ids的最小值
- max_trx2_id:生成ReadView，系统应该分配给下一个事务的事务id值

> max_trx_id不是m_ids的最大值，事务id是递增分配的，下面有id为1,2,3的事务，如果事务3提交了，新的读事务生成ReadView,m_ids包括1，2，max_trx_id = 4

- creator_trx_id：生成ReadView事务的事务id

> 只有对表中记录进行改动才会分配唯一事务id值(Insert,Delete,Update) , 否则事务id默认值为0

**ReadView判断版本是否可见**

- 如果被访问的版本trx_id和ReadView和creator_trx_id相同,**意味着当前的事务在访问他自己修改过的记录，**所以该版本可以被当前事务访问

- 被访问的trx_id小于min_trx_id，说明生成版本前ReadView已经提交，可以访问

- 被访问的trx_id大于或等于ReadView中的max_trx_id，说明该版本事务在当前事务生成ReadView后才开启， 所以这个版本不允许电气概念事务访问

- 如果被访问的trx_id在min_trx_id和max_trx_id之间，则需要看trx_id是否在m_ids中

  如果在，说明创建ReadView时候该版本还是活跃的，该版本不可以被访问

  如果不在,创建ReadView该版本已经提交，可以被访问

如果某个版本对当前事务不可见，就顺着版本链找下一个版本的数据，并继续执行上面的步骤来判断记录可见性，如果记录最后一个版本也不可见，说明这条记录对当前事务完全不可见

##### **READ COMMITED ——每次读取数据都会生成一个ReadView**



![image-20220624194114280](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220624194114280.png)

> **第一次**真正修改记录(Insert,Delete,Update)才会分配唯一事务id，而且事务id是递增的，Transaction 200更新别的表记录是为了分配事务id

~~~mysql
#使用 READ COMMITED隔离级别的事务
BEGIN;
#SELECT1:Transaction 100 ,200
SELECT * FROM hero WHERE number =1; 
~~~

**SELECT的执行过程**

**步骤1**:在执行SELECT生成ReadView文件

- m_ids ->[100,200]
- min_trx_id :100
- max_trx_id: 201
- creator_trx_id : 0

**步骤2**:顺着版本链挑选可见的记录. name = ‘张飞’,版本的trx_id值为100,在m_ids列表(是活跃的，不符合要求),根据roll_pointer看下一个版本

**步骤3**  下一个版本name='关羽',trx_id=100,不符合要求

**步骤4**  下一个版本name = '刘备', trx_id=80，小于ReadView中的min_trx_id,符合要求，最后返回给用户的就是 name = '刘备'的记录

##### **REPEATABLE READ——第一次读取数据生成一个ReadView**

> 对于RepeatableRead来说，只会在第一次查询数据生成ReadView，之后就不会生成ReadView了



#### 3.二级索引和MVCC

只有聚簇索引有trx_id 和roll_pointer隐藏列

1.二级索引有个Page Header部分有个名为PAGE_MAX_TRX_ID的属性，每次进行CRUD都需要先判断min_trx_id>PAGE_MAX_TRX_ID，如果是则保证所有记录对ReadView可见，不是就步骤2

2.用二级索引进行徽标操作，找到对ReadView可见的第一个版本

#### 4.MVCC小结

**READ UNCOMMITED**:可以直接读为提交事务的记录

**SERIALIZABLE**:通过加锁的形式进行反问事务记录

**READ COMMITED和REPEATABLE READ**:需要用到ReadView（一致性视图）

**RepeatableRead和ReadCommited的区别**：

- READ COMMITED每次查询的时候都会创建ReadView
- REPEATABLE READ只会在第一次查询的时候创建ReadView(之后的查询都是复用前面的ReadView)

> 20章讲到过，如果执行delete或者update操作，并不会将对应的记录(聚簇索引和二级索引)从页面删除掉而是会标记一个 **delete_mark**，这就是为MVCC服务的 ——>
>
> 如果级别是REPEATABLE READ，同时有事务T1,T2，  T1刚开始根据某些搜索条件读取一条记录，然后T2删除，接着T1又读，就再也读不到该记录了

截止目前，我们遇见的SELECT都是普通SELECT，MVCC只对普通SELECT生效

## 5.总结

- **脏写**： 一个事务修改了另一未提交事务修改的数据
- **脏读**： 一个事务读到了另一未提交事务修改的数据
- **不可重复读**:一个事务修改了另一未提交事务的读取的数据
- **幻读**： 一个事务先根据搜索条件查询一些记录，然后另一个事务写入一些符合搜索条件的记录



| 隔离界别        | 脏读   | 不可重复读 | 幻读   |
| --------------- | ------ | ---------- | ------ |
| READ UNCOMMITED | 可能   | 可能       | 可能   |
| READ COMMITED   | 不可能 | 可能       | 可能   |
| REPEATABLE READ | 不可能 | 不可能     | 可能   |
| SERIALIZABLE    | 不可能 | 不可能     | 不可能 |



设置事务的隔离级别

~~~mysql
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level;
~~~

# 22 工作面试老大难——锁

## 1 解决并发事务的两种基本方式

### 1.写——写情况

写写是一种脏写情况，任何一个隔离级别都不允许该现象发生

在多个未提交事务相继提交一条记录，需要让他们排队执行。 **这种排队过程实际上是加锁实现的**

**这种"锁"本质是一个内存结构**

**锁结构主要信息**

- trx信息——表示和那些事务关联
- is_waiting：当前事务是否在等待

![image-20220626165022314](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220626165022314.png)

**关于锁的几种说法**

**获取锁成功**：内存产生了对应锁结构，锁结构的is_waiting=false,表示事务可以继续执行

**获取锁失败或者加锁失败或者没有获取到锁**:内存中生成锁结构，但是锁的is_waiting=true

**不加锁**:   不需要内存生成对应锁，直接执行操作 (不包括加隐式锁的情况)

### 2  读写——写读

**读写——写读情况会出现脏读，不可重复读，幻读情况**

MySQL的REPEATABLE READ在很大程度上避免了幻读的情况

> ReadView本身保证不可以读取到为提交事务的更改，避免了脏读现象

> 不可重复读是，当前事务先读取了一条记录，然后另一个事务对当前记录进行修改
>
> 幻读是当前事务读取若干条相关记录，然后插入了若干条相关记录

### 3.一致性读

事务利用MVCC进行读取称为一致性读(快照读)

### 4. 锁定读

#### 1.共享锁和独占锁

加锁解决问题的时候，既需要读-读情况不受影响，又要写-读，写-写，读-写情况相互阻塞，所以InnoDB进行一下分类

- **共享锁(Shared Lock)**:S锁，事务读取需要获取S锁
- **独占锁**:X锁

| 兼容性 | S锁    | X锁    |
| ------ | ------ | ------ |
| S锁    | 兼容   | 不兼容 |
| X锁    | 不兼容 | 不兼容 |

#### 2.锁定读

我们如果想读取记录就获取X锁，防止其他事务进行读写，我们把这种记录方式成为锁定读

- 为读取记录加S锁

~~~mysql
SELECT ... LOCK IN SHARE MODE;
~~~

这条事务执行该语句，允许其他事务获取S锁，不允许获取X锁

- 对读取事务加X锁

~~~mysql
SELECT ... FOR UPDATE
~~~

- DELETE:先定位到这个节点在B+树的位置，然后deletemark
- 

## 2.多粒度锁

  给表加的锁分为共享锁（S锁）和独占锁(X 锁)

**给表加S锁**

- 别的事务可以获得该表的S锁和该表中记录的S锁
- 不可以获得该表的X锁和表中记录的X锁

**给表加X锁**

- 别的事务不可以获得该表的X锁或某些记录的X锁
- 别的事务不可以获得该表的S锁或某些记录的S锁

**意向共享锁**：IS锁，事务准备在某条记录加S锁，需要现在表级别加IS锁

**意向独占锁**：IX锁，事务准备在某条记录加X锁，需要现在表级别加IX锁

| 兼容性 | S      | X      | IS     | IX     |
| ------ | ------ | ------ | ------ | ------ |
| S      | 兼容   | 不兼容 | 兼容   | 不兼容 |
| X      | 不兼容 | 不兼容 | 不兼容 | 不兼容 |
| IS     | 兼容   | 不兼容 | 兼容   | 兼容   |
| IX     | 不兼容 | 不兼容 | 兼容   | 兼容   |

## 3.MySQL的行级锁和表级锁

### 2.InnoDB引擎中的锁

#### **1.InnoDB表级别的锁**

对表执行 **SELECT INSERT DELETE UPDATE**，InnoDB不为给这个表添加S锁或X锁

对一个表执行DDL（ALTER TABLE,DROP TABLE）会发生阻塞。

- LOCK TABLES t READ :InnoDB 引擎会对表t加表级别的S锁
- LOCK TABLES t WRITE:InnoDB 引擎会对表t加表级别的X锁

**表级别的AUTO-INC锁**

是在AUTO_INCREMENT之前给列上的锁，其他事务插入语句都要被阻塞，为了保证事务分配的递增值是联系的

> 这个AUTO-INC是作用于单个插入语句，在插入语句执行之后就被释放了

#### 2.InnoDB行级锁

- **Record Lock**

正经记录所是有S锁和X锁的区分，我们分为S型正经记录锁，X型正经记录锁，

- 加S锁，其他事务可以获取S锁，不能获取X锁
- 加X锁，其他事务既不能获取S锁，也不能获取X锁



- **Cap Lock**

MySQL在REPETABLE READ可以很大程度解决幻读

解决方案：

- MVCC解决
- 加锁解决

![image-20220629220223244](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220629220223244.png)

gap可以解决幻影记录的问题，我们把gap插入number=8的前面,这就代表(3,8)不允许主键插入，我们如果想插入主键为4的记录进来，也就不允许了

**gap锁的提出仅仅是为了解决幻影记录问题,对一个事务加gap锁，不会限制其他事务加gap锁或者正经记录锁**



- **Next-Key Lock**

我们既想锁住记录，又想阻止其他事务在该记录前的间隙插入新纪录

next-key锁的本质就是正经记录锁和gap锁的结合，**既能保护这个记录，又能阻止其他事务将新纪录插入到被保护记录前面**

- **Insert Intention Lock**

一个事务在插入一条记录，需要判断插入位置是否被别的事务加了gap锁， 如果有的话，插入需要等待，插入等待也需要在内存中生成锁结构

![image-20220630143337667](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220630143337667.png)



- **隐式锁**

一个事务对新插入的记录可以先不显式的加锁(生成一个锁结构),但是由于事务id这个角色，相当于加了一个隐式锁，别的事务对这个记录加S锁或者X锁的时候，由于隐式锁的存在，会先给当前记录生成锁结构，然后自己生成锁结构然后进入等待状态

### 3.InnoDB的内存结构

InnoDB对记录加锁不可能1000记录加1000个锁，所以对下面的情况，这些记录的锁可以放在一个锁结构里面：

- 在同一事务中进行加锁操作
- 被加锁的记录记录在同一个页面
- 加锁的类型是一样的
- 等待状态是一样的

![image-20220630153837528](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220630153837528.png)

- 锁所在的事务信息： 无论是行级锁还是表级锁，这里记载事务信息

- 索引：对于行级锁，需要记录加锁的位置属于哪个索引

- **表锁/行锁信息**

  - 表级锁记录的是对哪个表加的锁
  - 行级锁：
    - SpaceID:记录表空间
    - **Page Number**：记录所在页号
    -  **n_bits**: 一个页面包含很多记录，一条记录对应一个比特，用不同比特来区分是哪条记录加锁
  
- type_mode : 32bit数，包括lock_mode,lock_type,sec_lock_mode![image-20220630155513915](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220630155513915.png)

 lock_mode 占用低4比特

- LOCK_IS（十进制0）：表示共享意向锁，IS
- LOCK_IX(十进制1)：表示独占意向锁，IX
- LOCK_S(十进制2)：表示共享锁，S
- LOCK_X(十进制3)：表示独占锁X
- LOCK_AUTO_INC(十进制的4)：AUTO_INC锁

> InnoDB中，LOCK_IS,LOCK_IX,LOCK_AUTO_INC是表级锁，
>
> LOCK_S,LOCK_X既可以是表级锁，也可以是行级锁

lock_type（锁类型）占5-8位，目前只占用第5比特和第6比特

- **LOCK_TABLE(十进制的16)**:表示表级锁

- LOCK_REC：表示行级锁

`rec_lock_type`（行锁的具体类型），只有LOCK_REC ，才会细分出更多的类型

- `LOCK_ORDINARY`(十进制的0)：表示next-key锁
- `LOCK_GAP`（十进制的512）:表示gap锁
- `LOCK_REC_NOT_GAP`:(十进制的1024):11比特为1，表示正经记录锁
- `LOCK_INSERT_INTENTION`:(十进制的2048):12比特为1，表示插入意向锁
- `LOCK_WAIT`:(十进制的256):第9比特设置为1，表示is_waiting设置为true

​    ![image-20220823215402655](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220823215402655.png)

### 4.语句加锁分析

#### 1.普通SELECT语句

![image-20220824082346648](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220824082346648.png)

> 在 `RR`隔离级别下，T1执行普通`SELECT`生成一个`ReadView`,T2向hero插入记录并提交，`ReadView`不能阻止T1进行UPDATE和DELETE对这个记录改动，但是这样 新记录的 trx_id变成了T1事务的id

![image-20220824090917324](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220824090917324.png)

  

####   2.锁定读

![image-20220824095558515](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220824095558515.png)



  

  

  

  
