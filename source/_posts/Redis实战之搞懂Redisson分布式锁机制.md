---
title: Redis实战之搞懂Redisson分布式锁机制
date: 2024-01-30
---

### 一、简介
Redisson是什么？以下是官网的简介，相信各位彦祖们都能看懂，就不翻译了。
> Redisson is a Redis Java client with features of In-Memory Data Grid. It provides more convenient and easiest way to work with Redis. Redisson objects provides a separation of concern, which allows you to keep focus on the data modeling and application logic.

作为Redis广受欢迎的客户端，Redisson具有以下特性：

- 线程安全的实现；
- 支持多种Redis种模式，集群模式、哨兵模式、主从模式、单机模式等；
- 支持主动重连、失败自动重试；
- 丰富的数据类型：Object, Binary stream, BitSet, AtomicLong, Bloom filter, Map, Set, List, SortedSet, Queue, Deque等；
- 多样化的锁结构：Lock, FairLock, MultiLock, RedLock, ReadWriteLock, Semaphore, CountDownLatch等;
- 异步API实现：Asynchronous、RxJava3、Reactive Streams；
- Spring生态支持：Spring Cache、Spring Transaction API、Spring Data Redis、Spring Boot Starter、Spring Session；

### 二、使用方式
#### 1.基本使用
基本使用方法和API，官方文档有比较详细的描述，就不多加赘述了。
https://github.com/redisson/redisson/wiki/Table-of-Content
#### 2.与Spring Boot集成
在实例化Bean的方式上、配置文件格式上有些许区别，文档有具体介绍。
https://github.com/redisson/redisson/tree/master/redisson-spring-boot-starter

### 三、分布式锁
使用Redis来实现分布式锁，大家所熟知的SET NX命令，存在以下问题：
1. 不可重入：同一个线程无法多次获取同一把锁；
2. 不可重试：只能操作获取锁一次，失败无法重试；
3. 超时释放：如果业务耗时较长，超过失效时间会自动释放锁，此时其他线程可以获取到锁，破坏了唯一性(同一时间，只有一个线程获取到锁)；

从源码的角度，来分析一下Redisson是如何解决这些问题的。
#### 1.可重入机制
从 redissonClient的tryLock()方法源码一路跟踪，tryLockInnerAsync()方法的实现代码如下：
```
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {  
    return commandExecutor.syncedEval(getRawName(), LongCodec.INSTANCE, command,  
        "if ((redis.call('exists', KEYS[1]) == 0) " +  
        "or (redis.call('hexists', KEYS[1], ARGV[2]) == 1)) then " +  
        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +  
        "redis.call('pexpire', KEYS[1], ARGV[1]); " +  
        "return nil; " +  
        "end; " +  
        "return redis.call('pttl', KEYS[1]);",  
    Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));  
}

protected RFuture<Boolean> unlockInnerAsync(long threadId) {  
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,  
        "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +  
        "return nil;" +  
        "end; " +  
        "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +  
        "if (counter > 0) then " +  
        "redis.call('pexpire', KEYS[1], ARGV[2]); " +  
        "return 0; " +  
        "else " +  
        "redis.call('del', KEYS[1]); " +  
        "redis.call(ARGV[4], KEYS[2], ARGV[1]); " +  
        "return 1; " +  
        "end; " +  
        "return nil;",  
    Arrays.asList(getRawName(), getChannelName()),  
    LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId), getSubscribeService().getPublishCommand());  
}
```
这里用到了Lua脚本，判断key不存在或者存在且等于当前线程id时，使用了Hash结构来记录重入的次数，实现思路类似于ReentrantLock。而在解锁时，从unlockInnerAsync()方法中可以看到，对重入次数进行了减1操作，并且推送了订阅事件。因此可以得知进行了N次加锁后，需要操作N次解锁，才能释放锁。
#### 2.失败重试机制
在tryLock()方法中，返回值ttl为空时说明已经成功获取到锁，失败的情况会先判断是否已经超时，如果没有超时则会通过subscribe()方法进行订阅锁事件，当锁被释放时会再次尝试获取。
```
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();
    Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    // lock acquired
    if (ttl == null) {
        return true;
    }
    
    time -= System.currentTimeMillis() - current;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }

    current = System.currentTimeMillis();
    CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    try {
        subscribeFuture.get(time, TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        if (!subscribeFuture.completeExceptionally(new RedisTimeoutException(
                "Unable to acquire subscription lock after " + time + "ms. " +
                        "Try to increase 'subscriptionsPerConnection' and/or 'subscriptionConnectionPoolSize' parameters."))) {
            subscribeFuture.whenComplete((res, ex) -> {
                if (ex == null) {
                    unsubscribe(res, threadId);
                }
            });
        }
        acquireFailed(waitTime, unit, threadId);
        return false;
    } catch (ExecutionException e) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
}    
```
#### 3.解决超时释放问题
在文档的分布式锁章节，每一种类型的锁都有以下描述：
> If Redisson instance which acquired lock crashes then such lock could hang forever in acquired state. To avoid this Redisson maintains lock watchdog, it prolongs lock expiration while lock holder Redisson instance is alive. By default lock watchdog timeout is 30 seconds and can be changed through [Config.lockWatchdogTimeout](https://github.com/redisson/redisson/wiki/2.-Configuration#lockwatchdogtimeout) setting.

Ression引入了wathdog机制，在锁持有者Redisson实例处于活动状态时，延长锁的过期时间，默认情况下，锁定看门狗的超时时间为30s。
从tryLock()方法的源码追踪，在tryAcquireAsync()方法中进行了定时延长有效期的操作。

```
private RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    RFuture<Long> ttlRemainingFuture;
    if (leaseTime > 0) {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    } else {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    }
    CompletionStage<Long> s = handleNoSync(threadId, ttlRemainingFuture);
    ttlRemainingFuture = new CompletableFutureWrapper<>(s);

    CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
        // lock acquired
        if (ttlRemaining == null) {
            if (leaseTime > 0) {
                internalLockLeaseTime = unit.toMillis(leaseTime);
            } else {
                scheduleExpirationRenewal(threadId);
            }
        }
        return ttlRemaining;
    });
    return new CompletableFutureWrapper<>(f);
}
```
通过scheduleExpirationRenewal()方法设置定时任务，底层是基于Netty的HashedWheelTimer时间轮函数，这里就不展开讨论了，感兴趣的话可以去翻翻源码。
### 四、总结
Redisson分布式锁的实现原理：
- 可重入：记录线程id和重入次数；
- 可重试：利用订阅功能实现等待、唤醒机制，达到失败重试的目的；
- 延长有效期：使用watchdog方式，通过定时任务延长有效期。

