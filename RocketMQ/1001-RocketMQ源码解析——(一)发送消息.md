```Java
// MQClientInstance.java
private void updateTopicRouteInfoFromNameServer() {
   Set<String> topicList = new HashSet<String>();
   // Consumer 获取topic数组
   {
       Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
       while (it.hasNext()) {
           Entry<String, MQConsumerInner> entry = it.next();
           MQConsumerInner impl = entry.getValue();
           if (impl != null) {
               Set<SubscriptionData> subList = impl.subscriptions();
               if (subList != null) {
                   for (SubscriptionData subData : subList) {
                       topicList.add(subData.getTopic());
                   }
               }
           }
       }
   }
   // Producer 获取topic数组
   {
       Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
       while (it.hasNext()) {
           Entry<String, MQProducerInner> entry = it.next();
           MQProducerInner impl = entry.getValue();
           if (impl != null) {
               Set<String> lst = impl.getPublishTopicList();
               topicList.addAll(lst);
           }
       }
   }
   // 逐个topic更新
   for (String topic : topicList) {
       this.updateTopicRouteInfoFromNameServer(topic);
   }
}
```

