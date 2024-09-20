# 为什么会关注这个问题

## 背景

线上有个场景使用了redis缓存，在使用这个缓存之前会首先判断这个缓存是否存在。但是线上的错误日志提示获取这个缓存时不存在，为什么已经提前判断了key是否存在还会出现取值的时候拿不到呢？

判断是否存在使用的是`stringRedisTemplate.hasKey`方法，后续通过补充日志确定了这个方法返回了true，但是get拿到的值是空的。

## 问题的答案

`stringRedisTemplate.hasKey` 底层使用的是Redis的`EXISTS` 命令，问题就出在这个命令上，问题的原因跟Redis的key过期策略和`EXISTS` 命令的实现有关系。

# Redis的key过期策略

Redis的key过期策略主要时为了平衡key过期的效率和系统资源的占用（例如CPU资源和内存资源），如果不考虑资源占用的问题，肯定就是用个定时任务实时扫描来的更及时。

## 惰性删除/被动删除

所谓惰性删除就是当一条Redis命令访问到某一个key时，会首先判断key是否已经过期，如果已经过期，那么就先删除，然后再执行命令的具体内容。

过期键的惰性删除策略由`db.c/expireIfNeeded`函数实现，所有读写数据库的Redis命令在执行之前都会调用`expireIfNeeded`函数对输入键进行检查。

## 定期清理/主动删除

单纯依靠惰性删除显然无法满足需求，如果一个过期的key永远都不会再被访问，依靠惰性删除就无法完成这个key的清理，所以Redis还有一个定期清理的策略。

过期键的定期删除策略由`redis.c/activeExpireCycle`函数实现，每当Redis的服务器周期性操作`redis.c/serverCron`函数执行时，`activeExpireCycle`函数就会被调用，它在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的expires字典中随机检查一部分键的过期时间，并删除其中的过期键。

# EXISTS命令的问题

`EXISTS`命令的问题是他不会触发Redis的惰性删除，所以当一个key过期以后的较短的一个时间内，用`EXISTS` 命令来判断key是否存在就很有可能拿到key还存在的结果。

# 问题的解决

这个Redis的一个bug，或者说一系列的bug，关于master-slave查询的问题：

slave 查询过期 key，经历了 3 个阶段：

1. 3.2 以下版本，key 过期未被清理，无论哪个命令，查询 slave，均正常返回 value
2. 3.2 - 4.0.11 版本，查询数据返回 NULL，但 EXISTS 依旧返回 true
3. 4.0.11 以上版本，所有命令均已修复，过期 key 在 slave 上查询，均返回「不存在」

但是问题并没有完全解决，主从key不一致的清空仍然有可能发生：
参考资料：
http://kaito-kidd.com/2021/03/14/redis-trap/