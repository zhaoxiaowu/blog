## 什么是队列

队列(`Queue`)是一种常见的数据结构，其最大的特性就是先进先出(`Firist In First Out`)，作为最基础的数据结构，队列应用很广泛，比如我们熟知的`Redis`基础数据类型`List`，其底层数据结构就是队列。

## 什么是消息队列

消息队列(`Messaeg Queue`)是一种使用队列(Queue)作为底层存储数据结构，

可用于解决不同进程与应用之间通讯的分布式消息容器，也称为消息中间件。

目前使用得比较多的消息队列有`ActiveMQ`，`RabbitMQ`，`Kafka`，`RocketMQ`等。

## 消息队列的应用场景 及 优点

**1.异步**      **2.系统解耦**  **3.流量削峰**（设置请求的最大值，超过阈值抛弃 或转到错误页面)  

**4.rabbitmq采用信道通信。不采用tcp直接通信**

     1).tcp的创建和销毁开销大，创建3次握手，销毁4四次分手
    
        2).高峰时成千上万条的链接会造成资源的巨大浪费，而且操作系统没秒处理tcp的数量也是有数量限制的，必定造成性能瓶颈
    
        3).一条线程一条信道，多条线程多条信道，公用一个tcp连接。一条tcp连接可以容纳无限条信道（硬盘容量足够的话），不会造成性能瓶颈。
#### 5.**消息通信**

消息队列最主要功能收发消息，其内部有高效的通讯机制，因此非常适合用于消息通讯。

我们可以基于消息队列开发点对点聊天系统

也可以开发广播系统，用于将消息广播给大量接收者

## 什么是RabbitMQ

`RabbitMQ`是用`Erlang`语言开发的一个实现了`AMQP协议`的消息队列服务器

## 核心概念

我们知道无论是生产者还是消费者，都需要和 RabbitMQ Broker 建立连接，这个连接就是一条 TCP 连接，也就是 Connection。

一旦 TCP 连接建立起来，客户端紧接着可以创建一个 AMQP 信道（Channel），每个信道都会被指派一个唯一的 ID。

信道是建立在 Connection 之上的虚拟连接，RabbitMQ 处理的每条 AMQP 指令都是通过信道完成的。

我们完全可以使用 Connection 就能完成信道的工作，为什么还要引入信道呢？

由于tcp的创建和销毁比较耗费开销，遇到高峰，会出现性能瓶颈。

rabbit 使用信道 达到TCP连接复用

当每个信道的流量不是很大时，复用单一的 Connection 可以在产生性能瓶颈的情况下有效地节省 TCP 连接资源。但是信道本身的流量很大时，这时候多个信道复用一个 Connection 就会产生性能瓶颈，进而使整体的流量被限制了。此时就需要开辟多个 Connection，将这些信道均摊到这些 Connection 中，至于这些相关的调优策略需要根据业务自身的实际情况进行调节。

### Broker

RabitMQ Server

### Producer与Consumer

生产者与消费者相对于`RabbitMQ`服务器来说，都是`RabbitMQ`服务器的客户端。

- 生产者(Producer)：连到`RabbitMQ`服务器，将消息发送到`RabbitMQ`服务器的队列，是消息的发送方。
- 消费者(Consumer)：连接到`RabbitMQ`则是为了消费队列中的消息，是消息的接收方。

生产者与消费者一般由我们的应用程序充当。

### Connection

`Connection`是`RabbitMQ`内部对象之一，用于管理每个到`RabbitMQ`的`TCP`网络连接。

### Channel

`Channel`是我们与`RabbitMQ`打交道的最重要的一个接口，我们大部分的业务操作是在`Channel`这个接口中完成的，包括定义`Queue`、定义`Exchange`、绑定`Queue`与`Exchange`、发布消息等

**Exchange**

消息交换机，作用用于接受生产者的消息，并根据路由键转发消息到所绑定的队列

生产者发送上的消息，就是先通过`Exchnage`按照绑定(`binding`)规则转发到队列的。

交换机类型(`Exchange Type`)有四种：`fanout`、`direct`、`topic`，`headers`，其中`headers`并不常用。

- `fanou`t：这种类型不处理路由键(`RoutingKey`)，很像子网广播，每台子网内的主机都获得了一份复制的消息，**发布/订阅模式**就是指使用`fanout`交换机类型，`fanout`类型交换机转发消息是最快的。
- `direct`：模式处理路由键，需要路由键完全匹配的队列才能收到消息，**路由模式**使用的是`direct`类型的交换机。
- `topic`：将路由键和某模式进行匹配。**主题模式**使用的是`topic`类型的交换机。

### Queue

`Queue`，即队列，`RabbitMQ`内部用于存储消息的对象，是真正用存储消息的结构，在生产端，生产者的消息最终发送到指定队列，而消费者也是通过订阅某个队列，达到获取消息的目的。

### Binding

`Binding`是一种操作，其作用是建立消息从`Exchange`转发到`Queue`的规则，在进行`Exchange`与`Queue`的绑定时，需要指定一个`BindingKey`，Binding操作一般用于`RabbitMQ`的路由工作模式和主题工作模式。

### Virtual Host

`Virutal host`也叫虚拟主机，一个`VirtualHost`下面有一组不同`Exchnage`与`Queue`，不同的`Virtual host`的`Exchnage`与`Queue`之间互相不影响。

应用隔离与权限划分，`Virtual host`是RabbitMQ中最小颗粒的权限单位划分。

如果要类比的话，我们可以把`Virtual host`比作`MySQL`中的数据库，通常我们在使用`MySQL`时，会为不同的项目指定不同的数据库，同样的，在使用`RabbitMQ`时，我们可以为不同的应用程序指定不同的`Virtual host`。
