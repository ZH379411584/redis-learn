## 所有资料
[cluster-tutorial](https://redis.io/topics/cluster-tutorial)   
[cluster-tutorial翻译](http://redisdoc.com/topic/cluster-tutorial.html#)  
[Redis Cluster Specification](https://redis.io/topics/cluster-spec)  
[Redis Cluster Specification翻译](http://redisdoc.com/topic/cluster-spec.html)    

## [cluster-tutorial](https://redis.io/topics/cluster-tutorial)   

This document is a gentle introduction to Redis Cluster, that does not use complex to understand distributed systems concepts. It provides instructions about how to setup a cluster, test, and operate it, without going into the details that are covered in the [Redis Cluster specification](https://redis.io/topics/cluster-spec) but just describing how the system behaves from the point of view of the user.（本文档是对Redis Cluster的简要介绍，它不使用复杂知识来理解分布式系统的概念。它提供了有关如何设置集群，测试和操作集群的说明，而没有涉及Redis集群规范中涵盖的细节，而只是从用户的角度描述了系统的行为。）

However this tutorial tries to provide information about the availability and consistency characteristics of Redis Cluster from the point of view of the final user, stated in a simple to understand way.

Note this tutorial requires Redis version 3.0 or higher.

If you plan to run a serious Redis Cluster deployment, the more formal specification is a suggested reading, even if not strictly required. However it is a good idea to start from this document, play with Redis Cluster some time, and only later read the specification.

#### Redis Cluster 101
Redis Cluster provides a way to run a Redis installation where data is automatically sharded across multiple Redis nodes.

Redis Cluster also provides some degree of availability during partitions(分区), that is in practical terms the ability to continue the operations when some nodes fail or are not able to communicate. However the cluster stops to operate in the event of larger failures (for example when the majority of masters are unavailable).

So in practical terms, what do you get with Redis Cluster?

- The ability to automatically split your dataset among multiple nodes.（自动在多个节点之间分割数据集的能力）
- The ability to continue operations when a subset of the nodes are experiencing failures or are unable to communicate with the rest of the cluster.（当一部分节点出现故障或无法与集群的其余部分通信时，继续运行的能力。）

#### Redis Cluster TCP ports
Every Redis Cluster node requires two TCP connections open. The normal Redis TCP port used to serve clients, for example 6379, plus the port obtained by adding 10000 to the data port, so 16379 in the example.

This second high port is used for the Cluster bus（集群总线）, that is a node-to-node communication channel using a binary protocol. The Cluster bus is used by nodes for failure detection, configuration update, failover authorization and so forth.（节点将群集总线用于故障检测，配置更新，故障转移授权等） Clients should never try to communicate with the cluster bus port, but always with the normal Redis command port, however make sure you open both ports in your firewall, otherwise Redis cluster nodes will be not able to communicate.

The command port and cluster bus port offset is fixed and is always 10000.

Note that for a Redis Cluster to work properly you need, for each node:

The normal client communication port (usually 6379) used to communicate with clients to be open to all the clients that need to reach the cluster, plus all the other cluster nodes (that use the client port for keys migrations).
The cluster bus port (the client port + 10000) must be reachable from all the other cluster nodes.
If you don't open both TCP ports, your cluster will not work as expected.

The cluster bus uses a different, binary protocol, for node to node data exchange, which is more suited to exchange information between nodes using little bandwidth and processing time

#### Redis Cluster and Docker
Currently Redis Cluster does not support NATted environments and in general environments where IP addresses or TCP ports are remapped.(当前，Redis Cluster不支持NATted环境以及在重新映射IP地址或TCP端口的常规环境中。)

Docker uses a technique called port mapping: programs running inside Docker containers may be exposed with a different port compared to the one the program believes to be using. This is useful in order to run multiple containers using the same ports, at the same time, in the same server.

In order to make Docker compatible with Redis Cluster you need to use the host networking mode(主机联网模式) of Docker. Please check the --net=host option in the Docker documentation for more information.
#### Redis Cluster data sharding
Redis Cluster does not use consistent hashing(一致性哈希), but a different form of sharding where every key is conceptually part of what we call an hash slot(哈希槽).

There are 16384(2^14) hash slots in Redis Cluster, and to compute what is the hash slot of a given key, we simply take the CRC16 of the key modulo 16384.

Every node in a Redis Cluster is responsible for a subset of the hash slots, so for example you may have a cluster with 3 nodes, where:

Node A contains hash slots from 0 to 5500.
Node B contains hash slots from 5501 to 11000.
Node C contains hash slots from 11001 to 16383.

This allows to add and remove nodes in the cluster easily. For example if I want to add a new node D, I need to move some hash slot from nodes A, B, C to D. Similarly if I want to remove node A from the cluster I can just move the hash slots served by A to B and C. When the node A will be empty I can remove it from the cluster completely.

Because moving hash slots from a node to another does not require to stop operations, adding and removing nodes, or changing the percentage of hash slots hold by nodes, does not require any downtime.(因为将哈希槽从一个节点移动到另一个节点不需要停止操作，所以添加和删除节点或更改节点持有的哈希槽的百分比都不需要停机)

Redis Cluster supports multiple key operations as long as all the keys involved into a single command execution (or whole transaction, or Lua script execution) all belong to the same hash slot. The user can force multiple keys to be part of the same hash slot by using a concept called hash tags.(只要单个命令执行（或整个事务或Lua脚本执行）中涉及的所有键都属于同一哈希槽，Redis Cluster就支持多种键操作.用户可以通过使用称为“哈希标签”的概念来强制多个键成为同一哈希槽的一部分)

Hash tags are documented in the Redis Cluster specification, but the gist is that if there is a substring between {} brackets in a key, only what is inside the string is hashed, so for example this{foo}key and another{foo}key are guaranteed to be in the same hash slot, and can be used together in a command with multiple keys as arguments.

#### Redis Cluster master-slave model
In order to remain available when a subset of master nodes are failing or are not able to communicate with the majority of nodes, Redis Cluster uses a master-slave model where every hash slot has from 1 (the master itself) to N replicas (N-1 additional slaves nodes).

In our example cluster with nodes A, B, C, if node B fails the cluster is not able to continue, since we no longer have a way to serve hash slots in the range 5501-11000.

However when the cluster is created (or at a later time) we add a slave node to every master, so that the final cluster is composed of A, B, C that are masters nodes, and A1, B1, C1 that are slaves nodes, the system is able to continue if node B fails.

Node B1 replicates B, and B fails, the cluster will promote node B1 as the new master and will continue to operate correctly.

However note that if nodes B and B1 fail at the same time Redis Cluster is not able to continue to operate.
#### Redis Cluster consistency guarantees
Redis Cluster is not able to guarantee strong consistency.(Redis Cluster无法保证强一致性) In practical terms this means that under certain conditions it is possible that Redis Cluster will lose writes that were acknowledged by the system to the client.(实际上，这意味着在某些情况下，Redis Cluster可能会丢失系统认可给客户端的写入)

The first reason why Redis Cluster can lose writes is because it uses asynchronous replication.(异步复制) This means that during writes the following happens:

Your client writes to the master B.
The master B replies OK to your client.
The master B propagates the write to its slaves B1, B2 and B3.
As you can see, B does not wait for an acknowledgement from B1, B2, B3 before replying to the client(B在回复客户端之前不等待B1，B2，B3的确认), since this would be a prohibitive latency penalty for Redis(因为这会对Redis造成极大的延迟损失
), so if your client writes something, B acknowledges the write, but crashes before being able to send the write to its slaves, one of the slaves (that did not receive the write) can be promoted to master, losing the write forever.

This is very similar to what happens with most databases that are configured to flush data to disk every second, so it is a scenario you are already able to reason about because of past experiences with traditional database systems not involving distributed systems. Similarly you can improve consistency by forcing the database to flush data on disk before replying to the client, but this usually results into prohibitively low performance. That would be the equivalent of synchronous replication in the case of Redis Cluster.

Basically there is a trade-off to take between performance and consistency.(基本上，需要在性能和一致性之间进行权衡。)

Redis Cluster has support for synchronous writes when absolutely needed, implemented via the WAIT command, this makes losing writes a lot less likely, however note that Redis Cluster does not implement strong consistency even when synchronous replication is used: it is always possible under more complex failure scenarios that a slave that was not able to receive the write is elected as master.

There is another notable scenario where Redis Cluster will lose writes, that happens during a network partition where a client is isolated with a minority of instances including at least a master.(还有另一种值得注意的情况，Redis Cluster将丢失写操作，这种情况发生在网络分区期间，在该分区中，客户端与少数实例（至少包括一个主实例）隔离。)

Take as an example our 6 nodes cluster composed of A, B, C, A1, B1, C1, with 3 masters and 3 slaves. There is also a client, that we will call Z1.

After a partition occurs, it is possible that in one side of the partition we have A, C, A1, B1, C1, and in the other side we have B and Z1.

Z1 is still able to write to B, that will accept its writes. If the partition heals in a very short time, the cluster will continue normally. However if the partition lasts enough time for B1 to be promoted to master in the majority side of the partition, the writes that Z1 is sending to B will be lost.

Note that there is a maximum window to the amount of writes Z1 will be able to send to B: if enough time has elapsed for the majority side of the partition to elect a slave as master, every master node in the minority side stops accepting writes.

This amount of time is a very important configuration directive of Redis Cluster, and is called the node timeout.

After node timeout has elapsed, a master node is considered to be failing, and can be replaced by one of its replicas. (节点超时结束后，将主节点视为故障节点，并且可以用其副本之一替换该主节点。)Similarly after node timeout has elapsed without a master node to be able to sense the majority of the other master nodes, it enters an error state and stops accepting writes.

#### Redis Cluster configuration parameters
We are about to create an example cluster deployment. Before we continue, let's introduce the configuration parameters that Redis Cluster introduces in the redis.conf file. Some will be obvious, others will be more clear as you continue reading.
- cluster-enabled <yes/no>: If yes, enables Redis Cluster support in a specific Redis instance. Otherwise the instance starts as a stand alone instance as usual.
- cluster-config-file <filename>: Note that despite the name of this option, this is not a user editable configuration file, but the file where a Redis Cluster node automatically persists the cluster configuration (the state, basically) every time there is a change, in order to be able to re-read it at startup. The file lists things like the other nodes in the cluster, their state, persistent variables, and so forth. Often this file is rewritten and flushed on disk as a result of some message reception.
- cluster-node-timeout <milliseconds>: The maximum amount of time a Redis Cluster node can be unavailable, without it being considered as failing. If a master node is not reachable for more than the specified amount of time, it will be failed over by its slaves. This parameter controls other important things in Redis Cluster. Notably, every node that can't reach the majority of master nodes for the specified amount of time, will stop accepting queries.
- cluster-slave-validity-factor <factor>: If set to zero, a slave will always try to failover a master, regardless of the amount of time the link between the master and the slave remained disconnected. If the value is positive, a maximum disconnection time is calculated as the node timeout value multiplied by the factor provided with this option, and if the node is a slave, it will not try to start a failover if the master link was disconnected for more than the specified amount of time. For example if the node timeout is set to 5 seconds, and the validity factor is set to 10, a slave disconnected from the master for more than 50 seconds will not try to failover its master. Note that any value different than zero may result in Redis Cluster to be unavailable after a master failure if there is no slave able to failover it. In that case the cluster will return back available only when the original master rejoins the cluster.
cluster-migration-barrier <count>: Minimum number of slaves a master will remain connected with, for another slave to migrate to a master which is no longer covered by any slave. See the appropriate section about replica migration in this tutorial for more information.
- cluster-require-full-coverage <yes/no>: If this is set to yes, as it is by default, the cluster stops accepting writes if some percentage of the key space is not covered by any node. If the option is set to no, the cluster will still serve queries even if only requests about a subset of keys can be processed.
- cluster-allow-reads-when-down <yes/no>: If this is set to no, as it is by default, a node in a Redis Cluster will stop serving all traffic when when the cluster is marked as fail, either when a node can't reach a quorum of masters or full coverage is not met. This prevents reading potentially inconsistent data from a node that is unaware of changes in the cluster. This option can be set to yes to allow reads from a node during the fail state, which is useful for applications that want to prioritize read availability but still want to prevent inconsistent writes. It can also be used for when using Redis Cluster with only one or two shards, as it allows the nodes to continue serving writes when a master fails but automatic failover is impossible
  
#### Creating and using a Redis Cluster

#### Replicas migration
In Redis Cluster it is possible to reconfigure a slave to replicate with a different master at any time just using the following command:

```
CLUSTER REPLICATE <master-node-id>

```
However there is a special scenario where you want replicas to move from one master to another one automatically, without the help of the system administrator. The automatic reconfiguration of replicas is called replicas migration and is able to improve the reliability of a Redis Cluster.

Note: you can read the details of replicas migration in the Redis Cluster Specification, here we'll only provide some information about the general idea and what you should do in order to benefit from it.

The reason why you may want to let your cluster replicas to move from one master to another under certain condition, is that usually the Redis Cluster is as resistant to failures as the number of replicas attached to a given master.

For example a cluster where every master has a single replica can't continue operations if the master and its replica fail at the same time, simply because there is no other instance to have a copy of the hash slots the master was serving. However while net-splits are likely to isolate a number of nodes at the same time, many other kind of failures, like hardware or software failures local to a single node, are a very notable class of failures that are unlikely to happen at the same time, so it is possible that in your cluster where every master has a slave, the slave is killed at 4am, and the master is killed at 6am. This still will result in a cluster that can no longer operate.

To improve reliability of the system we have the option to add additional replicas to every master, but this is expensive. Replica migration allows to add more slaves to just a few masters. So you have 10 masters with 1 slave each, for a total of 20 instances. However you add, for example, 3 instances more as slaves of some of your masters, so certain masters will have more than a single slave.

With replicas migration what happens is that if a master is left without slaves, a replica from a master that has multiple slaves will migrate to the orphaned master. So after your slave goes down at 4am as in the example we made above, another slave will take its place, and when the master will fail as well at 5am, there is still a slave that can be elected so that the cluster can continue to operate.

So what you should know about replicas migration in short?

- The cluster will try to migrate a replica from the master that has the greatest number of replicas in a given moment.
- To benefit from replica migration you have just to add a few more replicas to a single master in your cluster, it does not matter what master.
- There is a configuration parameter that controls the replica migration feature that is called cluster-migration-barrier: you can read more about it in the example redis.conf file provided with Redis Cluster.
