---
title: "eclipse paho mqtt使用记录"
date: "2023-05-25"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

eclipse paho mqtt
公司用了这个mqtt服务器去连接设备去做双向通讯
服务端订阅了设备的topic得到上报的报文
记录一下这个库

```
public boolean connect() {
    MemoryPersistence persistence = new MemoryPersistence();
    try {
        client = new MqttClient(MQTT_HOST, NotificationConstant.CLIENT_ID, persistence);  // 如果不加第三个参数，在MCOS上，TOMCAT环境里会丢异常
    } catch (MqttException e) {
        log.error(ExceptionUtils.getStackTrace(e));
        return false;
    }

    MqttConnectOptions options = new MqttConnectOptions();
    options.setCleanSession(true);        //断线后清除session
    options.setAutomaticReconnect(true);    //自动重连
    options.setConnectionTimeout(NotificationConstant.CONNECTION_TIMEOUT);
    options.setKeepAliveInterval(NotificationConstant.KEEP_ALIVE_INTERVAL);
    options.setUserName(NotificationConstant.MQTT_USER_NAME);
    options.setPassword(NotificationConstant.MQTT_PASSWORD.toCharArray());
    client.setCallback(new MqttPushCallback());
    try {
        client.connect(options);
    } catch (MqttException e) {
        log.error(ExceptionUtils.getStackTrace(e));
        return false;
    }
    subscribeTopics();
    return true;
}
public boolean subscribeTopics() {
    try{
        client.subscribe(NotificationConstant.DEVICE_STATUS_TOPIC, NotificationConstant.MQTT_QUALITY);
    }catch (Exception e){
        log.error(ExceptionUtils.getStackTrace(e));
        return false;
    }
    return true;
}
public boolean pushMsg(PushMsgDTO pendingMsg) {
    MqttMessage message = new MqttMessage();
    message.setQos(mqttQuality);
    message.setPayload(pendingMsg.getMessage().getBytes());
    try {
        client.publish(pendingMsg.getTopic(), message);
    } catch(MqttException e) {
        log.error(ExceptionUtils.getStackTrace(e));
        return false;
    }
    log.info("publish mqtt successfully, topic: " + pendingMsg.getTopic());

    return true;
}
```