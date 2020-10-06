## 所有资料
[sentinel](https://redis.io/topics/sentinel)

## [sentinel](https://redis.io/topics/sentinel)

Redis Sentinel provides high availability for Redis. In practical terms this means that using Sentinel you can create a Redis deployment that resists without human intervention certain kinds of failures.

Redis Sentinel also provides other collateral tasks such as monitoring, notifications and acts as a configuration provider for clients.

This is the full list of Sentinel capabilities at a macroscopical level (i.e. the big picture):

- Monitoring. Sentinel constantly checks if your master and replica instances are working as expected.
- Notification. Sentinel can notify the system administrator, or other computer programs, via an API, that something is wrong with one of the monitored Redis instances.
- Automatic failover. If a master is not working as expected, Sentinel can start a failover process where a replica is promoted to master, the other additional replicas are reconfigured to use the new master, and the applications using the Redis server are informed about the new address to use when connecting.
- Configuration provider. Sentinel acts as a source of authority for clients service discovery: clients connect to Sentinels in order to ask for the address of the current Redis master responsible for a given service. If a failover occurs, Sentinels will report the new address.
#### Distributed nature of Sentinel
Redis Sentinel is a distributed system:

Sentinel itself is designed to run in a configuration where there are multiple Sentinel processes cooperating together. The advantage of having multiple Sentinel processes cooperating are the following:

Failure detection is performed when multiple Sentinels agree about the fact a given master is no longer available. This lowers the probability of false positives.

Sentinel works even if not all the Sentinel processes are working, making the system robust against failures. There is no fun in having a failover system which is itself a single point of failure, after all.
The sum of Sentinels, Redis instances (masters and replicas) and clients connecting to Sentinel and Redis, are also a larger distributed system with specific properties. In this document concepts will be introduced gradually starting from basic information needed in order to understand the basic properties of Sentinel, to more complex information (that are optional) in order to understand how exactly Sentinel works.



#### 

sentinel config

```
sentinel monitor <master-group-name> <ip> <port> <quorum>

```
For the sake of clarity, let's check line by line what the configuration options mean:

The first line is used to tell Redis to monitor a master called mymaster, that is at address 127.0.0.1 and port 6379, with a quorum of 2. Everything is pretty obvious but the quorum argument:

- The quorum is the number of Sentinels that need to agree about the fact the master is not reachable, in order to really mark the master as failing, and eventually start a failover procedure if possible.
- However the quorum is only used to detect the failure. In order to actually perform a failover, one of the Sentinels need to be elected leader for the failover and be authorized to proceed. This only happens with the vote of the majority of the Sentinel processes.
So for example if you have 5 Sentinel processes, and the quorum for a given master set to the value of 2, this is what happens:
- If two Sentinels agree at the same time about the master being unreachable, one of the two will try to start a failover.
- If there are at least a total of three Sentinels reachable, the failover will be authorized and will actually start.
In practical terms this means during failures Sentinel never starts a failover if the majority of Sentinel processes are unable to talk (aka no failover in the minority partition).
