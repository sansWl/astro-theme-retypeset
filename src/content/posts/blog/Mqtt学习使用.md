---
 title: Mqtt学习使用
 published: 2025-03-04
 tags:
  - BlogPost
 lang: zh
 abbrlink:  fbf8bdcc-8f34 
---

##1. 依赖导入
```xml
//V3
<dependency>
            <groupId>org.eclipse.paho</groupId>
            <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
            <version>1.2.0</version>
</dependency>
// V5
<dependency>
            <groupId>org.eclipse.paho</groupId>
            <artifactId>org.eclipse.paho.mqttv5.client</artifactId>
            <version>1.2.5</version>
</dependency>
```
##2. 客户端连接
```java
 String broker = "tcp://broker.emqx.io:1883";
    String clientId = "demo_client";
    MqttClient client;

    public MqttServerClient()  {
       try {
           MqttClient client = new MqttClient(broker, clientId);
        //MqttAsyncClient aClient = new MqttAsyncClient(broker, clientId); //异步通信客户端
          MqttConnectOptions options = new MqttConnectOptions();
        // 连接 MQTT Broker 的用户名密码
          options.setUserName("username");
          options.setPassword("password".toCharArray());
        // 是否清除会话
          options.setCleanSession(true);
        // 心跳间隔，单位为秒
          options.setKeepAliveInterval(300);
        // 连接超时时间，单位为秒
          options.setConnectionTimeout(30);
        // 是否自动重连
          options.setAutomaticReconnect(true);

           client.connect(options);

           //this.client = client;
           //init();
       }catch (MqttException e) {
           e.printStackTrace();
       }
    }

```
##3. 客户端回调V3（V5增加了数个额外实现）
```java
//方法在 CommsCallback.class  执行回调
client.setCallback(new MqttCallback() {
          //消息发送被接收到
            public void messageArrived(String topic, MqttMessage message) throws Exception {
                System.out.println("topic: " + topic);
                System.out.println("qos: " + message.getQos());
                System.out.println("message content: " + new String(message.getPayload()));
            }

            public void connectionLost(Throwable cause) {
                System.out.println("connectionLost: " + cause.getMessage());
            }

            public void deliveryComplete(IMqttDeliveryToken token) {
                System.out.println("deliveryComplete: " + token.isComplete());
            }
        });
```
**注意**：  
`1.`订阅的clientId 与 发布的clientId 需要保持不同，否则会发生客户端断连问题；
`2.` 使用的ClientId 尽可能复杂些，避免连接失败（使用官方提供的broker = "tcp://broker.emqx.io:1883";）

##4. 订阅Topic
```java
 public void subscribe(String topic) throws MqttException {
       // qos的数量需要与topic一致，存在方法签名为 public void subscribe(String[] topicFilters, int[] qos) throws MqttException【批量】
        int qos = 2;
       client.subscribe(topic, qos);
//允许插入回调 IMqttMessageListener
   }
//走的是异步客户端处理
	    MqttToken token = new MqttToken(getClientId());
		token.setActionCallback(callback);
		token.setUserContext(userContext);
		token.internalTok.setTopics(topicFilters);

		MqttSubscribe register = new MqttSubscribe(topicFilters, qos);

		comms.sendNoWait(register, token);
```
> `+`：单层通配，必须占据一个层级，例如 test/+/aa <=\=\=> {test/1/aa,test/abc/aa}、+、test/+
  `#`：多层通配，必须占用一个层级，且是最后一个字符，例如 test/#  <=\=\=> {test/aa，test/aa/bb... } 、test/demo/#、#
  `$share/{Share Name}/{Topic Filter}`: 共享订阅，{shareName}定义的共享组名，{topicFilter}主题名与publish的一致，例如 $share/test1/demo(sub)、 demo(pub)  
  `$queue/{Topic File}`: MQTT3.1.1 
**负载均衡** EMQX
```vim
# etc/emqx.conf

# 均衡策略
broker.shared_subscription_strategy = random

# 当设备离线，或者消息等级为 QoS1、QoS2，因各种各样原因设备没有回复 ACK 确认，消息会被重新派发至群组内其他的设备。
broker.shared_dispatch_ack_enabled = false
```
##5. 发布msg
```java
 public void publish(String topic, String msg) throws MqttException {
       int qos = 1;
       MqttMessage message = new MqttMessage(msg.getBytes());
       message.setQos(qos);
       client.publish(topic, message);
//允许插入回调 IMqttActionListener
   }
//同样走的是异步客户端处理
		MqttDeliveryToken token = new MqttDeliveryToken(getClientId());
		token.setActionCallback(callback);
		token.setUserContext(userContext);
		token.setMessage(message);
		token.internalTok.setTopics(new String[] { topic });

		MqttPublish pubMsg = new MqttPublish(topic, message);
		comms.sendNoWait(pubMsg, token);
```
> `Retain(保留消息)` 生产者publish的消息会保存一份最新消息，当订阅此topic后会拿取到这份消息，且消息的retain属性为true，注意必须是后订阅读取到的消息才是保留消息
`Will(遗嘱消息)`  在连接到指定Broker前指定，options.setWill()，会创建一个独立的Topic