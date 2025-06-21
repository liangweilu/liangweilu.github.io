---
layout: post
title: RabbitMQ系列（三）---基于Spring boot的rabbitmq实现消息异步处理
date: 2018-8-15
categories:
  - 后端开发
tags:
  - RabbitMQ
---
### 概述
&emsp;&emsp;前面已经对rabbitmq的安装和基本概念进行了讲解，那么在实际的项目实战中，我们如何利用rabbitmq来实现一个消息异步处理的功能呢。本文就将基于Spring Boot 中的rabbitmq来实现一个简单的例子，并对实现过程中遇到的问题进行讲解。如果对spring boot还不了解的话，请自行上网学习，这里重点是对rabbitmq的实现进行讲解。  
### 应用场景
&emsp;&emsp;本文主要演示的应用场景是：客户端调用后端接口，传入一个User对象，这个对象中包含了姓名，年龄，地址，手机号四个属性，我们假设这个User对象需要在服务端进行一个非常耗时的任务。服务端接收到这个User对象之后，通过生产者将其投递到队列queue中，队列异步处理这个User对象（这里演示的是在控制台打印了User中的消息），然后完成。

### 依赖配置
&emsp;&emsp;由于Spring Boot为我们提供了许多方便的stater，所以这里我们直接使用它为我们提供的依赖。本例子是基于Spring Boot 1.5.15.RELEASE版本进行演示的。废话不多说，直接上pom依赖：
```xml
<!-- 继承parent依赖属性 -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.15.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>

	<dependencies>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<!-- rabbitmq -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-amqp</artifactId>
		</dependency>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<scope>test</scope>
		</dependency>

	</dependencies>
```
&emsp;&emsp;配置文件application.yml中的内容如下,关于下面注释掉的rabbitmq地址配置部分，我后面会做讲解，
```yaml
# 当前应用名称
spring:
  application:
    name: spring-rabbit-demo


# 当前应用端口
server:
  port: 8080

# rabbitmq地址
#---
#spring:
#  rabbitmq:
#    addresses: 10.10.9.153
#    port: 5672
#    username: test
#    password: 123456a
```

### 队列与交换机配置
&emsp;&emsp;这部分配置主要是创建一个Direct类型的交换机，声明一个队列Queue，并将该队列绑定在这个交换机上，同时这部分还可以声明rabbitmq的链接信息，消息转换器等。下面对每个部分的声明进行讲解，完整的代码可以到[我的GitHub](https://github.com/byeluliangwei/spring-rabbitmq-demo) `clone`或者下载。  
- 配置连接信息  
&emsp;&emsp;根据第一篇文章rabbitmq的安装，我们可以在本地安装好rabbitmq服务，这里所说的配置连接信息，就是使我们的应用连接到已经安装好的rabbitmq服务上。如果在`application.yml`文件中已经配置了rabbitmq的地址（默认是本地127.0.0.1:5672），那么直接在配置类中注入`ConnectionFactory`即可。如果没有配置，那么就需要在代码中进行如下的配置，连接到rabbitmq服务。
```java
    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory conn = new CachingConnectionFactory("10.10.9.153", 5672);
        conn.setUsername("test");
        conn.setPassword("123456a");
        return conn;
    }
```
- 公共配置  
&emsp;&emsp;上面的配置连接信息其实也属于是公共配置，但是为了展示，我将它分开了。这部分主要配置了Json消息转换器，用于生产者和消费者之间传递对象时可以序列化，配置对@RabbitListener的支持，保证消费者在监听队列的时候能够正确的接收到消息以，还有就是rabbitmq管理员，默认使用连接信息connectionFactory中的用户作为管理员,如果需要设置其他队列的管理员，需要用这个AmqpAdmin进行配置。
```java
@Bean(name = "rabbitListenerContainerFactory")
    public SimpleRabbitListenerContainerFactory listenerContainerFactory(){
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        //json消息转换器,如果不设置会报错： org.springframework.amqp.AmqpException: No method found for class [B
        factory.setMessageConverter(jsonMessageConverter);
        //对消息消费后进行手动确认
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return factory;
    }

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public AmqpAdmin amqpAdmin(){
        return new RabbitAdmin(connectionFactory);
    }
```
- 配置交换机和队列  
&emsp;&emsp;此处我配置了一个Direct类型的交换机，以及两个队列，这两个队列均绑定在此交换机上，具体配置如下：
```java
    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange("test-direct-exchange1", true, false);
    }

    @Bean
    public Queue messageQueue() {
        return new Queue("test-message-queue1", true);
    }
    @Bean
   public Binding bind() {
       return BindingBuilder.bind(messageQueue()).to(directExchange()).with("test-message-queue1");
   }

   @Bean
    public Queue messageQueue2() {
        return new Queue("test-message-queue2", true);
    }

    @Bean
    public Binding bind2() {
        return BindingBuilder.bind(messageQueue2()).to(directExchange()).with("test-message-queue2");
    }
```

&emsp;&emsp;到这里我们的基础配置就已经配置完成了，这个时候只需要在生产者中注入发送消息的模板`AmqpTemplate`然后就可以给往队列中发送消息了，如下所示：
```java
public class MessageSendProducer {

    @Autowired
    private AmqpTemplate amqpTemplate;

    public boolean send(User user) {
        amqpTemplate.convertAndSend("test-direct-exchange1", "test-message-queue1", user);
        amqpTemplate.convertAndSend("test-direct-exchange1", "test-message-queue2", user);
        return true;
    }
}
```
&emsp;&emsp;完成上述配置之后，就可以启动服务，然后通过postman进行一波测试了。这里我通过psotman传递了一个user对象，请求如下图所示
![](/images/rabbitmq-impl/step1.png)  
然后你可以看到如下的效果，表示队列1和队列2均已收到了生产者发送过来的消息，队列1和队列2的消费者都将这个消息打印到了控制台上。  
![](/images/rabbitmq-impl/step2.png)  
但是你也可能看到的结果是这样的，因为队列是异步处理，可能有个队列先处理，或者处理的快了，输出的结果就会如下所示。
![](/images/rabbitmq-impl/step3.png)

### 踩过的坑
&emsp;&emsp;上文中提到了发送消息的时候，需要注入`AmqpTemplate`模板。不注入这个可以不呢？当然可以，因为我们还可以使用`RabbitTemplate`来进行消息发送，我们只需在配置文件中声明一个`RabbitTemplate`的对象，然后在发送消息的时候，直接注入该bean，和`AmqpTemplate`用法一样。声明`RabbitTemplate`如下所示
```java
@Bean
    public RabbitTemplate queueOneTemplate() {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setExchange("test-direct-exchange1");
        template.setRoutingKey("test-message-queue1");
        //json消息转换器,如果不设置会报错： org.springframework.amqp.AmqpException: No method found for class [B
        template.setMessageConverter(jsonMessageConverter);
        return template;
    }
```
这样有什么问题呢，当我们只有一个队列的时候，完全没有问题。但是假如我们有两个队列，比如我这个例子中，如果我们声明两个`RabbitTemplate`对象，这时就会报错了，因为`RabbitTemplate`是单例bean。但是这个时候我们仍然想要为每个队列声明一个`RabbitTemplate`模板来发送消息怎么办呢？大家可以去[我的GitHub](https://github.com/byeluliangwei/spring-rabbitmq-demo)上看看我写的这个例子，例子中我就是为两个队列使用了两个`RabbitTemplate`的解决办法。  
&emsp;&emsp;这里再说一下消费者处理部分的手动确认消息。rabbitmq发送消息之后，默认是自动进行确认的（ack），但是实际生产者，我们在队列中异步处理消息过程中很可能会出现异常，或者消费不成功的情况，如果这个时候使用自动确认，那么rabbitmq就会认为该消息已成功处理， 但实际上是没有的，所以一般我们都会设置手动确认`AcknowledgeMode.MANUAL`。这里只是提醒一下需要注意这个问题，关于具体的原理，详细的需要大家自己去网上学习一下。  

### 总结
&emsp;&emsp;至此基于spring boot搭建一个简单的生产者消费者队列已经完成了，其实还有使用最原始的rabbitmq来实现这个场景，还可以使用rabbitmq的客户端和服务端模式来实现，总的来说原理是一样的，感兴趣的同学可以自行去研究。只是spring boot的starter为我们提供了许多的自动配置，减少了复杂的配置，使得我们可以更加关注业务处理。其实还有一个基于spring cloud stream消息驱动来实现这个功能的，更加的简单，后面会写一个简单的例子。
