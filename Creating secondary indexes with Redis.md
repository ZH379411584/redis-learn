## 所有资料
[Secondary indexing with Redis](https://redis.io/topics/indexes)
## [Secondary indexing with Redis](https://redis.io/topics/indexes)
### Secondary indexing with Redis 使用Redis进行二级索引
Redis is not exactly a key-value store, since values can be complex data structures. However it has an external key-value shell: at API level data is addressed by the key name. It is fair to say that, natively, Redis only offers primary key access. However since Redis is a data structures server, its capabilities can be used for indexing, in order to create secondary indexes of different kinds, including composite (multi-column) indexes.   

This document explains how it is possible to create indexes in Redis using the following data structures:
- Sorted sets to create secondary indexes by ID or other numerical fields.
- Sorted sets with lexicographical ranges for creating more advanced secondary indexes, composite indexes and graph traversal indexes.
- Sets for creating random indexes.
- Lists for creating simple iterable indexes and last N items indexes.

Implementing and maintaining indexes with Redis is an advanced topic, so most users that need to perform complex queries on data should understand if they are better served by a relational store. However often, especially in caching scenarios, there is the explicit need to store indexed data into Redis in order to speedup common queries which require some form of indexing in order to be executed.
#### Simple numerical indexes with sorted sets
The simplest secondary index you can create with Redis is by using the sorted set data type, which is a data structure representing a set of elements ordered by a floating point number which is the score of each element. Elements are ordered from the smallest to the highest score.

Since the score is a double precision float, indexes you can build with vanilla sorted sets are limited to things where the indexing field is a number within a given range.

The two commands to build these kind of indexes are ZADD and ZRANGEBYSCORE to respectively add items and retrieve items within a specified range.

For instance, it is possible to index a set of person names by their age by adding element to a sorted set. The element will be the name of the person and the score will be the age.
```
ZADD myindex 25 Manuel
ZADD myindex 18 Anna
ZADD myindex 35 Jon
ZADD myindex 67 Helen
```
In order to retrieve all persons with an age between 20 and 40, the following command can be used:
```
ZRANGEBYSCORE myindex 20 40
1) "Manuel"
2) "Jon"
```
By using the WITHSCORES option of ZRANGEBYSCORE it is also possible to obtain the scores associated with the returned elements.

The ZCOUNT command can be used in order to retrieve the number of elements within a given range, without actually fetching the elements, which is also useful, especially given the fact the operation is executed in logarithmic time regardless of the size of the range.

Ranges can be inclusive(包括) or exclusive（不包括）, please refer to the ZRANGEBYSCORE command documentation for more information.

Note: Using the ZREVRANGEBYSCORE it is possible to query a range in reversed order, which is often useful when data is indexed in a given direction (ascending or descending) but we want to retrieve information the other way around.
#### Using objects IDs as associated values
In the above example we associated names to ages. However in general we may want to index some field of an object which is stored elsewhere.(但是，一般而言，我们可能希望索引存储在其他位置的对象的某些字段。) Instead of using the sorted set value directly to store the data associated with the indexed field, it is possible to store just the ID of the object.

For example I may have Redis hashes representing users. Each user is represented by a single key, directly accessible by ID:
```
HMSET user:1 id 1 username antirez ctime 1444809424 age 38
HMSET user:2 id 2 username maria ctime 1444808132 age 42
HMSET user:3 id 3 username jballard ctime 1443246218 age 33
```
If I want to create an index in order to query users by their age, I could do:
```
ZADD user.age.index 38 1
ZADD user.age.index 42 2
ZADD user.age.index 33 3
```
This time the value associated with the score in the sorted set is the ID of the object. So once I query the index with ZRANGEBYSCORE I'll also have to retrieve the information I need with HGETALL or similar commands. The obvious advantage is that objects can change without touching the index, as long as we don't change the indexed field.

In the next examples we'll almost always use IDs as values associated with the index, since this is usually the more sounding design, with a few exceptions.

#### Updating simple sorted set indexes

Often we index things which change over time. In the above example, the age of the user changes every year. In such a case it would make sense to use the birth date as index instead of the age itself, but there are other cases where we simple want some field to change from time to time, and the index to reflect this change.

The ZADD command makes updating simple indexes a very trivial(不重要的) operation since re-adding back an element with a different score and the same value will simply update the score and move the element at the right position, so if the user antirez turned 39 years old, in order to update the data in the hash representing the user, and in the index as well, we need to execute the following two commands:
```
HSET user:1 age 39
ZADD user.age.index 39 1
```
The operation may be wrapped in a MULTI/EXEC transaction in order to make sure both fields are updated or none.
#### Turning multi dimensional data into linear data(将多维数据转换为线性数据)
Indexes created with sorted sets are able to index only a single numerical value. Because of this you may think it is impossible to index something which has multiple dimensions using this kind of indexes, but actually this is not always true. If you can efficiently represent something multi-dimensional in a linear way, they it is often possible to use a simple sorted set for indexing.

For example the Redis geo indexing API uses a sorted set to index places by latitude and longitude using a technique called Geo hash. The sorted set score represents alternating bits of longitude and latitude(经度和纬度的交替位), so that we map the linear score of a sorted set to many small squares in the earth surface. By doing an 8+1 style center plus neighborhoods search it is possible to retrieve elements by radius.

#### Limits of the score
Sorted set elements scores are double precision floats. It means that they can represent different decimal or integer values with different errors, because they use an exponential representation internally.(内部指数表示) However what is interesting for indexing purposes is that the score is always able to represent without any error numbers between -9007199254740992 and 9007199254740992, which is -/+ 2^53.

When representing much larger numbers, you need a different form of indexing that is able to index numbers at any precision, called a lexicographical(词典) index.
#### summary
