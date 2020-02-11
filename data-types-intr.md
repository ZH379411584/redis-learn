## 所有资料
[data-types-intro](https://redis.io/topics/data-types-intro)
## [data-types-intro](https://redis.io/topics/data-types-intro)
### An introduction to Redis data types and abstractions
Redis is not a plain key-value store, it is actually a data structures server, supporting different kinds of values. What this means is that, while in traditional key-value stores you associated string keys to string values, in Redis the value is not limited to a simple string, but can also hold more complex data structures. The following is the list of all the data structures supported by Redis, which will be covered separately in this tutorial:

- Binary-safe strings.
- Lists: collections of string elements sorted according to the order of insertion. They are basically linked lists.
- Sets: collections of unique, unsorted string elements.
- Sorted sets, similar to Sets but where every string element is associated to a floating number value, called score. The elements are always taken sorted by their score, so unlike Sets it is possible to retrieve a range of elements (for example you may ask: give me the top 10, or the bottom 10).
- Hashes, which are maps composed of fields associated with values. Both the field and the value are strings. This is very similar to Ruby or Python hashes.
- Bit arrays (or simply bitmaps)位数组（或简称为位图）: it is possible, using special commands, to handle String values like an array of bits: you can set and clear individual bits, count all the bits set to 1, find the first set or unset bit, and so forth.

- HyperLogLogs: this is a probabilistic data structure which is used in order to estimate the cardinality of a set. (这是一个概率数据结构，用于估计集合的基数。)Don't be scared, it is simpler than it seems... See later in the HyperLogLog section of this tutorial.

- Streams: append-only collections of map-like entries that provide an abstract log data type. (提供抽象日志数据类型的类地图项的仅追加集合。)They are covered in depth in the Introduction to Redis Streams.
It's not always trivial to grasp how these data types work and what to use in order to solve a given problem from the [command reference](https://redis.io/commands), so this document is a crash course to Redis data types and their most common patterns.(从命令参考中掌握这些数据类型的工作方式以及使用什么来解决给定问题并不总是那么容易，因此，本文档是Redis数据类型及其最常见模式的速成教程。)

For all the examples we'll use the redis-cli utility, a simple but handy command-line utility, to issue commands against the Redis server.
#### Redis keys
Redis keys are binary safe(Redis key 是二进制安全的), this means that you can use any binary sequence as a key, from a string like "foo" to the content of a JPEG file. The empty string is also a valid key.

A few other rules about keys:

- Very long keys are not a good idea. For instance a key of 1024 bytes is a bad idea not only memory-wise, but also because the lookup of the key in the dataset may require several costly key-comparisons.（例如，1024字节的key是一个坏主意，不仅是内存方面的问题，而且还因为在数据集中查找密钥可能需要进行几次代价高昂的key比较） Even when the task at hand is to match the existence of a large value, hashing it (for example with SHA1) is a better idea, especially from the perspective of memory and bandwidth.（即使任务是匹配一个大值的存在，对它进行散列（例如使用SHA1）也是一个更好的主意，尤其是从内存和带宽的角度来看。 ）

- Very short keys are often not a good idea. There is little point in writing "u1000flw" as a key if you can instead write "user:1000:followers". The latter is more readable and the added space is minor compared to the space used by the key object itself and the value object. While short keys will obviously consume a bit less memory, your job is to find the right balance.

- Try to stick with a schema. For instance "object-type:id" is a good idea, as in "user:1000". Dots or dashes are often used for multi-word fields, as in "comment:1234:reply.to" or "comment:1234:reply-to".

- The maximum allowed key size is 512 MB. key最大限制为512MB。
#### Redis Strings
The Redis String type is the simplest type of value you can associate with a Redis key. It is the only data type in Memcached, so it is also very natural for newcomers to use it in Redis.

Since Redis keys are strings, when we use the string type as a value too, we are mapping a string to another string. The string data type is useful for a number of use cases, like caching HTML fragments or pages.

Let's play a bit with the string type, using redis-cli (all the examples will be performed via redis-cli in this tutorial).
```java
> set mykey somevalue
OK
> get mykey
"somevalue"
```
As you can see using the SET and the GET commands are the way we set and retrieve a string value. Note that SET will replace any existing value already stored into the key, in the case that the key already exists, even if the key is associated with a non-string value. So SET performs an assignment.

Values can be strings (including binary data) of every kind, for instance you can store a jpeg image inside a value. A value can't be bigger than 512 MB.

The SET command has interesting options, that are provided as additional arguments. For example, I may ask SET to fail if the key already exists, or the opposite, that it only succeed if the key already exists:
```
> set mykey newval nx
(nil)
> set mykey newval xx
OK
```
Even if strings are the basic values of Redis, there are interesting operations you can perform with them. For instance, one is atomic increment:
```
> set counter 100
OK
> incr counter
(integer) 101
> incr counter
(integer) 102
> incrby counter 50
(integer) 152
```
The INCR command parses the string value as an integer, increments it by one, and finally sets the obtained value as the new value. There are other similar commands like INCRBY, DECR and DECRBY. Internally it's always the same command, acting in a slightly different way.

What does it mean that INCR is atomic? That even multiple clients issuing INCR against the same key will never enter into a race condition.(即使使用相同KEY发出INCR的多个客户也永远不会进入竞争状态。) For instance, it will never happen that client 1 reads "10", client 2 reads "10" at the same time, both increment to 11, and set the new value to 11. The final value will always be 12 and the read-increment-set operation is performed while all the other clients are not executing a command at the same time.

There are a number of commands for operating on strings. For example the GETSET command sets a key to a new value, returning the old value as the result. You can use this command, for example, if you have a system that increments a Redis key using INCR every time your web site receives a new visitor. You may want to collect this information once every hour, without losing a single increment. You can GETSET the key, assigning it the new value of "0" and reading the old value back.

The ability to set or retrieve the value of multiple keys in a single command is also useful for reduced latency. For this reason there are the MSET and MGET commands:
