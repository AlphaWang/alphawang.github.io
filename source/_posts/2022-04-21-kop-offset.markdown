---
layout: post
title: "Offset Implementation in Kafka-on-Pulsar"
date: 2022-04-21 18:30:10 +0800
comments: true
categories: [messaging, distributed system, translation]
tags: [distributed system, translation, Pulsar, messaging]
description: KoP 突破性进展：2.8.0 及更高版本的连续偏移量实现
---

> 本文翻译自《Offset Implementation in Kafka-on-Pulsar》  
> - 译文发表于 Apache Pulsar 公众号：https://mp.weixin.qq.com/s/JXLquQkJFAzu8uw_lJaIcg



[Protocol handlers](https://github.com/apache/pulsar/wiki/PIP-41%3A-Pluggable-Protocol-Handler) were introduced in Pulsar 2.5.0 (released in January 2020) to extend Pulsar’s capabilities to other messaging domains. By default, Pulsar brokers only support Pulsar protocol. With protocol handlers, Pulsar brokers can support other messaging protocols, including Kafka, AMQP, and MQTT. This allows Pulsar to interact with applications built on other messaging technologies, expanding the Pulsar ecosystem. 

[协议处理器](https://github.com/apache/pulsar/wiki/PIP-41%3A-Pluggable-Protocol-Handler) 是 2020 年一月份发布的 Pulsar 2.5.0 所引入的新功能，目的是将 Pulsar 的能力扩展到其他消息领域。默认情况下 Pulsar Broker 仅支持 Pulsar 协议。而通过协议处理器，Pulsar Broker 就可以支持其他消息协议，包括 Kafka、AMQP 以及 MQTT（现已新增 RocketMQ）。这使得 Pulsar 可以与基于其他消息技术的应用进行交互，从而扩展 Pulsar 生态系统。



[Kafka-on-Pulsar (KoP)](https://github.com/streamnative/kop) is a protocol handler that brings native Kafka protocol into Pulsar. It enables developers to publish data into or fetch data from Pulsar using existing Kafka applications without code change. KoP significantly lowers the barrier to Pulsar adoption for Kafka users, making it one of the most popular protocol handlers.

[Kafka-on-Pulsar (KoP)](https://github.com/streamnative/kop) 就是一种协议处理协议，它将原生 Kafka 协议引入了 Pulsar，使得开发人员能够使用现有的 Kafka 应用将数据发布到 Pulsar 或从 Pulsar 读取数据，而无需更改代码。KoP 极大降低了 Kafka 用户使用 Pulsar 的壁垒，这让 KoP 成为最受欢迎的协议处理器之一。

<!--more-->



KoP works by parsing Kafka protocol and accessing BookKeeper storage directly via streaming storage abstraction provided by Pulsar. While Kafka and Pulsar share many common concepts, such as topic and partition, there is no corresponding concept of Kafka’s offset in Pulsar. Early versions of KoP tackled this problem with a simple conversion method, which did not allow continuous offset and was prone to problems.

KoP 解析 Kafka 协议，并通过 Pulsar 提供的流式存储抽象接口直接访问 BookKeeper。虽然 Kafka 和 Pulsar 有许多通用的概念，例如主题和分区，但在 Pulsar 中没有对应 Kafka 偏移量的概念。KoP 的早期版本通过一种简单的转换来应对这个问题，但这种转换不支持连续偏移量，同时也容易出现问题。 



To solve this pain point, broker entry metadata was introduced in KoP 2.8.0 to enable continuous offset. With this update, KoP is available and production-ready. It is important to note that with this update backward compatibility is broken. In this blog, we dive into how KoP implemented offset before and after 2.8.0. and explain the rationale behind the breaking change. 

为了解决这个痛点，KoP 2.8.0 引入了 Broker Entry Metadata，以实现连续偏移量。这个更新使得 KoP 可用并且生产就绪。需要特别注意的是，这个更新破坏了向后兼容性。本文将深入探讨 KoP 2.8.0 之前和之后分别是如何实现偏移量的，并解释该突破性变化背后的基本原理。

> **Note on Version Compatibility** 
>
> Since Pulsar 2.6.2, KoP version has been updated with Pulsar version accordingly. The version of KoP x.y.z.m conforms to Pulsar x.y.z, while m is the patch version number. For instance, the latest KoP release 2.8.1.22 is compatible with Pulsar 2.8.1. In this blog, 2.8.0 refers to both Pulsar 2.8.0 and KoP 2.8.0.
>
> **版本兼容性说明**
>
> Pulsar 2.6.2 版本之后，KoP 版本即随着相应的 Pulsar 版本而更新。KoP x.y.z.m 版本对应 Pulsar x.y.z 版本，其中 m 是补丁版本号。例如，最新的 KoP 2.8.1.22 版本与 Pulsar 2.8.1 版本兼容。本文中 2.8.0 同时指代 Pulsar 2.8.0 和 KoP 2.8.0。



# Message Identifier in Kafka and Pulsar - Kafka 和 Pulsar 的消息标识符

## Offset in Kafka - Kafka 偏移量

In Kafka, offset is a 64-bit integer that represents the position of a message in a specific partition. Kafka consumers can commit an offset to a partition. If the offset is committed successfully, after the consumer restarts, it can continue consuming from the committed offset.

在 Kafka 中，偏移量是一个 64 位整数，表示消息在特定分区中的位置。Kafka 消费者可以向分区提交偏移量。如果偏移量提交成功，那么消费者重启后就能够从已提交的偏移量位置继续消费。

1. The first message's offset is 0.
   第一条消息的偏移量为 0。
2. If the latest message's offset is N, then the next message's offset will be N+1.
   如果最后一条消息的偏移量为 N，那么下一条消息的偏移量将会是 N + 1。



Kafka stores messages in each broker's file system:
Kafka 将消息存储在每个 broker 的文件系统中：



Since each message records the message size in the header, for a given offset, Kafka can easily find its segment file and position. 

由于每条消息的头部都记录了消息大小，所以对于给定偏移量，Kafka 可以很容易地找到其分片文件以及位置。



## Message ID in Pulsar - Pulsar 消息 ID

Unlike Kafka, which stores messages in each broker's file system, Pulsar uses BookKeeper as its storage system. In BookKeeper:

Kafka 将消息存储到每个 Broker 上的文件系统，而 Pulsar 则不同，它使用 BookKeeper 作为其存储系统。在 BookKeeper 中：

- Each log unit is an **entry**
  每个日志单元称为一个 **Entry**
- Streams of log entries are **ledgers**
  日志 Entry 流称为 **Ledger**
- Individual servers storing ledgers of entries are called **bookies**
  存储 Entry Ledger 的单独的服务器称为 **Bookie**



A bookie can find any entry via a 64-bit ledger ID and a 64-bit entry ID. Pulsar can store a message or a batch (one or more messages) in an entry. Therefore, Pulsar finds a message via its message ID that consists of a ledger ID, an entry ID, and a batch index (-1 if it’s not batched). In addition, the message ID also contains the partition number.

Bookie 可以通过 64 位 Ledger ID 和 64 位 Entry ID 找到任何 Entry。Pulsar 可以在一个 Entry 中存储单条消息或一批消息。因此，Pulsar 的消息 ID 由 Ledger ID、Entry ID、 批索引（如果不是批量消息则为 -1）以及分区编号组成，Pulsar 可通过这种消息 ID 找到一条消息。



Just like a Kafka consumer can commit an offset to record the consumer position, a Pulsar consumer can acknowledge a message ID to record the consumer position.

就像 Kafka 消费者可以提交偏移量来记录消费位置一样，Pulsar 消费者可以确认消息 ID 来记录消费位置。



## How Does KoP Deal with a Kafka Offset - KoP 如何处理 Kafka 偏移量

KoP needs the following Kafka requests to deal with a Kafka offset:

KoP 需要如下 Kafka 请求来处理 Kafka 偏移量：

- PRODUCE: After messages from a Kafka producer are persisted, KoP needs to tell the Kafka producer the offset of the first message. However, the BookKeeper client only returns a message ID.
  PRODUCE：当 Kafka 生产者生产的消息被持久化之后，KoP 需要告诉 Kafka 生产者第一条消息的偏移量。然而 BookKeeper 客户端只返回一个消息 ID。
- FETCH : When a Kafka consumer wants to fetch some messages from a given offset, KoP needs to find the corresponding message ID to read entries from the ledger.
  FETCH：当 Kafka 消费者想要从指定偏移量开始获取消息时，KoP 需要找到对应的消息 ID 并从 Ledger 中读取相应的 Entry。
- LIST_OFFSET: Find the earliest or latest available message, or find a message by timestamp. 
  We must support computing a specific message offset or locating a message by a given offset.
  LIST_OFFSET：查找最早或最新的可用消息，或者按时间戳查找消息。



We must support computing a specific message offset or locating a message by a given offset.

我们必须支持计算特定消息的偏移量，或通过给定的偏移量定位消息。



# How KoP Implements Offset before 2.8.0 -KoP 2.8.0 之前版本如何实现偏移量



## The Implementation - 实现细节

As explained earlier, Kafka locates a message via **a partition number and an offset**, while Pulsar locates a message via **a message ID**. Before Pulsar 2.8.0, KoP simply performed conversions between Kafka offsets and Pulsar message IDs. A 64-bit offset is mapped into a 20-bit ledger ID, a 32-bit entry id, and a 12-bit batch index. Here is a simple Java implementation.

如前文所述，Kafka 通过 **分区编号和偏移量** 来定位消息，而 Pulsar 通过 **消息 ID** 来定位消息。在 Pulsar 2.8.0 之前，KoP 简单地在 Kafka 偏移量和 Pulsar 消息 ID 之间进行一个转换。将 64 位的偏移量映射为 20 位 Ledger ID、32 位 Entry ID 以及 12 位批索引。如下是一个简单的 Java 实现。

```

    public static long getOffset(long ledgerId, long entryId, int batchIndex) {
        return (ledgerId << (32 + 12) | (entryId << 12)) + batchIndex;
    }

    public static PositionImpl getPosition(long offset) {
        long ledgerId = offset >>> (32 + 12);
        long entryId = (offset & 0x0F_FF_FF_FF_FF_FFL) >>> BATCH_BITS;
        // BookKeeper only needs a ledger id and an entry id to locate an entry
        return new PositionImpl(ledgerId, entryId);
    }

```



In this blog, we use `(ledger id, entry id, batch index)` to represent a message ID. For example, assuming a message's message ID is `(10, 0, 0)`, the converted offset will be 175921860444160. This works in some cases because the offset is monotonically increasing. But it’s problematic when a ledger rollover happens or the application wants to manage offsets manually. The section below goes into details about the problems of this simple conversion implementation. 

在本文中，我们使用 `(ledger id, entry id, batch index)` 来表示一个消息 ID。例如，假设一个消息的 ID 是 `(10, 0, 0)`，则转换后的偏移量为 175921860444160。在一些情况下这样的数值能正常工作，因为偏移量是单调递增的。然而当发生 Ledger 翻转，或应用程序想要手动管理偏移量时，就会出现问题。下面详细介绍这种简单转换方法存在的问题。

## The Problems of the Simple Conversion - 简单转换存在的问题

The converted offset **is not continuous**, which causes many serious problems. 

转换后的偏移量**不连续**，这会导致许多严重问题。



For example, let’s assume the current message's ID is `(10, 5, 100)`. The next message's ID could be `(11, 0, 0)` if a ledger rollover happens. In this case, the offsets of these two messages are 175921860464740 and 193514046488576. The delta value between the two is 17,592,186,023,836.

例如，假设当前消息 ID 是 `(10, 5, 100)`。如果发生 Ledger 翻转，则下一条消息的 ID 可能是 `(11, 0, 0)`。在这种情况下，两条消息的偏移量分别为 175921860464740 和 193514046488576，两者差了 17,592,186,023,836。



KoP leverages Kafka's `MemoryRecordBuilder` to merge multiple messages into a batch. The `MemoryRecordBuilder` must ensure the batch size is less than the maximum value of a 32-bit integer (4,294,967,296). In our example, the delta value of the two continuous offsets is much greater than 4,294,967,296. This will result in an exception that says `Maximum offset delta exceeded`.

KoP 利用 Kafka 的 `MemoryRecordBuilder` 将多条消息合并为一个批量消息。 `MemoryRecordBuilder` 必须确保批量大小小于 32 位整数的最大值 (4,294,967,296)。在上文示例中，两个连续偏移量的差值远大于 4,294,967,296。这将导致抛出 `Maximum offset delta exceeded` 异常。



To avoid the exception, before KoP 2.8.0, we must configure `maxReadEntriesNum` (this config limits the maximum number of entries read by the BookKeeper client) to 1. Naturally, reading only one entry for each FETCH request worsens the performance significantly.

为了避免该异常，在使用 KoP 2.8.0 之前版本时，我们必须配置 `maxReadEntriesNum` 为 1 (此配置限制 BookKeeper 客户端读取的最大 Entry 条数)。如此一来，每个 FETCH 请求只读取一个 Entry，会显著降低性能。



However, even with the workaround of `maxReadEntriesNum=1`, this conversion implementation doesn’t work in some scenarios. For example, the Kafka integration with Spark relies on the continuance of Kafka offsets. After it consumes a message with an offset of N, it will seek the next offset (N+1). However, the offset N+1 might not be able to convert to a valid message ID.

然而，即使使用 `maxReadEntriesNum=1` 这种变通方法，这种转换实现在某些场景下也不能正常工作。例如，Kafka 与 Spark 的集成依赖于 Kafka 偏移量的连续性。当消费偏移量为 N 的消息后，Spark 会寻找下一个偏移量 (N + 1)。但是偏移量 N + 1 可能无法转换为有效的消息 ID。



There are other problems caused by the conversion method. And before 2.8.0, there is no good way to implement the continuous offset.

转换方法还存在其他问题。而在 2.8.0 之前版本，没有好办法实现连续偏移量。



# The Continuous Offset Implementation since 2.8.0 - 自 KoP 2.8.0 版本的连续偏移量实现

The solution to implement continuous offset is to record the offset into the metadata of a message. However, an offset is determined at the broker side before publishing messages to bookies, while the metadata of a message is constructed at the client side. To solve this problem, we need to do some extra jobs at the broker side:

实现连续偏移量的解决方案是将偏移量记录到消息的元数据中。然而，偏移量是由 Broker 端在将消息发布到 Bookie 之前决定的，而消息的元数据则是在客户端构建的。为了解决这个问题，我们需要在 Broker 端做一些额外的工作：

1. Deserialize the metadata
   反序列化元数据

2. Set the "offset" property of metadata
   设置元数据的“偏移量”属性

3. Serialize the metadata again, including re-computing the checksum value
   再次序列化元数据，包括重新计算校验和值

   

This results in a significant increase in CPU overhead on the broker side.

这会导致 Broker 端的 CPU 开销显著增加。



## Broker Entry Metadata - 轻量级 Broker Entry 元数据

[PIP 70](https://github.com/apache/pulsar/wiki/PIP-70:-Introduce-lightweight-broker-entry-metadata) introduced lightweight broker entry metadata. It's a metadata of a BookKeeper entry and should only be visible inside the broker.

[PIP 70](https://github.com/apache/pulsar/wiki/PIP-70:-Introduce-lightweight-broker-entry-metadata) 引入了轻量级 Broker Entry 元数据。它是 BookKeeper Entry 的元数据，并且只在 Broker 内部可见。
The default message flow is:默认的消息流如下图所示：



![img](https://lh3.googleusercontent.com/eS6w7ZRshhoBcsk3t0Q7O9njkV-Wej0b6n00rbcxnfW2BKK6qYxIbAlWsbZT4M1yh-9H534GpTM3YWd3Wg9lal4O26HmNjX1qDsS7Xk6FiS3k9kmk1QAS39Jk22bsGN6z9X4unD7WBeBK-9MUN1jnQ)

If you configured `brokerEntryMetadataInterceptors`, which represents a list of **broker entry metadata interceptors**, then the message flow would be: 

如果配置了 `brokerEntryMetadataInterceptors`，即配置一组 **Broker Entry 元数据拦截器**，那么消息流将会是：

![img](https://lh3.googleusercontent.com/eNYmZkvZx1ZDWCsr-XOLw1B2OZ6ku4_mx02uWZuZbIJvFWWYRmeCWV5nbVmI_zaZreo0_syq3bowX0wTNAk-Ls9wFl32q6XeKYjnbueQ02uhqIX01F6gueJgqC2DTj6tiTP1Rdm4Cd3Wb9ZWlt0flA)
We can see the broker entry metadata is stored in bookies, but is not visible to a Pulsar consumer.

可以看到 Broker Entry 元数据存储在 Bookie 上，但对 Pulsar 消费者不可见。



> From 2.9.0, a Pulsar consumer can be configured to read the broker entry metadata.
>
> 2.9.0 版本之后，可以将 Pulsar 消费者配置为可以读取 Broker Entry 元数据。



Each broker entry metadata interceptor adds the specific metadata (called "broker entry metadata") before the message metadata. Since the broker entry metadata is independent of the message metadata, the broker does not need to deserialize the message metadata. In addition, the BookKeeper client supports sending a Netty `CompositeByteBuf` that is a list of `ByteBuf` without any copy operation. From the perspective of a BookKeeper client, only some extra bytes are sent into the socket buffer. Therefore, the extra overhead is low and acceptable.

每个 Broker Entry 元数据拦截器都在消息元数据前面加上特定的元数据（称之为 “Broker Entry 元数据”）。由于 Broker Entry 元数据和消息元数据是独立的，所以 Broker 无需反序列化消息元数据。此外，BookKeeper 客户端支持发送包含多个 `ByteBuf` 的 Netty `CompositeByteBuf`，而无需任何复制操作。从 BookKeeper 客户端角度看，只是将一些额外字节发送到套接字缓冲区。因此，额外的开销会很低且可接受。



## The Index Metadata - 索引元数据

We need to configure the `AppendIndexMetadataInterceptor` (we say **index metadata interceptor**) to support the Kafka offset.

我们需要配置 `AppendIndexMetadataInterceptor` (即 **索引元数据拦截器**) 来支持 Kafka 偏移量。

```
brokerEntryMetadataInterceptors=org.apache.pulsar.common.intercept.AppendIndexMetadataInterceptor
```



In Pulsar brokers, there is a component named "managed ledger" that manages all ledgers of a partition. The index metadata interceptor maintains an index that starts from 0. The "index" term is used instead of "offset".

Pulsar Broker 中有个名为 “Managed Ledger” 的组件，它管理分区中的所有 Ledger。索引元数据拦截器维护了一个从 0 开始的索引。Pulsar 使用术语“索引”而不是“偏移量”。



Each time before an entry is written to bookies, the following two things happen:

每次将 Entry 写入 Bookie 之前，都会发生如下两件事：

1. The index is serialized into the broker entry metadata.
   将索引序列化到 Broker Entry 元数据中。
2. The index increases by the number of messages in the entry.
   将索引自增 Entry 中的消息数目。



After that, each entry records the first message's index, which is equivalent to the "base offset" concept in Kafka.

之后，每个 Entry 记录第一条消息的索引，相当于 Kafka 中的“基础偏移量”概念。



Now, we must make sure even if the partition's owner broker was down, the index metadata interceptor would recover the index from somewhere.

现在，我们需要保证即使分区的 owner Broker 宕机，索引元数据拦截器也能从某个地方恢复索引。



There are some cases where the managed ledger needs to store its metadata (usually in ZooKeeper). For example, when a ledger is rolled over, the managed ledger must archive all ledger IDs in a z-node. Here we don't look deeper into the metadata format. We only need to know there is a property map in the managed ledger's metadata.

在某些场景下，Managed Ledger 需要将其元数据存储起来（通常存储到 ZooKeeper）。例如，当一个 Ledger 发生翻转，Managed Ledger 需要将所有 Ledger ID 归档到一个 z-node。这里我们不深入研究元数据的格式，只需要知道在 Managed Ledger 元数据中有一个属性映射。



Before metadata is stored in ZooKeeper (or another metadata store):

在将元数据存储到 ZooKeeper (或其他元数据存储) 之前：

1. Retrieve the index from the index metadata interceptor, which represents the latest message's index.
   从索引元数据拦截器中检索索引，该索引代表了最新消息的索引。
2. Add the property whose key is "index" and value is the index to the property map.
   向属性映射中添加一条属性，属性名为 “index”，属性值为索引值。



Each time a managed ledger is initialized, it will restore the metadata from the metadata store. At that time, we can set the index metadata intercerptor's index to the value associated with the "index" key.

每次初始 Managed Ledger 时，都会从元数据存储中恢复元数据。那时，我们可以将索引元数据拦截器中的索引设置为“index”键关联值。



## How KoP Implements the Continuous Offsets - KoP 如何实现连续偏移量

Let's look back to the **How does KoP deal with a Kafka offset** section and review how we deal with the offset in following Kafka requests.

让我们回顾一下 **KoP 如何处理 Kafka 偏移量** 一节，看看在如下 Kafka 请求中如何处理偏移量。

- PRODUCE

When KoP handles PRODUCE requests, it leverages the managed ledger to write messages to bookies. The API has a callback that can access the entry's data.

当 KoP 处理 PRODUCE 请求时，它利用 Managed Ledger 将消息写入 Bookie。相关 API 有一个回调可以访问 Entry 数据。

```
      @Override
      public void addComplete(Position pos, ByteBuf entryData, Object ctx) {
```

 We only need to parse the broker entry metadata from `entryData` and then retrieve the index. The index is just the base offset returned to the Kafka producer.

我们只需要从 `entryData` 中解析出 Broker Entry 元数据，然后检索索引即可。该索引就是返回给 Kafka 生产者的基础偏移量。



- FETCH

The task is to find the position (ledger id and entry id) for a given offset. KoP implements a callback that reads the index from the entry and compares it with the given offset. It then passes the callback to a class named `OpFindNewest`, which uses binary search to find an entry.

 The binary search could take some time. But it only happens on the initial search unless the Kafka consumer disconnects. After the position is found, a non-durable cursor will be created to record the position. The cursor will move to a newer position as the fetch offset increases.

FETCH 是通过给定偏移量找到消息位置 (Ledger ID 和 Entry ID)。KoP 实现了一个回调，从 Entry 中读取索引并与给定的偏移量进行比较。然后将回调传给 `OpFindNewest` 类，该类使用二分查找算法来查找 Entry。

二分查找可能要花一些时间。但它仅发生在初始搜索中，除非 Kafka 消费者断开连接。当找到位置后，会创建一个非持久化的游标来记录该位置。随着 fetch 偏移量的增加，游标会移动到更新的位置。



- LIST_OFFSET
  - **Earliest**: Get the first valid position in a managed ledger. Then read the entry of the position, and parse the index.
    最早：获得 Managed Ledger 中的第一个有效位置，然后读取该位置的 Entry，并解析索引。
  - **Latest**: Retrieve the index from the index metadata interceptor and increase by one. It should be noted that the latest offset (or called LEO) in Kafka is the next offset to be assigned to a message, while the index metadata interceptor's index is the offset assigned to the latest message.
    最新：从索引元数据拦截器中检索索引，并加一。需要注意的是，Kafka 中的最新偏移量（也被称为 LEO）是下一个将要分配给消息的偏移量，而索引元数据拦截器中的索引则是分配给最新消息的偏移量。
  - **By timestamp**: First leverage broker's timestamp based binary search to find the target entry. Then parse the index from the entry.
    按时间戳：首先利用 Broker 的基于时间戳的二分查找找到目标 Entry，然后从 Entry 中解析出索引。



# Upgrade from KoP Version before 2.8.0 to 2.8.0 or Higher - 从 KoP 2.8.0 之前的版本升级到 2.8.0 或更高版本

KoP 2.8.0 implements continuous offset with a tradeoff – [the backward compatibility is broken](https://github.com/streamnative/kop/blob/master/docs/upgrade.md). The offset stored by KoP versions before 2.8.0 cannot be recognized by KoP 2.8.0 or higher.

KoP 2.8.0 实现的连续偏移量是有折衷的 —— [向后兼容性被破坏](https://github.com/streamnative/kop/blob/master/docs/upgrade.md)。KoP 2.8.0 之前版本存储的偏移量无法被 KoP 2.8.0 或更高版本识别。



If you have not tried KoP, please upgrade your Pulsar to 2.8.0 or higher and then use the corresponding KoP. As of this writing, the latest KoP release for Pulsar 2.8.1 is 2.8.1.22.

如果在此之前你还没有使用过 KoP，需将 Pulsar 升级到 2.8.0 或更高版本后使用相应版本的 KoP。



If you have already tried KoP before 2.8.0, you need to know that there's a breaking change from version less than 2.8.0 to version 2.8.0 or higher. You must delete the `__consumer_offsets` topic and all existing topics previously used by KoP.

如果你在此之前已经使用过 2.8.0 之前版本的 KoP，则需要知道从低于 2.8.0 版本到 2.8.0 或更高版本有突破性变化。使用新版本前，你必须删除 `__consumer_offsets` 主题以及 KoP 之前使用过的所有主题。



There is a latest feature in KoP that can skip these old messages by enabling a config. It would be included in 2.8.1.23 or later. Note that the old messages still won’t be accessible. It just saves the work of deleting old topics.

KoP 中有一个最新的功能，可以通过启用配置来跳过这些旧消息。这个功能将包含在 2.8.1.23 或更高版本。注意：旧消息仍将无法访问，这个功能只是节省了删除旧主题的工作量。



# Summary - 总结
In this blog, we first explained the concept of Kafka offset and the similar concept of message ID in Pulsar. Then we talked about how KoP implemented Kafka offset before 2.8.0 and the related problems.

本文首先解释了 Kafka 偏移量的概念，以及 Pulsar 类似的消息 ID 概念。然后讲了 KoP 在 2.8.0 版本之前是如何实现 Kafka 偏移量的及其带来的相关问题。



To solve these problems, the broker entry metadata was introduced from Pulsar 2.8.0. Based on this feature, index metadata is implemented via a corresponding interceptor. After that, KoP can leverage the index metadata interceptor to implement the continuous offset. 

为了解决这些问题，Pulsar 2.8.0 引入了 Broker Entry 元数据。基于此特性，通过相应的拦截器实现了索引元数据。之后，KoP 可以利用索引元数据拦截器来实现连续偏移量。



Finally, since it's a breaking change, we talked about the upgrade from KoP version less than 2.8.0 to 2.8.0 or higher. It's highly recommended to try KoP 2.8.0 or higher directly.

最后，由于这是一个突破性的变化，我们谈了从 2.8.0 之前版本到 2.8.0 或更高版本的升级。强烈建议直接尝试 KoP 2.8.0 或更高版本。

