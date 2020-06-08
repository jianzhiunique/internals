# Publishing Messages into RabbitMQ 发布消息到RabbitMQ#

One of the best ways to cover the various parts of RabbitMQ's
architecture is to see what happens when a message gets published into
the broker. In this document we are going to visit the different
subsystems a message crosses inside the broker. Let's start by
`rabbit_reader`.

最好的覆盖RabbitMQ架构的方式是看消息被发送到服务器之后发生了什么。在这篇文档里，
我们将访问服务器内部不同的子系统。让我们从`rabbit_reader`开始。

The `rabbit_reader` process is the one that takes care of reading data
from the network and forwarding it to the respective channel
process. Messages get into the channel when the reader calls the
function `rabbit_channel:do_flow/3`, this function will call the
`credit_flow` module to track that a message was received from the
reader, so it could eventually throttle down the reader in case the
message publisher is sending more messages in than the amount the
broker can handle at a particular time. Read more about Credit Flow
[here](./credit_flow.md). More information about the Reader process
can be found in the
[Networking and Connections guide](./networking_and_connections.md#rabbit_reader).

`rabbit_reader`进程是负责从网络中读取数据，然后将它转发到相应channel中的程序。
消息通过reader调用`rabbit_channel:do_flow/3`进入channel。这个方法将会调用
`credit_flow`模块来追踪从reader中接收的消息，所以可能降低reader的速度，当发布者
发送比服务器在特定时间内可以处理的数量还多的消息时。在Credit Flow中阅读更多。更多
与reader进程相关的信息可以在[网络和连接指南](./networking_and_connections.md#rabbit_reader).中找到。

## Arriving into the Channel Process 到达Channel进程##

Once Credit Flow is accounted for, then the `do_flow/3` function will
issue an asynchronous `gen_server:cast/2` into the channel process
passing in this Erlang message: `{method, Method, Content,
flow}`. There we have the AMQP `Method`, then method `Content`, and
the atom `flow` indicating the channel that credit flow is in use.

一旦Credit Flow被计入，`do_flow/3`方法将引发一个异步的`gen_server:cast/2`到channel进程，
传递这样的Erlang消息`{method, Method, Content,
flow}`。这里包含AMQP的`Method`、方法的`Content`，以及一个`flow`代表channel正在使用credit flow

When the cast reaches the `handle_cast/2` function inside the channel
module, we are finally inside the channel process memory and execution
path. If `flow` was in use, as is the case here, then the channel will
issue a `credit_flow:ack/1` to the reader process. Then the AMQP
method that's being processed will be passed to the
[Interceptor](./interceptors.md) defined for the channel, in case
there are any. After the Interceptors are done processing the AMQP
method, then the channel process will continue processing the method,
in our case the function `handle_method/3` will be called, with a
`basic.publish` record.

当转换到达channel中的`handle_cast/2`模块时，我们最终位于channel进程的内存和执行路径中。如果`flow`
在使用中，正如此处的情况，channel将引发一个`credit_flow:ack/1`给reader进程，然后正在处理的AMQP方法
将会传递给channel定义的[拦截器](./interceptors.md)，如果有的话。在拦截器处理完成后，channel进程
将会继续处理这个方法，在当前的情况下，`handle_method/3`将会被调用，并带有一个`basic.publish`记录。

## Inside basic.publish basic.publish内部##

basic.publish works by receiving an AMQP message, an Exchange and a
Routing Key, and it will use the exchange to route the message to one
or various queues, based on the routing key. Let's see how's that
accomplished.

basic.publish的工作是接收AMQP消息、Exchange、Routing Key，他将使用exchange来路由消息
到一个或多个队列，基于routing key，让我们看看他是如何实现的

The first thing the function does is to check the size of the message 
since RabbitMQ has an upper limit of 2GB for messages.

方法首先要做的事情是检查消息的大小，因为RabbitMQ有一个2GB的消息上限。

Then the function needs to build the resource record for the
Exchange. Exchanges and Queues are represented internally with a
resource record that keeps track of the name, and the vhost where the
exchange or queue was declared. The type declaration record looks like
this:

然后方法需要组建资源记录给exchange，exchange和queues在内部被资源记录所代表，资源记录中保留
了名称、exchange和queue被声明的vhost。声明的记录像是下面的样子

```erlang
#resource{virtual_host :: VirtualHost,
          kind         :: Kind,
          name         :: Name}
```

So if a message was published to the default vhost to an exchange
called `"my_exchange"`, we will end up with the following record:

所以如果消息是被发送到默认vhost和叫做my_exchange的exchange时，最终将用以下记录：

```erlang
#resource{virtual_host = <<"/">>
          kind         = exchange,
          name         = <<"my_exchange">>}
```

Resources like that one are used everywhere in RabbitMQ, so it's a
good idea to study their parts in the
[rabbit_types](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_types.erl)
module where this declarations are defined.

像这样的资源在RabbitMQ中到处使用，所以学习rabbit_types模块中研究他们的各组成部分。

Once we have the exchange record, `basic.publish` will use it to see
if the user publishing the message has write permissions to this
particular exchange by calling the function
`check_write_permitted/2`. Read more about the different kind of
permissions here:
[access-control](https://www.rabbitmq.com/access-control.html)

当我们获得exchange记录后， `basic.publish`会使用它来调用`check_write_permitted/2`，
看用户发布的消息是否有指定的exchange的写权限。在[权限控制](https://www.rabbitmq.com/access-control.html)阅读更多不同的权限

If the user does have permission to publish messages to this exchange,
then the channel will query the Mnesia database trying to find out if
the exchange actually exists, so the function
`rabbit_exchange:lookup_or_die/1` is called in order to retrieve the
actual exchange record from the database, if the exchange is not found,
then a channel error is raised by `lookup_or_die/1`. Keep in mind that
one thing is the exchange resource we mentioned above, and another
much different is the exchange record stored in mnesia. The latter
holds up much more information about the actual exchange, like it's
type for example (direct, fanout, topic, etc). Here's the exchange
record definition from `rabbit.hrl`:

如果用户确实拥有发布到exchange消息的权限，channel将会查询Mnesia数据库来找到exchange是否已经存在，
所以`rabbit_exchange:lookup_or_die/1`被调用，来获取数据库中真正的exchange 记录，如果exchange不存在，
`lookup_or_die/1`会引发一个channel错误。记住，我们上面提到的exchange资源，和mnesia中一个exchange记录
是不同的，后者持有真正exchange的更多信息，像是他的类型（direct, fanout, topic等）。
这里是`rabbit.hrl`中exchange记录定义：

```erlang
%% fields described as 'transient' here are cleared when writing to
%% rabbit_durable_<thing>
-record(exchange, {
          name, type, durable, auto_delete, internal, arguments, %% immutable
          scratches,    %% durable, explicitly updated via update_scratch/3
          policy,       %% durable, implicitly updated when policy changes
          decorators}). %% transient, recalculated in store/1 (i.e. recovery)
```

Then we need to check that the record returned by Mnesia is not an 
internal exchange, otherwise an error will be raised and the publish 
will fail.

然后我们需要检查Mnesia返回的记录中不是内部exchange，否则将会触发一个错误，发布会失败

The next thing to do is to validate the user id provided with the
basic publish, if any. If provided, this user id has to be validated
against the user that created the channel where the message is being
published. More details
[here](https://www.rabbitmq.com/validated-user-id.html)

下一步是验证提供的用户ID，如果有的话，则必须验证用户id与创建channel的用户是否一致，
在消息发布时。[更多细节](https://www.rabbitmq.com/validated-user-id.html)

Then we need to validate if the message expiration header that the
user provided is correct. More info about the Per-Message-TTL


然后我们需要验证用户提供的消息的expiration header是否正确。
[Per-Message-TTL](https://www.rabbitmq.com/ttl.html#per-message-ttl)

Then it's time to check if the message was published as _Mandatory_ or
if the channel is in _Transaction_ or _Confirm Mode_. If this is the
case, then the `publish_seqno` field on the channel state will be
incremented to account for the new publish that's being handled. This
Message Sequence Number will be later used to reply back to the
publisher in case the message was Mandatory and/or the channel was in
[Confirm Mode](https://www.rabbitmq.com/confirms.html). See also the
document [Delivering Messages to Queues](./deliver_to_queues.md).

然后需要检查消息在发布时是否包含_Mandatory_、或者channel在_Transaction_、_Confirm Mode_下。
如果是这种情况，channel状态的`publish_seqno`字段将会自增，来计量新的发布正在处理。
这个消息序列号将在之后被使用，回复给发布者，在消息是Mandatory，且/或 channel是[Confirm Mode](https://www.rabbitmq.com/confirms.html)。
同时阅读[投递消息到队列](./deliver_to_queues.md)文档

After all these steps have been completed, it's time to route the AMQP
message, but in order to do that we need to wrap the message first
into a `#basic_message` record, and then pass it to the exchange and
queues as a `#delivery{}` record:

在以上所有的步骤完成后，该路由AMQP消息了，但是为了完成路由，我们需要首先包装消息为`#basic_message`记录，
然后作为`#delivery{}`记录，传递给exchange和queue。

```erlang
-record(basic_message,
        {exchange_name,     %% The exchange where the message was received
         routing_keys = [], %% Routing keys used during publish
         content,           %% The message content
         id,                %% A `rabbit_guid:gen()` generated id
         is_persistent}).   %% Whether the message was published as persistent

-record(delivery,
        {mandatory,  %% Whether the message was published as mandatory
         confirm,    %% Whether the message needs confirming
         sender,     %% The pid of the process that created the delivery
         message,    %% The #basic_message record
         msg_seq_no, %% Msg Sequence Number from the channel publish_seqno field
         flow}).     %% Should flow control be used for this delivery
```

## Message Routing ##

The `#delivery` we just created on the previous step is now passed to
the exchange via the function `rabbit_exchange:route/2`. If the
exchange name used during `basic.publish` is the empty string
`<<"">>`, then the `default` exchange is assumed, and the `route/2`
will just return the queue name associated with the routing key, per
AMQP spec. If that's not the case, then the delivery will be processed
first by the [exchange decorators](./exchange_decorators.md) that are
configured to the exchange that's handling the routing. The decorators
will send back a list of _destinations_. At this point, delivery will
finally reach the exchange, where the routing algorithm implemented by
the exchange will take place. This process will return a new list of
_destinations_ which will be merged and deduplicated with the list
returned before by the decorators. At this point, all the destinations
proposed by the
[Exchange To Exchange](https://www.rabbitmq.com/e2e.html) bindings are
also included in the list of destinations that will be returned to the
channel.

我们刚刚创建的`#delivery`现在通过方法`rabbit_exchange:route/2`传递给exchange，
如果`basic.publish`的exchange名称是空串`<<"">>`，那么`default`exchange被假定，
`route/2`，根据AMQP协议，将直接返回与routingkey相关的队列名。如果不是这种情况，那么
投递将首先被配置的exchange的[exchange decorators](./exchange_decorators.md)处理，
装饰器将发送回一个_destinations_列表。此时，投递将最终到达exchange，这是路由算法
实现的地方。这个进程将返回一个新的目标列表，将与装饰器之前所返回的列表进行合并。此时，所有
[Exchange To Exchange](https://www.rabbitmq.com/e2e.html)绑定的也被包含在目标列表中，
然后返回给channel

## Processing Routing Results 处理路由结果##

Now the channel has a list of queues to which it should deliver the
messages. Before doing that, we need to see if the channel is in
transaction mode, if that's the case, then the `#delivery` and the
list of queues are enqueued for later until the transaction is
committed. Keep in mind that transaction support in RabbitMQ are a
very simple form of
[message batching](https://www.rabbitmq.com/semantics.html). If the
channel is not in transaction mode, then the message will be delivered
to the queues returned by the routing function.

现在channel有了一个queues列表，他应该把消息投递过去。在投递之前，我们需要看一下channel是否在
事务模式下，如果是，`#delivery`和queues列表将会放入队列中供之后使用，直到事务被提交。记住RabbitMQ的事务支持
是[消息批处理](https://www.rabbitmq.com/semantics.html)的一种简单的形式。
如果channel不是事务模式，消息将会被投递到由路由方法返回的列表中。


## Summary 总结##

We saw in this guide that messages arrive via the network into the
`rabbit_reader` process. This process forwards commands to
`rabbit_channel` processes who take care of processing the various
AMQP methods. In this case, we are seeing what happens when a message
is published into RabbitMQ. Once credit flow has been acked back to
the reader process, then it's time to take care of handling the
message. First it will go to the interceptors, who might modify or
augment the AMQP method received from the _reader_. Then the channel
must make sure the message complies to the size limits set at the
broker side. Once that's done, we need to see if the user has
permission to publish message to the selected exchange. If that's fine
and the `user_id` and `expiration` headers of the message are
validated, then it's time to route the message. The exchange who
handles the message will return back a list of queues to which the
message must be delivered to. At this point we are done with the
message and the channel is ready to keep processing commands.

我们看到，在此指南中，通过网络到达的消息，被`rabbit_reader`程序接收，然后转发命令给
`rabbit_channel`进程，这是考虑处理多种AMQP方法的，在这种情况下，我们看到了消息发送至RabbitMQ后
发生了什么。一旦信用额流被确认返回给reader进程，那么就该考虑处理消息了。首先，他将进入拦截器，
这里可能会修改或增强从_reader_接收的AMQP方法。然后channel必须确保消息符合服务端的消息大小限制。
这些结束后，我们需要查看用户是否有发往exchange的权限，如果ok并且`user_id` 和 消息`expiration`
header都合法，那么就该路由消息了，处理消息的exchange将返回一个消息应该投递到的queues列表，
此时我们完成了消息处理，channel准备好继续处理命令。

Now we can continue with the next guide and see what happens when
messages are delivered to queues:
[Delivering Messages to Queues](./deliver_to_queues.md)

现在我们继续阅读下一个指南来看看消息被投递到队列时发生了什么。
[Delivering Messages to Queues](./deliver_to_queues.md)