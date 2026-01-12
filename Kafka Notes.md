# Kafka Notes

## 目录

- [Kafka Notes](#kafka-notes)
  - [目录](#目录)
  - [一、架构概览](#一架构概览)
  - [二、核心概念](#二核心概念)
    - [2.1 Topic](#21-topic)
    - [2.2 Partition](#22-partition)
    - [2.3 Offset](#23-offset)
    - [2.4 Replication](#24-replication)
  - [三、生产机制](#三生产机制)
    - [3.1 发送方式](#31-发送方式)
    - [3.2 Message Key](#32-message-key)
    - [3.3 消息结构](#33-消息结构)
    - [3.4 序列化](#34-序列化)
  - [四、消费机制](#四消费机制)
    - [4.1 消费模型](#41-消费模型)
    - [4.2 消费者组](#42-消费者组)
    - [4.3 Offset 管理](#43-offset-管理)
    - [4.4 反序列化](#44-反序列化)
  - [五、投递语义](#五投递语义)
  - [六、核心配置](#六核心配置)
  - [七、架构关系图](#七架构关系图)

---

## 一、架构概览

**Kafka 是什么？**

分布式消息队列系统，用于处理实时数据流。

**核心架构**

```
Producer → Broker Cluster → Consumer
   ↑            ↑              ↑
 生产消息      存储消息       消费消息
```

**基本组件**

| 组件 | 职责 |
|------|------|
| Producer | 生产消息，发送到 Broker |
| Broker | 存储消息，处理读写请求 |
| Consumer | 从 Broker 拉取并处理消息 |
| ZooKeeper | 管理集群元数据（旧版本） |

[↑ 返回目录](#目录)

---

## 二、核心概念

### 2.1 Topic

**定义**

消息的逻辑分类容器，类似数据库中的表。

**特性**

- 命名具有业务含义（如 `orders`、`user-events`）
- 一个 Topic 可包含多个 Partition
- Producer/Consumer 通过 Topic 名称进行消息收发

[↑ 返回目录](#目录)

---

### 2.2 Partition

**定义**

Topic 的物理分片，实现并行处理和扩展。

**关键特性**

| 特性 | 说明 |
|------|------|
| 单分区有序 | 同一分区内消息按顺序存储 |
| 多分区并行 | 不同分区可并行处理 |
| 负载均衡 | 分区分布在不同 Broker 上 |

**分区示例**

```
Topic: "orders" (3个分区)
├── Partition 0  [Offset: 0,1,2,3...]
├── Partition 1  [Offset: 0,1,2,3...]
└── Partition 2  [Offset: 0,1,2,3...]
```

**消费规则**

消费者数量 ≤ 分区数量（一个分区同时只能被一个消费者消费）

[↑ 返回目录](#目录)

---

### 2.3 Offset

**定义**

分区内消息的唯一位置标识，从 0 开始递增。

**作用**

- 标识消息在分区中的位置
- 追踪消费者的消费进度
- 支持消息回溯和重新消费

**特性**

- 分区独立，每个分区有独立的 Offset 序列
- Consumer 通过 Offset 控制消费位置

[↑ 返回目录](#目录)

---

### 2.4 Replication

**目的**

数据冗余，保证高可用性。

**核心概念**

| 概念 | 说明 |
|------|------|
| Leader | 处理所有读写请求 |
| Follower | 只从 Leader 同步数据，不处理客户端请求 |
| ISR (In-Sync Replicas) | 与 Leader 保持同步的副本集合 |
| Replication Factor | 如果为3: 每个分区被3个broker拥有 最多可以丢失2个broker

**分区分布示例**

```
Topic: "orders" (3 partitions, 2 replicas)

Broker 1         Broker 2         Broker 3
│                 │                 │
├─ P0 (Leader)   ├─ P0 (Replica)  ├─ P1 (Leader)
├─ P1 (Replica)  ├─ P2 (Leader)   ├─ P2 (Replica)
└─ P2 (Replica)  └─ P1 (Replica)  └─ P0 (Replica)
```

[↑ 返回目录](#目录)

---

## 三、生产机制

### 3.1 发送方式

| 方式 | 特点 | 风险 |
|------|------|------|
| Fire-and-forget | 发送后不等待响应 | 可能丢失数据 |
| Synchronous | 同步等待确认 | 吞吐量较低 |
| Asynchronous | 异步回调处理 | 需处理失败情况 |

**对应的acks 参数（确认级别）**

| 值 | 含义 | 数据安全 | 性能 | 适用场景 |
|------|------|----------|------|----------|
| 0 | 不等待任何确认 | 最低（可能丢数据） | 最高 | 允许丢失的日志 |
| 1 | Leader 写入成功即可 | 中等（Leader 挂了会丢） | 较高 | 一般业务 |
| -1 (all) | 所有 ISR 副本都确认 | 最高（不丢数据） | 较低 | 金融、支付等 |

**acks 选择策略**

| 考量因素 | 选择 acks=0/1 | 选择 acks=all |
|----------|---------------|---------------|
| 数据重要性 | 可容忍丢失 | 不可丢失 |
| 吞吐量要求 | 高吞吐优先 | 可接受延迟 |
| 网络环境 | 稳定 | 不稳定时更安全 |
| 典型场景 | 用户行为日志 | 订单、支付 |

**acks 与 ISR 的关系**

acks=all 只需等待 ISR 副本确认，无需等待所有副本：
- ISR 副本数受 `min.insync.replicas` 参数控制
- 建议 min.insync.replicas ≥ 2，保证至少 2 个副本有数据

[↑ 返回目录](#目录)

---

### 3.2 Message Key

**作用**

决定消息路由到哪个分区。

**路由规则**

```
无 Key → 随机选择分区（负载均衡）
有 Key → Hash(Key) % 分区数（固定路由）
```

**Hash 算法**

- 使用 Murmur2 算法
- 相同 Key 永远路由到同一分区
- **增加分区数会改变路由！**

**使用场景**

| 场景 | 需要 Key |
|------|----------|
| 日志收集 | 否 |
| 用户行为事件 | 是（同一用户保序） |
| 订单状态变更 | 是（同一订单保序） |

[↑ 返回目录](#目录)

---

### 3.3 消息结构

**生产者发送时**

```
Topic + Key + Value + Headers
 ↑      ↑     ↑        ↑
必填   可选  必填     可选元数据
```

**Broker 存储时**

```
Offset + Key + Value + Headers
  ↑      ↑     ↑        ↑
位置   路由  业务数据  元数据
```

**Headers 用途**

- 消息追踪（Trace ID）
- 版本控制
- 其他元数据（Kafka 0.11+ 支持）

[↑ 返回目录](#目录)

---

### 3.4 序列化

**定义**

将对象转换为字节流，便于网络传输和存储。

**常用序列化器**

| 序列化器 | 适用场景 |
|---------|----------|
| StringSerializer | JSON、文本数据 |
| ByteArraySerializer | 二进制数据 |
| 自定义 JSON 序列化器 | 跨语言场景（推荐） |

**注意**

- Producer 的 Serializer 与 Consumer 的 Deserializer 必须匹配
- 同一个 Topic 的序列化格式不可改变

[↑ 返回目录](#目录)

---

## 四、消费机制

### 4.1 消费模型

**基本流程**

```
Consumer → poll() → 批量拉取 → 处理消息 → 提交 Offset
```

**特点**

- 主动拉取模式（Pull-based）
- 批量拉取提高效率
- 需要手动控制消费进度

[↑ 返回目录](#目录)

---

### 4.2 消费者组

**定义**

多个消费者组成的逻辑组，共同消费一个 Topic。

**消费规则**

- 一个分区在同一时间只能被组内一个消费者消费
- 不同消费者组互不影响，可独立消费同一 Topic

**示例**

```
Topic: "orders" (3 partitions)

Group A (3 consumers):
├── C1 → Partition 0
├── C2 → Partition 1
└── C3 → Partition 2

Group B (1 consumer):
└── C1 → Partition 0,1,2 (消费所有分区)
```

**作用**

- **负载均衡**：多消费者并行处理
- **容错**：消费者挂掉时自动重新分配分区
- **独立消费**：不同组可各自处理相同消息

[↑ 返回目录](#目录)

---

### 4.3 Offset 管理

**存储位置**

存储在内部 Topic `__consumer_offsets` 中：
- Key: `group.id + topic + partition`
- Value: `offset + metadata`

**提交策略**

| 策略 | 优点 | 缺点 |
|------|------|------|
| 自动提交 | 简单易用 | 可能重复消费或丢失 |
| 手动同步提交 | 精确控制 | 阻塞处理，吞吐量低 |
| 手动异步提交 | 高吞吐量 | 可能提交失败 |

**auto.offset.reset 参数**

| 值 | 行为 | 场景 |
|------|------|------|
| earliest | 从最早的消息开始 | 新消费者组 |
| latest | 从最新的消息开始 | 默认值 |
| none | 报错 | 必须已有 Offset |

[↑ 返回目录](#目录)

---

### 4.4 反序列化

**定义**

将字节流转换为对象，与序列化过程相反。

**要求**

- 必须与 Producer 的序列化格式一致
- 使用对应的 Deserializer

**常用反序列化器**

| 反序列化器 | 对应序列化器 |
|-----------|-------------|
| StringDeserializer | StringSerializer |
| ByteArrayDeserializer | ByteArraySerializer |

[↑ 返回目录](#目录)

---

## 五、投递语义

| 语义 | 提交时机 | 风险 | 适用场景 |
|------|----------|------|----------|
| At most once | 接收后立即提交 | 可能丢消息 | 允许数据丢失 |
| At least once | 处理完成后提交 | 可能重复消费 | **推荐**（需幂等） |
| Exactly once | 事务机制 | 实现复杂 | Kafka → Kafka 场景 |

**At least once（推荐）**

- 处理完成后才提交 Offset
- 处理失败会重试，确保至少处理一次
- 需要业务层实现幂等性

**At most once**

- 接收后立即提交 Offset
- 处理失败则丢失，不会重试
- 适用于可容忍数据丢失的场景

**Exactly once**

- Kafka → Kafka: 使用 Transactional API 或 Kafka Streams
- Kafka → 外部系统: 需要幂等消费者

[↑ 返回目录](#目录)

---

## 六、核心配置

**Producer 关键配置**

| 参数 | 作用 |
|------|------|
| bootstrap.servers | Broker 集群地址 |
| key.serializer | Key 序列化器 |
| value.serializer | Value 序列化器 |
| acks | 确认级别（0/1/all） |
| retries | 重试次数 |

**Consumer 关键配置**

| 参数 | 作用 |
|------|------|
| bootstrap.servers | Broker 集群地址 |
| key.deserializer | Key 反序列化器 |
| value.deserializer | Value 反序列化器 |
| group.id | 消费者组 ID |
| auto.offset.reset | 初始 Offset 策略 |
| enable.auto.commit | 是否自动提交 |

**bootstrap.servers 说明**

- 只需配置部分 Broker 地址
- 客户端会自动发现集群中的所有 Broker
- 建议配置多个以提高可用性

[↑ 返回目录](#目录)

---

## 七、架构关系图

**完整数据流**

```
┌─────────────────────────────────────────────────────────┐
│                    Kafka Cluster                         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │              │
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
    │         │                      │ Group   │
    └─────────┘                      └─────────┘
```

**概念关系**

```
Topic
  ├─ Partition (并行扩展)
  │   ├─ Offset (位置追踪)
  │   └─ Leader/Follower (高可用)
  └─ Replication (数据冗余)

Producer
  ├─ Message Key (路由策略)
  ├─ Serialization (数据转换)
  └─ acks (可靠性)

Consumer
  ├─ Consumer Group (负载均衡)
  ├─ Offset 提交 (消费进度)
  └─ Deserialization (数据还原)
```

[↑ 返回目录](#目录)