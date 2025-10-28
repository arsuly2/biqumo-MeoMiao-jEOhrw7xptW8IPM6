## 前言

今天我们来聊聊一个让很多开发者头疼的话题——MQ消息丢失问题。

有些小伙伴在工作中，一提到消息队列就觉得很简单，但真正遇到线上消息丢失时，排查起来却让人抓狂。

其实，我在实际工作中，也遇到过MQ消息丢失的情况。

今天这篇文章，专门跟大家一起聊聊这个话题，希望对你会有所帮助。

## 一、消息丢失的三大环节

在深入解决方案之前，我们先搞清楚消息在哪几个环节可能丢失：

![](https://files.mdnice.com/user/5303/c8a81514-c7dc-4e59-bb15-755d73f9acd1.png)

### 1. 生产者发送阶段

* 网络抖动导致发送失败
* 生产者宕机未发送
* Broker处理失败未返回确认

### 2. Broker存储阶段

* 内存消息未持久化，重启丢失
* 磁盘故障导致数据丢失
* 集群切换时消息丢失

### 3. 消费者处理阶段

* 自动确认模式下处理异常
* 消费者宕机处理中断
* 手动确认但忘记确认

理解了问题根源，接下来我们看5种实用的解决方案。

## 二、方案一：生产者确认机制

### 核心原理

生产者发送消息后等待Broker确认，确保消息成功到达。

这是防止消息丢失的第一道防线。

![](https://files.mdnice.com/user/5303/3f315e85-f6e8-4e1a-b28a-38115253ac68.png)

### 关键实现

```
// RabbitMQ生产者确认配置
@Bean
public RabbitTemplate rabbitTemplate() {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    template.setConfirmCallback((correlationData, ack, cause) -> {
        if (ack) {
            // 消息成功到达Broker
            messageStatusService.markConfirmed(correlationData.getId());
        } else {
            // 发送失败，触发重试
            retryService.scheduleRetry(correlationData.getId());
        }
    });
    return template;
}

// 可靠发送方法
public void sendReliable(String exchange, String routingKey, Object message) {
    String messageId = generateId();
    // 先落库保存发送状态
    messageStatusService.saveSendingStatus(messageId, message);
    
    // 发送持久化消息
    rabbitTemplate.convertAndSend(exchange, routingKey, message, msg -> {
        msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
        msg.getMessageProperties().setMessageId(messageId);
        return msg;
    }, new CorrelationData(messageId));
}
```

### 适用场景

* 对消息可靠性要求高的业务
* 金融交易、订单处理等关键业务
* 需要精确知道消息发送结果的场景

## 三、方案二：消息持久化机制

### 核心原理

将消息保存到磁盘，确保Broker重启后消息不丢失。

这是防止Broker端消息丢失的关键。

![]()

### 关键实现

```
// 持久化队列配置
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")  // 队列持久化
            .deadLetterExchange("order.dlx")    // 死信交换机
            .build();
}

// 发送持久化消息
public void sendPersistentMessage(Object message) {
    rabbitTemplate.convertAndSend("order.exchange", "order.create", message, msg -> {
        msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT); // 消息持久化
        return msg;
    });
}

// Kafka持久化配置
@Bean
public ProducerFactory producerFactory() {
    Map props = new HashMap<>();
    props.put(ProducerConfig.ACKS_CONFIG, "all"); // 所有副本确认
    props.put(ProducerConfig.RETRIES_CONFIG, 3);   // 重试次数
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // 幂等性
    return new DefaultKafkaProducerFactory<>(props);
}
```

### 优缺点

**优点：**

* 有效防止Broker重启导致的消息丢失
* 配置简单，效果明显

**缺点：**

* 磁盘IO影响性能
* 需要足够的磁盘空间

## 四、方案三：消费者确认机制

### 核心原理

消费者处理完消息后手动向Broker发送确认，Broker收到确认后才删除消息。

这是保证消息不丢失的最后一道防线。

![]()

### 关键实现

```
// 手动确认消费者
@RabbitListener(queues = "order.queue")
public void handleMessage(Order order, Message message, Channel channel) {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    
    try {
        // 业务处理
        orderService.processOrder(order);
        
        // 手动确认
        channel.basicAck(deliveryTag, false);
        log.info("消息处理完成: {}", order.getOrderId());
        
    } catch (Exception e) {
        log.error("消息处理失败: {}", order.getOrderId(), e);
        
        // 处理失败，重新入队
        channel.basicNack(deliveryTag, false, true);
    }
}

// 消费者容器配置
@Bean
public SimpleRabbitListenerContainerFactory containerFactory() {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL); // 手动确认
    factory.setPrefetchCount(10); // 预取数量
    factory.setConcurrentConsumers(3); // 并发消费者
    return factory;
}
```

### 注意事项

* 确保业务处理完成后再确认
* 合理设置预取数量，避免内存溢出
* 处理异常时要正确使用NACK

## 五、方案四：事务消息机制

### 核心原理

通过事务保证本地业务操作和消息发送的原子性，要么都成功，要么都失败。

![]()

### 关键实现

```
// 本地事务表方案
@Transactional
public void createOrder(Order order) {
    // 1. 保存订单到数据库
    orderRepository.save(order);
    
    // 2. 保存消息到本地消息表
    LocalMessage localMessage = new LocalMessage();
    localMessage.setBusinessId(order.getOrderId());
    localMessage.setContent(JSON.toJSONString(order));
    localMessage.setStatus(MessageStatus.PENDING);
    localMessageRepository.save(localMessage);
    
    // 3. 事务提交，本地业务和消息存储保持一致性
}

// 定时任务扫描并发送消息
@Scheduled(fixedDelay = 5000)
public void sendPendingMessages() {
    List pendingMessages = localMessageRepository.findByStatus(MessageStatus.PENDING);
    
    for (LocalMessage message : pendingMessages) {
        try {
            // 发送消息到MQ
            rabbitTemplate.convertAndSend("order.exchange", "order.create", message.getContent());
            
            // 更新消息状态为已发送
            message.setStatus(MessageStatus.SENT);
            localMessageRepository.save(message);
            
        } catch (Exception e) {
            log.error("发送消息失败: {}", message.getId(), e);
        }
    }
}

// RocketMQ事务消息
public void sendTransactionMessage(Order order) {
    TransactionMQProducer producer = new TransactionMQProducer("order_producer");
    
    // 发送事务消息
    Message msg = new Message("order_topic", "create", 
                             JSON.toJSONBytes(order));
    
    TransactionSendResult result = producer.sendMessageInTransaction(msg, null);
    
    if (result.getLocalTransactionState() == LocalTransactionState.COMMIT_MESSAGE) {
        log.info("事务消息提交成功");
    }
}
```

### 适用场景

* 需要严格保证业务和消息一致性的场景
* 分布式事务场景
* 金融、电商等对数据一致性要求高的业务

## 六、方案五：消息重试与死信队列

### 核心原理

通过重试机制处理临时故障，通过死信队列处理最终无法消费的消息。

![]()

### 关键实现

```
// 重试队列配置
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
            .withArgument("x-dead-letter-exchange", "order.dlx") // 死信交换机
            .withArgument("x-dead-letter-routing-key", "order.dead")
            .withArgument("x-message-ttl", 60000) // 60秒后进入死信
            .build();
}

// 死信队列配置
@Bean
public Queue orderDeadLetterQueue() {
    return QueueBuilder.durable("order.dead.queue").build();
}

// 消费者重试逻辑
@RabbitListener(queues = "order.queue")
public void handleMessageWithRetry(Order order, Message message, Channel channel) {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    
    try {
        orderService.processOrder(order);
        channel.basicAck(deliveryTag, false);
        
    } catch (TemporaryException e) {
        // 临时异常，重新入队重试
        channel.basicNack(deliveryTag, false, true);
        
    } catch (PermanentException e) {
        // 永久异常，直接确认进入死信队列
        channel.basicAck(deliveryTag, false);
        log.error("消息进入死信队列: {}", order.getOrderId(), e);
    }
}

// 死信队列消费者
@RabbitListener(queues = "order.dead.queue")
public void handleDeadLetterMessage(Order order) {
    log.warn("处理死信消息: {}", order.getOrderId());
    // 发送告警、记录日志、人工处理等
    alertService.sendAlert("死信消息告警", order.toString());
}
```

### 重试策略建议

1. **指数退避**：1s, 5s, 15s, 30s
2. **最大重试次数**：3-5次
3. **死信处理**：人工介入或特殊处理流程

## 七、方案对比与选型指南

为了帮助大家选择合适的方案，我整理了详细的对比表：

| 方案 | 可靠性 | 性能影响 | 复杂度 | 适用场景 |
| --- | --- | --- | --- | --- |
| 生产者确认 | 高 | 中 | 低 | 所有需要可靠发送的场景 |
| 消息持久化 | 中 | 中 | 低 | Broker重启保护 |
| 消费者确认 | 高 | 低 | 中 | 确保消息被成功处理 |
| 事务消息 | 最高 | 高 | 高 | 强一致性要求的业务 |
| 重试+死信 | 高 | 低 | 中 | 处理临时故障和最终死信 |

### 选型建议

**初创项目/简单业务：**

* 生产者确认 + 消息持久化 + 消费者确认
* 满足大部分场景，实现简单

**电商/交易系统：**

* 生产者确认 + 事务消息 + 重试机制
* 保证数据一致性，处理复杂业务

**大数据/日志处理：**

* 消息持久化 + 消费者确认
* 允许少量丢失，追求吞吐量

**金融/支付系统：**

* 全方案组合使用
* 最高可靠性要求

## 总结

消息丢失问题是消息队列使用中的常见挑战，通过今天介绍的5种方案，我们可以构建一个可靠的消息系统：

1. **生产者确认机制** - 保证消息成功发送到Broker
2. **消息持久化机制** - 防止Broker重启导致消息丢失
3. **消费者确认机制** - 确保消息被成功处理
4. **事务消息机制** - 保证业务和消息的一致性
5. **重试与死信队列** - 处理异常情况和最终死信

有些小伙伴可能会问："我需要全部使用这些方案吗？

"我的建议是：**根据业务需求选择合适的组合**。

对于关键业务，建议至少使用前三种方案；对于普通业务，可以根据实际情况适当简化。

记住，没有完美的方案，只有最适合的方案。

## 最后说一句(求关注，别白嫖我)

如果这篇文章对您有所帮助，或者有所启发的话，帮忙关注一下我的同名公众号：苏三说技术，您的支持是我坚持写作最大的动力。

求一键三连：点赞、转发、在看。

关注公众号：【苏三说技术】，在公众号中回复：进大厂，可以免费获取我最近整理的10万字的面试宝典，好多小伙伴靠这个宝典拿到了多家大厂的offer。

更多精彩内容在我的技术网站：[http://www.susan.net.cn](https://github.com)

本博客参考[slower加速器](https://slowerss.com)。转载请注明出处！
