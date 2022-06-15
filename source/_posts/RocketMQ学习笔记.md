---
title: RocketMQ学习笔记
date: 2022-06-15 22:35:55
top: true
tags:
- 学习笔记

categories:
- MQ
  - RocketMQ
---


  一种提供消息队列服务的中间件，是一套提供了消息生产、存储、消费全过程API的软件系统。消息即数据，一般消息的体量不会很大。
主要用途为限流削峰、异步处理、服务解耦。

<!-- more -->

##### 1、MQ 简介

一种提供消息队列服务的中间件，是一套提供了消息生产、存储、消费全过程API的软件系统。消息即数据，一般消息的体量不会很大

##### 2、用途

**限流削峰**

MQ可以将系统的超量请求暂存其中，以便系统后期可以慢慢进行处理，从而避免了请求的丢失或系统被压垮

![image-20220330211205452](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220330211205452.png)



**异步处理**

上游系统对下游系统的调用如果为同步调用，会大大降低系统的吞吐量和并发度，异步调用会解决这些问题。所有两层之间若要实现同步到异步的转化，一般做法就是在这两层之间添加一个MQ层。

**服务解耦**

##### 3、MQ选择

中间件的考量维度：可靠性，性能，功能，可运维行，可拓展性，是否开源及社区活跃度

**rabbitmq：**

优点：轻量，迅捷，容易部署和使用，拥有灵活的路由配置

缺点：性能和吞吐量较差，不易进行二次开发

**rocketmq：**

优点：性能好，稳定可靠，有活跃的中文社区，特点响应快

缺点：兼容性较差，但随意影响力的扩大，该问题会有改善

**kafka：**

优点：拥有强大的性能及吞吐量，兼容性很好

缺点：由于“攒一波再处理”导致延迟比较高

**pulsar：** 采用存储和计算分离的设计，是消息队里产品中黑马，值得持续关

![image-20220330213904530](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220330213904530.png)

### 二、RocketMQ 概述

##### 1、简介

RocketMQ是一个统一消息引擎、轻量级数据处理平台。

是阿里巴巴开源的消息中间件。2016年11月28日，阿里巴巴向Apache基金会捐赠该项目。2017年9月25，成为Apache顶级项目。

##### 2、基本概念

![image-20220330223452188](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220330223452188.png)

**消息模型**

RocketMQ主要由Producer、Broker、Consumer三部分组成，其中Producer负责生产消息、Consumer负责消费消息、Broker负责存储消息。Broker在实际部署中对应一天服务器，每个Broker可以存储多个Topic的消息，每个Topic可以分片存储于不同的Broker。

**队列**

Message Queue用于存储消息的物理地址。每个Topic中的消息地址 存储在多个MessageQueue中。ConsumerGroup由多个Consumer实例构成

**消息生产者**

Producer 负责生产消息，一般由业务系统负责生产消息。一个消息生产者会把业务应用系统里产生的消息发送到broker服务器。RocketMQ提供多种发送方式，同步发送、异步发送、顺序发送、单向发送。同步和异步方式均需要Broker返回确认信息，单向发送不需要。

**消息消费者（Consumer）**

负责消费消息，一般是后台系统负责异步消费。一个消息消费者会从Broker服务器拉取消息、并将其提供给应用程序。从用户应用的角度而言提供了两种消费形式：拉取式消费、推动式消费。

**代理服务器（Broker Server）**

消息中转角色，负责存储消息、转发消息。代理服务器在RocketMQ系统中负责接收从生产者发送来的消息并存储、同时为消费者的拉取请求作准备。代理服务器也存储消息相关的元数据，包括消费者组、消费进度偏移和主题和队列消息等。

**名字服务（Name Server）**

名称服务充当路由消息的提供者。生产者或消费者能够通过名字服务查找各主题相应的Broker IP列表。多个Namesrv实例组成集群，但相互独立，没有信息交换。

**拉取式消费（Pull Consumer）**

Consumer消费的一种类型，应用通常主动调用Consumer的拉消息方法从Broker服务器拉消息、主动权由应用控制。一旦获取了批量消息，应用就会启动消费过程。

**推动式消费（Push Consumer）**

Consumer消费的一种类型，该模式下Broker收到数据后会主动推送给消费端，该消费模式一般实时性较高。

**生产者组（Producer Group）**

同一类Producer的集合，这类Producer发送同一类消息且发送逻辑一致。如果发送的是事务消息且原始生产者在发送之后崩溃，则Broker服务器会联系同一生产者组的其他生产者实例以提交或回溯消费。

**消费者组（Consumer Group）**

同一类Consumer的集合，这类Consumer通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现负载均衡和容错的目标变得非常容易。要注意的是，消费者组的消费者实例必须订阅完全相同的Topic。RocketMQ 支持两种消息模式：集群消费（Clustering）和广播消费（Broadcasting）。

**集群消费**（Clustering）

集群消费模式下,相同Consumer Group的每个Consumer实例平均分摊消息。

**广播消费**（Broadcasting）

广播消费模式下，相同Consumer Group的每个Consumer实例都接收全量的消息。

**普通顺序消息**（Normal Ordered Message）

普通顺序消费模式下，消费者通过同一个消息队列（ Topic 分区，称作 Message Queue） 收到的消息是有顺序的，不同消息队列收到的消息则可能是无顺序的。

**严格顺序消息（Strictly Ordered Message）**

严格顺序消息模式下，消费者收到的所有消息均是有顺序的。

**消息**

消息是指消息系统所传输信息的物理载体，生产和消费数据的最小单位，每条消息必须属于一个主题。RocketMQ中每个消息拥有唯一的MessageID，可以携带具有业务标识的Key。系统提供了通过MessageId和Key查询消息的功能。

**主题**

表示一类消息的集合，每个主题包含若干条消息，每条消息只能属于一个主题，是进行消息订阅的基本单位i

**标签**

为消息设置的标志，用以同一主题下区分不同类型的消息。来自同一业务单元的消息，可以根据不用业务目的在同一主题下设置不同标签。标签能够有效保持代码的清晰度和连贯性，并优化RocketMQ提供的查询系统。消费者可以根据Tag实现对不同主题的不同消费逻辑，实现更好的扩展

##### NameServer

NameServer主要作用是为生产者和消费者提供关于Topic的路由信息

主要包括两个功能：

Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；

路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。

NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。

**路由注册**

路由注册时通过Broker与NameServer的心跳功能实现的。Broker启动时向集群中所有的NameServer保持长连接，每隔30秒向所有的NameServer发送心跳包，NameServer收到心跳包时会更新brokerLiveTable缓存中BrokerLiveInfo的lastUpdateTimestamp，

**路由删除**

NameServer和Broker保持长连接，Broker状态存储在BrokerLiveTable里，NameServer会每10s扫描一次NameServer

一旦发现已经有120s没有收到Broker发送过来的心跳信息，就移除该Broker并关闭与Broker的连接，同时更新路由元信息。 还有一种情况是Broker正常关闭，会执行unRegisterBroker指令。

**路由发现**

路由发现并不是实时的，当Topic路由出现变化后，NameServer不主动推送给客户端。由客户端定时拉取Topic最新的路由

##### 3、特性

**订阅发布**

消息的发布是指某个生产者向某个topic发送消息；消息的订阅是指某个消费者关注了某个topic中带有某些tag的消息，进而从该topic消费数据

**消息顺序**

消息有序是指一类消息消费时，能够按照发送的的顺序来消费。如：：一个订单产生了三条消息分别是订单创建、订单付款、订单完成。消费时要按照这个顺序消费才能有意义，但是同时订单之间是可以并行消费的。

RocketMQ可以按照严格的保证消息有序。

顺序消息分为全局顺序消息与分区顺序消息。

全局顺序是指某个topic下的所有消息都要保证顺序，部分顺序消息只要保证每一组消息被顺序消费即可。

- 全局顺序：对于指定的一个topic，所有消息按照严格的先入先出的顺序进行发布和消费。适用场景：性能要求不高，所有的消息严格按照 FIFO 原则进行消息发布和消费的场景
- 分区顺序：对于指定的一个Topic，所有消息根据sharding key进行区块分区。同一个分区消息按照严格先入先出顺序进行发布和消费。Sharding key是顺序消息中用来区分不同分区的关键字段，和普通消息的key完全不同的概念。用场景：性能要求高，以 sharding key 作为分区字段，在同一个区块中严格的按照 FIFO 原则进行消息发布和消费的场景。

**消息过滤**

RocketMQ的消费者根据Tag进行消息过滤，支持自定义属性过滤。消息过滤目前是在Broker端实现的，优点是减少了Consumer无用消息的网络传输，缺点是增加了Broker的负担、而且实现相对复杂。

**消息可靠性**

RocketMQ支持消息的高可靠，影响消息可靠性的几种情况：

1. Broker非正常关闭
2. Broker异常Crash
3. OS Crash
4. 机器掉电，但是能立即恢复供电情况
5. 机器无法开机（可能是cpu、主板、内存等关键设备损坏）
6. 磁盘设备损坏

1)、2)、3)、4) 四种情况都属于硬件资源可立即恢复情况，RocketMQ在这四种情况下能保证消息不丢，或者丢失少量数据（依赖刷盘方式是同步还是异步）。

5)、6)属于单点故障，且无法恢复，一旦发生，在此单点上的消息全部丢失。RocketMQ在这两种情况下，通过异步复制，可保证99%的消息不丢，但是仍然会有极少量的消息可能丢失。通过同步双写技术可以完全避免单点，同步双写势必会影响性能，适合对消息可靠性要求极高的场合，例如与Money相关的应用。注：RocketMQ从3.0版本开始支持同步双写。

**至少一次**

至少一次(At least Once)指每个消息必须投递一次。Consumer先Pull消息到本地，消费完成后，才向服务器返回ack，如果没有消费一定不会ack消息，所以RocketMQ可以很好的支持此特性。

**回溯消费**

回溯消费是指Consumer已经消费成功的消息，由于业务上需求需要重新消费，要支持此功能，Broker在向Consumer投递成功消息后，消息仍然需要保留。并且重新消费一般是按照时间维度，例如由于Consumer系统故障，恢复后需要重新消费1小时前的数据，那么Broker要提供一种机制，可以按照时间维度来回退消费进度。RocketMQ支持按照时间回溯消费，时间维度精确到毫秒。

**事务消息**

RocketMQ事务消息（Transactional Message）是指应用本地事物和发送消息操作可以被定义到全局事务中，要么同时成功，要么同时失败。RocketMQ的事务消息提供类似 X/Open XA 的分布事务功能，通过事务消息能达到分布式事务的最终一致

**定时消息**

定时消息（延迟队列）是指消息发送broker后，不会立即被消费，等待特定时间投递给真正的topic。broker有配置项messageDelayLevel，默认值为“1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h”，18个level。

可以自定义messageDelayLevel。messageDelayLevel是broker的属性，不属于某个topic。发消息时，设置delayLevel等级即可：msg.setDelayLevel(level)。level有以下三种情况：

- level == 0，消息为非延迟消息
- 1<=level<=maxLevel，消息延迟特定时间，例如level==1，延迟1s
- level > maxLevel，则level== maxLevel，例如level==20，延迟2h

定时消息会暂存在名为SCHEDULE_TOPIC_XXXX的topic中，并根据delayTimeLevel存入特定的queue，queueId = delayTimeLevel – 1，即一个queue只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。broker会调度地消费SCHEDULE_TOPIC_XXXX，将消息写入真实的topic。

需要注意的是，定时消息会在第一次写入和调度写入真实topic时都会计数，因此发送数量、tps都会变高。

**消息重试**

Consumer消费消息失败后，要提供一种重试机制，令消息再消费一次。Consumer消费消息失败通常可以认为有以下几种情况：

- 由于消息本身的原因，例如反序列化失败，消息数据本身无法处理（例如话费充值，当前消息的手机号被注销，无法充值）等。这种错误通常需要跳过这条消息，再消费其它消息，而这条失败的消息即使立刻重试消费，99%也不成功，所以最好提供一种定时重试机制，即过10秒后再重试。
- 由于依赖的下游应用服务不可用，例如db连接不可用，外系统网络不可达等。遇到这种错误，即使跳过当前失败的消息，消费其他消息同样也会报错。这种情况建议应用sleep 30s，再消费下一条消息，这样可以减轻Broker重试消息的压力。

RocketMQ会为每个消费组都设置一个Topic名称为“%RETRY%+consumerGroup”的重试队列（这里需要注意的是，这个Topic的重试队列是针对消费组，而不是针对每个Topic设置的），用于暂时保存因为各种异常而导致Consumer端无法消费的消息。考虑到异常恢复起来需要一些时间，会为重试队列设置多个重试级别，每个重试级别都有与之对应的重新投递延时，重试次数越多投递延时就越大。RocketMQ对于重试消息的处理是先保存至Topic名称为“SCHEDULE_TOPIC_XXXX”的延迟队列中，后台定时任务按照对应的时间进行Delay后重新保存至“%RETRY%+consumerGroup”的重试队列中。

**消息重投**

生产者在发送消息时，同步消息失败会重投，异步消息有重试，oneway没有任何保证。消息重投保证消息尽可能发送成功、不丢失，但可能会造成消息重复，消息重复在RocketMQ中是无法避免的问题。消息重复在一般情况下不会发生，当出现消息量大、网络抖动，消息重复就会是大概率事件。另外，生产者主动重发、consumer负载变化也会导致重复消息。如下方法可以设置消息重试策略：

- retryTimesWhenSendFailed:同步发送失败重投次数，默认为2，因此生产者会最多尝试发送retryTimesWhenSendFailed + 1次。不会选择上次失败的broker，尝试向其他broker发送，最大程度保证消息不丢。超过重投次数，抛出异常，由客户端保证消息不丢。当出现RemotingException、MQClientException和部分MQBrokerException时会重投。
- retryTimesWhenSendAsyncFailed:异步发送失败重试次数，异步重试不会选择其他broker，仅在同一个broker上做重试，不保证消息不丢。
- retryAnotherBrokerWhenNotStoreOK:消息刷盘（主或备）超时或slave不可用（返回状态非SEND_OK），是否尝试发送到其他broker，默认false。十分重要消息可以开启。

**流量控制**

生产者流控，因为broker处理能力达到瓶颈；消费者流控，因为消费能力达到瓶颈。

生产者流控：

- commitLog文件被锁时间超过osPageCacheBusyTimeOutMills时，参数默认为1000ms，返回流控。
- 如果开启transientStorePoolEnable == true，且broker为异步刷盘的主机，且transientStorePool中资源不足，拒绝当前send请求，返回流控。
- broker每隔10ms检查send请求队列头部请求的等待时间，如果超过waitTimeMillsInSendQueue，默认200ms，拒绝当前send请求，返回流控。
- broker通过拒绝send 请求方式实现流量控制。

注意，生产者流控，不会尝试消息重投。

消费者流控：

- 消费者本地缓存消息数超过pullThresholdForQueue时，默认1000。
- 消费者本地缓存消息大小超过pullThresholdSizeForQueue时，默认100MB。
- 消费者本地缓存消息跨度超过consumeConcurrentlyMaxSpan时，默认2000。

消费者流控的结果是降低拉取频率。

**死信队列**

死信队列用于处理无法被正常消费的消息。当一条消息初次消费失败，消息队列会自动消息重试；达到最大重试次数后，如果消费依然失败，表明消费者在正常情况下无法正确的消费该消息，此时，消息队列不会立刻将消息丢弃，而是将其发送到该消费者对应的特殊队列中。

RocketMQ将这种正常情况下无法被消费的消息称为死信消息，将存储死信消息的特殊队列称为死信队列。在RocketMQ中，可以通过console控制台对死信队列中的消息进行重发使得消费者实例再次进行消费。

### 三、数据复制与刷盘策略

![image-20220403140959146](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220403140959146.png)

#### 复制策略

复制策略是Broker的Master与Slave之间的数据同步方式。分为同步复制和异步复制：

同步复制：消息写入master后，master会等待slave同步数据成功后才向producer返回ACK

异步复制：消息写入master后，master立即向producer返回成功ACK，无须等待slave数据成功

#### 刷盘策略

![image-20220404161605051](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220404161605051.png)

同步刷盘：只有在消息真正持久化至磁盘后RocketMQ的Broker端才会真正返回给Producer端一个成功的ACK响应。同步刷盘对MQ消息可靠性来说是一种不错的保障，但是性能上会有较大影响，一般适用于金融业务应用该模式较多。

异步刷盘：能够充分利用OS的PageCache的优势，只要消息写入PageCache即可将成功的ACK返回给Producer端。消息刷盘采用后台异步线程提交的方式进行，降低了读写延迟，提高了MQ的性能和吞吐量。

#### 通信机制

RocketMQ消息队列集群主要包括NameServer、Broker(Master/Slave)、Producer、Consumer4个角色，基本通讯流程如下：

1. Broker启动后需要完成一次将自己注册到NameServer的操作，随后每隔30s时间定时向NameServer上报Topic路由信息
2. 消息生产者Producer作为客户端发送消息时候，需要根据消息的Topic从本地缓存的TopicPublishInfoTable获取路由信息。如果没有从NameServer上重新拉取路由信息。同时Producer会默认每隔30s向NameServer拉取一次路由信息
3. 消息生产者根据路由信息选择一个队列进行消息发送；Broker作为消息的接收者接收消息并落盘存储。
4. 消费者根据路由信息，在完成客户端的负载均衡后，选择其中一个或某几个消息队列拉取消息并进行消费。

rocketmq-remoting 模块是 RocketMQ消息队列中负责网络通信的模块，它几乎被其他所有需要网络通信的模块（诸如rocketmq-client、rocketmq-broker、rocketmq-namesrv）所依赖和引用。为了实现客户端与服务器之间高效的数据请求与接收，RocketMQ消息队列自定义了通信协议并在Netty的基础之上扩展了通信模块

####  负载均衡

RocketMQ中的负载均衡都在Client端完成，具体来说的话，主要可以分为Producer端发送消息时候的负载均衡和Consumer端订阅消息的负载均衡。

#### 事务消息

Apache RocketMQ在4.3.0版中已经支持分布式事务消息，这里RocketMQ采用了2PC的思想来实现了提交事务消息，同时增加一个补偿逻辑来处理二阶段超时或者失败的消息，如下图所示。

![image-20220404162721289](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220404162721289.png)

上图说明了事务消息的大致方案，其中分为两个流程：正常事务消息的发送及提交、事务消息的补偿流程。

1. 事务消息发送及提交：

   (1) 发送消息（half消息）。

   (2) 服务端响应消息写入结果。

   (3) 根据发送结果执行本地事务（如果写入失败，此时half消息对业务不可见，本地逻辑不执行）。

   (4) 根据本地事务状态执行Commit或者Rollback（Commit操作生成消息索引，消息对消费者可见）。

2. 补偿流程：

   (1) 对没有Commit/Rollback的事务消息（pending状态的消息），从服务端发起一次“回查”

   (2) Producer收到回查消息，检查回查消息对应的本地事务的状态

   (3) 根据本地事务状态，重新Commit或者Rollback

   其中，补偿阶段用于解决消息Commit或者Rollback发生超时或者失败的情况。

### 四、工作原理

#### 一、消息的生产

##### 消息的生产过程

Producer可以将消息写入到Broker中的某Queue中，经历了如下过程：

- Producer发送消息之前，会向NameServer发出获取消息Topic的路由信息请求
- Namserver 返回该Topic的路由表和Broker列表
- Producer根据代码中指定的Queue选择策略，从Queue列表中选择一个队列，用于后续存储消息
- Producer对消息做一些特殊处理，如：消息超过4M，会对其进行压缩
- Producer向选择出的Queue所在的Broker发出RPC请求，将消息发送到选择出Queue

##### Queue选择算法

对于无序消息，Queue选择算法（消息投递算法），常见的有两种：

**轮训算法**

默认选择算法，该算法保证了每个Queue中可以均匀的获取到消息

**最小投递延迟算法**



##### 二、消息存储

abort：该文件在Broker启动后会自动创建，正常关闭Broker该文件会自动消失。若没有启动Broker的情况下，发现这个文件是存在的，说明之前Broker的关闭是非正常关闭。

checkpoint：存储commitlog、consumequeue、index文件的最后刷盘时间戳

commitlog：存储着commitlog文件、消息写在commitlog文件中

config：存放着Broker运行期间的一些配置数据

Consumerqueue：存放着consumerqueue文件，队列存放在这个目录中。

index：其中存放着消息索引文件indexFile

lock：运行期间使用到的全局资源锁

**Commitlog文件**

commitlog目录中存放很多的mappedFile文件，当前Broker中的所有消息都是落盘到这些mappedFile文件中的。mappedFile文件早打小为1G，文件名由20位十进制数构成，表示当前文件的第一条消息的起始位移偏移量。一个Broker中仅包含一个commitlog目录，所有的mappedFile文件都是存放在该目录中的。即无论当前Broker中存放多少Topic的消息，这些消息都是被顺序写入到了mappedFile文件中的。也就是说，这些消息在Broker中存放时并没有被按照Topic进行分类存放

> mappedFile是顺序读写文件，访问效率很高

为了提高效率，会为每个topic在~/store/consumequeue中创建一个目录，目录名为Topic名称。在该Topic目录下，会再为每个Topic的Queue建立一个目录，目录名为queueId。每个目录中存放若干consumequeue文件，consumequeue文件是commitlog的索引文件，可以根据consumequeue定位到具体的消息。

**索引条目**



![image-20220404181640638](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220404181640638.png)

每个consumequeue文件可以包含30w个索引条目，每个索引条目包含了三个消息重要属性：消息在mappedFile文件中的偏移量Commitlog Offset、消息长度、消息Tag的HashCode。



**消息单元**

![image-20220404175556071](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20220404175556071.png)

mappedFIle文件内容由一个个消息单元构成。每个消息单元中包含消息总长度MsgLen、消息的物理位置physicalOffset、消息内容body、消息总长度BodyLength、消息主题topic、topic长度topicLength、消息生产者Bornhost、消息发送时间戳BornTimestamp、消息所在队列QueueId消息在Queueu中存储的偏移量QueueOffset等相关属性。



