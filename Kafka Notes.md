# Kafka Notes

## 核心概念

| 概念 | 作用 | 核心特性 |
|------|------|----------|
| **Topic** | 消息分类 | 逻辑容器 |
| **Partition** | 并行扩展 | 单分区有序，多分区并行 |
| **Offset** | 位置追踪 | 唯一标识 + 消费进度 |

---

### Topic & Partition & Offset

```
Topic: "orders"
├── Partition 0  [Offset: 0,1,2,3...]
├── Partition 1  [Offset: 0,1,2,3...]
└── Partition 2  [Offset: 0,1,2,3...]

消费者数 ≤ 分区数 （一个分区只允许有一个消费者）
```

**Partition**: 单个分区有序，跨分区无序

**Offset**: 分区内唯一位置标识，从 0 开始

---

### Producer

**发送方式**
- Fire-and-forget: 发送不确认（可能丢数据）
- Synchronous: 同步等待响应
- Asynchronous: 异步回调

**核心配置**
```properties
bootstrap.servers=localhost:9092
key.serializer=StringSerializer
value.serializer=StringSerializer
acks=1  # 0/1/all
retries=3
```

---

### Message Key

```
无 Key → 随机分区（负载均衡）
有 Key → Hash(Key) % 分区数（固定分区，保序）
```

**何时用 Key？**

| 场景 | 需要 Key |
|------|----------|
| 日志收集 | ❌ |
| 用户事件 | ✅ 同一用户需保序 |
| 订单状态 | ✅ 同一订单需保序 |

```java
new ProducerRecord<>("orders", "order-123", "{\"status\":\"shipped\"}");
//                    ↑Topic     ↑Key        ↑Value
```

---

### Serialization

**对象 → 字节流**

| 序列化器 | 适用场景 |
|---------|----------|
| StringSerializer | JSON、文本 |
| ByteArraySerializer | 二进制 |
| 自定义 JSON | 推荐（跨语言） |

---

### Key Hashing（Murmur2 alg）

```java
partition = abs(murmur2(key)) % partitionCount
```

**增加分区数会改变路由！**

---

### Message Structure

**ProducerRecord（生产者发送时）**
```
Topic + Key + Value + Headers
 ↑     ↑     ↑        ↑
必填  可选  必填     可选
```

**Broker 存储（追加 Offset 后）**
```
Offset + Key + Value + Headers
  ↑      ↑     ↑        ↑
位置   路由  业务数据  元数据
```

**Headers**: 追踪、版本控制等元数据（Kafka 0.11+）

---

### Consumer

**消费模型**
```
Consumer → poll() → 批量拉取消息 → 处理 → 提交 Offset
```

**核心配置**
```properties
bootstrap.servers=localhost:9092
key.deserializer=StringDeserializer
value.deserializer=StringDeserializer
group.id=my-group
auto.offset.reset=earliest  # earliest/latest/none
enable.auto.commit=false    # 手动提交更安全
```

**消费者组（Consumer Group）**
```
Topic: "orders" (3 partitions)

Group A (3 consumers):
├── C1 → Partition 0
├── C2 → Partition 1
└── C3 → Partition 2

Group B (1 consumer):
└── C1 → Partition 0,1,2 (消费所有分区)
```


**消费者组作用**
- **负载均衡**：多消费者并行处理
- **容错**：消费者挂掉，分区重新分配
- **独立消费**：不同组互不影响

---

### Consumer Offset

**Offset 存储位置**
```
__consumer_offsets topic (内部 topic)
├── Key: group.id + topic + partition
└── Value: offset + metadata
```

**Offset 提交策略**

| 策略 | 优点 | 缺点 |
|------|------|------|
| 自动提交 | 简单 | 可能重复消费或丢失 |
| 手动同步 | 精确控制 | 阻塞，吞吐低 |
| 手动异步 | 高吞吐 | 可能提交失败 |

---

### Delivery Semantics（投递语义）

| 语义 | 提交时机 | 风险 | 适用场景 |
|------|----------|------|----------|
| **At most once** | 接收后立即提交 | 可能丢消息 | 允许丢失 |
| **At least once** ✅ | 处理完成后提交 | 可能重复 | 推荐（需幂等） |
| **Exactly once** | 事务 API | 复杂 | Kafka→Kafka 用 Streams |

**At least once**（默认推荐）
```java
// 处理完成后提交，处理失败会重试
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);  // 需幂等性
    }
    consumer.commitSync();
}
```

**At most once**
```java
// 接收后立即提交，处理失败则丢失
consumer.commitSync();  // 先提交
process(records);       // 后处理
```

**Exactly once**
- Kafka → Kafka: 使用 Transactional API / Kafka Streams
- Kafka → 外部系统: 幂等消费者

**auto.offset.reset**
- `earliest`: 从最早开始（新组）
- `latest`: 从最新开始（默认）
- `none`: 报错（必须已有 offset）

---

### Deserialization

**字节流 → 对象**（与 Producer 序列化格式对应）

**Producer/Consumer 序列化格式必须一致！**

**同一个topic中serializer和deserializer的格式不可以改变！**

```java
// Consumer 配置
Properties props = new Properties();
props.put("key.deserializer", "StringDeserializer");
props.put("value.deserializer", "StringDeserializer");

Consumer<String, String> consumer = new KafkaConsumer<>(props);
```

---

### Broker & Cluster

**Kafka Client → Broker Cluster 关系**
```
┌─────────────────────────────────────────────────────────┐
│                    Kafka Cluster                         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │  (3节点集群)  │
│  │ :9092    │  │ :9092    │  │ :9092    │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│       │             │             │                      │
│       └─────────────┴─────────────┘                      │
│                     │                                    │
│               ZooKeeper (元数据)                         │
└─────────────────────────────────────────────────────────┘
                          ↑
                          │ bootstrap.servers
                          │
         ┌────────────────┴────────────────┐
         │                                 │
    ┌─────────┐                      ┌─────────┐
    │Producer │                      │Consumer │
    └─────────┘                      └─────────┘
```

**Broker 职责**
- 接收 Producer 发送的消息
- 存储分区数据（日志文件）
- 响应 Consumer 拉取请求
- 分区 Leader 选举

**bootstrap.servers**
```properties
# 客户端连接配置（只需配置部分 Broker）
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092

# 客户端会自动发现集群中的所有 Broker
```

**Partition 分布**
```
Topic: "orders" (3 partitions, 2 replicas)

Broker 1         Broker 2         Broker 3
│                 │                 │
├─ P0 (Leader)   ├─ P0 (Replica)  ├─ P1 (Leader)
├─ P1 (Replica)  ├─ P2 (Leader)   ├─ P2 (Replica)
└─ P2 (Replica)  └─ P1 (Replica)  └─ P0 (Replica)
```

**关键概念**
- **Leader**: 处理读写请求
- **Replica/Follower**: 只复制数据，不处理客户端请求
- **ISR**: In-Sync Replicas（与 Leader 同步的副本集）

---

### 总结

| 概念 | 决策 |
|------|------|
| Producer | 同步/异步？重试？ |
| Consumer | 手动/自动提交？ |
| Key | 是否保序？ |
| Serialization | JSON/Avro？ |
| Hashing | 预估分区数，避免变更 |