---
layout: post
title: "Kafka Rebalance机制深度解析"
date: 2026-06-24
categories: [Kafka]
---

> Rebalance是Kafka消费者组的自我修复机制，也是生产环境中消费端抖动的头号嫌疑人

## 一、Rebalance是什么

Kafka内置了组协调协议（Group Coordination Protocol），集群中某个Broker会被选举为组协调者（Group Coordinator），负责管理消费者组的成员关系和分区分配。当消费者组内的成员发生变化，或者订阅关系发生变更时，Coordinator会触发一次重新平衡（Rebalance），组织组内所有成员重新协商分区分配方案，确保每个分区都有且仅有一个消费者负责消费。

简单来说，Rebalance就是"重新分活"——把Topic的分区在消费者之间重新分配。

## 二、触发Rebalance的时机

Rebalance并非凭空发生，只有在以下三种情况下才会被触发：

### 1. 组成员数发生变更

新消费者实例加入组，或已有消费者实例离开组（主动离开或崩溃超时）。

```java
// 新消费者加入——比如运维扩容，新增了一个consumer实例
Properties props = new Properties();
props.put("bootstrap.servers", "kafka1:9092,kafka2:9092,kafka3:9092");
props.put("group.id", "order-service-group");
props.put("key.deserializer", StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("order-topic"));
// 此刻，order-service-group的成员数+1，触发Rebalance

// 消费者主动离开
consumer.close(); // 触发Rebalance，剩余成员重新分配分区
```

### 2. 订阅Topic数发生变更

消费者订阅了新的Topic，或取消了对已有Topic的订阅。

```java
// 原来只订阅了order-topic
consumer.subscribe(Arrays.asList("order-topic"));

// 运行时动态新增订阅——触发Rebalance
consumer.subscribe(Arrays.asList("order-topic", "payment-topic"));
```

### 3. 订阅Topic的分区数发生变更

通过命令行或管理工具对Topic进行了分区扩容。

```bash
# 将order-topic从6个分区扩容到12个分区
kafka-topics.sh --bootstrap-server kafka1:9092 \
  --alter --topic order-topic --partitions 12
# 分区数变更后，消费者组触发Rebalance，重新分配新增的6个分区
```

## 三、Rebalance的执行过程

Rebalance发生时，组下所有消费者实例必须全部参与协调，共同完成分区分配的新方案。整个过程分为以下几个阶段：

### 1. FindCoordinator

消费者向集群中的任意Broker发送FindCoordinator请求，找到负责自己所在消费者组的Group Coordinator。

### 2. JoinGroup

所有消费者向Coordinator发送JoinGroup请求。Coordinator从中选出一个消费者作为Leader（注意：这个Leader是消费者组的Leader，不是Broker的Leader），由它来制定分区分配方案。

```java
// 消费者组Leader收到JoinGroup响应后，可以看到所有成员的订阅信息
// 假设有3个消费者C0、C1、C2，订阅了order-topic（6个分区）
// Leader需要决定：谁消费哪个分区
```

### 3. SyncGroup

消费者组Leader根据分配策略（Range、RoundRobin、Sticky等）计算分区分配方案，然后通过SyncGroup请求将方案提交给Coordinator。其他消费者也发送SyncGroup请求，从Coordinator获取属于自己的分区分配结果。

```java
// Leader计算的分配方案示例（RoundRobin策略）
// C0: partition-0, partition-3
// C1: partition-1, partition-4
// C2: partition-2, partition-5

// Leader将方案通过SyncGroup发给Coordinator
// Coordinator再通过SyncGroup响应分发给各个消费者
```

### 4. Heartbeat

Rebalance完成后，消费者通过定期发送心跳来维持与Coordinator的会话。如果心跳停止，Coordinator会认为该消费者已死亡，再次触发Rebalance。

**关键问题：Rebalance期间所有消费者都会暂停消费，等待重新分配完成。** 如果消费者数量多、分区数量大，或者消费者处理逻辑耗时较长，整个Rebalance过程可能持续数分钟甚至更久。

## 四、消费者"崩溃"与关键参数

消费者并非真的"崩溃"才会离开组，很多时候只是因为参数配置不当导致Coordinator误判。理解以下三个参数至关重要：

### 1. session.timeout.ms

**默认值：10s（Kafka 3.x为45s）**

Coordinator在超过该时间未收到消费者的心跳信号后，会认定该消费者已死亡，将其从组中移除并触发Rebalance。

### 2. heartbeat.interval.ms

**默认值：3s（Kafka 3.x为15s）**

消费者向Coordinator发送心跳信号的频率。一般建议设为session.timeout.ms的1/3左右，确保在超时前有足够的心跳机会。

### 3. max.poll.interval.ms

**默认值：5min**

消费者两次调用`poll()`方法的最大允许间隔。如果消费者在拉取消息后处理时间过长，超过该阈值，Consumer会主动向Coordinator发送LeaveGroup请求，离开消费者组。

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka1:9092,kafka2:9092,kafka3:9092");
props.put("group.id", "order-service-group");

// 心跳相关参数
props.put("session.timeout.ms", "60000");      // 60秒无心跳才判定死亡
props.put("heartbeat.interval.ms", "20000");    // 每20秒发一次心跳

// 消费处理相关参数
props.put("max.poll.interval.ms", "600000");    // 允许10分钟处理完一批消息
props.put("max.poll.records", "100");           // 每次poll最多拉100条，降低处理压力
```

**一个典型的误判场景**：消费者处理消息较慢（比如每批消息需要3分钟处理），但max.poll.interval.ms保持默认5分钟。大多数时候能处理完，偶尔遇到一批慢消息耗时6分钟，超过了5分钟限制，消费者主动离开组，触发Rebalance。而其他消费者本来正常消费，也被迫暂停参与Rebalance。

## 五、Rebalance引发的问题

### 1. 资源浪费

Rebalance需要Coordinator协调所有消费者，涉及大量的网络通信和计算，占用Kafka Broker的CPU和网络资源。频繁的Rebalance会显著影响集群性能。

### 2. 消息积压

Rebalance期间，所有消费者暂停消费。如果Rebalance持续30秒，对于一个日均千万级消息的Topic来说，就是近350条消息的积压。如果频繁Rebalance，积压会持续增长。

```
Rebalance前:
  C0 -> partition-0, partition-3  (正常消费)
  C1 -> partition-1, partition-4  (正常消费)
  C2 -> partition-2, partition-5  (正常消费)

Rebalance中（所有消费者暂停消费）:
  C0 -> 暂停，等待分配
  C1 -> 暂停，等待分配
  C2 -> 暂停，等待分配

Rebalance后:
  C0 -> partition-0, partition-1  (恢复消费，但分区变了)
  C1 -> partition-2, partition-3  (恢复消费)
  C2 -> partition-4, partition-5  (恢复消费)
```

### 3. 消息重复

消息重复是Rebalance最常见的问题，根本原因是**offset提交滞后于消息处理**。具体有三种场景：

**场景一：手动提交offset的间隙触发Rebalance**

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        processMessage(record);  // 处理完成
    }
    // 危险窗口：消息已处理，但offset还未提交
    // 如果此时触发Rebalance，新消费者会从上次提交的offset开始消费
    // 导致已处理的消息被重复消费
    consumer.commitSync();
}
```

**场景二：消费超时被踢**

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        processMessage(record);  // 处理耗时过长，超过max.poll.interval.ms
        // 消费者已被踢出组，但消息处理还在继续
        // 处理完后提交offset会失败：CommitFailedException
    }
    try {
        consumer.commitSync();  // 抛出CommitFailedException，offset提交失败
    } catch (CommitFailedException e) {
        // offset没提交成功，下次其他消费者会重新消费这批消息
        log.error("提交offset失败，消息可能被重复消费", e);
    }
}
```

**场景三：找不到已提交的offset**

如果消费者组的`auto.offset.reset`设为`earliest`，当Rebalance后找不到已提交的offset（如offset数据过期被清理），消费者会从最早的消息开始消费，导致大量重复。

```java
// 配置auto.offset.reset为earliest时需格外小心
props.put("auto.offset.reset", "earliest");
// 如果offset数据损坏或过期，Rebalance后会从头开始消费
```

### 4. 消息丢失

消息丢失比重复更严重，发生在offset提交先于消息处理完成的场景。

**场景一：自动提交时丢数据**

```java
// 默认配置：自动提交开启，每5秒提交一次
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "5000");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        // 假设在第3秒时自动提交了offset=100
        // 但消息96-100还没处理完
        // 此时触发Rebalance，新消费者从offset=101开始消费
        // 消息96-100丢失！
        processMessage(record);
    }
}
```

**场景二：手动提交时提交时机错误**

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    // 错误做法：先提交offset再处理消息
    consumer.commitSync();  // offset已提交
    for (ConsumerRecord<String, String> record : records) {
        processMessage(record);  // 如果处理失败，消息丢失
    }
}
```

## 六、Rebalance优化策略

### 1. 调整核心参数

合理调整三个关键参数，避免误判消费者死亡：

```java
Properties props = new Properties();
props.put("session.timeout.ms", "120000");      // 2分钟无心跳才判定死亡
props.put("heartbeat.interval.ms", "40000");     // 40秒发一次心跳（约1/3）
props.put("max.poll.interval.ms", "900000");     // 15分钟处理窗口
props.put("max.poll.records", "50");             // 减少每次拉取量，降低处理压力
```

核心原则：**session.timeout.ms设大一些（1-2分钟），给消费者足够的容错空间；max.poll.interval.ms根据业务处理耗时来设，确保最慢的消息处理也能在超时前完成。**

### 2. 安全处理offset提交

**关闭自动提交，改为手动提交，确保消息处理完成后再提交offset：**

```java
props.put("enable.auto.commit", "false");  // 关闭自动提交

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("order-topic"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        try {
            processMessage(record);
        } catch (Exception e) {
            log.error("消息处理失败, offset={}, retrying...", record.offset(), e);
            // 处理失败不提交，等下次poll重新消费
            continue;
        }
    }
    // 所有消息处理完成后再提交
    try {
        consumer.commitSync();
    } catch (CommitFailedException e) {
        log.error("offset提交失败，可能正在Rebalance", e);
    }
}
```

**更精细的控制——按分区粒度提交offset：**

```java
for (TopicPartition partition : records.partitions()) {
    List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
    for (ConsumerRecord<String, String> record : partitionRecords) {
        processMessage(record);
    }
    // 每个分区处理完就提交该分区的offset
    long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
    consumer.commitSync(Collections.singletonMap(
        partition,
        new OffsetAndMetadata(lastOffset + 1)
    ));
}
```

**结合Kafka事务，确保消息处理和offset提交的原子性：**

```java
// 生产者端：开启事务
props.put("transactional.id", "order-txn-1");
KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();

    // 处理消息 + 发送结果到下游Topic + 提交offset
    producer.send(new ProducerRecord<>("result-topic", key, value));

    // 事务性提交offset——与消息发送是原子的
    producer.sendOffsetsToTransaction(
        offsets,
        consumer.groupMetadata()
    );

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

### 3. 优化分区分配策略

Kafka提供了三种内置的分区分配策略：

| 策略 | 特点 | Rebalance时的行为 |
|---|---|---|
| RangeAssignor | 按分区数值范围连续分配 | 分区大范围变动 |
| RoundRobinAssignor | 轮询分配 | 所有分区重新洗牌 |
| **StickyAssignor** | 尽量保持原有分配 | **只移动必须变动的分区** |

```java
// 使用粘性分配策略，Rebalance时尽量保留原有分配，减少分区变动
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.StickyAssignor");
```

举个例子，假设有3个消费者C0、C1、C2，6个分区p0-p5：

```
原始分配:
  C0: p0, p3
  C1: p1, p4
  C2: p2, p5

C2离开组后:

RoundRobinAssignor（全量重分配）:
  C0: p0, p2, p4    ← 全变了
  C1: p1, p3, p5    ← 全变了

StickyAssignor（只移动C2的分区）:
  C0: p0, p3, p2    ← 保留p0,p3，新增p2
  C1: p1, p4, p5    ← 保留p1,p4，新增p5
```

StickyAssignor的优势显而易见：C0和C1原有的分区分配不变，只需要把C2的分区均匀分配给剩余消费者即可。这大幅减少了分区迁移带来的重复消费和状态重建开销。

### 4. 做好消费幂等性

无论怎么优化，Rebalance都无法完全避免，因此消费逻辑的幂等性是最后的防线。**幂等性保证了即使消息被重复消费，业务结果也不会出错。**

**基于数据库唯一键的幂等：**

```java
public void processMessage(ConsumerRecord<String, String> record) {
    OrderEvent event = JSON.parseObject(record.value(), OrderEvent.class);
    String orderId = event.getOrderId();

    // 利用数据库唯一索引保证幂等
    // 如果orderId已存在，INSERT会失败，不会重复处理
    String sql = "INSERT INTO processed_orders (order_id, status, amount, processed_at) " +
                 "VALUES (?, ?, ?, NOW())";
    try {
        jdbcTemplate.update(sql, orderId, event.getStatus(), event.getAmount());
    } catch (DuplicateKeyException e) {
        // 订单已处理过，直接跳过——这就是幂等
        log.info("订单已处理，跳过: orderId={}", orderId);
        return;
    }

    // 执行业务逻辑
    orderService.handleOrder(event);
}
```

**基于Redis的幂等检查：**

```java
public void processMessage(ConsumerRecord<String, String> record) {
    OrderEvent event = JSON.parseObject(record.value(), OrderEvent.class);
    String orderId = event.getOrderId();
    String dedupeKey = "dedupe:order:" + orderId;

    // SETNX：如果key不存在则设置成功返回true，已存在返回false
    Boolean isNew = redisTemplate.opsForValue()
        .setIfAbsent(dedupeKey, "1", Duration.ofHours(24));

    if (Boolean.FALSE.equals(isNew)) {
        log.info("订单已处理，跳过: orderId={}", orderId);
        return;
    }

    try {
        orderService.handleOrder(event);
    } catch (Exception e) {
        // 处理失败，删除去重标记，允许下次重试
        redisTemplate.delete(dedupeKey);
        throw e;
    }
}
```

**基于状态机的幂等（适用于有状态流转的业务）：**

```java
public void processPayment(PaymentEvent event) {
    Order order = orderRepository.findById(event.getOrderId());

    // 状态机天然幂等：只有当前状态为PENDING_PAYMENT时才执行支付逻辑
    // 如果重复消费，第二次进来时状态已经是PAID，直接跳过
    if (order.getStatus() == OrderStatus.PENDING_PAYMENT) {
        order.pay(event.getPaymentId());
        orderRepository.save(order);
    } else {
        log.info("订单状态不是待支付，跳过: orderId={}, status={}",
                 order.getId(), order.getStatus());
    }
}
```

## 七、Kafka 2.4+ 的 CooperativeStickyAssignor

Kafka 2.4引入了增量协作式的Rebalance协议（KIP-429），对应的分配器是`CooperativeStickyAssignor`，它解决了传统Rebalance"Stop The World"的问题。

```java
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

传统Rebalance（Eager协议）的流程：

```
1. 所有消费者撤销全部分区 → 全部暂停消费
2. 重新JoinGroup、SyncGroup → 重新分配
3. 所有消费者按新分配方案消费 → 全部恢复
```

CooperativeStickyAssignor的流程：

```
1. 只有需要变动的消费者撤销分区 → 大部分消费者不受影响
2. 增量式重新分配 → 只分配必须变动的分区
3. 只有受影响的消费者切换分区 → 平滑过渡
```

举例说明：C0、C1、C2消费6个分区，C3新加入组：

```
Eager协议：
  全部消费者撤销分区 → 全部暂停 → 重新分配 → 全部恢复
  C0: p0,p3 → 暂停 → p0,p1
  C1: p1,p4 → 暂停 → p2,p3
  C2: p2,p5 → 暂停 → p4,p5
  C3: (新加入)       → p0  ← 需要两轮Rebalance

CooperativeStickyAssignor：
  只有需要变动的消费者调整，其他不受影响
  第一轮：C2把p2让给C3 → C0和C1完全不受影响
  C0: p0,p3 （不变）
  C1: p1,p4 （不变）
  C2: p5     （让出p2）
  C3: p2     （新增）
```

CooperativeStickyAssignor将"全员暂停"变为"按需调整"，大幅降低了Rebalance对消费的影响。

## 八、监控与排查

生产环境中，及时发现和定位Rebalance问题同样重要。以下是关键的监控指标和排查思路：

### 1. 关键JMX指标

| 指标 | 含义 | 告警阈值建议 |
|---|---|---|
| `commit-rate` | offset提交速率 | 骤降可能正在Rebalance |
| `join-rate` | JoinGroup请求速率 | 频繁则说明反复Rebalance |
| `sync-rate` | SyncGroup请求速率 | 频繁则说明反复Rebalance |
| `last-rebalance-seconds-ago` | 距上次Rebalance的秒数 | 频繁出现需排查 |

### 2. 排查Rebalance原因

```bash
# 查看消费者组状态
kafka-consumer-groups.sh --bootstrap-server kafka1:9092 \
  --describe --group order-service-group

# 查看Coordinator日志，关注Rebalance触发原因
# 常见日志关键字：
#   "Member xxx leaving group" → 消费者主动离开
#   "Session timeout expired"  → 心跳超时
#   "max.poll.interval.ms exceeded" → 处理超时
```

### 3. 常见Rebalance模式与排查方向

| 现象 | 可能原因 | 排查方向 |
|---|---|---|
| 每隔约5分钟Rebalance一次 | max.poll.interval.ms超时 | 检查消费处理耗时 |
| 每隔约10秒Rebalance一次 | session.timeout.ms超时 | 检查消费者是否GC或网络抖动 |
| 新实例上线后反复Rebalance | 分配策略不合理或消费者逻辑异常 | 切换StickyAssignor，检查消费逻辑 |
| 所有消费者同时Rebalance | Eager协议全员参与 | 升级到CooperativeStickyAssignor |

## 写在最后

Rebalance是Kafka消费者组自我修复的核心机制，但它也是一把双刃剑：在保障高可用的同时，又可能带来消息积压、重复消费甚至丢失等问题。

理解Rebalance的触发条件、执行过程和关键参数，是做好Kafka消费端稳定性的基础。在此基础上，通过合理调参、安全提交offset、选择粘性分配策略、保证消费幂等性，以及升级到增量协作式Rebalance协议，可以最大程度地降低Rebalance对业务的影响。

**Rebalance不可怕，可怕的是不知道它为什么发生、不知道它带来了什么影响。**
