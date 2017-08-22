---
layout: post
title:  "Kafka Consumer启动时不能接收历史消息"
date:   2017-08-22 08:00:00 +0800
categories: 中间件
---
今天测试Kafka的时候发现一个问题，Consumer启动后接收不到历史消息。比如Producer生产若干消息至Broker后下线，这时再启动Consumer无法接收Producer之前发送的消息。

**问题原因**

参数 auto.offset.reset

    官方解释
    What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server
    (e.g. because that data has been deleted):

    - earliest: automatically reset the offset to the earliest offset
    - latest: automatically reset the offset to the latest offset
    - none: throw exception to the consumer if no previous offset is found for the consumer's group
    - anything else: throw exception to the consumer.</li></ul>

如果kafka中没有当前消费组对应的offset，或offset被删除，那么当`auto.offset.reset`：

* earliest 从最小的offset开始消费
* latest（默认值） 从最大的offset开始消费
* none或其它值 抛出异常
