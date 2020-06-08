# RabbitMQ Internals #

This project aims to explain how RabbitMQ works internally. The goal
is to make it easier to contribute for newcomers to the project, and
at the same time have a common repository of knowledge to be shared
across the project contributors.

本项目的目标是解释RabbitMQ内部是如何工作的。目的是让项目的新的参与者更容易贡献，
同时让其成为一个贡献者之间共享的通用知识库。

## Purpose 目的##

Most interesting modules in RabbitMQ projects have documentation
essays, sometimes quite extensive, at the top. The aim here is not to
duplicate what's there, but to provide the highest-level overview as
to the overall architecture.

最有趣的模块在RabbitMQ项目里有文档，并且相当丰富。这里不是要复制那些内容，而是提供
整个体系的最高级别的概览。

## Guides 向导##

In order to understand how RabbitMQ's internals work, it's better to
try to follow the logic of how a message progresses through
RabbitMQ as it is handled by the broker, otherwise, you would end up
navigating through many guides without a clear context of what's going
on, or without knowing what to read next. Therefore we have prepared
the following guides to help you understand how RabbitMQ works:

为了理解RabbitMQ内部是如何工作的，最好是尝试跟随消息是如何经过RabbitMQ，以及如何被
服务器处理的过程，否则，你将会在没有明确上下文的情况下阅读很多向导，或者不知道接下来该
阅读什么。因此，我们准备了以下指南来帮助你理解RabbitMQ如何工作：

### Basic Publish Guide 基本的发布指南###

Here we follow the life of a message since it's received from the
network until it has been routed by the exchanges. We take a look at
the various processing steps that happen to a message right until it
is delivered to one or perhaps many queues.

这里我们跟随消息的声明周期，从它从网络中接收到被exchange路由完成。我们来看看消息在投递
到一个或多个队列之前发生的一系列的处理步骤。

[Basic Publish](./basic_publish.md)

### Deliver To Queues Guide 投递到队列向导###

After the message has been routed, the broker needs to deliver that
message to the respective queues. Here not only the message has to be
sent to queues, but also mandatory messages and publisher confirms
need to be taken into account. Also, the queue needs to try to deliver
the message to prospective consumer, otherwise the message ends up
queued.

在消息被路由完成后，服务器需要投递消息到相应的队列。这里不仅包含消息被发送到队列，也包含
mandatory messages（强制消息）和publisher confirms（发布者确认）。同时队列需要尝试
投递消息到可能的消费者，否则消息最终排队。

[Deliver To Queues](./deliver_to_queues.md)

### Queues and Message Store 队列和消息存储###

Provides an overview of the Erlang processes that back queues
and how they interact with the message store, message index and so on.

提供队列背后的Erlang进程的概览，以及他们如何与消息存储交互，消息如何所以等等。

[Queues and Message Store](./queues_and_message_store.md)

### Variable Queue Guide 可变队列指南###

Ultimately, messages end up queued at the
[backing queue](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_backing_queue.erl). From
here they can be retrieved, acked, purged, and so on. The most common
implementation of the backing queue behaviour is the
`rabbit_variable_queue`
[module](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_variable_queue.erl),
explained in the following guide:

最后，消息最终在backing queue中进行排队，在这里，他们可以被获取、确认、清空等等。backing queue最常见的实现
是rabbit_variable_queue这个模块。在以下的指南中进行了解释。

[Variable Queue](./variable_queue.md)

### Mandatory Messages and Publisher Confirm Guides ###

As explained on the [Deliver To Queues](./deliver_to_queues.md) guide,
a channel has to handle messages published as mandatory and also take
care of publisher confirms. These processes are explained in the
following guides:

在发送到队列指南里我们说过，一个处理消息以mandatory发送的channel上，需要考虑进行发布者确认。
下面的指南解释了这些过程。

- [Mandatory Message Handling](./mandatory_message_handling.md)
- [Publisher Confirms](./publisher_confirms.md)

### Authentication and Authorization 身份认证与鉴权###

As explained in the [Basic Publish](./basic_publish.md), there are
some rules to see if a message can be accepted by the broker from a
certain publisher. This is explained in the following guide:

在基本发送中我们提到，由发布者发送的消息，是否能被服务端接收，有一些规则。以下的指南是解释这个的。

[Authorization and Authentication Backends](./authorization_and_authentication.md)

### Internal Event Subsystem 内部事件子系统###

In some cases components in a running node communicate via events.
Some events are consumed by other nodes.

在某些场合下，正在运行的节点中的组件通过事件进行通讯，许多事件被其他节点消费。

[Internal Events 内部事件](./internal_events.md)

### Management Plugin 管理插件###

An architectural overview of the v3.6.7+ version of the management plugin.

v3.6.7+ 版本中的管理插件的架构概览。

[Metrics and Management Plugin](./metrics_and_management_plugin.md)

## Maturity and Completeness 成熟度和完整性##

These guides are not complete, haven't been edited, and are work in
progress in general.

这些指南不完整，尚未编辑，并且总体上还在进行中。

So if you find yourself wanting more detail, check the code first!

因此如果你想要更多细节，请先看代码！

## License 

(c) Pivotal Software Inc, 2015-2016

Released under the
[Creative Commons Attribution-ShareAlike 3.0 Unported](https://creativecommons.org/licenses/by-sa/3.0/)
license.
