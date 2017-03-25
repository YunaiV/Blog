# 03-26

- [ ] Producer发送消息失败，报`org.apache.rocketmq.client.exception.MQClientException: No route info of this topic`
    
    操作步骤：
    
    1. 启动Namesrv：[NamesrvControllerTest#testStart](https://github.com/YunaiV/incubator-rocketmq/blob/1e1fec48631c69bcd5d1daac406fc934c78a4971/namesrv/src/test2/java/org/apache/rocketmq/namesrv/yunai/NamesrvControllerTest.java) 
    2. 启动Broker：[BrokerControllerTest#testBrokerRestart](https://github.com/YunaiV/incubator-rocketmq/blob/1e1fec48631c69bcd5d1daac406fc934c78a4971/namesrv/src/test2/java/org/apache/rocketmq/namesrv/yunai/NamesrvControllerTest.java)
    3. Producer发送消息：[Producer#main](https://github.com/YunaiV/incubator-rocketmq/blob/1e1fec48631c69bcd5d1daac406fc934c78a4971/example/src/main/java/org/apache/rocketmq/example/quickstart/Producer.java) 发送失败，报`org.apache.rocketmq.client.exception.MQClientException: No route info of this topic`

    解决过程：
    1. 使用 [quick-start](http://rocketmq.apache.org/docs/quick-start/) 启动 Namesrv 、 Broker ，使用上述 Producer 进行发送消息，结果成功。关闭 Namesrv ，使用上述 Namesrv 启动，使用上述 Producer 进行发送消息，结果发送成功。猜测上述 Broker 启动失败，原因排查中。

- [ ] 

