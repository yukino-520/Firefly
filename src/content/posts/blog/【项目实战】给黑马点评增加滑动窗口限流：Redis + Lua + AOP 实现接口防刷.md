---
title: 【项目实战】给黑马点评增加滑动窗口限流：Redis + Lua + AOP 实现接口防刷
published: 2026-06-08
updated: 2026-06-08
description: 给黑马点评增加滑动窗口
tags:
  - Java
  - 项目实战
category: 后端开发
draft: false
author: yukino
---
**前言**

> 基础版黑马点评项目里，秒杀优惠券接口是一个高并发入口。虽然项目已经用了 Redis、Lua、分布式锁、异步下单等方案来保证库存安全，但如果短时间内大量请求直接打到秒杀接口，后端仍然会承受很大压力。
> 
> 所以我在项目中增加了一套基于 **Redis ZSet + Lua + Spring AOP** 的滑动窗口限流组件，用注解的方式对接口进行限流。

最终使用效果如下：

```java
@RateLimiter(

        window = 10,

        limit = 10,

        message = "秒杀活动太火爆，请稍后再试"

)

@PostMapping("seckill/{id}")

public Result seckillVoucher(@PathVariable("id") Long voucherId) {

    return voucherOrderService.seckillVoucher(voucherId);

}
```

表示： **10 秒内最多允许访问 10 次，超过后直接限流。**

## 为什么选择滑动窗口 ？

常见限流算法有：

> 固定窗口  
> 滑动窗口  
> 令牌桶  
> 漏桶

固定窗口实现简单，但边界问题明显。比如限制 10 秒 10 次，用户可以在第 9.9 秒请求 10 次，再在第 10.1 秒请求 10 次，短时间内实际打进来 20 次请求。

滑动窗口则会根据当前时间动态统计最近一段时间内的请求数，更平滑，也更适合秒杀、防刷这 类 场景。

具体可看我的另一篇博客 [《一篇文章搞懂常见限流算法：固定窗口、滑动窗口、漏桶和令牌桶》\_漏桶 令牌桶-CSDN博客](https://blog.csdn.net/2503_91308599/article/details/161208882?spm=1001.2014.3001.5501 "《一篇文章搞懂常见限流算法：固定窗口、滑动窗口、漏桶和令牌桶》_漏桶 令牌桶-CSDN博客")

## 实现思路

整体流程如下：

> 用户请求接口  
> ↓  
> AOP 拦截带 @RateLimiter 的方法  
> ↓  
> 读取注解中的 window、limit、type 等参数  
> ↓  
> 构造 Redis 限流 key  
> ↓  
> 执行 Lua 脚本  
> ↓  
> Redis ZSet 统计当前窗口内请求数量  
> ↓  
> 未超限：放行  
> 超限：抛出 RateLimitException

这里用 Redis ZSet 存请求记录：

> key：限流 key  
> score：请求时间戳  
> member：请求唯一标识

每次请求进来时：

> 1\. 删除窗口外的数据  
> 2\. 统计当前窗口内请求数  
> 3\. 如果数量小于限制，写入当前请求  
> 4\. 如果数量达到限制，返回 0 表示限流

## 第一步：定义限流注解

先定义一个 @RateLimiter 注解：

```java
@Target(ElementType.METHOD)

@Retention(RetentionPolicy.RUNTIME)

@Documented

public @interface RateLimiter {

 

    /**

     * 限流 key 前缀

     */

    String key() default "rate_limit:";

 

    /**

     * 时间窗口大小，单位秒

     */

    int window() default 10;

 

    /**

     * 时间窗口内允许的请求数

     */

    int limit() default 20;

 

    /**

     * 限流提示信息

     */

    String message() default "系统繁忙，请稍后再试";

 

    /**

     * 限流维度

     */

    LimitType type() default LimitType.METHOD;

 

    enum LimitType {

        IP,

        USER,

        METHOD

    }

}
java运行
```

这里我设计了三种限流维度：

> METHOD：按方法限流  
> IP：按用户 IP 限流  
> USER：按当前登录用户限流

比如秒杀接口可以按方法全局限流，也可以改成按用户限流，防止单个用户疯狂刷接口。

## 第二步：编写 Lua 脚本

Lua 脚本 放在：

> src/main/resources/limiter.lua

核心代码如下：

```java
local key = KEYS[1]

local window = tonumber(ARGV[1])

local limit = tonumber(ARGV[2])

local now = tonumber(ARGV[3])

 

if not window or not limit or not now then

  return redis.error_reply("Invalid input parameters")

end

 

window = window * 1000

 

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

 

local current = redis.call('ZCARD', key)

 

if current < limit then

  math.randomseed(now)

  local random = math.random(1000000)

  redis.call('ZADD', key, now, now .. '-' .. random)

  redis.call('EXPIRE', key, window / 1000)

  return current + 1

else

  return 0

end
java运行
```

为什么要用 Lua？

因为限流过程包含多个 Redis 操作：

> 删除过期请求  
> 统计当前数量  
> 写入当前请求  
> 设置过期时间

如果这些操作分多次执行，在高并发下可能出现并发安全问题。Lua 脚本在 Redis 中是原子执行的，可以保证整个限流判断过程不会被其他请求打断。

## 第三步：编写 AOP 切面

然后定义切面，拦截所有带 @RateLimiter 的方法：

```java
@Aspect

@Component

public class RateLimiterAspect {

 

    @Resource

    private StringRedisTemplate stringRedisTemplate;

 

    private static final DefaultRedisScript<Long> SLIDING_WINDOW_SCRIPT;

 

    static {

        SLIDING_WINDOW_SCRIPT = new DefaultRedisScript<>();

        SLIDING_WINDOW_SCRIPT.setLocation(new ClassPathResource("limiter.lua"));

        SLIDING_WINDOW_SCRIPT.setResultType(Long.class);

    }

 

    @Before("@annotation(rateLimiter)")

    public void doBefore(JoinPoint point, RateLimiter rateLimiter) {

        String key = rateLimiter.key();

        long window = rateLimiter.window();

        long limit = rateLimiter.limit();

 

        String fullKey = buildRateLimitKey(point, rateLimiter, key);

 

        Long result = executeSlidingWindowScript(fullKey, window, limit);

 

        if (result != null && result == 0) {

            throw new RateLimitException(rateLimiter.message());

        }

    }

 

    public Long executeSlidingWindowScript(String key, Long window, Long limit) {

        long now = System.currentTimeMillis();

 

        return stringRedisTemplate.execute(

                SLIDING_WINDOW_SCRIPT,

                Collections.singletonList(key),

                window.toString(),

                limit.toString(),

                Long.toString(now)

        );

    }

}
```

这里的重点是：

```java
@Before("@annotation(rateLimiter)")
```

它会拦截所有加了 @RateLimiter 注解的方法，并且可以直接拿到注解对象 rateLimiter，从而读取限流配置。

## 第四步：构造限流 Key

不同限流维度，对应不同 Redis key。

```java
private String buildRateLimitKey(JoinPoint point, RateLimiter rateLimiter, String baseKey) {

    StringBuilder keyBuilder = new StringBuilder(baseKey);

 

    MethodSignature signature = (MethodSignature) point.getSignature();

    Method method = signature.getMethod();

 

    keyBuilder.append(method.getDeclaringClass().getName())

            .append(":")

            .append(method.getName());

 

    switch (rateLimiter.type()) {

        case IP:

            keyBuilder.append(":ip:").append(getClientIp());

            break;

        case USER:

            keyBuilder.append(":user:").append(getCurrentUserId());

            break;

        case METHOD:

        default:

            break;

    }

 

    return keyBuilder.toString();

}
```

如果是方法级限流，key 大概是：

> rate\_limit:com.hmdp.controller.VoucherOrderController:seckillVoucher

## 第五步：定义限流异常

当 Lua 脚本返回 0 时，说明当前窗口内请求次数已经达到上限，直接抛出异常：

```java
public class RateLimitException extends RuntimeException {

    public RateLimitException(String message) {

        super(message);

    }

}
```

建议在全局异常处理器中单独处理这个异常：

```java
@ExceptionHandler(RateLimitException.class)

public Result handleRateLimitException(RateLimitException e) {

    return Result.fail(e.getMessage());

}
```

如果不单独处理，可能会被普通 RuntimeException 捕获，最后返回“服务器异常”， 用户体验 不太好。

## 第六步：在秒杀接口上使用

最后，在秒杀接口上加注解即可：

```java
@RestController

@RequestMapping("/voucher-order")

public class VoucherOrderController {

 

    @Resource

    private IVoucherOrderService voucherOrderService;

 

    @RateLimiter(

            window = 10,

            limit = 10,

            message = "秒杀活动太火爆，请稍后再试"

    )

    @PostMapping("seckill/{id}")

    public Result seckillVoucher(@PathVariable("id") Long voucherId) {

        return voucherOrderService.seckillVoucher(voucherId);

    }

}
```

这样，请求进入秒杀业务之前，会先经过限流切面。

如果 10 秒内请求次数小于 10 次，就正常进入秒杀逻辑。

如果超过 10 次，就直接返回：

> {  
> "success": false,  
> "errorMsg": "秒杀活动太火爆，请稍后再试",  
> "data": null,  
> "total": null  
> }

## 总结

这次给基础黑马点评 项目 增加的滑动窗口限流，核心由四部分组成：

> @RateLimiter 注解：声明限流规则  
> RateLimiterAspect 切面：拦截接口请求  
> Redis ZSet：记录请求时间  
> Lua 脚本：原子完成滑动窗口判断

它的优点是：

> 接入简单，一个注解即可使用  
> 限流粒度灵活，支持方法、IP、用户维度  
> Redis + Lua 保证高并发下判断准确  
> 适合秒杀、防刷、验证码等高频接口

相比基础版黑马点评，这个改造让系统在面对突发流量时多了一层保护。尤其是秒杀场景，限流可以把大量无效请求挡在业务逻辑之前，减少 Redis、数据库和 消息队列 的压力。