# Nginx

## 1.什么是Nginx



> **Nginx(engine X)是一个高性能的HTTP和反向代理web服务器**，同时提供了**IMAP/POP3/SMTP**服务。Nginx是由伊戈尔·赛索耶夫为俄罗斯访问量第二的Rambler.ru站点开发的，第一个版本在2004年，2011年6月1日，nginx发布

>  Nginx是安装简单，配置文件简洁，Bug少的服务。Nginx启动容易，并且可以7*24不间断运行

> Nginx由C语言协程，官方测试并发数可高达50000  

## 2.Nginx作用

### 1.正向代理

![image-20220704185433440](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704185433440.png)

>  **正向代理**:为了从原始服务器获取内容，客户端向代理服务器发送请求并且确定目标(原始服务器)，然后代理将原始服务器转交请求并将内容返回给客户端
>
> 比如：你想访问外网，就需要VPN然后联络外部服务器然后让外部服务器处理请求并且返回给你结果

### 2.反向代理

![image-20220704185840632](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704185840632.png)

> 正向代理是代理客户端的，反向代理是代理服务端的
>
> 正向代理：代理的是客户端，服务端不知道响应请求是谁发出的
>
> 方向代理：代理的是服务端，客户端不知道是那一台服务器提供的服务

## 3.负载均衡

### 1.加权轮询

![image-20220704190401383](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704190401383.png)

>  加权轮询算法是：给不同的服务器划分权重，给服务器承受能力强的服务器划分权重比高一些

### 2.iphash

![image-20220704191246168](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704191246168.png)

## 4.动静分离

> 动静分离，我们有些请求是需要后台处理的，有些请求是不需要经过后台处理的(如:css,html,jps,js等文件)，这个不需要后台处理的文件叫做静态文件，**让动态网站的动态网页把不经常变的资源和经常变的资源区分开，拆分之后，我们可以根据静态资源将其做缓存操作，提高资源相应速度**

![image-20220704191656862](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704191656862.png)

## 5. Nginx常用命令

![image-20220704193203742](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704193203742.png)

![image-20220704193135802](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704193135802.png)

![image-20220704194915813](C:/Users/%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7/AppData/Roaming/Typora/typora-user-images/image-20220704194915813.png)

> 实战 

反向代理和负载均衡

![image-20220704203120460](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220704203120460.png)

## 6.一致性哈希

#### 应用场景

> 如果有3w张图片需要放在服务器0,1,2上，我们如果采用 **遍历**，效率和时间复杂度很长，我们不能接口，  我们可以通过 **Hash操作**直接求出Hash的键值，然后对取余的值进行判断再哪个服务器上

![image-20220822103843347](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220822103843347.png)



#### 试想

> 如果服务器数量不够，我们3台缓存服务器就需要变成4台，那我们刚才的 **Hash缓存是不是就失效了**，因为Hash需要重新计算，这就导致之前计算的缓存几乎全部都失效了

#### 问题

问题1：服务器数量发生变化，引起缓存的雪崩，大量缓存同一时间失效

问题2：服务器数量变化，缓存的位置几乎所有都会发生变化，如何减少缓存受到的影响

#### 解决方案：一致性哈希

**对服务器IP地址进行取模，对2^32次方进行取模，将2的^ 32的首尾相连想象成一个环**

![image-20220822105011860](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220822105011860.png)

> **计算服务器位置** 我们计算出A，B，C服务器ip对2^32取模 

![image-20220822105351208](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220822105351208.png)

> **计算服务器位置** 我们计算出A，B，C服务器ip对2^32取模 

##### 计算方式

> 从被缓存对象位置出发，顺时针找到遇到的第一个服务器

##### 优点

> 有效的解决大面积缓存失效的问题，如果服务器B被移除，我们直接在这个Hash环上剔除这个B就行

![image-20220822110133974](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220822110133974.png)

#### Hash环偏斜

##### 问题提出：

> 一致性哈希概念中，我们 **理想化**将3台服务器均匀映射到hash环中，但实际情况和现实往往不一样，如果是下图情况那就糟糕了

![image-20220822110421834](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220822110421834.png)

##### 虚拟节点

> 我们想尽量让这3个服务器节点均匀分布在这个环中，我们没有多余物理节点，但是我们可以在 **虚拟节点层面下功夫**，创建尽可能多的虚拟节点(实际是Hash环的复制品)，虚拟节点越多，hash环节点越多，均匀分布概率越大









