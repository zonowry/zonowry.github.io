---
name: generate-serial-number
title: 并发安全的生成一个时间相关的订单流水号
description: 总结下利用 redis 生成流水号的一个解决方案。
date: 2025-04-13
isCJKLanguage: true
toc: true
tags:
  - redis
  - concurrent
keywords:
  - serial number generator
  - order number generator
---

## 序列号部分

得益于 redis 的原子性与其自增方法 `INCR`，我们业务方法并不需要线程锁，即可获取一个并发安全的自增序列号；随后将序列号的 key 精确到秒，我们就可以获得一个秒级别的自增序列号。

```kotlin
val dateStr = now.format(DateTimeFormatter.ofPattern("yyMMddhhmmss"))


val redisKey = "order_seq:$dateStr"
val seq = redis.opsForValue().increment(redisKey)
    ?: throw IllegalStateException("Redis INCR 失败")

// MAX_SEQ = (2 shl 14) -1
require(seq <= MAX_SEQ) { "超过每秒最大订单数限制" }

if (seq == 1L) {
    // 设置过期时间，避免 Redis 键爆炸（设置为2秒即可）
    redis.expire(redisKey, Duration.ofSeconds(2))
}
```

## 时间戳部分

再将序列号拼接上时间戳就是一个相对完整的流水号了；时间戳很好获取，根据当前服务器系统时间计算即可。

这里我们的时间戳格式为 `yyMMddhhmmss`。这样的话 `250413120000` 就代表 25 年 4 月 13 号 12 点 0 分 0 秒，再拼接上序列号 `0001` ，流水号就是 `25_04_13_12_00_00_0001`。

```kotlin
val dateInt = dateStr.toInt()
// 假定序列号只有 14 位
val currentNumber = (dateInt.toLong() shl 14) or seq
```

## 时钟回拨问题

如此简单就基本完成了核心逻辑，但很不「高可用」。如果业务服务器（即调用 redis `INCR` 的服务器）的系统时间回拨了呢？那么**新序列号可能会比老序列号更小**，例如系统时间回拨一年，重启服务，则会生成 24 年的流水号 `24_04_13...`。

可见让这个流水号生成器「高可用」的主要的挑战在于预防「时钟回拨」。

当服务器的系统时间异常了，原因可能会五花八门，不过导致的根本问题都是「新序列号比老序列号更小」。

> 至于 redis 重启问题，在我们这个生成逻辑下，redis 得是秒级重启。如果 redis 花费了一秒多的时间重启成功，那么序列号可以透明的、自动的从零重新开始自增，基本不用考虑这个问题。

我们需要将「新序列号」与「（上一个）旧序列号」比较，才能知道新序列号是否正确。

```kotlin
private val prev = -1L
```

```kotlin
if (currentNumber < prev) {
    // TODO: 解决时钟回拨问题
    throw IllegalStateException("时钟回拨：$currentNumber < $prev")
}
```

现在的问题变成了 `prev` 怎么读写。

## 即时更新序列号

很方便的是，我们（业务服务器）是生成者，生成新序列号后更新下变量即可。

```kotlin
if (currentNumber < prev) {
	throw IllegalStateException("时钟回拨：$currentNumber < $prev")
}
prev = currentNumber
```

出现了，竟态条件！前面说到我们的生成器只依赖 redis 的 `INCR`，本来是不需要线程锁的，但现在 `prev` 变量的出现，导致生成器线程不安全了。

可以直接给生成器方法上锁，但不够轻便和**优雅**。因为我们的序列号是秒级别内并发才需要同步，不需要时时刻刻同步，可以乐观一点，使用 `AtomicLong`。

```kotlin
private val lastNumber = AtomicLong(0)
```

简单的实现，利用原子变量尝试单次更新。

```kotlin
val updated = lastNumber.updateAndGet { prev ->
    if (currentNumber > prev) currentNumber else prev
}

if (currentNumber < updated) {
    throw IllegalStateException("时钟回拨，无法生成订单号")
}
```

当我们需要在时钟回拨时做些处理的时候，我们就可以基于原子变量封装一个乐观锁。

```kotlin
// 保证线程安全地维护 lastNumber，避免时钟回拨
while (true) {
    val prev = lastNumber.get()
    if (currentNumber < prev) {
       // throw IllegalStateException("时钟回拨，无法生成订单号")
       // 或者等待时间修复，重新生成 currentNumber
       currentNumber = waitClockSyncAndGenerate()
       continue
    }

    if (lastNumber.compareAndSet(prev, currentNumber)) {
        break
    }
}
```

## 启动时恢复序列号

及时更新很完美，但当业务服务器重启，就会丢失 `lastNumber` 值，需要一个行为在服务启动时恢复 `lastNumber`。

### plan 1 - Redis 缓存

首当其冲，我们的 redis 服务器还健在，直接从 redis 服务器中恢复。

```kotlin
fun postInit() {
	val last = redis.opsForValue().get(LAST_NUMBER_KEY)
	if (last != null) {
	    lastNumber.set(last)
	    return
	}
}
```

### plan 2 - 生命周期 Hook

如果 redis 服务也重启了，还是要想办法持久化 `lastNumber`。什么时候持久化比较好呢，因为持久化是一个 IO 操作，在每次生成时即时持久化不够优雅，最好是通过各种手段监控到业务服务的销毁后，在业务服务启动前持久化 `lastNumber`。

当然设计允许的话，也可以直接从相关业务表里恢复，例如 `select max(orderNumber) from order`，获取最大（新）的订单号。

```kotlin
val last = sql.query("select max(orderNumber) from order")
if (last != null) {
	lastNumber.set(last)
	return
}
```

如果这个生成器专门为这个业务服务，这样做没什么不好。若是一个通用生成器，就不够优雅了，会使我们的生成器要和某个业务强绑定。

紧随其后的就是业务服务器本地文件，当业务服务是正常停止的，一般都会给我们提供一些 hook，例如 spring 的 `@PreDestory` 注解。

```kotlin
@PreDestory
fun preDestory() {
	try {
	    LAST_NUMBER_FILE.parentFile.mkdirs()
	    LAST_NUMBER_FILE.writeText(lastNumber.get().toString())
	} catch (e: Exception) {
	    println("写入本地 lastNumber 失败：${e.message}")
	}
}

@PostConstructor
fun postInit() {
	// Plan 2: 尝试从本地文件恢复
	if (LAST_NUMBER_FILE.exists()) {
	    val fileVal = LAST_NUMBER_FILE.readText().trim().toLongOrNull()
	    if (fileVal != null) {
	        lastNumber.set(fileVal)
	    }
	}
}
```

### plan 3 - 守卫服务

但前面的还是不靠谱，单机服务、机房淹水，来不及正常停止服务，这些容错方案就不起作用了。

我们可以抽象一个 `StateStorage` 出来，交给下游实现，主要作用是持久化 `lastNumber`。至于怎么实现就敬请想象了，可以利用心跳检测，服务监控等各种中间件，部署一个外部监控服务（守卫），在一系列连环措施下，终于可以安全的 hook 掉业务服务宕机了。最终我们可以简单的启动时恢复，例如：

```kotlin
@PostConstruct
fun initLastNumber() {
	// Plan 3: 尝试从状态存储器恢复  ，大概会是这样
	val last = stateStorage.getLast()
	lastNumber.set(fileVal)
}
```

不过依靠中间件服务器的异构架构过于庞大，可以简单一点的实现，让业务服务自己监控自己，实现一个内部的 scheduler 。例如以 heartbeat 的形式持久化 `lastNumber`，也足够应付。

最后想说关于启动时恢复，是为了解决新序列号过小牵扯出来的问题。那只要我们的业务服务的系统时间恢复正常，解决掉「新序列号 < 旧序列号」的问题。新旧序列号大小比较通过，此时启动时恢复的靠不靠谱就不重要了。

或者不够优雅也无所谓了，我这个业务很重要，不太关心性能，那就每次生成后即时持久化。

如果再增加一个 machineid bit，就很像分布式环境下的 snowflake id 了。不过也要面临分布式下才需要考虑的诸多问题。

## 参考

[SnowflakeId | CosId](https://cosid.ahoo.me/guide/snowflake.html)
