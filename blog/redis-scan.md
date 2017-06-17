#摘要#

本文主要是介绍使用redis scan命令遇到的一些问题总结，scan命令本身没有什么问题，主要是spring-data-redis的问题。


#需求#

需要遍历redis中key，找到符合某些pattern的所有keys。第一反应当然是

` KEYS "ABC* ` 

可以找到前缀是ABC的所有KEYS,时间复杂度O(N)。可以使用，但是在生产环境中，这么使用肯定是不行的，因为生产环境的key的数量比较多，一次查询会block其他操作。而更重要的是一次性返回这么多的key，数据量比较大，网络传输成本高。所以一般生产环境中去找符合某些条件的KEYS一般使用SCAN 或 Sets。

集合来操作比较好理解，一个个的pop出来，但是相当于在原有的数据结构上多了一个keys的set集合。SCAN的不需要多维护这份列表。

#SCAN 命令#

SCAN命令的有SCAN,SSCAN,HSCAN,ZSCAN。
SCAN的话就是遍历所有的keys

其他的SCAN命令的话是SCAN选中的集合。

SCAN命令是增量的循环，每次调用只会返回一小部分的元素。所以不会有`KEYS`命令的坑。

SCAN命令返回的是一个游标，从0开始遍历，到0结束遍历。

	scan 0
	1) "655"
	2)  1) "test1"
	    2) "test2"

返回值一个array，一个是下次循环的cursorId,一个是元素数组。SCAN命令不能保证每次返回的值都是有序的，另外同一个key有可能返回多次，不做区分，需要应用程序去处理。

另外SCAN命令可以指定COUNT,默认是10。但是这个并不是指定多少，就能返回多少，这只是一个提示，并不能保证一定返回这么多条。


#spring-data-redis SCAN命令的坑#

**抛出NoSuchElementException 错误**

```

	RedisConnection redisConnection = redisTemplate.getConnectionFactory().getConnection();
	    Cursor c = redisConnection.scan(scanOptions);
	    while (c.hasNext()) {
	        c.next();
	    }

    java.util.NoSuchElementException at java.util.Collections$EmptyIterator.next(Collections.java:4189) at org.springframework.data.redis.core.ScanCursor.moveNext(ScanCursor.java:215) at org.springframework.data.redis.core.ScanCursor.next(ScanCursor.java:202)

```

这个错误发生在spring-data-redis-1.6版本中。已经被修掉了，
https://github.com/spring-projects/spring-data-redis/pull/154

看到最后comments 1.5.x 和1.6.x中都修复了，但是不知道为什么1.6.0没有修复。

看下ScanCursor.java 源码，异常时next()方法抛出来的，产生的原因是没有next的元素了。在前面介绍过，SCAN命令返回两个一个cursorId,一个是值数组。即使你指定了返回多少条（COUNT）,也不能保证实际会返回多少条，当然包括返回0条。这种情况不会经常发生，当你redis server中有大量小的集合时，而扫描时又扫不到匹配的keys,就会返回0个结果，但这并不表示扫描结束，扫描结束的唯一判断依据是扫描结果返回的cursor = 0


	@Override
	public T next() {

		assertCursorIsOpen();
		
		if (!hasNext()) {
			throw new NoSuchElementException("No more elements available for cursor " + cursorId + ".");
		}

		T next = moveNext(delegate);
		position++;

		return next;
	}


这个错误最好的解决办法是升级spring-data-redis版本。如果没法升级，只能在程序中捕获这个异常，再发一次scan请求。而不是依赖spring-data-redis中的scan请求发送。

**多线程环境使用的坑**

返回这种错误，
```
	java.lang.ClassCastException: java.lang.Long cannot be cast to java.util.List
    at redis.clients.jedis.Connection.getRawObjectMultiBulkReply(Connection.java:230)
    at redis.clients.jedis.Connection.getObjectMultiBulkReply(Connection.java:236)

```

或者unknown reply错误。

这个的原因是在一次full 扫描期间，发送一次scan请求，返回游标结果，connection释放掉了，再发送scan请求时，又拿到一个新的连接。这个在单线程环境下，没有问题，但是在多线程环境下，一般来说没有问题，因为scan 命令server没有状态，只有一个cursorId。一个线程scan一次完了，释放掉连接，再发送时，拿到一个新的连接，没有问题，但是如果拿到其他线程的连接就会出现上述问题。

这个问题在spring-data-redis 1.8 RC1 版本修复。就是每个scan操作的cursor维护一个connection。

如果低版本需要修复的话，就是连接不要交给spring-data-redis管理了，获取一个连接，自己维护。







#参考#

https://stackoverflow.com/questions/28814502/spring-data-redis-multi-threading-issue-with-jedis

https://github.com/spring-projects/spring-data-redis/pull/218

https://jira.spring.io/browse/DATAREDIS-531