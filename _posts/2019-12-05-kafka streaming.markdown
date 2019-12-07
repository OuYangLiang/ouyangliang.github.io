---
layout: post
title:  "kafka streaming"
date:   2019-12-05 08:00:00 +0800
categories: streaming
keywords: streaming
description: kafka streaming
commentId: 2019-12-05
---

写一个小小的demo调研一下kafka streaming，模拟订单消息并进行统计。

## 数据源

---

每秒钟生成约30个随机订单，数据落地到数据库，并推送到Kafka。

数据推送到order_queue，配置16个partitions。根据客户ID取模决定分片，record时间取下单时间。

**订单内容**

* 订单ID，DB自增长
* 商品Code：100种商品，1 ~ 100
* 支付金额：随机：100 〜 5000之间
* 优惠金额：随机：小于订单金额的10%
* 订单总金额：支付金额 + 优惠金额
* 客户ID：共1000个客户：范围：1 〜 1000
* 客户姓名：考虑不要，或姓名 + ID
* 下单时间：当前时间
* 支付时间：当前时间

**订单消息格式**

```json
{
    "orderId": 103832,
    "productCode": 12,
    "payAmt": 1000.00,
    "discount": 99.00,
    "totalAmt": 1099.00,
    "custId": 997,
    "orderTime": "2019-09-10 00:09:08",
    "payTime": "2019-09-10 00:09:08"
}
```

## 计算

---

* 每分钟下单量
* 每分钟下单金额
* 每分钟下单客户数量
* 可查询任意时刻，某客户在某分钟内的下单量、下单金额

## 设计

---

**数据生成**

一、kafka topic名称：order_queue，partition数量：16

二、根据客户ID值hash取模确定partition

三、创建topic

```
./bin/kafka-topics.sh --zookeeper localhost:2181 --create --replication-factor 1 --partitions 16 --topic order_queue
```

四、监听topic

```
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic order_queue --from-beginning
```

五、订单明细落地到DB表，方便验证

```
create table `order`(
    order_id                    int             not null         auto_increment,
    prod_code                   varchar(20)     not null,
    pay_amt                     numeric(15,2)   not null,
    discount                    numeric(15,2)   not null,
    total_amt                   numeric(15,2)   not null,
    cust_id                     int             not null,
    order_time                  datetime        not null,
    pay_time                    datetime        not null,
    primary key (order_id)
)engine=innodb;
```

**统计**

一：从Kafka接收订单信息，计算 & 结果写入kafka topic，最终写入db

二：基于Order的order time字段值计算时间窗口

三：每分钟下单量、每分钟下单金额、每分钟下单客户数量，结果写入minute_statistics topic，最终结果入minute_statistics表

```
./bin/kafka-topics.sh --zookeeper localhost:2181 --create --replication-factor 1 --partitions 1 --topic minute_statistics
```

```
create table `minute_statistics`(
    id                          int             not null         auto_increment,
    minute                      varchar(20)     not null,
    num_of_orders               int             not null,
    total_order_amt             numeric(15,2)   not null,
    num_of_ordered_customer     numeric(15,2)   not null,
    primary key (id),
    unique key (minute)
)engine=innodb;
```

四：可查询任意时刻，某客户在5分钟内的下单量、下单金额，结果写入user_statistics topic与结果写入user_statistics表

```
./bin/kafka-topics.sh --zookeeper localhost:2181 --create --replication-factor 1 --partitions 1 --topic user_statistics
```

```
create table `user_statistics`(
    id                          int             not null         auto_increment,
    cust_id                     int             not null,
    minute                      varchar(20)     not null,
    num_of_orders               int             not null,
    total_order_amt             numeric(15,2)   not null,
    primary key (id),
    unique key (cust_id, minute)
)engine=innodb;
```

## 说明

---

代码库地址：[https://github.com/OuYangLiang/kafka-streaming](https://github.com/OuYangLiang/kafka-streaming)

`OrderGenerator`会生成订单并写入order_queue和数据库；

`StreamingComputer`从order_queue取消息，并进行计算，结果会写入user_statistics和minute_statistics两个topic，这里是保证了exactly once的递交语义；

`sink`从user_statistics和minute_statistics两个topic获取最终计算结果，并写入数据表，这里实现是at least once的递交语义。

使用<kbd>Suppressed.untilWindowCloses</kbd>的方式，只有在时间窗口关闭，才会输出结果，且只会输出一次。<kbd>grace(Duration.ofSeconds(10)</kbd>会延迟等待10秒输出，超过10秒后的消息数据会被丢弃。如果想近实时查询当前分钟统计情况的话，可以使用<kbd>Suppressed.untilTimeLimit</kbd>控制窗口输出频率，但sink端需要作好对应的处理（持续更新，而不是新增）。

另外，最好关闭全局缓存，否则会影响对窗口的细粒度制。

```java
props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 0L);
```

kafka 0.11 release发布后，支持了两个新特性：
* **Idempotence:** Exactly-once in order semantics per partition
* **Transactions:** Atomic writes across multiple partitions

这两个新特性使得kafka streaming支持了exactly once的递交语义。使用时只需要增加一个配置项即可：

```java
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE);
```

注册，exactly once只是针对kafka本身，即消息的产生与结果的写入，都是对kafka topic而言。如果最终结果是输出到其它地方，比如数据库，那最终端到端的递交语义取决于最末端消费者的行为。
详情可以参考[Kafka深度解析]({{site.baseurl}}/2018/06/Kafka深度解析)
