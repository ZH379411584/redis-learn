## 所有资料
[Redis Persistence](https://redis.io/topics/persistence)   
[Redis 持久化](http://redisdoc.com/topic/persistence.html)   
[redis-persistence-demystified ](http://oldblog.antirez.com/post/redis-persistence-demystified.html)  


## [Redis Persistence](https://redis.io/topics/persistence)   
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

## [redis-persistence-demystified 揭开redis持久化神秘的面纱](http://oldblog.antirez.com/post/redis-persistence-demystified.html)  
Part of my work on Redis is reading blog posts, forum messages, and the twitter time line for the "Redis" search. It is very important for a developer to have a feeling about what the community of users, and the community of non users, think about the product he is developing. And my feeling is that there is no Redis feature that is as misunderstood as its persistence.（我的感觉是没有比persistence更让人误解的特性了）

In this blog post I'll do an effort to be truly impartial: no advertising of Redis, no attempt to skip the details that may put Redis in a bad light.（在这篇博客文章中，我将尽力做到真正公正：不要发布Redis广告，不要试图跳过可能会使Redis陷入困境的细节。） All I want is simply to provide a clear, understandable picture of how Redis persistence works, how much reliable is, and how it compares to other database systems.（我只想提供一个清晰易懂的思路，以了解Redis持久性的工作方式，可靠程度以及与其他数据库系统的比较。）
#### The OS and the disk
The first thing to consider is what we can expect from a database in terms of durability 就持久性而言的话. In order to do so we can visualize what happens during a simple write operation（为此，我们可以可视化在简单的写操作期间发生的情况）:
1. The client sends a write command to the database (data is in client's memory).(客户端将写命令发送到数据库（数据在客户端的内存中）)
2. The database receives the write (data is in server's memory).(数据库接收到写入（数据在服务器的内存中）)
3. The database calls the system call that writes the data on disk (data is in the kernel's buffer).(数据库调用将数据写入磁盘的系统（数据在内核缓冲区中)
4. The operating system transfers the write buffer to the disk controller (data is in the disk cache).(操作系统将写缓冲区传输到磁盘控制器（数据在磁盘缓存中）。)
5. The disk controller actually writes the data into a physical media (a magnetic disk, a Nand chip, ...).(磁盘控制器实际上将数据写入物理介质（磁盘，Nand芯片等）中。)

Note: the above is an oversimplification in many ways, because there are more levels of caching and buffers than that.（上面列出来的内容只是一个大概，因为实际上有很多级别的缓存和缓冲区）

Step 2 is often implemented as a complex caching system inside the database implementation, and sometimes writes are handled by a different thread or process. （步骤2通常是作为数据库实现内部的复杂缓存系统实现的，有时，写入是由不同的线程或进程处理的）However soon or later, the database will have to write data to disk, and this is what matters from our point of view. That is, data from memory has to be transmitted to the kernel (step 3) at some point.（但是，迟早，数据库将不得不将数据写入磁盘，这对我们而言很重要。也就是说，必须将来自内存的数据传输到内核（步骤3））

Another big omission（省略） of details is about step 3. The reality is more complex since most advanced kernels implement different layers of caching, that usually are the file system level caching (called the page cache in Linux) and a smaller buffer cache that is a buffer containing the data that waits to be committed to the disk.（由于大多数高级内核实现了不同的缓存层，因此实际情况更加复杂。通常是文件系统级缓存（在Linux中称为页面缓存）和较小的缓冲区高速缓存，缓冲区高速缓存是包含等待提交到磁盘的数据的缓冲区） Using special APIs is possible to bypass both (see for instance O_DIRECT and O_SYNC flags of the open system call on Linux). However from our point of view we can consider this as an unique layer of opaque caching (that is, we don't know the details). It is enough to say that often the page cache is disabled when the database already implements its caching to avoid that both the database and the kernel will try to do the same work at the same time (with bad results). The buffer cache is usually never turned off because this means that every write to the file will result into data committed to the disk that is too slow for almost all the applications.

What databases usually do instead is to call system calls that will commit the buffer cache to the disk, only when absolutely needed, as we'll see later in a more detailed way.

#### When is our write safe along the line?
If we consider a failure that involves just the database software (the process gets killed by the system administrator or crashes) and does not touch the kernel, the write can be considered safe just after the step 3 is completed with success, that is after the write(2) system call (or any other system call used to transfer data to the kernel) returns successfully. After this step even if the database process crashes, still the kernel will take care of transferring data to the disk controller.
（假设发生了一次故障，该故障只影响数据库软件（例如进程被系统管理员杀死或崩溃）没有影响到内核，成功完成第3步后，就可以认为此次写入是安全的。在此步骤之后，即使数据库进程崩溃，内核仍然会负责将数据传输到磁盘控制器。）

If we consider instead a more catastrophic event（灾难性的事件） like a power outage, we are safe only at step 5 completion, that is, when data is actually transfered to the physical device memorizing it.（只有在第5步完成时我们才是安全的，也就是说，当数据实际传输到存储它的物理设备时）

We can summarize that the important stages in data safety are the 3, 4, and 5. That is:
- How often the database software will transfer its user-space buffers to the kernel buffers using the write (or equivalent) system call?(数据库软件使用写（或等效）系统调用将用户空间缓冲区转移到内核缓冲区的频率)
- How often the kernel will flush the buffers to the disk controller?(内核多久刷新一次缓冲区到磁盘控制器)
- And how often the disk controller will write data to the physical media?(以及磁盘控制器将数据写入物理介质的频率)

Note: when we talk about disk controller we actually mean the caching performed by the controller or the disk itself. In environments where durability is important system administrators usually disable this layer of caching.(注意：当我们谈论磁盘控制器时，实际上是指由控制器或磁盘本身执行的缓存。在持久性很重要的环境中，系统管理员通常禁用此缓存层。)

Disk controllers by default only perform a write through caching for most systems (i.e. only reads are cached, not writes).(默认情况下，磁盘控制器仅对大多数系统执行直写式高速缓存（即仅高速缓存读操作，而不高速写）) It is safe to enable the write back mode (caching of writes) only when you have batteries or a super-capacitor device protecting the data in case of power shutdown.(仅当电池或超级电容器设备在电源关闭的情况下保护数据时，才可以启用写回模式（写缓存）)
#### POSIX API
From the point of view of the database developer the path that the data follows before being actually written to the physical device is interesting, but even more interesting is the amount of control the programming API provides along the path.

Let's start from step 3. We can use the write system call to transfer data to the kernel buffers, so from this point of view we have a good control using the POSIX API. However we don't have much control about how much time this system call will take before returning successfully. The kernel write buffer is limited in size, if the disk is not able to cope with the application write bandwidth, the kernel write buffer will reach it's maximum size and the kernel will block our write. When the disk will be able to receive more data, the write system call will finally return. After all the goal is to, eventually, reach the physical media.

Step 4: in this step the kernel transfers data to the disk controller. By default it will try to avoid doing it too often, because it is faster to transfer it in bigger pieces. For instance Linux by default will actually commit writes after 30 seconds. This means that if there is a failure, all the data written in the latest 30 seconds can get potentially lost.

The POSIX API provides a family of system calls to force the kernel to write buffers to the disk: the most famous of the family is probably the fsync system call (see also msync and fdatasync for more information). Using fsync the database system has a way to force the kernel to actually commit data on disk, but as you can guess this is a very expensive operation: fsync will initiate a write operation every time it is called and there is some data pending on the kernel buffer. Fsync() also blocks the process for all the time needed to complete the write, and if this is not enough, on Linux it will also block all the other threads that are writing against the same file.(Fsync（）还会在完成写入所需的所有时间内阻止该进程，如果这还不够，那么在Linux上，它也会阻止针对同一文件进行写操作的所有其他线程。)
#### What we can't control
So far we learned that we can control step 3 and 4, but what about 5? Well formally speaking we don't have control from this point of view using the POSIX API. Maybe some kernel implementation will try to tell the drive to actually commit data on the physical media, or maybe the controller will instead re-order writes for the sake of speed, and will not really write data on disk ASAP, but will wait a couple of milliseconds more. This is simply out of our control.

In the rest of this article we'll thus simplify our scenario to two data safety levels:
- Data written to kernel buffers using the write(2) system call (or equivalent) that gives us data safety against process failure.(使用write（2）系统调用（或等效调用）将数据写入内核缓冲区，这使我们可以安全地防止进程失败)
- Data committed to the disk using the fsync(2) system call (or equivalent) that gives us, virtually, data safety against complete system failure like a power outage.(使用fsync（2）系统调用（或等效调用）提交到磁盘的数据实际上使我们免受完全系统故障（例如停电）的数据安全) We actually know that there is no guarantee because of the possible disk controller caching, but we'll not consider this aspect because this is an invariant among all the common database systems. Also system administrators often can use specialized tools in order to control the exact behavior of the physical device.

Note: not all the databases use the POSIX API. Some proprietary database use a kernel module that will allow a more direct interaction with the hardware. However the main shape of the problem remains the same. You can use user-space buffers, kernel buffers, but soon or later there is to write data on disk to make it safe (and this is a slow operation). A notable example of a database using a kernel module is Oracle.
#### Data corruption（数据损坏）
In the previous paragraphs we analyzed the problem of ensuring data is actually transfered to the disk by the higher level layers: the application and the kernel. However this is not the only concern about durability. Another one is the following: is the database still readable after a catastrophic event, or its internal structure can get corrupted in some way so that it may no longer be read correctly, or requires a recovery step in order to reconstruct a valid representation of data?（另一个是：灾难性事件后数据库是否仍然可读，或者数据库的内部结构可能以某种方式损坏，因此可能无法正确读取，或需要恢复步骤以重建数据的有效表示）

For instance many SQL and NoSQL databases implement some form of on-disk tree data structure that is used to store data and indexes.（例如，许多SQL和NoSQL数据库实现某种形式的磁盘树数据结构，用于存储数据和索引。） This data structure is manipulated（操纵） on writes. If the system stops working in the middle of a write operation, is the tree representation still valid?（如果系统在写操作过程中停止工作，树表示是否仍然有效？）

In general there are three levels of safety against data corruption:
- Databases that write to the disk representation not caring about what happens in the event of failure, asking the user to use a replica for data recovery, and/or providing tools that will try to reconstruct a valid representation if possible.（写入磁盘表示形式的数据库不关心发生故障时会发生什么，要求用户使用副本进行数据恢复，和/或提供尽可能尝试重建有效表示形式的工具。）
- Database systems that use a log of operations (a journal) so that they'll be able to recover to a consistent state after a failure.（使用操作日志（日志）的数据库系统，以便在发生故障后能够恢复到一致状态）
- Database systems that never modify already written data, but only work in append only mode, so that no corruption is possible.（数据库系统从不修改已经写入的如数，但只能在“仅追加”模式下工作，因此不可能损坏）

Now we have all the elements we need to evaluate a database system in terms of reliability of its persistence layer. It's time to check how Redis scores in this regard. Redis provides two different persistence options, so we'll examine both one after the other.

#### Snapshotting
Redis snapshotting is the simplest Redis persistence mode. It produces point-in-time snapshots of the dataset when specific conditions are met, for instance if the previous snapshot was created more than 2 minutes ago and there are already at least 100 new writes, a new snapshot is created. This conditions can be controlled by the user configuring the Redis instance, and can also be modified at runtime without restarting the server. Snapshots are produced as a single .rdb file that contains the whole dataset.

The durability of snapshotting is limited to what the user specified as save points. If the dataset is saved every 15 minutes, than in the event of a Redis instance crash or a more catastrophic event, up to 15 minutes of writes can be lost. From the point of view of Redis transactions snapshotting guarantees that either a MULTI/EXEC transaction is fully written into a snapshot, or it is not present at all (as already said RDB files represent exactly point in time images of the dataset).

The RDB file can not get corrupted, because it is produced by a child process in an append-only way, starting from the image of data in the Redis memory.（RDB文件不会损坏，因为它从一个子进程中采用追加的方式生成，） A new rdb snapshot is created as a temporary file, and gets renamed into the destination file using the atomic rename(2) system call once it was successfully generated by a child process (and only after it gets synched on disk using the fsync system call).

Redis snapshotting does NOT provide good durability guarantees if up to a few minutes of data loss is not acceptable in case of incidents, so it's usage is limited to applications and environments where losing recent data is not very important.

However even when using the more advanced persistence mode that Redis provides, called "AOF", it is still advisable to also turn snapshotting on, because to have a single compact RDB file containing the whole dataset is extremely useful to perform backups, to send data to remote data centers for disaster recovery, or to easily roll-back to an old version of the dataset in the event of a dramatic software bug that compromised the content of the database in a serious way.（但是，即使使用Redis提供的更高级的持久性模式，也称为“ AOF”，还是建议您同时打开快照，因为只有一个包含整个数据集的紧凑型RDB文件对于执行备份非常有用，将数据发送到远程数据中心以进行灾难恢复，或者在发生严重的软件错误严重破坏数据库内容的情况下轻松回滚到数据集的旧版本）

It's worth to note that RDB snapshots are also used by Redis when performing a master -> slave synchronization.（值得注意的是，Redis在执行主->从同步时也使用RDB快照。）

One of the additional benefits of RDB is the fact for a given database size, the number of I/Os on the system is bound, whatever the activity on the database is. This is a property that most traditional database systems (and the Redis other persistence, the AOF) do not have.（RDB的其他好处之一是，对于给定的数据库大小，无论系统上的活动是什么，都必须绑定系统上的I / O数量。这是大多数传统数据库系统（以及Redis其他持久性，AOF）所不具备的属性）

#### Append only file
The Append Only File, usually called simply AOF, is the main Redis persistence option. The way it works is extremely simple: every time a write operation that modifies the dataset in memory is performed, the operation gets logged. The log is produced exactly in the same format used by clients to communicate with Redis, so the AOF can be even piped via netcat to another instance, or easily parsed if needed. At restart Redis re-plays all the operations to reconstruct the dataset.（日志的生成格式与客户端与Redis进行通信所使用的格式完全相同，因此AOF甚至可以通过netcat通过管道传输到另一个实例，或者在需要时轻松进行解析。重新启动时，Redis重播所有操作以重建数据集）

To show how the AOF works in practice I'll do a simple experiment, setting up a new Redis 2.6 instance with append only file enabled:
```
```
Now it's time to send a few write commands to the instance:
```
redis 127.0.0.1:6379> set key1 Hello
OK
redis 127.0.0.1:6379> append key1 " World!"
(integer) 12
redis 127.0.0.1:6379> del key1
(integer) 1
redis 127.0.0.1:6379> del non_existing_key
(integer) 0
```
The first three operations actually modified the dataset, the fourth did not, because there was no key with the specified name. This is how our append only file looks like:

```
$ cat appendonly.aof 
*2
$6
SELECT
$1
0
*3
$3
set
$4
key1
$5
Hello
*3
$6
append
$4
key1
$7
 World!
*2
$3
del
$4
key1
```
As you can see the final DEL is missing, because it did not produced any modification to the dataset.

It is as simple as that, new commands received will get logged into the AOF, but only if they have some effect on actual data. However not all the commands are logged as they are received. For instance blocking operations on lists are logged for their final effects as normal non blocking commands. Similarly INCRBYFLOAT is logged as SET, using the final value after the increment as payload, so that differences in the way floating points are handled by different architectures will not lead to different results after reloading an AOF file.

So far we know that the Redis AOF is an append only business, so no corruption is possible. However this desirable feature can also be a problem: in the above example after the DEL operation our instance is completely empty, still the AOF is a few bytes worth of data. The AOF is an always growing file, so how to deal with it when it gets too big?

#### AOF rewrite
When an AOF is too big Redis will simply rewrite it from scratch in a temporary file. The rewrite is NOT performed by reading the old one, but directly accessing data in memory, so that Redis can create the shortest AOF that is possible to generate, and will not require read disk access while writing the new one.(重写不是通过读取旧的AOF来执行的，而是直接访问内存中的数据，以便Redis可以创建可能生成的最短的AOF，并且在写入新的AOF时不需要读取磁盘。)

Once the rewrite is terminated, the temporary file is synched on disk with fsync and is used to overwrite the old AOF file.(重写终止后，临时文件将使用fsync同步到磁盘上，并用于覆盖旧的AOF文件。)  

You may wonder what happens to data that is written to the server while the rewrite is in progress. This new data is simply also written to the old (current) AOF file, and at the same time queued into an in-memory buffer, so that when the new AOF is ready we can write this missing part inside it, and finally replace the old AOF file with the new one.(只需将新数据也写入旧的（当前）AOF文件中，并同时排队到内存缓冲区中，这样，当新的AOF准备就绪时，我们可以在其中写入缺少的部分，最后用新的AOF文件替换旧的AOF文件)

As you can see still everything is append only, and when we rewrite the AOF we still write everything inside the old AOF file, for all the time needed for the new to be created. This means that for our analysis we can simply avoid considering the fact that the AOF in Redis gets rewritten at all. So the real question is, how often we write(2), and how often we fsync(2).(如您所见，所有内容仅是追加内容，并且当我们重写AOF时，我们仍然将所有内容写入旧的AOF文件中，直到创建新的AOF文件为止.这意味着对于我们的分析，我们可以完全避免考虑Redis中的AOF完全被重写的事实。因此，真正的问题是，我们多久编写一次（2），以及我们多久进行一次fsync（2）)

AOF rewrites are generated only using sequential I/O operations, so the whole dump process is efficient even with rotational disks (no random I/O is performed.（因此，即使使用旋转磁盘，整个转储过程也是有效的（不执行随机I / O）。） This is also true for RDB snapshots generation. The complete lack of Random I/O accesses is a rare feature among databases, and is possible mostly because Redis serves read operations from memory, so data on disk does not need to be organized for a random access pattern, but just for a sequential loading on restart.（完全缺乏随机I / O访问是数据库中很少见的功能，这可能是因为Redis从内存中读取操作，因此，磁盘上的数据不需要为随机访问模式进行组织，而只需在重新启动时进行顺序加载即可。）

#### AOF durability

This whole article was written to reach this paragraph. I'm glad I'm here, and I'm even more glad you are still here with me.

The Redis AOF uses an user-space buffer that is populated with new data as new commands are executed. The buffer is usually flushed on disk every time we return back into the event loop, using a single write(2) call against the AOF file descriptor, but actually there are three different configurations that will change the exact behavior of write(2), and especially, of fsync(2) calls.

This three configurations are controlled by the appendfsync configuration directive, that can have three different values: no, everysec, always. This configuration can also be queried or modified at runtime using the CONFIG SET command, so you can alter it every time you want without stopping the Redis instance

#### appendfsync no
In this configuration Redis does not perform fsync(2) calls at all.（在此配置中，Redis根本不执行fsync（2）调用。） However it will make sure that clients not using pipelining, that is, clients that wait to receive the reply of a command before sending the next one, will receive an acknowledge that the command was executed correctly only after the change is transfered to the kernel by writing the command to the AOF file descriptor, using the write(2) system call.

Because in this configuration fsync(2) is not called at all, data will be committed to disk at kernel's wish, that is, every 30 seconds in most Linux systems.

#### appendfsync everysec
in this configuration data will be both written to the file using write(2) and flushed from the kernel to the disk using fsync(2) one time every second. Usually the write(2) call will actually be performed every time we return to the event loop, but this is not guaranteed.

However if the disk can't cope with the write speed, and the background fsync(2) call is taking longer than 1 second, Redis may delay the write up to an additional second (in order to avoid that the write will block the main thread because of an fsync(2) running in the background thread against the same file descriptor). If a total of two seconds elapsed without that fsync(2) was able to terminate, Redis finally performs a (likely blocking) write(2) to transfer data to the disk at any cost.

So in this mode Redis guarantees that, in the worst case, within 2 seconds everything you write is going to be committed to the operating system buffers and transfered to the disk. In the average case data will be committed every second.(因此，在这种模式下，Redis保证在最坏的情况下，在2秒钟之内，您写入的所有内容都将提交给操作系统缓冲区并传输到磁盘。通常情况下，每秒将提交一次数据）

#### appednfsync always
In this mode, and if the client does not use pipelining but waits for the replies before issuing new commands, data is both written to the file and synched on disk using fsync(2) before an acknowledge is returned to the client.(在将确认返回给客户端之前，数据既被写入文件，又使用fsync（2）在磁盘上同步。)

This is the highest level of durability that you can get, but is slower than the other modes.

The default Redis configuration is appendfsync everysec that provides a good balance between speed (is almost as fast as appendfsync no) and durability.

What Redis implements when appendfsync is set to always is usually called group commit. This means that instead of using an fsync call for every write operation performed, Redis is able to group this commits in a single write+fsync operation performed before sending the request to the group of clients that issued a write operation during the latest event loop iteration.

In practical terms it means that you can have hundreds of clients performing write operations at the same time: the fsync operations will be factorized - so even in this mode Redis should be able to support a thousand of concurrent transactions per second while a rotational device can only sustain 100-200 write op/s.(fsync操作将被分解-因此，即使在这种模式下，Redis也应该能够每秒支持一千个并发事务，而旋转设备只能维持100-200个写op/s。)

This feature is usually hard to implement in a traditional database, but Redis makes it remarkably more simple.
#### Why is pipelining different?
The reason for handling clients using pipelining in a different way is that clients using pipelining with writes are sacrificing the ability to read what happened with a given command, before executing the next one, in order to gain speed. There is no point in committing data before replying to a client that seems not interested in the replies before going forward, the client is asking for speed. However even if a client is using pipelining, writes and fsyncs (depending on the configuration) always happen when returning to the event loop.

#### AOF and Redis transactions
AOF guarantees a correct MULTI/EXEC transactions semantic, and will refuse to reload a file that contains a broken transaction at the end of the file. An utility shipped with the Redis server can trim the AOF file to remove the partial transaction at the end.

Note: since the AOF file is populated using a single write(2) call at the end of every event loop iteration, an incomplete transaction can only appear if the disk where the AOF resides gets full while Redis is writing.

#### Comparison with PostrgreSQL
So how durable is Redis, with its main persistence engine (AOF) in its default configuration?  
- Worst case: It guarantees that write(2) and fsync(2) are performed within two seconds.   
- Normal case: it performs write(2) before replying to client, and performs an fsync(2) every second.   
What is interesting is that in this mode Redis is still extremely fast, for a few reasons. One is that fsync is performed on a background thread, the other is that Redis only writes in append only mode, that is a big advantage.

However if you need maximum data safety and your write load is not high, you can still have the best of the durability that is possible to obtain in any database system using fsync always.

How this compares to PostgreSQL, that is (with good reasons) considered a good and very reliable database?

Let's read some PostgreSQL documentation together (note, I'm only citing the interesting pieces, you can find the full documentation here in the PostgreSQL official site)

fsync (boolean)

If this parameter is on, the PostgreSQL server will try to make sure that updates are physically written to disk, by issuing fsync() system calls or various equivalent methods (see wal_sync_method). This ensures that the database cluster can recover to a consistent state after an operating system or hardware crash.

[snip]

In many situations, turning off synchronous_commit for noncritical transactions can provide much of the potential performance benefit of turning off fsync, without the attendant risks of data corruption.


So PostgreSQL needs to fsync data in order to avoid corruptions. Fortunately with Redis AOF we don't have this problem at all, no corruption is possible. So let's check the next parameter, that is the one that more closely compares with Redis fsync policy, even if the name is different:

##### synchronous_commit (enum)

Specifies whether transaction commit will wait for WAL records to be written to disk before the command returns a "success" indication to the client. Valid values are on, local, and off. (指定在命令向客户端返回“成功”指示之前，事务提交是否将等待WAL记录写入磁盘。有效值为on，local和off。)The default, and safe, value is on. When off, there can be a delay between when success is reported to the client and when the transaction is really guaranteed to be safe against a server crash. (The maximum delay is three times wal_writer_delay.) Unlike fsync, setting this parameter to off does not create any risk of database inconsistency: an operating system or database crash might result in some recent allegedly-committed transactions being lost, but the database state will be just the same as if those transactions had been aborted cleanly.

Here we have something much similar to what we can tune with Redis. Basically the PostgreSQL guys are telling you, want speed? Probably it is a good idea to disable synchronous commits. That's like in Redis: want speed? Don't use appendfsync always.

Now if you disable synchronous commits in PostgreSQL you are in a very similar affair as with Redis appendfsync everysec, because by default wal_writer_delay is set to 200 milliseconds, and the documentation states that you need to multiply it by three to get the actual delay of writes, that is thus 600 milliseconds, very near to the 1 second Redis default.

MySQL InnoDB has similar parameters the user can tune. From the documentation:

If the value of innodb_flush_log_at_trx_commit is 0, the log buffer is written out to the log file once per second and the flush to disk operation is performed on the log file, but nothing is done at a transaction commit. When the value is 1 (the default), the log buffer is written out to the log file at each transaction commit and the flush to disk operation is performed on the log file. When the value is 2, the log buffer is written out to the file at each commit, but the flush to disk operation is not performed on it. However, the flushing on the log file takes place once per second also when the value is 2. Note that the once-per-second flushing is not 100% guaranteed to happen every second, due to process scheduling issues.

You can read more here.


Long story short: even if Redis is an in memory database it offers good durability compared to other on disk databases.

From a more practical point of view Redis provides both AOF and RDB snapshots, that can be enabled simultaneously (this is the advised setup, when in doubt), offering at the same time easy of operations and data durability.

Everything we said about Redis durability can also be applied not only when Redis is used as a datastore but also when it is used to implement queues that needs to persist on disk with good durability.
