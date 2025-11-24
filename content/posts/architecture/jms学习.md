---
title: "jms学习"
date: "2017-04-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

公司使用了hornetq作为消息队列 有个客户环境没有安装hornetq 只能使用activemq   
我们无需修改代码 直接替换了jar包 因为activemq的api和hornetq的api是一致的 它们都是JMS规范的实现  
这也是面向接口编程的一种体现  

JMS规范主要定义了两种消息传递模式，你可以根据业务需求选择或组合使用：  
点对点模式（Queue）：消息生产者将消息发送到特定的队列，只有一个消费者能成功接收并处理该消息。即使有多个消费者同时监听同一个队列，一条消息也只会被其中一个消费。这种模式常用于任务分发。  
发布/订阅模式（Topic）：消息生产者将消息发布到一个主题，所有订阅了该主题的消费者都会收到这条消息的副本。这种模式适用于需要广播消息或事件通知的场景。  

```

// 1. 创建JNDI上下文（实际参数取决于你的JMS提供商）
Properties env = new Properties();
// 这些配置应该外部化到配置文件中
env.put(Context.INITIAL_CONTEXT_FACTORY, 
       "org.apache.activemq.artemis.jndi.ActiveMQInitialContextFactory");
env.put(Context.PROVIDER_URL, "tcp://localhost:61616");
env.put("queue.myQueue", "exampleQueue");

context = new InitialContext(env);

// 2. 查找ConnectionFactory和Destination（标准JMS接口）
ConnectionFactory connectionFactory = 
    (ConnectionFactory) context.lookup("ConnectionFactory");
Destination queue = (Destination) context.lookup("myQueue");

// 3. 创建连接和会话
connection = connectionFactory.createConnection("admin", "password");
Session session = connection.createSession(
    false, Session.AUTO_ACKNOWLEDGE);

// 4. 创建消息生产者
MessageProducer producer = session.createProducer(queue);

// 设置消息持久化（JMS标准属性）
producer.setDeliveryMode(DeliveryMode.PERSISTENT);

// 5. 创建并发送文本消息
TextMessage message = session.createTextMessage();
message.setText("Hello JMS! 当前时间: " + System.currentTimeMillis());
message.setStringProperty("MessageType", "Greeting");

producer.send(message);

// 6. 发送一个MapMessage示例
MapMessage mapMessage = session.createMapMessage();
mapMessage.setString("userName", "张三");
mapMessage.setInt("userAge", 25);
mapMessage.setDouble("salary", 5000.0);

producer.send(mapMessage);

```

```
// 4. 创建消息消费者
    MessageConsumer consumer = session.createConsumer(queue);
// 5. 同步接收消息（设置超时时间）
while (true) {
    Message message = consumer.receive(3000); // 3秒超时
    
    if (message == null) {
        System.out.println("⏰ 接收超时，结束监听");
        break;
    }
    
    // 6. 根据消息类型处理消息
    if (message instanceof TextMessage) {
        TextMessage textMessage = (TextMessage) message;
        System.out.println("📨 收到文本消息: " + textMessage.getText());
        System.out.println("  消息属性: " + 
            textMessage.getStringProperty("MessageType"));
        
    } else if (message instanceof MapMessage) {
        MapMessage mapMessage = (MapMessage) message;
        System.out.println("📊 收到Map消息: ");
        System.out.println("  用户名: " + mapMessage.getString("userName"));
        System.out.println("  年龄: " + mapMessage.getInt("userAge"));
        System.out.println("  薪资: " + mapMessage.getDouble("salary"));
        
    } else {
        System.out.println("ℹ️ 收到其他类型消息: " + message.getClass().getSimpleName());
    }
}
```