---
title: "spring message container"
date: "2020-11-28"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
# 接口与抽象层（顶层设计）
MessageListenerContainer：所有容器的根接口，定义了生命周期管理（start()/stop()）和设置监听器的基本契约。

AbstractMessageListenerContainer：实现了大部分通用逻辑（如生命周期状态管理、监听器设置），是所有具体容器的父类。它持有：

ConnectionFactory：连接消息代理的工厂

Destination：消息目的地（队列、主题）

MessageListener：实际处理消息的监听器

AbstractPollingMessageListenerContainer：为采用轮询机制的容器（如基于JMS的容器）提供了模板方法receiveAndExecute()，并管理着TaskExecutor（线程池）。

# RabbitMQ实现分支

AbstractRabbitListenerContainer：继承了AbstractMessageListenerContainer，添加了RabbitMQ特有的概念，如通道（Channel）、确认模式（AcknowledgeMode）等。

两个具体实现容器：

SimpleMessageListenerContainer：经典实现。在容器内部维护一个消费者线程池（通过concurrentConsumers和maxConcurrentConsumers配置）。它支持消费者数量的动态伸缩，但所有消费者共享同一个连接（Connection）。

DirectMessageListenerContainer：轻量级实现。为每个消费者使用独立的连接。它不直接管理线程，而是将消息处理任务提交给外部TaskExecutor。通常资源利用率更高，尤其适合短任务。

AbstractRabbitListenerContainerFactory：Spring Boot自动配置中用于按需创建和配置容器的工厂类。当你使用@RabbitListener注解时，就是由它背后负责创建和管理对应的容器实例。

# Kafka实现分支

KafkaMessageListenerContainer：单个监听器容器的具体实现。它内部包装了一个Kafka Consumer客户端，负责实际的拉取消息循环（doStart()方法）。它不直接实现顶层的MessageListenerContainer接口，而是作为ConcurrentMessageListenerContainer的组成部分。

ConcurrentMessageListenerContainer：容器的外包装和并发控制器。它实现了MessageListenerContainer接口，内部包含一个或多个KafkaMessageListenerContainer实例，通过concurrency属性来控制并发度（对应Kafka的分区数）。这是你实际通过@KafkaListener注解与之交互的容器对象。

ContainerProperties：Kafka容器的专属配置集，由ConcurrentMessageListenerContainer持有并传递给内部的KafkaMessageListenerContainer。它包含了如主题（topics）、监听器（messageListener）、提交偏移量的模式（ackMode）等关键配置。

AbstractKafkaListenerContainerFactory：类似于RabbitMQ的工厂，用于创建和配置ConcurrentMessageListenerContainer。

# 关键协作关系

工厂模式：AbstractRabbitListenerContainerFactory和AbstractKafkaListenerContainerFactory是创建容器的入口点。它们根据注解（如@RabbitListener）或编程式配置，组装出完整可用的容器实例。

组合模式：在Kafka分支中，ConcurrentMessageListenerContainer通过组合多个KafkaMessageListenerContainer来实现并发，这是与RabbitMQ分支（通过继承实现并发）的显著设计差异。

策略模式：消息的确认（Ack）、错误处理（ErrorHandler）、重试（Retry）等行为，通常作为可插拔的组件（如AcknowledgingMessageListener、RetryTemplate）注入到容器中，允许灵活替换策略。