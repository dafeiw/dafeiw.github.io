---
title: Message Queue
date: 2019-07-27 13:04:55
tags:
- Web
categories: MOM
sage: true
---

# Reasons to use message queue

* Traffic Spikes

  * By queuing the data we can be assured the data will be persisted and then be processed eventually, even if that means it takes a little longer than usual due to a high traffic spike.

* Asynchronous messaging
  * Improve performance

* Decouple

  * improve scalability: no hard dependencies between components, so one component can scale dynamically.
  * Create Resiliency: if one component fail, the other component can still write message into queue.

# Disadvantage
* if mq is down, the system can't send message to the mq. The whole system fail
* system becomes more complex. e.g. system A sends message twice to the queue or the message is lost in mq or the order of message is not correct. If receivers are down, bunch of messages are not consumed in mq. 
* consistency. If system A sends a message to mq, it expects system B, C, D all process the message successfully. But System D fails.


# ActiveMQ vs RabbitMQ vs RocketMQ vs Kafka
ActiveMQ and RabbitMQ: tens of thousands messages per sec.
  * RabbitMQ is developed in Erlang. 
RocketMQ vs Kafka: hundreds of thousands messages per sec.


# High availability

## Kafka
In Kafa, each topic is divided into multiple partition. Each partition is on different borkers (machine).

# Handle duplicate messages (idempotent)

Duplicate messages cannot be avoided, so we need to make system idempotent.

* use redis to store history
* use unique key in database. 

# no message get lost

## rabbitmq
几种可能
* 写消息过程中，消息都没到rabbitmq，在网络传输过程中就丢了。或者消息到了rabbitmq但是内部出错，没保存下来。
* rabbitmq挂掉了，内存中的数据丢了
* 消费者消费到了这个消息，但是还没来得及处理，自己就挂掉了，但是rabbitmq以为这个消费者已经处理完了。

## Kafka

### Consumer side
Trun off auto-ack. Manually ack after message has been processed.

Kafka broker sends ack only if message has been received by leader and copied to followers.

### broker side
Make sure at least 2 followers. Only ack when all follower get a copy of message.

# 保证消息的顺序性

## Rabbit MQ
保证数据只发到一个queue

## Kafka

指定一个key，保证他们都进到同一个partition，一个partion只能被一个consumer消费。如果一个消费者里面，有多个thread处理，the order也会变乱

# 消息积压，消息队列满了，

新建一个topic，partition是原来的10倍，让consumer consume messages and pass to new topic.


------------------

# Cache

## Why to use Cache
- 高性能 high performance

- 高并发 high concurrency
  - mysql can handle 4k reqs/s

## disadvantages
- cache and database data is inconsistent.
- 缓存雪崩
- 缓存穿透
- 缓存并发竞争

## Redis vs memcached
redis is single threaded. 单线程，nio

## Why Redis is so fast

* memory operation
* non-blocking IO multiplex
* single thread


## Cache Aside Pattern
- 读的时候，先读缓存，缓存没有，读数据库，然后写入缓存，返回
- 更新的时候，先删除缓存，然后更新数据库