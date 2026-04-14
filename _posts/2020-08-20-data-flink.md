---
title: Flink 算子实战 —— 基于交通数据的实时处理
author: morric
date: 2020-08-20 23:48:00 +0800
categories: [开发记录, 大数据]
tags: [flink, 大数据, 实时计算]
---

在交通数据平台的建设过程中，我们需要对车辆 GPS 轨迹、路口车流量、违章事件等多类数据进行实时处理。数据全部通过 Kafka 接入，经过 Flink 清洗、聚合、窗口计算和多流关联后，分别写入 Kafka、MySQL 和 Redis。

本文记录在这个场景下实际用到的 Flink DataStream 算子，结合真实的数据结构和处理逻辑进行说明。

---

## 数据结构

系统中主要涉及三类数据：

**车辆 GPS 轨迹**（来自车载设备，高频上报）：
```scala
case class GpsRecord(
  vehicleId: String,   // 车辆ID
  lon: Double,         // 经度
  lat: Double,         // 纬度
  speed: Double,       // 速度（km/h）
  timestamp: Long      // 上报时间戳
)
```

**路口车流量**（来自地感线圈或视频识别）：
```scala
case class TrafficFlow(
  intersectionId: String,  // 路口ID
  direction: String,        // 方向（东/南/西/北）
  count: Int,               // 过车数量
  timestamp: Long
)
```

**违章事件**（来自电子警察）：
```scala
case class ViolationEvent(
  vehicleId: String,
  violationType: String,  // 违章类型（闯红灯/超速/压线等）
  location: String,
  timestamp: Long
)
```

---

## 一、Source：从 Kafka 接入数据

三类数据分别对应三个 Kafka Topic，通过 `addSource` 接入：

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val properties = new Properties()
properties.setProperty("bootstrap.servers", "node01:9092,node02:9092")
properties.setProperty("group.id", "traffic-flink-consumer")
properties.setProperty("auto.offset.reset", "latest")

// GPS 数据流
val gpsStream: DataStream[String] = env.addSource(
  new FlinkKafkaConsumer011[String]("topic-gps", new SimpleStringSchema(), properties)
)

// 车流量数据流
val flowStream: DataStream[String] = env.addSource(
  new FlinkKafkaConsumer011[String]("topic-flow", new SimpleStringSchema(), properties)
)

// 违章事件流
val violationStream: DataStream[String] = env.addSource(
  new FlinkKafkaConsumer011[String]("topic-violation", new SimpleStringSchema(), properties)
)
```

---

## 二、Transform：数据处理

### 1. map —— 反序列化与数据转换

原始数据是 JSON 字符串，首先用 `map` 转换为样例类，同时分配事件时间戳和 Watermark：

```scala
val gpsRecords: DataStream[GpsRecord] = gpsStream
  .map(json => JSON.parseObject(json, classOf[GpsRecord]))
  .assignTimestampsAndWatermarks(
    new BoundedOutOfOrdernessTimestampExtractor[GpsRecord](Time.seconds(5)) {
      override def extractTimestamp(record: GpsRecord): Long = record.timestamp
    }
  )
```

> 交通数据上报存在网络延迟，Watermark 设置了 5 秒的乱序容忍窗口，避免迟到数据被直接丢弃。

### 2. filter —— 数据清洗过滤

GPS 数据中存在几类脏数据需要过滤：坐标为 0 的异常值、速度超出合理范围的记录、心跳包（无实际位移）：

```scala
val cleanGps: DataStream[GpsRecord] = gpsRecords.filter { record =>
  record.lon != 0.0 &&
  record.lat != 0.0 &&
  record.speed >= 0 &&
  record.speed <= 200  // 超过200km/h视为异常数据
}
```

违章事件同样需要过滤掉 `vehicleId` 为空的脏数据：

```scala
val cleanViolation: DataStream[ViolationEvent] = violationStream
  .map(json => JSON.parseObject(json, classOf[ViolationEvent]))
  .filter(event => event.vehicleId != null && event.vehicleId.nonEmpty)
```

### 3. keyBy —— 按车辆或路口分组

对 GPS 数据按车辆 ID 分组，后续计算每辆车的轨迹和行为：

```scala
val keyedGps: KeyedStream[GpsRecord, String] = cleanGps.keyBy(_.vehicleId)
```

对车流量数据按路口 ID 分组，统计各路口的实时流量：

```scala
val keyedFlow: KeyedStream[TrafficFlow, String] = flowStream
  .map(json => JSON.parseObject(json, classOf[TrafficFlow]))
  .keyBy(_.intersectionId)
```

### 4. window —— 窗口计算

**每 5 分钟统计各路口的过车总数**，使用滚动事件时间窗口：

```scala
val flowStat: DataStream[(String, Int)] = keyedFlow
  .window(TumblingEventTimeWindows.of(Time.minutes(5)))
  .reduce((a, b) => TrafficFlow(a.intersectionId, a.direction, a.count + b.count, b.timestamp))
  .map(r => (r.intersectionId, r.count))
```

**超速检测**：对每辆车使用滑动窗口，计算过去 1 分钟内的平均速度，超过限速阈值则告警：

```scala
val speedAlert: DataStream[String] = keyedGps
  .window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(30)))
  .apply(new WindowFunction[GpsRecord, String, String, TimeWindow] {
    override def apply(
      key: String,
      window: TimeWindow,
      input: Iterable[GpsRecord],
      out: Collector[String]
    ): Unit = {
      val records = input.toList
      val avgSpeed = records.map(_.speed).sum / records.size
      if (avgSpeed > 120) {
        out.collect(s"超速告警 车辆:$key 均速:${avgSpeed}km/h 时间窗口:${window.getEnd}")
      }
    }
  })
```

### 5. flatMap —— 轨迹点展开

部分设备会批量上报一段时间内的多个 GPS 点，使用 `flatMap` 将批量数据拆分为单条记录：

```scala
val flatGps: DataStream[GpsRecord] = gpsStream
  .flatMap(json => {
    val list = JSON.parseArray(json, classOf[GpsRecord])
    list.asScala
  })
```

### 6. connect + CoMap —— 多流关联

将超速告警流与违章事件流关联，对同一辆车在同一时间段内同时存在超速和违章的情况进行合并处理：

```scala
// 将两个流统一为 String 类型后 connect
val alertStream: DataStream[String] = speedAlert.map(msg => ("speed", msg))
val vioStream: DataStream[String] = cleanViolation.map(v => ("violation", s"${v.vehicleId}|${v.violationType}"))

val connectedStream: ConnectedStreams[(String, String), (String, String)] =
  alertStream.connect(vioStream)

val mergedStream = connectedStream.map(
  speedMsg => s"[超速] ${speedMsg._2}",
  vioMsg   => s"[违章] ${vioMsg._2}"
)
```

### 7. union —— 合并告警流

将超速告警和违章告警合并为统一的告警流，统一写入下游：

```scala
val allAlerts: DataStream[String] = speedAlert.union(mergedStream)
```

### 8. rebalance —— 解决数据倾斜

路口车流量数据中，城市主干道路口的数据量远大于普通路口，直接处理会出现严重数据倾斜。在进入窗口计算前做一次 `rebalance`：

```scala
val balancedFlow = keyedFlow.rebalance()
```

---

## 三、Sink：结果输出

处理结果分三个方向写出：

### 写入 Kafka（实时告警下发）

```scala
val prop = new Properties()
prop.setProperty("bootstrap.servers", "node01:9092")

val kafkaProducer = new FlinkKafkaProducer011[String](
  "topic-alert",
  new KeyedSerializationSchemaWrapper[String](new SimpleStringSchema()),
  prop
)

allAlerts.addSink(kafkaProducer)
```

### 写入 MySQL（路口流量统计持久化）

```scala
flowStat.addSink(new RichSinkFunction[(String, Int)] {
  var conn: Connection = _

  override def open(parameters: Configuration): Unit = {
    conn = DriverManager.getConnection(
      "jdbc:mysql://node01:3306/traffic", "root", "password"
    )
  }

  override def invoke(value: (String, Int), context: Context): Unit = {
    val sql = "INSERT INTO intersection_flow(intersection_id, flow_count, stat_time) VALUES (?, ?, NOW()) ON DUPLICATE KEY UPDATE flow_count = ?"
    val ps = conn.prepareStatement(sql)
    ps.setString(1, value._1)
    ps.setInt(2, value._2)
    ps.setInt(3, value._2)
    ps.executeUpdate()
  }

  override def close(): Unit = if (conn != null) conn.close()
})
```

### 写入 Redis（实时车辆位置缓存）

```scala
cleanGps.addSink(new RichSinkFunction[GpsRecord] {
  var jedis: Jedis = _

  override def open(parameters: Configuration): Unit = {
    jedis = new Jedis("node01", 6379)
  }

  override def invoke(record: GpsRecord, context: Context): Unit = {
    // 以车辆ID为key，缓存最新位置，过期时间5分钟
    val value = s"${record.lon},${record.lat},${record.speed},${record.timestamp}"
    jedis.setex(s"vehicle:loc:${record.vehicleId}", 300, value)
  }

  override def close(): Unit = if (jedis != null) jedis.close()
})
```

---

## 四、启动执行

```scala
env.execute("交通数据实时处理")
```

---

## 实践总结

在交通数据实时处理场景中，以下几点值得注意：

**Watermark 设置要贴合实际网络延迟**。车载设备通过 4G 上报数据，隧道、地库等弱网环境会导致数据延迟 10 秒以上，Watermark 容忍窗口不能设得太小，否则大量数据会被当作迟到数据丢弃。

**主干路口数据倾斜是常见问题**。早晚高峰期间，几个核心路口的数据量是普通路口的数十倍，`rebalance` 或自定义分区策略是必要的。

**GPS 数据脏数据比例较高**。实测中约有 3%～5% 的 GPS 记录存在坐标漂移或速度异常，`filter` 清洗这一步不能省略，否则会污染下游的统计结果。

**Redis 缓存实时位置要设置过期时间**。车辆停驶后不会再上报数据，如果不设 TTL，Redis 中会积累大量失效的位置记录。