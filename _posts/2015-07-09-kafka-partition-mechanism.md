---
layout: post
title: Kafka分配分区的机制
comments: true
---

---

本文阐述Kafka中Producer与Consumer分配分区的机制。

---


### 1.Consumer分区分配机制

分配partition的策略：

- range：对于每个topic，会将topic的partition编上序号排好序，然后consumer线程以字典序排序。然而把partition的总数除以consumer线程的总数来决定分配给每个线程的partition数目。如果无法除尽，将余数再均分给排序靠前的几个线程，即这些线程都会多出额外的一个partition。
- round-robin：这个策略把所有partition和所有consumer线程都列出来。然后它以循环制分配partition给线程。如果所有consumer实例的订阅是相同的，那么partition会均匀分布。这个分配策略只有当以下情况成立时才可用：a.每个topic在一个consumer实例中有同样的stream数目。b.在group中的每个consumer实例订阅的topic的集合是相同的。

如果所有consumer实例有相同的consumer group，那么这个就像传统的队列，负载均衡到所有consumer上。假如多个consumer实例都有多个线程，且属于同一个group，那么一个topic的所有partition会均匀分配给所有线程。

接收消息的顺序只能保证一个partition之内是有序的，一个consumer接收多个partition的话是无法保证消息全局有序的，即consumer接收的消息的顺序可能跟producer发送的顺序不同。

### 2.Producer分区机制

当指定partition key的时候，分配partition的策略：

- hash：由消息所提供的key来进行hash，然后分发到对应的partition。这是默认使用的partition机制。
- 自定义：自己实现partition接口，并在配置中用参数`partitioner.class`指定这个实现。

当没有指定partition key的时候，分配partition的策略：

- 随机：把每个消息随机分发到一个partition中。在10分钟内，该partition不会切换。所以，当producer数目小于partition时，在一定时间内会有部分partition没有收到数据。

### 3.参考

1. [参考链接1](http://my.oschina.net/u/591402/blog/152837);

2. [参考链接2](https://cwiki.apache.org/confluence/display/KAFKA/FAQ#FAQ-Whyisdatanotevenlydistributedamongpartitionswhenapartitioningkeyisnotspecified?);

3. [Kafka源码](https://github.com/apache/kafka/blob/0.8.1/core/src/main/scala/kafka/producer/async/DefaultEventHandler.scala)。

