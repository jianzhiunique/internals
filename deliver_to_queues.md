# Deliver To Queues 投递到队列#

In this document we will be going over quite a few RabbitMQ modules,
since a message crosses all of these once it enters the broker:

这篇文档中我们将会介绍许多RabbitMQ模块，因为一旦消息进入代理，将会穿越所有这些组件。

```
rabbit_reader -> rabbit_channel -> rabbit_amqqueue -> delegate -> rabbit_amqqueue_process -> rabbit_backing_queue
```

Let's see this process in more detail.

让我们更详细的了解这个过程。

The process of delivering messages to queues start during
`basic.publish`, right after the channel receives the result from
calling `rabbit_exchange:route/2`.

将消息投递到队列的过程开始于`basic.publish`，在channel接收到`rabbit_exchange:route/2`
调用的结果之后。

First we need to lookup the list of `#amqqueue` records based on the
destinations obtained from `route/2`. These records will be passed to
the function `rabbit_amqqueue:deliver/2` where they will be used to
obtain the _pids_ of the queue process where the message is going to
be delivered. Once the master and slave pids have been obtained, then
the message can start its way to be delivered to a queue process,
which consists of two parts: accounting for credit flow, and casting
the message into the queue process.

首先我们需要根据从`route/2`获得的目标，查找`#amqqueue`记录列表，记录将会被传递到
`rabbit_amqqueue:deliver/2`方法，在这里，记录将被用于获取队列进程的_pids_，这些进程
就是消息将要被投递到的地方。一旦master 和 slave的pid被获取到，然后消息可以开始他的投递路程，
这包含两个部分：计入信用流、将消息投射到队列进程里。

If the message delivery arrived with `flow = true`, then `credit_flow`
must be accounted for this message. One credit for each master Pid
where the message should arrive, plus one credit for each slave pid
that receives the message.

如果消息投递到来是带有`flow = true`的，那么`credit_flow`必须为这条消息计算。
消息要到达的master pid计1个信用、每个接收到消息slave Pid计1个信用。

Then the message delivery will be sent to master pids and slave pids,
via the `delegate` framework. The Erlang message will have this shape:

然后消息投递将会被发送至master pids和slave pids，通过`delegate`框架。Erlang消息将会是这个样子

```erlang
{deliver,            %% message tag
 Delivery,           %% The Delivery record
 SlaveWhenPublished} %% The Pid that received the message, was it a
                     %% slave when the deliver was published? This is
                     %% used in case of slave promotion
```

You can learn more about the delegate framework
[here](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/delegate.erl#L19).

你可以在[委托框架](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/delegate.erl#L19)中学到更多。

## AMQQueue Process Message Handling AMQQueue进程处理消息##

At this point the message delivery will finally arrive at the queue
process, implemented as a `gen_server2` callback inside the
`rabbit_amqqueue_process` module. The message from the delegate
framework will be received by the `handle_cast/2` callback. This
callback will ack the `credit_flow` issued in above, and it will
monitor the message sender. The message sender is usually the
`rabbit_channel` that received the process. This pid is tracked using
the
[pmon module](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/pmon.erl). The
state is kept as part of the `senders` field in the gen_server state
record. Once the message sender is accounted for the delivery is
passed to the function `deliver_or_enqueue/3`. There is where the
message will either be sent to a consumer or enqueued into the backing
queue.

此时，消息投递最终到达了队列进程，该进程作为`rabbit_amqqueue_process`模块内部的一个
`gen_server2`回调实现。来自有委托框架中的消息将会被`handle_cast/2`回调接收，这个回调
将会确认（ack）上文提到的`credit_flow`，然后它将监控消息的发送者。消息发送者通常是接收进程的
`rabbit_channel`，这个pid是通过[pmon module](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/pmon.erl)
进行追踪的。状态是被保留在gen_server 状态记录的`senders`字段里的。一旦消息的发送者被确认，
消息将传递到`deliver_or_enqueue/3`方法，在这里消息将被发送到消费者或排队到支持队列（也可以说底层队列）里。

### Mandatory Message Handling 强制消息处理###

The first thing `deliver_or_enqueue/3` does is to account for the
mandatory flag of the delivery. If the message was published as
mandatory, then at this point the queue process will consider the
message as routed to the queue. To that effect, the queue process will
cast the message `{mandatory_received, MsgSeqNo}` to the channel pid
that received the delivery. The channel process will the proceed to
forget the message, since from the point of view mandatory message
handling, there isn't anything left to do for that particular
delivery.

`deliver_or_enqueue/3`首先要做的事情是考虑交付此次投递的强制标志（mandatory flag ）。
如果消息是按照强制发布的，那么此时队列进程将考虑消息已经被路由到了队列。为此，队列进程
将投递消息`{mandatory_received, MsgSeqNo}`给接收到投递的channel的pid里。channel进程将
继续，忘掉消息，因为从强制消息处理的角度，本次投递已经没有其他事情要做了。

Take a look at the
[mandatory message handling guide](./mandatory_message_handling.md) for
more info.

更多内容，请看[mandatory 消息处理指南](./mandatory_message_handling.md)

### Message Confirm Handling 消息确认处理###

When handling confirms we need to take into account two things: is the
queue durable, and was the message published as persistent. If that's
the case, then the queue process will keep track of the `MsgId` in
order to confirm the message back later to the channel that received
it from a producer. To achieve that, the queue process keeps track of
a dictionary in the process state, using `msg_id_to_channel` record
field to hold it. As the name of the field implies, this dictionary
maps _msg ids_ to _channels_. When a message is finally persisted to
disk by the backing queue, then the BQ will notify the queue process,
which will send the confirm back to the channel using the
`msg_id_to_channel` dictionary just mentioned.

当处理确认时，我们需要考虑两个事情，队列是否是durable的？发布的消息是不是persistent的？
如果是那样，队列进程将继续追踪`MsgId`以便之后确认消息给从生产者获取消息的那个channel里。
为了达到这个目的，队列进程继续追踪一个进程状态中的字典，使用`msg_id_to_channel`记录来保存它。
如同这个字段名字所表达的，这个字典映射 _msg ids_ 到 _channels_ 。当消息最终被支持队列持久化到
磁盘上，然后支持队列将会通知队列进程，然后使用我们刚刚提到的`msg_id_to_channel`字典发送确认到
对应的channel

If the queue was non durable, or the message was published as
transient, then the queue process will proceed to issue a confirm back
to the channel that sent the message in.

如果队列不是耐久的（durable）的，或者消息被发布为临时的（transient），那么队列进程将会引发一个确认
给发送消息的channel

The function `rabbit_misc:confirm_to_sender/2` is the one taking care
of sending confirms back to channels.

`rabbit_misc:confirm_to_sender/2`这个方法就是发送确认回channel要关注的。

Take a look at the
[publisher confirm handling guide](./publisher_confirms.md) for more info.

更多请看[发布者确认处理指南](./publisher_confirms.md)

### Check for Message Duplicates 检查消息是否重复###

The next step is to check if the message has been seen by the queue
before. If the backing queue responds that the message is a duplicate,
then processing stops right here, since there's anything left to do
for this delivery, so `deliver_or_enqueue/3` simply returns.

下一个步骤是检查消息之前是否已经被看到了。如果支持队列的响应说消息是重复了，那么处理应该在此处停止。
因为此次投递已经没有事情要做了。所以`deliver_or_enqueue/3`简单地进行返回。

### Attempt to Deliver the Message to a Consumer 尝试投递消息到消费者###

To try to send the message delivery to a consumer, the function
`attempt_delivery/4` is called. This function will in turn call
`rabbit_queue_consumers:delivery/3` which takes a `FetchFun`, the
`QueueName`, and the `Consumers State` for this particular queue. The
Fetch Fun will return the message that will be delivered to the
consumer (if a consumer is available). This function deals with
message acknowledgment from the point of view of the queue. If the
consumer is in `ackmode = true`, then the message will be
`publish_delivered` into the backing queue, otherwise the message will
be discarded.

为了尝试将消息投递给消费者，`attempt_delivery/4`会被调用。这个方法将为特定的队列调用
`rabbit_queue_consumers:delivery/3`方法（方法的三个参数是`FetchFun`、`QueueName`、`Consumers State`），
Fetch Fun将会返回将要投递给消费者的消息（如果有消费者的话），这个方法将从队列的角度来处理消息承认（acknowledgment）
如果消费者在`ackmode = true`，那么消息将会`publish_delivered`到支持队列，否则消息会被丢弃。

Discarding a message involves confirming the message, in case that's
required for this particular delivery, and telling the backing queue
to discard it as well.

丢弃消息涉及到消息确认，以备特殊的投递需要，并告诉支持队列将其丢弃。

Once the queue attempted to deliver the message straight to a
consumer, it will call the function `maybe_notify_decorators/2` which
takes care of telling the queue decorators that the consumer state
might have changed. See the [queue decorators](./queue_decorators.md)
guide for more information on how decorators work.

一旦队列尝试直接投递消息给消费者，它将调用方法`maybe_notify_decorators/2`，这个方法关注告诉队列
装饰器消费者的状态可能发生了变化。关于装饰器的工作应该阅读更多[队列装饰器](./queue_decorators.md)

The `attempt_delivery/4` will return back to the
`deliver_or_enqueue/3` function telling it if the message was
`delivered` or if it is still `undelivered`. If the message was
delivered to a consumer, then there's nothing else to do, and
`deliver_or_enqueue/3` will simply return. Otherwise there's still
more to do.

`attempt_delivery/4`方法将返回给`deliver_or_enqueue/3`方法，告诉消息是否已经投递了`delivered`
或者依然未投递`undelivered`。如果消息投递给了消费者，那么没有其他事情要做了，`deliver_or_enqueue/3`
进行简单返回。否则还有事情要做。

### Handling Undelivered Messages 处理未投递的消息###

When handling undelivering messages, there's a special case that can
be considered an optimization. If the queue has a
[TTL](https://www.rabbitmq.com/ttl.html) of 0, and no
[DLX](https://www.rabbitmq.com/dlx.html) has been set up, then there
is no point in queueing this message, so it can be discarded in the
same way as explained above.

当处理未投递消息时，这里有个特殊的常见需要被考虑和优化。如果队列有一个值为0的 TTL，并且没有设置死信交换机
那么为这条消息排队是没有意义的，所以它将会被丢弃，就像之前所解释过的那样。

If a message cannot be discarded, then it has to be enqueued, so the
queue process will `publish` the message into the backing queue. After
the message has been published, we need to enforce the various
policies that might apply to this queue, like `max-length` for
example. This means we need to see if the queue head has to be
dropped. Once that's enforced, then we also have to check if we need
to drop expired messages. Both these functions work in conjunction
with the DLX feature mentioned above. At this point
`deliver_or_enqueue/3` returns.

如果消息不应该被丢弃，那么它将被排队，因此队列进程将发布（`publish`）消息到支持队列中
当消息被发布后，我们需要强制执行多个可能应用（apply）到队列的策略，像是`max-length`。
这意味着我们需要查看队列头部是否该被丢弃。一旦这个强制执行，我们也应该检查我们是否应该丢弃
已经过期的消息。这两个功能都与上面提到的死信交换机特效联合使用。此时，`deliver_or_enqueue/3`返回了。

## Bookkeeping 簿记##

Even if we are done with the delivery after this was handled by the
respective queue processes where it was sent, we still need to perform
some bookkeeping on the channel side. The `rabbit_amqqueue:deliver/2`
function will return a list of `QPids` that received the
messages. This list of pids will be used now for bookkeeping.

即使在各自的队列进程处理了这些发送之后，我们已经完成了投递，但是我们依然要在channel一侧做些簿记。
`rabbit_amqqueue:deliver/2`方法将返回一个收到消息的`QPids`列表。这个pid列表将会被簿记使用。

### Queue Monitoring 队列监控###

The first thing to do is to monitor the queue pids to which the
message was delivered. This is done among other things, to account for
credit flow in case the queue goes down. We don't want to block the
channel forever if a queue that's blocking it is actually down.

首先要做的事情是监控消息传递到的队列的pid。这个已经在其他事情中完成了，记录信用流以防队列挂掉。
我们不希望永远阻塞channel，如果一个阻塞它的队列确实已经down掉了。

Take a look at the `handle_info` channel callback for the case when a
`DOWN` message is
[received](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_channel.erl#L578).

看一下channel的`handle_info`回调，当收到一个`DOWN`消息的时候。

### Process Mandatory Messages 处理强制消息###

Here if the message wasn't delivered to any queue, then it's time to
issue `basic.return`s back to the publisher that sent them. If the
message was delivered to queues, then those `QPids` will be kept into
a dictionary for later processing.

这里如果消息不能投递到任何队列。是时候返回给发布者`basic.return`响应了。
如果消息投递到了队列，那么那些`QPids`将会保存到字典里以便后续处理。

As explained above, once the queue process receives the message
delivery, then it will take care of updating the `mandatory`
dictionary on the channel's state.

如上面解释的那样，一旦队列进程接收到消息投递，需要考虑channel状态上的`mandatory`字典。

### Process Confirms 处理确认消息###

Similar as with mandatory messages, if the message wasn't routed to
any queue, then it's time to record the message as confirmed. If the
message was delivered to some queues, then it will be tracked as
unconfirmed until the queue updates the message status.

与强制消息类似，如果消息不能路由到任何队列，是时候记录消息为confirmed了。如果消息投递到了很多队列，
那么它将被追踪为unconfirmed，直到队列更新了消息的状态。

### Stats Update 统计更新###

The final step for the channel is to account for stats, so it will
update the exchange stats, indicating that a message has been routed,
and then it will also update the queue stats, to indicate that a
message was delivered to this or that queue.

最后一步是为了channel来计入统计的，因此它将更新exchange的统计信息，指示消息已经被路由，
它将更新队列的统计信息，指示消息已经投递到那个队列。

## Summary 总结##

Delivering a message to a RabbitMQ queue is quite an involved process,
and we didn't even touch on queue mirroring! The main things to
account for when handling a delivery are mandatory messages and
message confirms. Both have to be handled accordingly, and the whole
process is coordinated between the channel process and the queue
process that receives the message. Other than that, the queue needs to
see if the message can be delivered to a consumer or if it has to be
enqueued for later. Once this is handled, the queue needs to enforce
the various policies that can be applied to it, like TTLs, or
max-lengths.

投递消息到RabbitMQ队列是一个相当复杂的过程，而且我们甚至没有涉及到队列镜像。
处理投递最主要的要考虑的是强制消息和消息确认。两者必须进行相应处理，并且在整个处理接收到的消息过程中，在channel进程和
队列进程之间进行协调。初此之外，队列需要去查看消息时候可以被投递发哦消费者，否则它将在之后排队。
一旦这个处理完成，队列需要强制执行应用于它的多个策略，比如TTL、最大长度等。

To understand what happens once a message arrives to a queue, take
look at the [variable queue](./variable_queue.md) guide.

为了理解当消息投递到队列之后发生了什么，请看[variable queue](./variable_queue.md)指南。