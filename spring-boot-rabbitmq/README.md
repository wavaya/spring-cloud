# 一、RabbitMQ介绍
RabbitMQ 即一个消息队列，主要是用来实现应用程序的异步和解耦，同时也能起到消息缓冲，消息分发的作用。

消息中间件在互联网公司的使用中越来越多，刚才还看到新闻阿里将RocketMQ捐献给了apache，当然了今天的主角还是讲RabbitMQ。消息中间件最主要的作用是解耦，中间件最标准的用法是生产者生产消息传送到队列，消费者从队列中拿取消息并处理，生产者不用关心是谁来消费，消费者不用关心谁在生产消息，从而达到解耦的目的。在分布式的系统中，消息队列也会被用在很多其它的方面，比如：分布式事务的支持，RPC的调用等等。

以前一直使用的是ActiveMQ，在实际的生产使用中也出现了一些小问题，在网络查阅了很多的资料后，决定尝试使用RabbitMQ来替换ActiveMQ，RabbitMQ的高可用性、高性能、灵活性等一些特点吸引了我们，查阅了一些资料整理出此文。

## 1、介绍
RabbitMQ是实现AMQP（高级消息队列协议）的消息中间件的一种，最初起源于金融系统，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。RabbitMQ主要是为了实现系统之间的双向解耦而实现的。当生产者大量产生数据时，消费者无法快速消费，那么需要一个中间层。保存这个数据。

AMQP，即Advanced Message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。

RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写，支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

## 2、RabbitMQ VS Kafka MQ
Kafka MQ是一个高吞吐量分布式消息系统。是由linkedin开源的消息中间件。 Kafka就跟这个名字一样，设计非常独特。
> 在应用场景方面

- RabbitMQ,遵循AMQP协议，由内在高并发的erlanng语言开发，用在实时的对可靠性要求比较高的消息传递上。(AMQP，即Advanced Message Queuing Protocol,)
- kafka是Linkedin于2010年12月份开源的消息发布订阅系统,它主要用于处理活跃的流式数据,大数据量的数据处理上。

> 在架构模型方面

- RabbitMQ遵循AMQP协议，RabbitMQ的broker由Exchange,Binding,queue组成，其中exchange和binding组成了消息的路由键；客户端Producer通过连接channel和server进行通信，Consumer从queue获取消息进行消费（长连接，queue有消息会推送到consumer端，consumer循环从输入流读取数据）。rabbitMQ以broker为中心；有消息的确认机制。
- kafka遵从一般的MQ结构，producer，broker，consumer，以consumer为中心，消息的消费信息保存的客户端consumer上，consumer根据消费的点，从broker上批量pull数据；无消息确认机制。

> 在吞吐量

- kafka具有高的吞吐量，内部采用消息的批量处理，zero-copy机制，数据的存储和获取是本地磁盘顺序批量操作，具有O(1)的复杂度，消息处理的效率很高。
- rabbitMQ在吞吐量方面稍逊于kafka，他们的出发点不一样，rabbitMQ支持对消息的可靠的传递，支持事务，不支持批量的操作；基于存储的可靠性的要求存储可以采用内存或者硬盘。

> 在可用性方面

- rabbitMQ支持miror的queue，主queue失效，miror queue接管。
- kafka的broker支持主备模式。
 
> 在集群负载均衡方面

- kafka采用zookeeper(ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务)对集群中的broker、consumer进行管理，可以注册topic到zookeeper上；通过zookeeper的协调机制，producer保存对应topic的broker信息，可以随机或者轮询发送到broker上；并且producer可以基于语义指定分片，消息发送到broker的某分片上。
- rabbitMQ的负载均衡需要单独的loadbalancer进行支持。

## 3、相关概念
通常我们谈到队列服务, 会有三个概念： 发消息者、队列、收消息者，RabbitMQ 在这个基本概念之上, 多做了一层抽象, 在发消息者和 队列之间, 加入了交换器 (Exchange). 这样发消息者和队列就没有直接联系, 转而变成发消息者把消息给交换器, 交换器根据调度策略再把消息再给队列。

![RabbitMQ](http://www.ityouknow.com/assets/images/2016/RabbitMQ01.png)

- 左侧 P 代表 生产者，也就是往 RabbitMQ 发消息的程序。
- 中间即是 RabbitMQ，其中包括了 交换机 和 队列。
- 右侧 C 代表 消费者，也就是往 RabbitMQ 拿消息的程序。
那么，其中比较重要的概念有 4 个，分别为：虚拟主机，交换机，队列，和绑定。

- 虚拟主机：一个虚拟主机持有一组交换机、队列和绑定。为什么需要多个虚拟主机呢？很简单，RabbitMQ当中，用户只能在虚拟主机的粒度进行权限控制。  因此，如果需要禁止A组访问B组的交换机/队列/绑定，必须为A和B分别创建一个虚拟主机。每一个RabbitMQ服务器都有一个默认的虚拟主机“/”。
- 交换机：Exchange 用于转发消息，但是它不会做存储 ，如果没有 Queue bind 到 Exchange 的话，它会直接丢弃掉 Producer 发送过来的消息。 这里有一个比较重要的概念：路由键 。消息到交换机的时候，交互机会转发到对应的队列中，那么究竟转发到哪个队列，就要根据该路由键。
- 绑定：也就是交换机需要和队列相绑定，这其中如上图所示，是多对多的关系。

### 交换机(Exchange)
交换机的功能主要是接收消息并且转发到绑定的队列，交换机不存储消息，在启用ack模式后，交换机找不到队列会返回错误。交换机有四种类型：Direct, topic, Headers and Fanout

- Direct：direct 类型的行为是”先匹配, 再投送”. 即在绑定时设定一个 routing_key, 消息的routing_key 匹配时, 才会被交换器投送到绑定的队列中去.
- Topic：按规则转发消息（最灵活）
- Headers：设置header attribute参数类型的交换机
- Fanout：转发消息到所有绑定队列

#### Direct Exchange
Direct Exchange是RabbitMQ默认的交换机模式，也是最简单的模式，根据key全文匹配去寻找队列。
![Direct Exchange](http://www.ityouknow.com/assets/images/2016/rabbitMq_direct.png)

第一个 X - Q1 就有一个 binding key，名字为 orange； X - Q2 就有 2 个 binding key，名字为 black 和 green。当消息中的 路由键 和 这个 binding key 对应上的时候，那么就知道了该消息去到哪一个队列中。

Ps：为什么 X 到 Q2 要有 black，green，2个 binding key呢，一个不就行了吗？ - 这个主要是因为可能又有 Q3，而Q3只接受 black 的信息，而Q2不仅接受black 的信息，还接受 green 的信息。

#### Topic Exchange

Topic Exchange 转发消息主要是根据通配符。 在这种交换机下，队列和交换机的绑定会定义一种路由模式，那么，通配符就要在这种路由模式和路由键之间匹配后交换机才能转发消息。

在这种交换机模式下：

- 路由键必须是一串字符，用句号（.） 隔开，比如说 agreements.us，或者 agreements.eu.stockholm 等。
- 路由模式必须包含一个 星号（*），主要用于匹配路由键指定位置的一个单词，比如说，一个路由模式是这样子：agreements..b.*，那么就只能匹配路由键是这样子的：第一个单词是 agreements，第四个单词是 b。 井号（#）就表示相当于一个或者多个单词，例如一个匹配模式是agreements.eu.berlin.#，那么，以agreements.eu.berlin开头的路由键都是可以的。
具体代码发送的时候还是一样，第一个参数表示交换机，第二个参数表示routing key，第三个参数即消息。如下：
```
rabbitTemplate.convertAndSend("testTopicExchange","key1.a.c.key2", " this is  RabbitMQ!");
```
topic 和 direct 类似, 只是匹配上支持了”模式”, 在”点分”的 routing_key 形式中, 可以使用两个通配符:

- *表示一个词.
- #表示零个或多个词.

#### Headers Exchange

headers 也是根据规则匹配, 相较于 direct 和 topic 固定地使用 routing_key , headers 则是一个自定义匹配规则的类型. 在队列与交换器绑定时, 会设定一组键值对规则, 消息中也包括一组键值对( headers 属性), 当这些键值对有一对, 或全部匹配时, 消息被投送到对应队列.

#### Fanout Exchange

Fanout Exchange 消息广播的模式，不管路由键或者是路由模式，会把消息发给绑定给它的全部队列，如果配置了routing_key会被忽略。

# 二、安装RabbitMQ

## 1、Mac安装RabbitMQ
> brew install rabbitmq

## 2、Mac安装RabbitMQ

看到如下的代码表示RabbitMQ安装成功

![install.png](http://upload-images.jianshu.io/upload_images/688387-d65d6c6da974fb48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>注意： rabbitmq的安装目录： /usr/local/Cellar/rabbitmq/3.7.3

## 3.RabbitMQ 的启动
进入到 /usr/local/Cellar/rabbitmq/3.7.3，执行
```
cd /usr/local/Cellar/rabbitmq/3.7.3 #切换目录
sbin/rabbitmq-server #启动rabbitmq
```

## 4.RabbitMQ 启动插件
待RabbitMQ 的启动完毕之后，另起终端进入cd /usr/local/Cellar/rabbitmq/3.7.3/sbin
 。启动插件：
```
cd /usr/local/Cellar/rabbitmq/3.7.3/sbin
rabbitmq-plugins enable rabbitmq_management
```
 
rabbitmq_management（执行一次以后不用再次执行）

## 5.登陆管理界面
打开浏览器并访问：http://localhost:15672/，并使用默认用户guest登录，密码也为guest。我们可以看到如下图的管理页面：

![login.png](http://upload-images.jianshu.io/upload_images/688387-acbbf48985dbf564.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从图中，我们可以看到之前章节中提到的一些基本概念，比如：Connections、Channels、Exchanges、Queue等。第一次使用的读者，可以都点开看看都有些什么内容，熟悉一下RabbitMQ Server的服务端。

点击Admin标签，在这里可以进行用户的管理。

# 三、springboot集成RabbitMQ
springboot集成RabbitMQ非常简单，如果只是简单的使用配置非常少，springboot提供了spring-boot-starter-amqp项目对消息各种支持。


## （一）简单实用

### 1、配置pom包，主要是添加spring-boot-starter-amqp的支持

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 2、配置文件
配置rabbitmq的安装地址、端口以及账户信息

```
spring.application.name=spirng-boot-rabbitmq

spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

### 3、队列配置
```
@Configuration
public class RabbitConfig {

    @Bean
    public Queue Queue() {
        return new Queue("hello");
    }

}
```

### 4、发送者
rabbitTemplate是springboot 提供的默认实现

```
public class HelloSender {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send() {
        String context = "hello " + new Date();
        System.out.println("Sender : " + context);
        this.rabbitTemplate.convertAndSend("hello", context);
    }

}
```

### 5、接受者
```
@Component
@RabbitListener(queues = "hello")
public class HelloReceiver {

    @RabbitHandler
    public void process(String hello) {
        System.out.println("Receiver  : " + hello);
    }

}
```

### 6、测试
> 注意，发送者和接收者的queue name必须一致，不然不能接收


```
@RunWith(SpringRunner.class)
@SpringBootTest
public class RabbitMqHelloTest {

    @Autowired
    private HelloSender helloSender;

    @Test
    public void hello() throws Exception {
        helloSender.send();
    }

}
```
> 结果如下

```
2018-03-06 12:27:42.417  INFO 72847 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 7.472 seconds (JVM running for 10.549)
Receiver  : hello Tue Mar 06 12:28:02 CST 2018
Receiver  : hello Tue Mar 06 12:29:29 CST 2018

```

## （二）多对多使用
一个发送者，N个接收者或者N个发送者和N个接收者会出现什么情况呢？

一对多发送     
对上面的代码进行了小改造,接收端注册了两个Receiver,Receiver1和Receiver2，发送端加入参数计数，接收端打印接收到的参数，下面是测试代码，发送一百条消息，来观察两个接收端的执行效果
```
@Test
public void oneToMany() throws Exception {
    for (int i=0;i<100;i++){
        neoSender.send(i);
    }
}
```

> 结果如下

```
Sender1 : spirng boot sagewang queue ****** 0
Sender1 : spirng boot sagewang queue ****** 1
Sender1 : spirng boot sagewang queue ****** 2
Sender1 : spirng boot sagewang queue ****** 3
Sender1 : spirng boot sagewang queue ****** 4
Sender1 : spirng boot sagewang queue ****** 5
Sender1 : spirng boot sagewang queue ****** 6
Sender1 : spirng boot sagewang queue ****** 7
Sender1 : spirng boot sagewang queue ****** 8
Sender1 : spirng boot sagewang queue ****** 9
Sender1 : spirng boot sagewang queue ****** 10
```

根据返回结果得到以下结论

> 一个发送者，N个接受者,经过测试会均匀的将消息发送到N个接收者中

多对多发送

复制了一份发送者，加入标记，在一百个循环中相互交替发送
```
@Test
    public void manyToMany() throws Exception {
        for (int i=0;i<100;i++){
            neoSender.send(i);
            neoSender2.send(i);
        }
}
```

> 结果如下

```
Receiver 1: spirng boot sagewang queue ****** 23
Receiver 1: spirng boot sagewang queue ****** 25
Receiver 1: spirng boot sagewang queue ****** 27
Receiver 1: spirng boot sagewang queue ****** 29
Receiver 1: spirng boot sagewang queue ****** 31
Receiver 1: spirng boot sagewang queue ****** 33
Receiver 2: spirng boot sagewang queue ****** 23
Receiver 1: spirng boot sagewang queue ****** 35
Receiver 2: spirng boot sagewang queue ****** 25
Receiver 1: spirng boot sagewang queue ****** 37
Receiver 2: spirng boot sagewang queue ****** 27
Receiver 1: spirng boot sagewang queue ****** 39
Receiver 1: spirng boot sagewang queue ****** 41
Receiver 1: spirng boot sagewang queue ****** 43
Receiver 2: spirng boot sagewang queue ****** 29
Receiver 1: spirng boot sagewang queue ****** 45
Receiver 1: spirng boot sagewang queue ****** 47
Receiver 1: spirng boot sagewang queue ****** 49
```

> 结论：和一对多一样，接收端仍然会均匀接收到消息


## （三）高级使用
对象的支持

springboot以及完美的支持对象的发送和接收，不需要格外的配置。

```
//发送者
public void send(User user) {
    System.out.println("Sender object: " + user.toString());
    this.rabbitTemplate.convertAndSend("object", user);
}

...

//接收者
@RabbitHandler
public void process(User user) {
    System.out.println("Receiver object : " + user);
}
```

> 结果如下

```
Sender object: User{name='neo', pass='123456'}
Receiver object : User{name='neo', pass='123456'}
```

## （四）Topic Exchange

topic 是RabbitMQ中最灵活的一种方式，可以根据routing_key自由的绑定不同的队列

首先对topic规则配置，这里使用两个队列来测试
```
@Configuration
public class TopicRabbitConfig {

    final static String message = "topic.message";
    final static String messages = "topic.messages";

    @Bean
    public Queue queueMessage() {
        return new Queue(TopicRabbitConfig.message);
    }

    @Bean
    public Queue queueMessages() {
        return new Queue(TopicRabbitConfig.messages);
    }

    @Bean
    TopicExchange exchange() {
        return new TopicExchange("exchange");
    }

    @Bean
    Binding bindingExchangeMessage(Queue queueMessage, TopicExchange exchange) {
        return BindingBuilder.bind(queueMessage).to(exchange).with("topic.message");
    }

    @Bean
    Binding bindingExchangeMessages(Queue queueMessages, TopicExchange exchange) {
        return BindingBuilder.bind(queueMessages).to(exchange).with("topic.#");
    }
}
```

使用queueMessages同时匹配两个队列，queueMessage只匹配”topic.message”队列

```
public void send1() {
    String context = "hi, i am message 1";
    System.out.println("Sender : " + context);
    this.rabbitTemplate.convertAndSend("exchange", "topic.message", context);
}

public void send2() {
    String context = "hi, i am messages 2";
    System.out.println("Sender : " + context);
    this.rabbitTemplate.convertAndSend("exchange", "topic.messages", context);
}
```
> 结果如下

```
Topic Receiver1  : hi, i am message 1
Topic Receiver2  : hi, i am message 1
Topic Receiver2  : hi, i am message all
```

发送send1会匹配到topic.#和topic.message 两个Receiver都可以收到消息，发送send2只有topic.#可以匹配所有只有Receiver2监听到消息

## （五）Fanout Exchange

Fanout 就是我们熟悉的广播模式或者订阅模式，给Fanout交换机发送消息，绑定了这个交换机的所有队列都收到这个消息。

Fanout 相关配置
```
@Configuration
public class FanoutRabbitConfig {

    @Bean
    public Queue AMessage() {
        return new Queue("fanout.A");
    }

    @Bean
    public Queue BMessage() {
        return new Queue("fanout.B");
    }

    @Bean
    public Queue CMessage() {
        return new Queue("fanout.C");
    }

    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange("fanoutExchange");
    }

    @Bean
    Binding bindingExchangeA(Queue AMessage,FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(AMessage).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeB(Queue BMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(BMessage).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeC(Queue CMessage, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(CMessage).to(fanoutExchange);
    }

}
```

这里使用了A、B、C三个队列绑定到Fanout交换机上面，发送端的routing_key写任何字符都会被忽略：

```
public void send() {
        String context = "hi, fanout msg ";
        System.out.println("Sender : " + context);
        this.rabbitTemplate.convertAndSend("fanoutExchange","", context);
}
```

> 结果如下

```
Topic Receiver1  : hi, i am message 1
Topic Receiver2  : hi, i am message 1
Topic Receiver2  : hi, i am message all
```

> 结果如下

```

fanout Receiver A  : hi, fanout msg 
fanout Receiver B: hi, fanout msg 
fanout Receiver C: hi, fanout msg 

```

结果说明，绑定到fanout交换机上面的队列都收到了消息

[示例代码-github](https://github.com/wsqat/spring-cloud/tree/master/spring-boot-rabbitmq)