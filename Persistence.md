[Redis Persistence](https://redis.io/topics/persistence)   
[Redis 持久化](http://redisdoc.com/topic/persistence.html)   
[redis-persistence-demystified ](http://oldblog.antirez.com/post/redis-persistence-demystified.html)  


## Redis Persistence
Redis provides a different range of persistence options(提供不同范围的持久性选项):

- The RDB persistence performs point-in-time snapshots of your dataset at specified intervals.(RDB持久性按指定的时间间隔执行数据集的时间点快照 )
- The AOF persistence logs every write operation received by the server, that will be played again at server startup, reconstructing the original dataset. Commands are logged using the same format as the Redis protocol itself, in an append-only fashion. Redis is able to rewrite the log in the background when it gets too big.(AOF持久性会记录服务器接收的每个写操作，这些操作将在服务器启动时再次播放，从而重建原始数据集.使用与Redis协议本身相同的格式记录命令，并且仅采用追加方式。 Redis太大时，Redis可以在后台重写日志)
- If you wish, you can disable persistence completely, if you want your data to just exist as long as the server is running.(如果希望，只要您的数据只需要在服务器运行时存在，则可以完全禁用持久性。)
- It is possible to combine both AOF and RDB in the same instance. Notice that, in this case, when Redis restarts the AOF file will be used to reconstruct the original dataset since it is guaranteed to be the most complete.（可以在同一实例中同时合并AOF和RDB。请注意，在这种情况下，当Redis重新启动时，AOF文件将用于重建原始数据集，因为它可以保证是最完整的）

The most important thing to understand is the different trade-offs（权衡） between the RDB and AOF persistence. Let's start with 

### RDB advantages
- RDB is a very compact single-file point-in-time representation of your Redis data. RDB files are perfect for backups. For instance you may want to archive your RDB files every hour for the latest 24 hours, and to save an RDB snapshot every day for 30 days. This allows you to easily restore different versions of the data set in case of disasters.(非常适合备份)
- RDB is very good for disaster recovery, being a single compact file that can be transferred to far data centers, or onto Amazon S3 (possibly encrypted).(非常适合灾难恢复)。
- RDB maximizes Redis performances since the only work the Redis parent process needs to do in order to persist is forking a child that will do all the rest. The parent instance will never perform disk I/O or alike.(RDB最大限度地提高了Redis的性能，因为Redis父进程为了持久化而需要做的唯一工作就是fork一个孩子，该孩子会做其余工作，父实例永远不会执行磁盘I/O或类似操作)
- RDB allows faster restarts with big datasets compared to AOF.(RDB在重启时比AOF快)

### RDB disadvantages
- RDB is NOT good if you need to minimize the chance of data loss in case Redis stops working (for example after a power outage). You can configure different save points where an RDB is produced (for instance after at least five minutes and 100 writes against the data set, but you can have multiple save points). However you'll usually create an RDB snapshot every five minutes or more, so in case of Redis stopping working without a correct shutdown for any reason you should be prepared to lose the latest minutes of data.(如果你需要在redis停止工作的时候确保数据最小的丢失，redis可能不是最好的。)
- RDB needs to fork() often in order to persist on disk using a child process. Fork() can be time consuming if the dataset is big, and may result in Redis to stop serving clients for some millisecond or even for one second if the dataset is very big and the CPU performance not great. AOF also needs to fork() but you can tune how often you want to rewrite your logs without any trade-off on durability.(RDB需要经常使用fork（）才能使用子进程将其持久化在磁盘上。如果数据集很大，Fork（）可能很耗时，并且如果数据集很大并且CPU性能不佳，则可能导致Redis停止为客户端服务一毫秒甚至一秒钟。)

###  AOF advantages
- Using AOF Redis is much more durable: you can have different fsync policies(同步策略): no fsync at all, fsync every second, fsync at every query. With the default policy of fsync every second write performances are still great (fsync is performed using a background thread and the main thread will try hard to perform writes when no fsync is in progress.fsync是使用后台线程执行的，并且在没有fsync进行时，主线程将尽力执行写操作。) but you can only lose one second worth of writes.（但您只能损失一秒钟的写入时间）
- The AOF log is an append only log, so there are no seeks, nor corruption problems if there is a power outage. Even if the log ends with an half-written command for some reason (disk full or other reasons) the redis-check-aof tool is able to fix it easily.(AOF日志仅是一个追加日志，因此，如果断电，也不会出现寻道或损坏问题。 即使由于某种原因（磁盘已满或其他原因）以半写命令结束日志，redis-check-aof工具也可以轻松修复它。)
- Redis is able to automatically rewrite the AOF in background when it gets too big. The rewrite is completely safe as while Redis continues appending to the old file, a completely new one is produced with the minimal set of operations needed to create the current data set, and once this second file is ready Redis switches the two and starts appending to the new one.(当日志变得太大时，redis会在后台自动重写。重写是完全安全的，因为Redis继续追加到旧文件时，会生成一个全新的文件，其中包含创建当前数据集所需的最少操作集，一旦准备好第二个文件，Redis会切换这两个文件并开始追加到 新的那一个。)
- AOF contains a log of all the operations one after the other in an easy to understand and parse format. You can even easily export an AOF file. For instance even if you flushed everything for an error using a FLUSHALL command, if no rewrite of the log was performed in the meantime you can still save your data set just stopping the server, removing the latest command, and restarting Redis again.(AOF以易于理解和解析的格式包含所有操作的日志。 您甚至可以轻松导出AOF文件。 例如，即使您使用FLUSHALL命令刷新了所有错误文件，如果在此期间未执行任何日志重写操作，您仍然可以保存数据集，只是停止服务器，删除最新命令并重新启动Redis。)
### AOF disadvantages
- AOF files are usually bigger than the equivalent RDB files for the same dataset.(对于相同的数据集，AOF文件通常大于等效的RDB文件)
- AOF can be slower than RDB depending on the exact fsync policy. In general with fsync set to every second performance is still very high, and with fsync disabled it should be exactly as fast as RDB even under high load. Still RDB is able to provide more guarantees about the maximum latency even in the case of an huge write load.(根据确切的fsync策略，AOF可能比RDB慢.通常，在将fsync设置为每秒的情况下，性能仍然很高，并且在禁用fsync的情况下，即使在高负载下，它也应与RDB一样快。 即使在巨大的写负载情况下，RDB仍然能够提供有关最大延迟的更多保证。)
- In the past we experienced rare bugs in specific commands (for instance there was one involving blocking commands like BRPOPLPUSH) causing the AOF produced to not reproduce exactly the same dataset on reloading. (过去，我们在特定命令中遇到过罕见的错误（例如，其中有一个涉及阻止命令，例如BRPOPLPUSH），导致生成的AOF在重载时无法重现完全相同的数据集)These bugs are rare and we have tests in the test suite creating random complex datasets automatically and reloading them to check everything is fine. However, these kind of bugs are almost impossible with RDB persistence. To make this point more clear: the Redis AOF works by incrementally updating an existing state, like MySQL or MongoDB does, while the RDB snapshotting creates everything from scratch again and again, that is conceptually more robust. However - 1) It should be noted that every time the AOF is rewritten by Redis it is recreated from scratch starting from the actual data contained in the data set, making resistance to bugs stronger compared to an always appending AOF file (or one rewritten reading the old AOF instead of reading the data in memory). 2) We have never had a single report from users about an AOF corruption that was detected in the real world.

### Ok, so what should I use?

The general indication is that you should use both persistence methods if you want a degree of data safety comparable to what PostgreSQL can provide you.(通常的指示是，如果您想要某种与PostgreSQL可以提供的功能相当的数据安全性，则应同时使用两种持久性方法。)

If you care a lot about your data, but still can live with a few minutes of data loss in case of disasters, you can simply use RDB alone.（如果您非常关心数据，但是在灾难情况下仍然可以承受几分钟的数据丢失，则只需使用RDB。）

There are many users using AOF alone, but we discourage it since to have an RDB snapshot from time to time is a great idea for doing database backups, for faster restarts, and in the event of bugs in the AOF engine.（这里有很多用户单独使用AOF，但是我们不建议这样做，使用RDB快照对于进行数据库备份，加快重新启动速度以及处理AOF引擎中的错误 会更好。）

Note: for all these reasons we'll likely end up unifying AOF and RDB into a single persistence model in the future (long term plan).（由于所有这些原因，我们将来可能最终将AOF和RDB统一为一个持久性模型）

The following sections will illustrate a few more details about the two persistence models.

### Snapshotting（快照）
By default Redis saves snapshots of the dataset on disk, in a binary file called dump.rdb.（默认情况下，Redis将数据集的快照保存在磁盘上的名为dump.rdb的二进制文件中） You can configure Redis to have it save the dataset every N seconds if there are at least M changes in the dataset, or you can manually call the SAVE or BGSAVE commands.

For example, this configuration will make Redis automatically dump the dataset to disk every 60 seconds if at least 1000 keys changed:
```
save 60 1000
```
This strategy is known as snapshotting.

How it works
Whenever Redis needs to dump the dataset to disk, this is what happens:
- Redis forks. We now have a child and a parent process.（Redis fork ，现在我们拥有一个孩子进程和父进程）
- The child starts to write the dataset to a temporary RDB file.（孩子进程开始将数据集写入临时RDB文件）
- When the child is done writing the new RDB file, it replaces the old one. （当孩子进程写完时，临时RDB文件替换旧的RDB文件）
This method allows Redis to benefit from copy-on-write semantics.（此方法使Redis可以受益于写时复制语义。）

### Append-only file（仅追加文件）
Snapshotting is not very durable. If your computer running Redis stops, your power line fails, or you accidentally kill -9 your instance, the latest data written on Redis will get lost. While this may not be a big deal for some applications, there are use cases for full durability, and in these cases Redis was not a viable option.（快照功能并不是非常耐久（durable）： 如果 Redis 因为某些原因而造成故障停机， 那么服务器将丢失最近写入、且仍未保存到快照中的那些数据。然而对某些应用来说，这可能是个大问题，在某些情况下，要实现完全的耐用性，如果是这样的话，Redis并不是可行的选择。）

The append-only file is an alternative, fully-durable strategy for Redis. It became available in version 1.1.

You can turn on the AOF in your configuration file:
```
appendonly yes

```
From now on, every time Redis receives a command that changes the dataset (e.g. SET) it will append it to the AOF. When you restart Redis it will re-play the AOF to rebuild the state.

#### Log rewriting
As you can guess, the AOF gets bigger and bigger as write operations are performed. For example, if you are incrementing a counter 100 times, you'll end up with a single key in your dataset containing the final value, but 100 entries in your AOF. 99 of those entries are not needed to rebuild the current state.

So Redis supports an interesting feature: it is able to rebuild the AOF in the background without interrupting service to clients.(它能够在后台重建AOF，而不会中断对客户端的服务。) Whenever you issue a BGREWRITEAOF Redis will write the shortest sequence of commands needed to rebuild the current dataset in memory. If you're using the AOF with Redis 2.2 you'll need to run BGREWRITEAOF from time to time. Redis 2.4 is able to trigger log rewriting automatically (see the 2.4 example configuration file for more information).

#### How durable is the append only file?
You can configure how many times Redis will fsync data on disk. There are three options:

- appendfsync always: fsync every time a new command is appended to the AOF. Very very slow, very safe.
- appendfsync everysec: fsync every second. Fast enough (in 2.4 likely to be as fast as snapshotting), and you can lose 1 second of data if there is a disaster.
- appendfsync no: Never fsync, just put your data in the hands of the Operating System. The faster and less safe method. Normally Linux will flush data every 30 seconds with this configuration, but it's up to the kernel exact tuning.
The suggested (and default) policy is to fsync every second. It is both very fast and pretty safe. The always policy is very slow in practice, but it supports group commit, so if there are multiple parallel writes Redis will try to perform a single fsync operation.

The suggested (and default) policy is to fsync every second. It is both very fast and pretty safe. The always policy is very slow in practice, but it supports group commit, so if there are multiple parallel writes Redis will try to perform a single fsync operation.
#### What should I do if my AOF gets truncated?（当AOF文件被截断，我该如何怎办？）
It is possible that the server crashed while writing the AOF file, or that the volume where the AOF file is stored was full at the time of writing. When this happens the AOF still contains consistent data representing a given point-in-time version of the dataset (that may be old up to one second with the default AOF fsync policy), but the last command in the AOF could be truncated. The latest major versions of Redis will be able to load the AOF anyway, just discarding the last non well formed command in the file. In this case the server will emit a log like the following:
```
* Reading RDB preamble from AOF file...
* Reading the remaining AOF tail...
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled

```
You can change the default configuration to force Redis to stop in such cases if you want, but the default configuration is to continue regardless the fact the last command in the file is not well-formed, in order to guarantee availability after a restart.

Older versions of Redis may not recover, and may require the following steps:

Make a backup copy of your AOF file.
Fix the original file using the redis-check-aof tool that ships with Redis:

$ redis-check-aof --fix

Optionally use diff -u to check what is the difference between two files.

Restart the server with the fixed file.

#### What should I do if my AOF gets corrupted?（当AOF文件损坏，我该如何怎办？）
If the AOF file is not just truncated, but corrupted with invalid byte sequences in the middle, things are more complex. Redis will complain at startup and will abort（如果AOF文件不仅被截断，而且在中间被无效字节序列损坏，则情况会更加复杂。 Redis将在启动时抱怨并中止）:
```
* Reading the remaining AOF tail...
# Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>

```
The best thing to do is to run the redis-check-aof utility, initially without the --fix option, then understand the problem, jump at the given offset in the file, and see if it is possible to manually repair the file: the AOF uses the same format of the Redis protocol and is quite simple to fix manually.（不使用 --fix参数，查看文件中出问题的那些抵挡，看看是否能够手动修复，AOF使用使用与Redis协议相同的格式，并且手动修复非常简单。） Otherwise it is possible to let the utility fix the file for us, but in that case all the AOF portion from the invalid part to the end of the file may be discarded, leading to a massive amount of data loss if the corruption happened to be in the initial part of the file.（否则，可以让该实用程序为我们修复文件，但是在这样的情况下从无效部分到文件末尾的AOF部分可能会被丢弃，如果损坏恰好发生在文件的初始部分，则会导致大量数据丢失。）

#### How it works
Log rewriting uses the same copy-on-write trick already in use for snapshotting. This is how it works:
- Redis forks, so now we have a child and a parent process.
- The child starts writing the new AOF in a temporary file.
- The parent accumulates all the new changes in an in-memory buffer (but at the same time it writes the new changes in the old append-only file, so if the rewriting fails, we are safe).（父进程将所有新更改累积在内存缓冲区中（但同时它会将新更改写入旧的仅追加文件中，因此，如果重写失败，我们是安全的）。）
- When the child is done rewriting the file, the parent gets a signal, and appends the in-memory buffer at the end of the file generated by the child.（当子进程完成文件的重写后，父进程会收到一个信号，并在子进程生成的文件末尾附加内存中的缓冲区。）
- Profit! Now Redis atomically renames the old file into the new one, and starts appending new data into the new file.（现在，Redis原子地将旧文件重命名为新文件，并开始将新数据追加到新文件中。）
#### How I can switch to AOF, if I'm currently using dump.rdb snapshots?
There is a different procedure to do this in Redis 2.0 and Redis 2.2, as you can guess it's simpler in Redis 2.2 and does not require a restart at all.
#### Redis >= 2.2
- Make a backup of your latest dump.rdb file.
- Transfer this backup into a safe place.
- Issue the following two commands:
- redis-cli config set appendonly yes
- redis-cli config set save ""
- Make sure that your database contains the same number of keys it contained.
- Make sure that writes are appended to the append only file correctly.
The first CONFIG command enables the Append Only File. In order to do so Redis will block to generate the initial dump, then will open the file for writing, and will start appending all the next write queries.

The second CONFIG command is used to turn off snapshotting persistence. This is optional, if you wish you can take both the persistence methods enabled.

#### IMPORTANT:
remember to edit your redis.conf to turn on the AOF, otherwise when you restart the server the configuration changes will be lost and the server will start again with the old configuration.
#### Redis 2.0
- Make a backup of your latest dump.rdb file.
- Transfer this backup into a safe place.
- Stop all the writes against the database!
- Issue a redis-cli BGREWRITEAOF. This will create the append only file.
- Stop the server when Redis finished generating the AOF dump.
- Edit redis.conf end enable append only file persistence.
- Restart the server.
- Make sure that your database contains the same number of keys it contained.
- Make sure that writes are appended to the append only file correctly.
#### Interactions between AOF and RDB persistence(AOF 和RDB持久化之间的互动)
Redis >= 2.4 makes sure to avoid triggering an AOF rewrite when an RDB snapshotting operation is already in progress, or allowing a BGSAVE while the AOF rewrite is in progress. This prevents two Redis background processes from doing heavy disk I/O at the same time.

When snapshotting is in progress and the user explicitly requests a log rewrite operation using BGREWRITEAOF the server will reply with an OK status code telling the user the operation is scheduled, and the rewrite will start once the snapshotting is completed.（当进行快照时，用户使用BGREWRITEAOF显式请求日志重写操作时，服务器将以OK状态代码答复，告知用户已安排该操作，并且快照完成后将开始重写。）

In the case both AOF and RDB persistence are enabled and Redis restarts the AOF file will be used to reconstruct the original dataset since it is guaranteed to be the most complete.

#### Backing up Redis data
Before starting this section, make sure to read the following sentence: Make Sure to Backup Your Database. Disks break, instances in the cloud disappear, and so forth: no backups means huge risk of data disappearing into /dev/null.

Redis is very data backup friendly since you can copy RDB files while the database is running: the RDB is never modified once produced, and while it gets produced it uses a temporary name and is renamed into its final destination atomically using rename(2) only when the new snapshot is complete.

This means that copying the RDB file is completely safe while the server is running. This is what we suggest:

- Create a cron job in your server creating hourly snapshots of the RDB file in one directory, and daily snapshots in a different directory.
- Every time the cron script runs, make sure to call the find command to make sure too old snapshots are deleted: for instance you can take hourly snapshots for the latest 48 hours, and daily snapshots for one or two months. Make sure to name the snapshots with data and time information.
- At least one time every day make sure to transfer an RDB snapshot outside your data center or at least outside the physical machine running your Redis instance.
If you run a Redis instance with only AOF persistence enabled, you can still copy the AOF in order to create backups. The file may lack the final part but Redis will be still able to load it (see the previous sections about truncated AOF files).（如果您在仅启用AOF持久性的情况下运行Redis实例，您仍然可以复制AOF以便创建备份。该文件可能缺少最后一部分，但是Redis仍然可以加载它（请参阅前面有关AOF文件被截断的部分）。）
#### Disaster recovery（灾难恢复）
Disaster recovery in the context of Redis is basically the same story as backups, plus the ability to transfer those backups in many different external data centers.（Redis上下文中的灾难恢复与备份基本相同，并且能够在许多不同的外部数据中心中传输这些备份。） This way data is secured even in the case of some catastrophic event affecting the main data center where Redis is running and producing its snapshots.

Since many Redis users are in the startup scene and thus don't have plenty of money to spend we'll review the most interesting disaster recovery techniques that don't have too high costs.（）

- Amazon S3 and other similar services are a good way for implementing your disaster recovery system. Simply transfer your daily or hourly RDB snapshot to S3 in an encrypted form. You can encrypt your data using gpg -c (in symmetric encryption mode). Make sure to store your password in many different safe places (for instance give a copy to the most important people of your organization). It is recommended to use multiple storage services for improved data safety.

- Transfer your snapshots using SCP (part of SSH) to far servers. This is a fairly simple and safe route: get a small VPS in a place that is very far from you, install ssh there, and generate an ssh client key without passphrase, then add it in the authorized_keys file of your small VPS. You are ready to transfer backups in an automated fashion. Get at least two VPS in two different providers for best results.

It is important to understand that this system can easily fail if not implemented in the right way. At least make absolutely sure that after the transfer is completed you are able to verify the file size (that should match the one of the file you copied) and possibly the SHA1 digest if you are using a VPS.

You also need some kind of independent alert system if the transfer of fresh backups is not working for some reason
