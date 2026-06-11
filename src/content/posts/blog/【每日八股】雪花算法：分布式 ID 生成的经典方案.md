---
title: 【每日八股】雪花算法：分布式 ID 生成的经典方案
published: 2026-06-11
updated: 2026-06-11
description: 雪花算法
image: https://tu.ztyukino.com/file/QIJn9Crp.jpeg
tags:
  - 八股
  - Java
category: 后端开发
draft: false
author: yukino
---
在单体系统里，我们通常可以直接使用数据库自增主键生成 ID。但在分布式系统中，多个服务实例、多个数据库分片同时写入数据，如果还依赖单点自增 ID，就会遇到性能瓶颈和 ID 冲突问题。

雪花算法，也就是 Snowflake，就是为了解决这个问题而出现的一种分布式 ID 生成方案。它最早由 Twitter 提出，核心思想是： **用一个 64 位整数，把时间戳、机器编号和序列号组合起来，从而在不同机器上并发生成全局唯一、趋势递增的 ID。**

## 一、为什么需要分布式 ID

在业务系统中，ID 通常有几个基本要求：

> 全局唯一  
> 高性能生成  
> 尽量有序  
> 方便数据库索引  
> 不依赖中心节点

数据库自增 ID 虽然简单，但在分库分表、微服务、多实例部署时会出现问题：

> 多个数据库自增 ID 可能重复  
> 集中式 ID 服务可能成为瓶颈  
> UUID 太长且无序，不适合作为数据库主键

雪花算法的优势就在于：它可以在本地内存中生成 ID，不需要每次访问数据库或 Redis，性能非常高。

## 二、雪花算法的 ID 结构

经典 Snowflake 使用 64 位 long 类型表示一个 ID，结构大致如下：

> 0 | 41位时间戳 | 10位机器ID | 12位序列号

各部分含义是：

> 最高位 1 位：  
> 固定为 0，保证生成的是正数。
> 
> 时间戳 41 位：  
> 通常表示当前时间相对于某个起始时间的毫秒差。  
> 41 位毫秒时间大约可以使用 69 年。
> 
> 机器 ID 10 位：  
> 用于区分不同机器或不同服务实例。  
> 10 位最多支持 1024 个节点。
> 
> 序列号 12 位：  
> 同一毫秒内的自增序列。  
> 12 位最多支持每毫秒 4096 个 ID。

所以一个雪花 ID 大概是这样生成的：

> ID = 时间戳部分 << 机器ID位数+序列号位数  
> | 机器ID << 序列号位数  
> | 序列号

## 三、生成流程

雪花算法的生成流程可以概括为：

> 1\. 获取当前毫秒时间戳  
> 2\. 判断是否和上一次生成 ID 的时间戳相同  
> 3\. 如果是同一毫秒，序列号加 1  
> 4\. 如果序列号超过上限，等待下一毫秒  
> 5\. 如果是新的毫秒，序列号重置为 0  
> 6\. 拼接时间戳、机器 ID、序列号，得到最终 ID

伪代码如下：

```java
public synchronized long nextId() {

    long timestamp = currentTimeMillis();

 

    if (timestamp < lastTimestamp) {

        throw new RuntimeException("Clock moved backwards");

    }

 

    if (timestamp == lastTimestamp) {

        sequence = (sequence + 1) & sequenceMask;

        if (sequence == 0) {

            timestamp = waitNextMillis(lastTimestamp);

        }

    } else {

        sequence = 0;

    }

 

    lastTimestamp = timestamp;

 

    return ((timestamp - epoch) << timestampShift)

            | (workerId << workerIdShift)

            | sequence;

}
java运行
```

## 四、雪花算法的优点

雪花算法最大的优点是快。

它不依赖数据库、不依赖 Redis，ID 生成只发生在本地内存中，所以性能非常高。

同时，它生成的 ID 具备趋势递增特性。虽然不是严格全局递增，但在同一个节点上基本按时间递增，这对数据库索引比较友好。

相比 UUID，雪花 ID 更短，通常是一个 long 类型数字，适合作为数据库主键。

它的典型优势是：

> 高性能  
> 全局唯一  
> 趋势递增  
> 可反推出生成时间  
> 适合分库分表  
> 适合高并发写入

## 五、雪花算法的问题

雪花算法并不是没有缺点。它最大的问题是依赖系统时钟。

如果服务器时间发生回拨，比如 NTP 校时导致当前时间小于上一次生成 ID 的时间，就可能出现 ID 重复风险。

常见处理方式有几种：

> 1\. 直接抛异常，拒绝生成 ID  
> 2\. 等待时钟追上来  
> 3\. 使用备用 workerId  
> 4\. 引入逻辑时钟  
> 5\. 对小幅回拨进行短暂 sleep

例如，如果回拨时间很短，可以等待几毫秒：

```java
if (timestamp < lastTimestamp) {

    long offset = lastTimestamp - timestamp;

    if (offset <= 5) {

        Thread.sleep(offset);

        timestamp = currentTimeMillis();

    } else {

        throw new RuntimeException("Clock moved backwards too much");

    }

}
java运行
```

如果回拨时间太长，就不应该继续生成 ID，否则可能破坏唯一性。

## 六、机器 ID 怎么分配

雪花算法要求每个节点的 workerId 不同。

如果两个服务实例使用了相同的 workerId，又在同一毫秒生成了相同序列号，就可能生成重复 ID。

常见分配方式有：

> 配置文件手动指定  
> 根据机器 IP/MAC 生成  
> 启动时从 Redis/Zookeeper 注册  
> Kubernetes 环境下用 Pod 序号  
> 数据库维护 workerId 分配表

生产环境中，不建议完全依赖随机数生成 workerId。随机数虽然简单，但节点多了之后存在冲突概率。

## 七、适合的业务场景

雪花算法适合这些场景：

> 订单 ID  
> 帖子 ID  
> 评论 ID  
> 消息 ID  
> 日志 ID  
> 分库分表主键  
> 高并发写入业务主键

比如发帖系统中，每篇帖子都需要一个全局唯一 ID。如果使用雪花算法，服务实例可以直接在本地生成 ID，然后写入数据库，不需要等待数据库返回自增主键。

> 用户发布内容  
> \-> 服务本地生成 postId  
> \-> 写入 knowpost 表  
> \-> 返回 postId 给前端

这对高并发写入非常友好。

## 八、总结

雪花算法是一种经典的分布式 ID 生成方案。它通过时间戳、机器 ID 和序列号组合出一个 64 位整数，在保证全局唯一的同时，也具备很高的生成性能和较好的数据库索引友好性。

它的核心价值可以总结为一句话：

**用本地计算代替中心化发号，在高并发分布式环境下生成趋势递增的全局唯一 ID。**

但在使用雪花算法时，一定要重点关注两个问题：

> 机器 ID 不能冲突  
> 系统时钟不能大幅回拨

只要这两个问题处理好，雪花算法就是一个非常实用、稳定、性能优秀的分布式 ID 方案。