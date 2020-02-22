## 所有资料
[Redis Mass Insertion](https://redis.io/topics/mass-insert)
## [Redis Mass Insertion](https://redis.io/topics/mass-insert)
### Redis Mass Insertion
Sometimes Redis instances need to be loaded with a big amount of preexisting or user generated data in a short amount of time, so that millions of keys will be created as fast as possible.(有时Redis实例需要在短时间内加载大量预先存在或用户生成的数据，以便尽可能快地创建数百万个key)

This is called a mass insertion, and the goal of this document is to provide information about how to feed Redis with data as fast as possible.(这称为批量插入，并且本文档的目的是提供有关如何尽快为Redis提供数据的信息。)
#### Use the protocol, Luke
Using a normal Redis client to perform mass insertion is not a good idea for a few reasons: the naive approach of sending one command after the other is slow because you have to pay for the round trip time for every command. It is possible to use pipelining, but for mass insertion of many records you need to write new commands while you read replies at the same time to make sure you are inserting as fast as possible.(由于以下几个原因，使用普通的Redis客户端执行批量插入并不是一个好主意：在一个命令之后发送一个命令的方法很慢，因为您必须为每个命令的往返时间会耗费时间。可以使用流水线操作，但是要大量插入许多记录，您需要在读取恢复健康的同时写入新命令，以确保尽快插入。)

Only a small percentage of clients support non-blocking I/O, and not all the clients are able to parse the replies in an efficient way in order to maximize throughput.（只有一小部分客户端支持非阻塞I / O，并且并非所有客户端都能够以有效的方式解析答复以最大化吞吐量。） For all of these reasons the preferred way to mass import data into Redis is to generate a text file containing the Redis protocol, in raw format, in order to call the commands needed to insert the required data.

For instance if I need to generate a large data set where there are billions of keys in the form: `keyN -> ValueN' I will create a file containing the following commands in the Redis protocol format:
```
SET Key0 Value0
SET Key1 Value1
...
SET KeyN ValueN
```
Once this file is created, the remaining action is to feed it to Redis as fast as possible. In the past the way to do this was to use the netcat with the following command:
```
(cat data.txt; sleep 10) | nc localhost 6379 > /dev/null

```
However this is not a very reliable way to perform mass import because netcat does not really know when all the data was transferred and can't check for errors.（但是，这不是执行批量导入的可靠方法，因为netcat并不真正知道何时传输所有数据，也无法检查错误。） In 2.6 or later versions of Redis the redis-cli utility supports a new mode called pipe mode that was designed in order to perform mass insertion.

Using the pipe mode the command to run looks like the following:
```
cat data.txt | redis-cli --pipe

```
That will produce an output similar to this:
```
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 1000000
```
The redis-cli utility will also make sure to only redirect errors received from the Redis instance to the standard output.
#### Generating Redis Protocol
The Redis protocol is extremely simple to generate and parse, and is Documented here. However in order to generate protocol for the goal of mass insertion you don't need to understand every detail of the protocol, but just that every command is represented in the following way:
```
*<args><cr><lf>
$<len><cr><lf>
<arg0><cr><lf>
<arg1><cr><lf>
...
<argN><cr><lf>
```
Where <cr> means "\r" (or ASCII character 13) and <lf> means "\n" (or ASCII character 10).

For instance the command SET key value is represented by the following protocol:
```
*3<cr><lf>
$3<cr><lf>
SET<cr><lf>
$3<cr><lf>
key<cr><lf>
$5<cr><lf>
value<cr><lf
```
Or represented as a quoted string:
```
"*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n"

```
The file you need to generate for mass insertion is just composed of commands represented in the above way, one after the other.

The following Ruby function generates valid protocol:
```
def gen_redis_proto(*cmd)
    proto = ""
    proto << "*"+cmd.length.to_s+"\r\n"
    cmd.each{|arg|
        proto << "$"+arg.to_s.bytesize.to_s+"\r\n"
        proto << arg.to_s+"\r\n"
    }
    proto
end

puts gen_redis_proto("SET","mykey","Hello World!").inspect
```
Using the above function it is possible to easily generate the key value pairs in the above example, with this program:
```
(0...1000).each{|n|
    STDOUT.write(gen_redis_proto("SET","Key#{n}","Value#{n}"))
}
```
We can run the program directly in pipe to redis-cli in order to perform our first mass import session.
```
$ ruby proto.rb | redis-cli --pipe
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 1000
```
#### How the pipe mode works under the hood ()
The magic needed inside the pipe mode of redis-cli is to be as fast as netcat and still be able to understand when the last reply was sent by the server at the same time.（redis-cli的管道模式内部需要的magic是与netcat一样快，并且仍然能够理解服务器何时同时发送了最后一个答复。）

This is obtained in the following way:

- redis-cli --pipe tries to send data as fast as possible to the server.
- At the same time it reads data when available, trying to parse it.
- Once there is no more data to read from stdin, it sends a special ECHO command with a random 20 bytes string: we are sure this is the latest command sent, and we are sure we can match the reply checking if we receive the same 20 bytes as a bulk reply.
- Once this special final command is sent, the code receiving replies starts to match replies with these 20 bytes. When the matching reply is reached it can exit with success.

Using this trick we don't need to parse the protocol we send to the server in order to understand how many commands we are sending, but just the replies.（使用此技巧，我们不需要解析我们发送到服务器的协议就可以了解我们要发送多少命令，而只需了解答复即可。）

However while parsing the replies we take a counter of all the replies parsed so that at the end we are able to tell the user the amount of commands transferred to the server by the mass insert session.（但是，在解析答复时，我们会对所有已解析的答复进行计数，以便最后我们能够告诉用户通过大容量插入会话传输到服务器的命令数量。）


#### summary
