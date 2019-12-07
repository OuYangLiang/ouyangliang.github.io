---
layout: post
title:  "kafka streaming"
date:   2019-12-05 08:00:00 +0800
categories: streaming
keywords: streaming
description: kafka streaming
commentId: 2019-12-05
---
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

代码库地址：https://github.com/OuYangLiang/kafka-streaming
