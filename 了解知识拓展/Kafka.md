# 一、Kafka概述

## 2.消息队列

### 1.传统消息队列场景

**消息队列的好处**

1.**解耦**

> 消息队列将处理过程分离出来，降低耦合性

2.**可恢复性**

> **系统部分组件失效并不影响整个系统**

3.**缓冲**

> 解决生产和消费速度不一致的问题

4.**灵活性&峰值处理能力**

> 访问量剧增情况下，我们需要进行削峰

5.**异步通信**

> 用户不需立即处理消息，消息队列提供了异步处理机制

### 2.消息队列的两种模式

#### 1.点对点模式(一对一，消费者主动拉取数据,消息接收后就清除)

> 一个producer对应一个consumer，如果新建一个consumer就得另外建立一个队列

![image-20220908144028827](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220908144028827.png)

#### 2.发布/订阅模式(一对多，消费者消费消息之后不会清除消息)

![image-20220908144213443](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220908144213443.png)

**两种情况**

1. 消费者主动拉取队列
2. 队列主动推送消息到consumer

### 3.基础架构

 ![image-20220908154320038](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220908154320038.png)

