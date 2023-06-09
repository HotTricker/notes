# 《RabbitMQ实战指南》笔记

- [《RabbitMQ实战指南》笔记](#rabbitmq实战指南笔记)
  - [消息中间件](#消息中间件)
  - [基础](#基础)
    - [交换器](#交换器)
    - [机制](#机制)
    - [生产和消费](#生产和消费)
  - [进阶](#进阶)
    - [消息何去何从](#消息何去何从)
    - [过期](#过期)
    - [死信队列、延迟队列、优先级队列](#死信队列延迟队列优先级队列)
    - [持久化](#持久化)
    - [生产者确认](#生产者确认)
    - [消息分发](#消息分发)
    - [消息传输保障](#消息传输保障)
  - [RabbitMQ管理](#rabbitmq管理)
  - [RabbitMQ配置](#rabbitmq配置)
  - [RabbitMQ运维](#rabbitmq运维)
  - [高阶](#高阶)
  - [网络分区](#网络分区)
  - [扩展](#扩展)

## 消息中间件

消息队列中间件（Message Queue Middleware，简称为MQ）是指利用高效可靠的消息传递机制进行与平台无关的数据交流，并基于数据通信来进行分布式系统的集成。

它一般有两种传递模式：点对点（P2P，Point-to-Point）模式和发布/订阅（Pub/Sub）模式。

主题可以认为是消息传递的中介，消息发布者将消息发布到某个主题，而消息订阅者则从主题中订阅消息

发送者将消息发送给消息服务器，消息服务器将消息存放在若干队列中，在合适的时候再将消息转发给接收者。

在消息路由的过程中，消息的标签会丢弃，存入到队列中的消息只有消息体

## 基础

### 交换器

生产者将消息发给交换器的时候，一般会指定一个RoutingKey，用来指定这个消息的路由规则，而这个Routing Key需要与交换器类型和绑定键（BindingKey）联合使用才能最终生效。

BindingKey并不是在所有的情况下都生效，它依赖于交换器类型

但是在topic交换器类型下，RoutingKey和BindingKey之间需要做模糊匹配，两者并不是相同的。

RabbitMQ常用的交换器类型有fanout、direct、topic、headers这四种。

BindingKey和RoutingKey一样也是点号“.”分隔的字符串；BindingKey中可以存在两种特殊字符串“*”和“＃”，用于做模糊匹配

headers类型的交换器不依赖于路由键的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配

headers类型的交换器性能会很差，而且也不实用，基本上不会看到它的存在

### 机制

和RabbitMQ Broker建立连接，这个连接就是一条TCP连接，也就是Connection。一旦TCP连接建立起来，客户端紧接着可以创建一个AMQP信道（Channel），每个信道都会被指派一个唯一的ID。信道是建立在Connection之上的虚拟连接，RabbitMQ处理的每条AMQP指令都是通过信道完成的。

选择TCP连接复用，不仅可以减少性能开销，同时也便于管理。

AMQP协议包括三层：Module Layer定义客户端调用的命令，Session Layer负责将客户端命令发给服务器以及服务端应答返回客户端，Transport Layer位于底层，传输二进制数据流。

RabbitMQ的消息存储在队列中，交换器并不真正耗费服务器的性能，而队列会。如果要衡量RabbitMQ当前的QPS（单位时间处理流量多少）只需看队列的即可。

按照RabbitMQ官方建议，生产者和消费者都应该尝试创建(这里指声明操作)队列，但预先创建好资源可以确保交换器和队列之间正确的绑定匹配。

### 生产和消费

RabbitMQ的消费模式分为推和拉两种。推模式通过持续订阅的方式来消费消息，拉模式单条地获取消息。

autoAck为false时，收到消息之后需要进行显式ack操作，可以防止消息不必要地丢失。

RabbitMQ会等待消费者显式地回复ack后才从内存(或者磁盘)中移去消息(实质上是先打上删除标记，之后再删除)。当autoAck等于true时，RabbitMQ会自动把发送出去的消息置为确认，然后从内存(或者磁盘)中删除，而不管消费者是否真正地消费到了这些消息。

RabbitMQ不会为未确认的消息设置过期时间，它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否己经断开，这么设计的原因是RabbitMQ允许消费者消费一条消息的时间可以很久很久。

可以让RabbitMQ重新发送还未被确认的消息，可以指定是发给之前的消费者还是可能分配给不同的消费者。

## 进阶

### 消息何去何从

交换器无法找到符合条件的队列时，允许返回消息给生产者，也可以直接丢弃。可以使用mandatory参数，也可以使用备份交换器，使用备份交换器会导致mandatory失效。如果备份交换器不存在/没有绑定队列/没有匹配的队列，不会有异常，消息会丢失。

### 过期

给消息设置过期时，可以通过队列属性或者对消息本身单独设置，如果同时设置以较小的为准。过期后消息变为“死信”，不设置TTL表示不会过期。

可以给队列本身设置超时，控制队列被自动删除前处于未使用状态（队列上没有消费者，队列没有被重新声明，过期时间段内未调用GET）的时间。RabbitMQ会确保在过期时间到达后将队列删除，但是不保障删除的动作有多及时。在RabbitMQ重启后，持久化的队列的过期时间会被重新计算。

### 死信队列、延迟队列、优先级队列

消息变为死信后能被发送到DLX死信交换器，绑定DLX的队列被称为死信队列。消息被拒绝且requeue为false/消息过期/队列达到最大长度都会导致消息变成死信。DLX能在任何队列上被指定，当队列中存在死信时自动发布到DLX上，进而被路由到死信队列。

延迟队列存储的对象是对应的延迟消息，所谓"延迟消息"是指当消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费，比如订单的过期等。可以通过DLX和TTL模拟出延迟队列。

可以设置队列的最大优先级和消息的优先级，优先级高的消息优先消费。

### 持久化

持久化可以提高RabbitMQ的可靠性，以防在异常情况(重启、关闭、右机等)下的数据丢失，这包括交换器、队列和消息的持久化。不做交换器的持久化不会导致重启后消息丢失，只会影响往交换器发送消息。不做队列和消息的持久化会导致重启后消息丢失。持久化会影响性能，需要在持久化和吞吐量之间做一个权衡。

持久化并不能保证消息不会丢失，可以使用镜像队列机制来提高可靠性。

### 生产者确认

通过事务机制或者发送方确认可以确保消息正确的到达服务器。可以将chan设置成事务模式，如果事务提交成功则一定到达了服务器，如果捕获到异常可以进行事务回滚，但使用事务发送后是阻塞的，会降低性能。
如果将chan设置成确认模式，所有的消息会指派一个唯一ID，投递到所有匹配的队列后RabbitMQ会向生产者发送一个确认，如果是可持久化的会在持久化完成后确认。可以不同步确认，而是采用批量确认或者异步确认，采用批量确认可以提高确认效率，但失败时需要全部重发，经常丢消息时性能不升反降；异步确认主要是编程复杂。

### 消息分发

当RabbitMQ队列拥有多个消费者时，队列收到的消息将以轮询的分发方式发送给消费者。每条消息只会发送给订阅列表里的一个消费者。这种方式非常适合扩展，而且它是专门为并发程序设计的。

默认情况下，如果有n个消费者，第m条消息会发给m%n（应该是限定推模式），如果消费者存在能力差异，可能会造成堆积。可以限制消费者能保持的最大未确认数量。

rabbitmq的消息顺序性有可能被打破，比如事务补偿、延迟队列、优先级、重发等等都会改变消息的顺序。

### 消息传输保障

一般分为三个层级，最多一次，消息可能会丢失但不会重复传输；最少一次，消息不会丢失但可能重复传输；恰好一次，肯定会且只会传输一次。RabbiMQ支持“最多一次”和“最少一次”。

通过事务/发送者确认可以保证消息传输到RabbitMQ，通过mandatory/备份交换器保证消息能够路由到队列，通过持久化保证异常情况时消息不会丢失，通过手动ack确认消息一定会正确消费，可以保证“最少一次”。“最多一次”则无需额外操作。一般通过业务的角度来处理重复实现恰好一次。

## RabbitMQ管理

每一个RabbitMQ服务器都能创建虚拟的消息服务器，我们称之为虚拟主机(virtual host)，简称vhost。vhost本质上是一个小型的RabbitMQ服务器，就像虚拟机一样在各个实例间提供逻辑上的隔离，既可以区分用户，又可以避免队列和交换器等命名冲突。RabbitMQ默认创建的vhost为“/”。

RabbitMQ的权限控制以vhost为单位，用户会被指派给至少一个vhost，并且只能访问被指派vhost内的队列、交换器、绑定关系等。用户角色上，none < management < policymaker < monitoring < administrator。RabbitMQ management 插件提供了Web管理界面和HTTP API接口来方便调用。

通常情况下，当关闭整个RabbitMQ集群时，重启的第一个节点应该是最后关闭的节点，因为它可以看到其他节点所看不到的事情。

## RabbitMQ配置

RabbitMQ提供了三种方式定制化服务：环境变量、配置文件、运行时参数和策略。

环境变量可以在Shell或者rabbitmq-env.conf中进行设置。优先级shell > 配置文件 > 默认配置。

默认配置文件的位置取决于不同的操作系统和安装包，最有效的方法是检查RabbitMQ的服务日志，在启动RabbitMQ服务的时候会打印相关信息。

配置文件中有一些敏感的配置项可以进行加密。通过调节缓冲区的大小可以调节吞吐量。rabbitmq.config需要重启broker才能生效。

## RabbitMQ运维

RabbitMQ集群中的所有节点都会备份所有的元数据信息，包括队列元数据、交换器、绑定关系元数据、vhost元数据，但不会备份消息，除了特殊配置比如镜像队列。

RabbitMQ集群对延迟非常敏感，应当只在本地局域网内使用。在广域网中不应该使用集群，而应该使用Federation或者Shovel来代替。

如果关闭了集群中的所有节点，则需要确保在启动的时候最后关闭的那个节点是最先启动的。如最后关闭的节点无法启动，通过命令将节点剔出集群。

节点类型分为内存节点和磁盘节点，在集群中可以配置部分节点为内存节点以获得更高的性能。

RabbitMQ要求在集群中至少有一个磁盘节点，所有其他节点可以是内存节点。当节点加入或者离开集群时，它们必须将变更通知到至少一个磁盘节点。如果只有一个磁盘节点，而且它刚好崩溃了，那么集群可以继续发送或者接收消息，但是不能执行创建队列、交换器、绑定关系、用户，以及更改权限、添加或删除集群节点的操作。在建立集群的时候应该保证有两个或者多个磁盘节点的存在。在内存节点重启后，它们会连接到预先配置的磁盘节点，下载当前集群元数据的副本。

RabbitMQ的日志默认存放在/var/log/rabbitmq文件夹，包括nodename-sasl.log和nodename.log，分别记录erlang和rabbitmq服务的日志。amq.rabbitmq.log是默认创建的收集日志的交换器，类型为topic，通过"#"可以收集所有级别的日志，但要注意，多节点的时候日志可能是交错的。

单点故障：硬件故障、断电、网络异常、服务进程异常。硬件故障后可以将节点剔除。断电后重启机器，但不要盲目重启服务，否则可能引起网络分区，首先将该节点从集群剔除，然后删除故障机器的Mnesia数据（相当于重置），然后重启rabbitmq服务，将此节点作为一个新的节点加入集群。

集群故障后可能需要集群迁移，通过web管理界面下载旧集群的元数据信息，然后在新集群的web管理界面上传，如果有冲突可能会报错。

自动化迁移作为容灾手段的一种。

通过HTTP API或者程序客户端可以提供监控数据。

注意元数据的监控与管理。

## 高阶

Federation插件的设计目标是使rbbitMQ在不同的Broker节点之间进行消息传递而无须建立集群，可以在不同管理域之间传递消息，可以容忍不稳定的网络连接，可以让多个交换器或队列进行联邦

Shovel够可靠、持续地从一个Broker中的队列拉取数据并转发至另一个Broker中的交换器，作为源端的队列和作为目的端的交换器可以同 时位于同一个Broker上，也可以位于不同的Broker上，就像一个“铲子”进行挖取。优势是松耦合，支持广域网，高度定制。

通常队列中的消息会尽可能存储在内存中，惰性队列会尽可能存入磁盘，以支持更长的队列。

默认情况下当RabbitMQ使用的内存超过40%时，就会产生内存告警井阻塞所有生产者的连接。

流控机制可以避免消息发送过快导致服务器难以支撑。

镜像队列通过将队列镜像到其他broker上，如果一个节点失效了，队列能自动切换到镜像中的另一个节点来保证服务可用性。每一个镜像队列都包含一个主节点和若干从节点，从按照主执行命令的顺序动作，如果主节点失效，加入最早的从升主。发送到队列的所有消息同时发送给主和所有的从，其他动作只向主发送，然后主将命令执行结果广播给从。如果消费者与从建立连接并消费，实质上都是从主获取消息。大多数压力都落在主上，但主从是对队列而言的，队列可以散布在各个节点以负载均衡。主断开之后，新的主会重新入队所有unack消息，因为无法确定这些消息是否已到达客户端，此时可能会导致重复消息。

## 网络分区

出现网络分区时，不同分区里的节点会认为不属于自身所在分区的节点都已经down 了，默认即使网络恢复了也不会自动处理网络分区。

网络分区造成的影响大多是负面的，引入这样设计的原因是rabbitMQ采用的镜像队列是一种环形的逻辑结构，如果某个节点网络异常会造成数据链堵塞，通过网络分区将异常节点剥离可以确保可用性和可靠性，通常会形成一个大分区和一个单节点的分区。

连续4次未应答的节点会被认为down并踢出分区。通过日志、rabbitmqctl命令、web管理界面、HTTP API可以查看是否发生了网络分区。

可以通过封禁/解封端口、开关网卡、挂起/恢复操作系统来模拟网络分区。

未配置镜像的集群，网络分区会导致队列随宿主节点分散在各自分区，发送方可能成功发送也可能路由失败，消费方可能会有诡异不可预知的现象发生。

网络分区可能会导致消息丢失，解决方法：发送端要有确认发送成功的能力，其次发生网络分区后挂起所有生产者，连接分区中的每个节点消费所有队列数据，消费完后处理网络分区，之后再恢复生产者进程。但会伴随大量的消息重复，消费者要做好幂等性处理。

手动处理时，先挑选信任分区，然后重启非信任节点，如果还有告警接着重启信任分区的节点。挑选时，优先选择有disc节点的、节点数多的、队列数多的、客户端连接数多的分区（指标有先后顺序）。恢复过程中可能会出现队列“漂移”，导致压力都集中到某个节点。

自动处理有三种方式，默认是不处理。pause-minority模式会关闭少数派，pause-if-all-down会有一个受信节点列表，如果其他节点和列表中任何节点不能交互时关闭。autoheal模式下会自动决定一个获胜分区，重启不在分区中的节点。会依次按照客户端连接数、节点数、节点名称的字典序来挑选。自动处理不一定会有正面的成效。

## 扩展

使用Firehose实现消息追踪，Firehose的原理是将生产者投递给RabbitMQ的消息，或者RabbitMQ投递给消费者的消息按照指定的格式发送到默认的交换器上。

客户端与集群建立的TCP连接不是与集群中所有的节点建立连接，而是挑选其中一个节点建立连接。对节点的负载均衡可以通过在客户端中使用轮询法、加权轮询法、随机法、加权随机法、源地址哈希法、最小连接数法进行。也可以使用HAProxy和Keepalived、LVS