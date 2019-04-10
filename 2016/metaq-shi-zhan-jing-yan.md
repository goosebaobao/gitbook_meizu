# MetaQ 实战经验

## 使用场景

mq 用于不同的程序之间交换数据，通常我们把这个数据称为消息。由于采用了排队机制，所以适合于实时性要求不高但数据量极大的场景。

典型的场景如下

* 日志收集
* 通知
* 广播，即一点发送多点接收

## 概念

* 生产者
* 消费者
* broker
* 消息
  * 广播消息：集群中每个消费者都能收到消息
  * 非广播消息：集群中任意一个消费者收到消息
* topic
* group

## 实战

### 申请 topic 

请联系运维同学

### 生产者

```markup
<!-- 配置 metaq 的 zk 地址 -->
<bean id="metaClientConfig" class="com.taobao.metamorphosis.client.MetaClientConfig">
    <property name="zkConfig">
        <bean id="zkConfig" class="com.taobao.metamorphosis.utils.ZkUtils.ZKConfig">
            <property name="zkConnect" value="${metaq.zk.server}" />
        </bean>
    </property>
</bean>

<!-- 广播消息会话工厂，既可以创建广播消息会话，也可以创建非广播消息会话 -->
<!-- metaq 建议一个应用只使用一个消息会话工厂，所以使用广播消息会话工厂是最吼滴 -->
<bean id="broadcastMessageSessionFactory"
    class="com.meizu.flymetv.video.metaq.patch.MetaBroadcastMessageSessionFactoryForWindows">
    <constructor-arg index="0">
        <ref bean="metaClientConfig" />
    </constructor-arg>
</bean>

<!-- 消息生产者，注入到业务代码后调用 messageProducer.publish(topic) 后就可以发射消息了 -->
<!-- 多个 topic 可以调用多次 publish 方法，所以一个应用只需要一个生产者 -->
<bean id="messageProducer" factory-bean="broadcastMessageSessionFactory"
    factory-method="createProducer" destroy-method="shutdown" />
```

{% hint style="info" %}
只创建了一个生产者，可用于多个 topic，参考官方文档：[最佳实践](https://github.com/killme2008/Metamorphosis/wiki/最佳实践)
{% endhint %}

### 消费者

```markup
<!-- 消费者，不同的 group 用不同的消费者 -->
<!-- 创建消费者：广播消息 createBroadcastConsumer(), 非广播消息 createConsumer() -->
<!-- 消息者注入业务代码后，需依次调用 subscribe(), completeSubscribe() 方法才能订阅并生效 -->

<!-- jetty group 广播消息 -->
<bean id="jettyMessageConsumer" factory-bean="broadcastMessageSessionFactory"
    factory-method="createBroadcastConsumer" destroy-method="shutdown">
    <constructor-arg>
        <bean id="jettyConsumerConfig"
            class="com.taobao.metamorphosis.client.consumer.ConsumerConfig">
            <property name="group" value="${metaq.group.jetty}" />
        </bean>
    </constructor-arg>
</bean>

<!-- redis group 广播消息 -->
<bean id="redisMessageConsumer" factory-bean="broadcastMessageSessionFactory"
    factory-method="createBroadcastConsumer" destroy-method="shutdown">
    <constructor-arg>
        <bean id="redisConsumerConfig"
            class="com.taobao.metamorphosis.client.consumer.ConsumerConfig">
            <property name="group" value="${metaq.group.redis}" />
        </bean>
    </constructor-arg>
</bean>

<!-- youku group 非广播消息 -->
<bean id="youkuMessageConsumer" factory-bean="broadcastMessageSessionFactory"
    factory-method="createConsumer" destroy-method="shutdown">
    <constructor-arg>
        <bean id="youkuConsumerConfig"
            class="com.taobao.metamorphosis.client.consumer.ConsumerConfig">
            <property name="group" value="${metaq.group.youku}" />
        </bean>
    </constructor-arg>
</bean>
```

{% hint style="info" %}
* 每个 topic 创建了一个消费者，参考官方文档：[最佳实践](https://github.com/killme2008/Metamorphosis/wiki/最佳实践)
* 广播消息和非广播消息的消费者是不同的
  * 广播消息 `broadcastMessageSessionFactory.createBroadcastConsumer()`
  * 非广播消息 `broadcastMessageSessionFactory.createConsumer()`
{% endhint %}

## 关于广播/非广播消息

首先要明白，消息是集中存储在 broker 上的，对消息的消费，并不意味着消息从 broker 上被删除。

消费者实际上是通过一个游标来记录自己已经消费的消息——该游标之前的消息是已消费的，该游标以后的消息是待消费的

那么广播消息和非广播消息的区别就是其维护游标的方式

* 广播消息，消费者把游标保存在本地——每个消费者都有自己的游标
* 非广播消息，消费者把游标保存在 zookeeper 上——所有消费者共享同一个游标

## 参考文档

* [Metaq 官方文档](https://github.com/killme2008/Metamorphosis/wiki)

