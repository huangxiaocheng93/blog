# 从一个线上问题说起

某天收到一个error日志告警，告警的代码如下（隐去了一些信息和无关的代码）：

```java
public List query() {

        ......
        
        // 缓存key
        String cacheKey = "test";
        
        // 判断缓存是否存在，如果不存在就重新加载缓存
        if (!Boolean.TRUE.equals(stringRedisTemplate.hasKey(cacheKey))) {
            return reload(businessId, pageIndex, defaultPageSize);
        }

        // 获取缓存数据，缓存是hash结构
        BoundHashOperations<String, String, Object> operations = stringRedisTemplate.boundHashOps(cacheKey);
        log.info("查询列表, 缓存详情, values:{}", JsonUtil.toJson(operations.entries()));
        
        
        ListCache listCache = JsonUtil.fromStr(
                Optional.ofNullable(operations.get(pageIndex.toString())).map(Object::toString).orElse(null),
                ListCache.class);

        // 获取total和ttl，total和ttl是hash的两个key
        Integer total = Optional.ofNullable(operations.get(CacheConstants.KEY_HASH_TOTAL))
                .map(obj -> Integer.valueOf(obj.toString())).orElse(null);
        Long ttl = Optional.ofNullable(operations.get(CacheConstants.LIST_KEY_HASH_TTL))
                .map(obj -> Long.valueOf(obj.toString())).orElse(null);
        log.info("查询列表, total:{}, ttl:{}, listCache:{}", total, ttl, JsonUtil.toJson(listCache));

        // 非预期情况，走重新加载缓存的逻辑
        if (total == null || ttl == null) {
            // 就是这条日志导致的告警
            log.error("查询列表, 从缓存获取到的total或ttl值为空, cacheKey:{}, total:{}, ttl:{}", cacheKey, total, ttl);
            return reload(businessId, pageIndex, defaultPageSize);
        }
        ......
    }
```

代码的大致意思是在读取一个缓存，缓存是hash结构，所有的hash key都是同时加载进去的，如果缓存不存在，那么走reload方法重新加载缓存并返回，如果缓存存在会直接返回。
那么问题来了，获取缓存已经判断了缓存key是否存在，不存在会直接走缓存reload的方法，存在的话才会走读取缓存的方法。那这行error日志为什么还会被打印呢？

> [!NOTE]
> 这里做的比较好的一点是：再次判断了total和ttl是否为空，为空的情况走了reload方法，所以虽然程序发生了非预期情况，但是返回结果仍然是对的。

## 获取的一瞬间key过期了？
告警的第一反应就是觉得是这个原因，但是查询日志以后立即就否定了，因为这行日志并不止出现了一次，天下哪有这么巧的事？

## 分析一下

所以这个问题就有两个可能性：
1、缓存建立的逻辑有问题，可能存在一个空的key，导致读取到的hash key都是空的；
2、`stringRedisTemplate.hasKey`判断key是否存在有问题；
分析起来非常简单，加日志就可以，打印整个Redis key，排查问题1；输出`stringRedisTemplate.hasKey`的结果，同时获取key的ttl输出，排查问题2；结果是`stringRedisTemplate.hasKey`有问题。
**`stringRedisTemplate.hasKey` 底层使用的是Redis的`EXISTS` 命令，问题就出在这个命令上，问题的原因跟Redis的key过期策略和`EXISTS` 命令的实现有关系。**

# 了解一下Redis的过期key清理策略

通常我们在把缓存数据写入Redis时都会设置一个过期时间，到时间后我们默认这个数据会被Redis清掉，那Redis是怎么清掉这个过期的key的呢？

> [!IMPORTANT]
> Redis里面不建议使用永不过期的key，因为后期想要处理这些key会变得非常麻烦，里面一些命令会导致key的过期时间丢失，变成永不过期的key，这一点也要非常注意。

思考一下，如果要我们来实现，无非就是两种方案：

1. 定时任务扫描，找到过期的key，然后删除，缺点是如果key的数量非常大，任务执行耗时就会非常长，长时间占用CPU资源；
2. 被动删除，也就是key被访问的时候发现过期了就删除，缺点假如key不再被访问，就删不掉了；

所以综合考虑下来，Redis采用了结合两个方案的方式，主要是为了平衡删key的效率和系统资源的占用（例如CPU资源和内存资源），如果不考虑资源占用的问题，肯定就是用个定时任务实时扫描来的更及时。

## 惰性删除/被动删除

所谓惰性删除就是当一条Redis命令访问到某一个key时，会首先判断key是否已经过期，如果已经过期，那么就先删除，然后再执行命令的具体内容。

过期键的惰性删除策略由`db.c/expireIfNeeded`函数实现，所有读写数据库的Redis命令在执行之前都会调用`expireIfNeeded`函数对输入键进行检查。

## 定期清理/主动删除

单纯依靠惰性删除显然无法满足需求，如果一个过期的key永远都不会再被访问，依靠惰性删除就无法完成这个key的清理，所以Redis还有一个定期清理的策略。

过期键的定期删除策略由`redis.c/activeExpireCycle`函数实现，每当Redis的服务器周期性操作`redis.c/serverCron`函数执行时，`activeExpireCycle`函数就会被调用，它在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的expires字典中随机检查一部分键的过期时间，并删除其中的过期键。

# EXISTS命令的问题

通过上面的说明可以看出来，过期key的清理很大程度上依赖命令执行的时候检查一次，也就是**惰性删除**策略，好巧不巧`EXISTS`命令没有触发Redis的惰性删除，所以当一个key过期以后如果没有被其他命令访问过，用`EXISTS` 命令来判断key是否存在就很有可能拿到key还存在的结果。

# 问题的解决

很明显这个Redis的一个bug，或者说一系列的bug，关于master-slave查询的问题：

slave 查询过期 key，经历了 3 个阶段：

3.2 以下版本 -> key 过期未被清理，无论哪个命令，查询 slave，均正常返回 value
3.2 - 4.0.11 版本 -> 查询数据返回 NULL，但 EXISTS 依旧返回 true
4.0.11 以上版本 -> 所有命令均已修复，过期 key 在 slave 上查询，均返回「不存在」