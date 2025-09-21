---
title: Kafka快速认知 (draft)
published: 2025-08-11
tags: 
- Kafka
lang: zh
abbrlink: kafkacognition-start 
---

### KafKa 结构


### KafKa Topic
> topic 主题为消息队列，内部Partition分区提高消息的吞吐；<br>
>文件结构 dir：<br>
>| --{topic1}-0   <br> 
> 	| -- 00000000xx.index <br>
> | -- 00000000xx.log <br>
> | -- ...  <br>
> index : N: 表示第N个Message，position表示消息的偏移地址

![分段日志文件](https://img2024.cnblogs.com/blog/3426265/202509/3426265-20250921081126954-365637170.png)

### KafKa 生产者
### KafKa 消费者

### KafKa 常见问题
##### 可用性和持久性保证
保证数据完整性，一致性：
ISR： Kafka 动态维护了一个同步状态的备份节点的集合 （a set of in-sync replicas）， 简称 ISR ，在这个集合中的节点都是和 leader 保持高度一致的，只有这个集合的成员才 有资格被选举为 leader，一条消息必须被这个集合 所有 节点读取并追加到日志中了，这条消息才能视为提交。
ISR控制参数：剔出ISR集合
`replica.lag.max.messages  before 0.9x`
`replica.lag.time.max.ms   after 0.9x`
producer：
ack： 0：不等待broker返回确认消息；
   1: leader保存成功返回或；
  -1(all): 所有备份都保存成功返回
注意：ack为all时，只要ISR同步完成就返回，例如多个副本下，可能没有全部副本完成同步  
方案：通过指定ISR集合大小减小数据丢失的可能性，仅在all下可用，降低可用性 
min.insync.replicas 配置ISR集合大小，一旦ISR大小少于配置数量，不再提供写能力
consumer:
1. 自动确认模式
● RECORD
每条消息处理完成后立即提交 Offset。
风险：若消费逻辑未完成时消费者宕机，消息可能丢失[12] [38]。
● BATCH
处理完一批消息后批量提交 Offset。
适用场景：批量消费时减少提交频率，提升性能[17] [30]。
● TIME
按固定时间间隔提交 Offset（如每 5 秒）。
缺点：若间隔过长且消费频率高，宕机时可能重复消费大量消息[41]。
● COUNT
累积处理指定数量消息后提交 Offset（如每 100 条）。
平衡点：在吞吐量和数据可靠性间折中[12] [33]。
2. 手动确认模式
● MANUAL
手动调用 acknowledge() 方法提交 Offset，通常在业务逻辑完成后执行。
示例：  优点：避免消息处理未完成时提交 Offset 导致数据丢失[19] [40]。
● MANUAL_IMMEDIATE
与 MANUAL 类似，但提交 Offset 的请求会立即发送至 Broker，而非等待批次处理。
性能影响：频繁提交可能降低吞吐量，适合对可靠性要求极高的场景[14] [38]。
死信队列 、延迟队列
死信队列: 利用消息消费errhandler，创建DeadLetterPublishingRecoverer对象，死信消费topic为对应topic.DLT;消费者消费异常
@Bean
@Primary
public ErrorHandler kafkaErrorHandler(KafkaTemplate<?, ?> template) {

    logger.warn("kafkaErrorHandler begin to Handle");

    // <1> 创建 DeadLetterPublishingRecoverer 对象
    ConsumerRecordRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
    // <2> 创建 FixedBackOff 对象   设置重试间隔 10秒 次数为 3次
    BackOff backOff = new FixedBackOff(10 * 1000L, 3L);
    // <3> 创建 SeekToCurrentErrorHandler 对象
    return new SeekToCurrentErrorHandler(recoverer, backOff);
}
Windows Kafka异常退出
kafka日志清理策略触发，在window环境下，在打开需要清理的日志的同时，对该文件进行重命名操作是不被允许的，从而导致kafka宕机。




消费打点
Callback:实现此接口，自定义逻辑，当消息消费完成执行
ProducerInterceptor:在生产者将消息发布到server之前拦截 
