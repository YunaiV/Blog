## 注册 Broker

* 协议号：103
* 枚举值：REGISTER_BROKER
* 场景：
    * Broker => Namesrv
* 时间
    * Broker 初始化启动时
    * Broker 每30秒注册次
* 请求
    * header
        * clusterName ： 集群名
        * brokerName ： broker名
        * brokerAddr ：broker地址
        * haServerAddr ： TODO
        * brokerId ： broker角色编号
    * body
        * TopicConfigSerializeWrapper ： Topic配置数组
        * filterServerList ： 过滤服务器数组
* 响应
    * header
        * brokerAddr ：broker地址
        * haServerAddr ： TODO
    * body TODO
* TODO
    * broker集群注册




 



