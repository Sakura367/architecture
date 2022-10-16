### 目标
1. 缓冲和削峰：上游数据时有突发流量，下游可能扛不住，或者下游没有足够多的机器来保证冗余，kafka在中间可以起到一个缓冲的作用，把消息暂存在kafka中，下游服务就可以按照自己的节奏进行慢慢处理。
2. 解耦和扩展性：项目开始的时候，并不能确定具体需求。消息队列可以作为一个接口层，解耦重要的业务流程。只需要遵守约定，针对数据编程即可获取扩展能力。
3. 冗余：可以采用一对多的方式，一个生产者发布消息，可以被多个订阅topic的服务消费到，供多个毫无关联的业务使用。
4. 健壮性：消息队列可以堆积请求，所以消费端业务即使短时间死掉，也不会影响主要业务的正常进行。
5. 异步通信：很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。

### 适用场景
1. 信息系统 Messaging 。 在这个领域中，kafka常常被拿来与传统的消息中间件进行对比，如RabbitMQ。
2. 网站活动追踪 Website Activity Tracking
3. 监控 Metrics
4. 日志收集 Log Aggregation
5. 流处理 Stream Processing
6. 事件溯源 Event Sourcing
7. 提交日志 Commit Log

### 组件
Producer ： 消息生产者，就是向 Kafka发送数据 ；
Consumer ： 消息消费者，从 Kafka broker 取消息的客户端；
Consumer Group （CG）： 消费者组，由多个 consumer 组成。 消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。 所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。
Broker ：经纪人 一台 Kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker可以容纳多个 topic。
Topic ： 话题，可以理解为一个队列， 生产者和消费者面向的都是一个 topic；
Partition： 为了实现扩展性，一个非常大的 topic 可以分布到多个 broker（即服务器）上，一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列；如果一个topic中的partition有5个，那么topic的并发度为5.
Replica： 副本（Replication），为保证集群中的某个节点发生故障时， 该节点上的 partition 数据不丢失，且 Kafka仍然能够继续工作， Kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本，一个 leader 和若干个 follower。
Leader： 每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是 leader。
Follower： 每个分区多个副本中的“从”，实时从 leader 中同步数据，保持和 leader 数据的同步。 leader 发生故障时，某个 Follower 会成为新的 leader。
Offset： 每个Consumer 消费的信息都会有自己的序号，我们称作当前队列的offset。即消费点位标识消费到的位置。每个消费组都会维护订阅的Topic 下每个队列的offset。
ZooKeeper：维护集群节点状态信息。主要储存存储各个节点的信息及状态、ISR的信息、consumer读取的消息的offset等（生产者不需要直接读取zookeeper，从broker里获取集群的状态信息）。

### 分配关系
kafka 为了保证同一类型的消息顺序性（FIFO），一个consumer group消费同一个topic，但一个partition只能被同一组的一个consumer消费。不同组的consumer可以消费同一个partition（重复消费），但是一个consumer可以消费多个partition（消费者同时消费多个partition容易导致负载过大）。

### 分区原则
1. 指明partition的情况下，使用指定的partition；
2. 没有指明partition，但是有key的情况下，将key的hash值与topic的partition数进行取余得到partition值；
3. 既没有指定partition，也没有key的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与topic可用的partition数取余得到partition值，也就是常说的round-robin算法。

### Consumer 与Consumer Group
Consumer Group与Consumer的关系是动态维护的：
当一个Consumer 进程挂掉 或者是卡住时，该consumer所订阅的partition会被重新分配到该group内的其它的consumer上。当一个consumer加入到一个consumer group中时，同样会从其它的consumer中分配出一个或者多个partition 到这个新加入的consumer。

当启动一个Consumer时，会指定它要加入的group，使用的是配置项：group.id。

为了维持Consumer 与 Consumer Group的关系，需要Consumer周期性的发送heartbeat到coordinator。当Consumer由于某种原因不能发Heartbeat到coordinator时，并且时间超过session.timeout.ms时，就会认为该consumer已退出，它所订阅的partition会分配到同一group 内的其它的consumer上。而这个过程，被称为rebalance。

### Coordinator
Coordinator 协调者，协调consumer、broker。早期版本中Coordinator，使用zookeeper实现，但是这样做，rebalance的负担太重。为了解决scalable的问题，不再使用zookeeper，而是让每个broker来负责一些group的管理，这样consumer就完全不再依赖zookeeper了。

#### Consumer连接到coordinator
从Consumer的实现来看，在执行poll或者是join group之前，都要保证已连接到Coordinator。连接到coordinator的过程是：
1. 连接到最后一次连接的broker（如果是刚启动的consumer，则要根据配置中的borker）。它会响应一个包含coordinator信息(host, port等)的response。
2. 连接到coordinator。

#### Consumer Group Management
Consumer Group 管理中，也是需要coordinator的参与。一个Consumer要join到一个group中，或者一个consumer退出时，都要进行rebalance。进行rebalance的流程是：
1. 会给一个coordinator发起Join请求（请求中要包括自己的一些元数据，例如自己感兴趣的topics）
2. Coordinator 根据这些consumer的join请求，选择出一个leader，并通知给各个consumer。这里的leader是consumer group 内的leader，是由某个consumer担任，不要与partition的leader混淆。
3. Consumer leader 根据这些consumer的metadata，重新为每个consumer member重新分配partition。分配完毕通过coordinator把最新分配情况同步给每个consumer。
4. Consumer拿到最新的分配后，继续工作。

同一个消费组共同管理一个offset，保存在Coordinator（也在broker中），Coordinator用来协调 消费组的不同consumer消费哪个partition、offset等。但不同的消费组的offset就不一样了，分别是独立的。

### partition数据一致性
为了提高消息的可靠性，Kafka每个topic的partition有N个副本(replicas)，其中N(大于等于1)是topic的复制因子(replica fator)的个数。

Kafka通过多副本机制实现故障自动转移，当Kafka集群中一个broker失效情况下仍然保证服务可用。在Kafka中发生复制时确保partition的日志能有序地写到其他节点上（N个replicas中），其中一个replica为leader，其他都为follower, leader处理partition的所有读写请求，与此同时，follower会被动定期地去复制leader上的数据。

如果leader发生故障或挂掉，Kafka会从ISR（副本同步队列）中选取一个follower替代成为leader。leader负责维护和跟踪ISR中所有follower滞后的状态。

当producer发送一条消息到broker后，leader写入消息并复制到所有follower。

消息复制延迟受最慢的follower限制，需要快速检测慢副本，如果follower“落后”太多或者失效，leader将会把它从ISR中删除。

#### SR
分区中的所有副本统称为AR（Assigned Repllicas）。所有与leader副本保持一定程度同步的副本（包括Leader）组成ISR（In-Sync Replicas），ISR集合是AR集合中的一个子集。消息会先发送到leader副本，然后follower副本才能从leader副本中拉取消息进行同步，同步期间内follower副本相对于leader副本而言会有一定程度的滞后。前面所说的“一定程度”是指可以忍受的滞后范围，这个范围可以通过参数进行配置。与leader副本同步滞后过多的副本（不包括leader）副本，组成OSR(Out-Sync Relipcas),由此可见：AR=ISR+OSR。在正常情况下，所有的follower副本都应该与leader副本保持一定程度的同步，即AR=ISR,OSR集合为空。

Leader副本负责维护和跟踪ISR集合中所有的follower副本的滞后状态，当follower副本落后太多或者失效时，leader副本会把它从ISR集合中剔除。如果OSR集合中follower副本“追上”了Leader副本，之后再ISR集合中的副本才有资格被选举为leader，而在OSR集合中的副本则没有机会（这个原则可以通过修改对应的参数配置来改变）

#### LEO和HW
每个Kafka副本对象都有两个重要的属性：LEO和HW。注意是所有的副本，而不只是leader副本。

LEO：即日志末端位移(log end offset)，记录了该副本底层日志(log)中下一条消息的位移值。注意是下一条消息！也就是说，如果LEO=10，那么表示该副本保存了10条消息，位移值范围是[0, 9]。

分区ISR集合中的每个副本都会维护自身的LEO，而ISR集合中最小的LEO即为分区的HW。

HW：对于同一个副本对象而言，其HW值不会大于LEO值。小于等于HW值的所有消息都被认为是“已备份”的（replicated），对消费者而言只能消费HW之前的消息。

### Kafka如何保证高吞吐
- Partition 顺序读写，充分利用磁盘特性，这是基础。
- Producer 生产的数据持久化到 Broker，采用 mmap 文件映射，实现顺序的快速写入。
- Customer 从 Broker 读取数据，采用 sendfile，将磁盘文件读到 OS 内核缓冲区后，直接转到 socket buffer 进行网络发送。
- Broker 性能优化：日志记录批处理、批量压缩、非强制刷新缓冲写操作等。
- 流数据并行
#### 数据顺序写入日志文件
Kafka 利用了一种分段式的、只追加 (Append-Only) 的日志，基本上把自身的读写操作限制为顺序 I/O，也就使得它在各种存储介质上能有很快的速度。

这种方法采用了只读设计 ，所以 Kafka 是不会修改、删除数据的，它会把所有的数据都保留下来，每个消费者（Consumer）对每个 Topic 都有一个 offset 用来表示读取到了第几条数据。

磁盘的顺序读写是磁盘使用模式中最有规律的，并且操作系统也对这种模式做了大量优化，Kafka 就是使用了磁盘顺序读写来提升的性能。Kafka 的 message 是不断追加到本地磁盘文件末尾的，而不是随机的写入，这使得 Kafka 写入吞吐量得到了显著提升。

#### 使用mmap映射文件
简称 mmap，简单描述其作用就是：将磁盘文件映射到内存，用户通过修改内存就能修改磁盘文件。

它的工作原理是直接利用操作系统的 Page 来实现磁盘文件到物理内存的直接映射。完成映射之后你对物理内存的操作会被同步到硬盘上（操作系统在适当的时候）。

通过 mmap，进程像读写硬盘一样读写内存（当然是虚拟机内存）。使用这种方式可以获取很大的 I/O 提升，省去了用户空间到内核空间复制的开销。

mmap 也有一个很明显的缺陷：不可靠，写到 mmap 中的数据并没有被真正的写到硬盘，操作系统会在程序主动调用 flush 的时候才把数据真正的写到硬盘。

Kafka 提供了一个参数 producer.type 来控制是不是主动 flush：
1. 如果 Kafka 写入到 mmap 之后就立即 flush，然后再返回 Producer 叫同步（sync）;
2. 写入 mmap 之后立即返回 Producer 不调用 flush 叫异步（async）。

#### 零拷贝
服务端提供文件传输功能的方式是：将磁盘上的文件读取出来，然后通过网络协议发送给客户端。

##### 传统IO方式
数据读取和写入是从用户空间到内核空间来回复制，而内核空间的数据是通过操作系统层面的 I/O 接口从磁盘读取或写入。

期间共发生了 4 次用户态与内核态的上下文切换，因为发生了两次系统调用，一次是 read() ，一次是 write()，每次系统调用都得先从用户态切换到内核态，等内核完成任务后，再从内核态切换回用户态。

还发生了 4 次数据拷贝，其中两次是 DMA 的拷贝，另外两次则是通过 CPU 拷贝的。
- 第一次拷贝，把磁盘上的数据拷贝到操作系统内核的缓冲区里，这个拷贝的过程是通过 DMA 搬运的。
- 第二次拷贝，把内核缓冲区的数据拷贝到用户的缓冲区里，于是我们应用程序就可以使用这部分数据了，这个拷贝到过程是由 CPU 完成的。
- 第三次拷贝，把刚才拷贝到用户的缓冲区里的数据，再拷贝到内核的 socket 的缓冲区里，这个过程依然还是由 CPU 搬运的。
- 第四次拷贝，把内核的 socket 缓冲区里的数据，拷贝到网卡的缓冲区里，这个过程又是由 DMA 搬运的。

##### mmap/write 方式
使用mmap/write方式替换原来的传统I/O方式，利用了虚拟内存的特性。

把数据读取到内核缓冲区后，应用程序进行写入操作时，直接把内核的Read Buffer的数据复制到Socket Buffer以便写入，这次内核之间的复制也是需要CPU的参与的。

上述流程就是少了一个 CPU COPY，提升了 I/O 的速度。不过发现上下文的切换还是4次并没有减少，这是因为还是要应用程序发起write操作。

##### sendfile 方式
从 Linux 2.1 版本开始，Linux 引入了 sendfile来简化操作。sendfile方式可以替换上面的mmap/write方式来进一步优化。
```
mmap();
write();
替换为：
sendfile();
减少了上下文切换，因为少了一个应用程序发起write操作，直接发起sendfile操作。
```
只有三次数据复制（其中只有一次 CPU COPY）以及2次上下文切换。

##### 带有 scatter/gather 的 sendfile方式
Linux 2.4 内核进行了优化，提供了带有 scatter/gather 的 sendfile 操作，这个操作可以把最后一次 CPU COPY 去除。其原理就是在内核空间 Read Buffer 和 Socket Buffer 不做数据复制，而是将 Read Buffer 的内存地址、偏移量记录到相应的 Socket Buffer 中，这样就不需要复制。其本质和虚拟内存的解决方法思路一致，就是内存地址的记录。

scatter/gather 的 sendfile 只有两次数据复制（都是 DMA COPY）及 2 次上下文切换。CUP COPY 已经完全没有。不过这一种收集复制功能是需要硬件及驱动程序支持的。

##### splice 方式
splice 调用和sendfile 非常相似，用户应用程序必须拥有两个已经打开的文件描述符，一个表示输入设备，一个表示输出设备。与sendfile不同的是，splice允许任意两个文件互相连接，而并不只是文件与socket进行数据传输。对于从一个文件描述符发送数据到socket这种特例来说，一直都是使用sendfile系统调用，而splice一直以来就只是一种机制，它并不仅限于sendfile的功能。也就是说 sendfile 是 splice 的一个子集。同时也不需要上下文切换。

在 Linux 2.6.17 版本引入了 splice，而在 Linux 2.6.23 版本中， sendfile 机制的实现已经没有了，但是其 API 及相应的功能还在，只不过 API 及相应的功能是利用了 splice 机制来实现的。和 sendfile 不同的是，splice 不需要硬件支持。

#### Broker性能
1. 日志记录批处理
顺序 I/O 在大多数的存储介质上都非常快，几乎可以和网络 I/O 的峰值性能相媲美。在实践中，这意味着一个设计良好的日志结构的持久层将可以紧随网络流量的速度。事实上，Kafka 的瓶颈通常是网络而非磁盘。因此，除了由操作系统提供的底层批处理能力之外，Kafka 的 Clients 和 Brokers 会把多条读写的日志记录合并成一个批次，然后才通过网络发送出去。日志记录的批处理通过使用更大的包以及提高带宽效率来摊薄网络往返的开销。
2. 批量压缩
当启用压缩功能时，批处理的影响尤为明显，因为压缩效率通常会随着数据量大小的增加而变得更高。特别是当使用 JSON 等基于文本的数据格式时，压缩效果会非常显著，压缩比通常能达到 5 到 7 倍。此外，日志记录批处理在很大程度上是作为 Client 侧的操作完成的，此举把负载转移到 Client 上，不仅对网络带宽效率、而且对 Brokers 的磁盘 I/O 利用率也有很大的提升。
3. 非强制刷新缓冲写操作
另一个助力 Kafka 高性能、同时也是一个值得更进一步去探究的底层原因：Kafka 在确认写成功 ACK 之前的磁盘写操作不会真正调用 fsync 命令；通常只需要确保日志记录被写入到 I/O Buffer 里就可以给 Client 回复 ACK 信号。这是一个鲜为人知却至关重要的事实：事实上，这正是让 Kafka 能表现得如同一个内存型消息队列的原因 —— 因为 Kafka 是一个基于磁盘的内存型消息队列 (受缓冲区/页面缓存大小的限制)。

#### 流数据并行
1. Topic 的分区方案。应该对 Topics 进行分区，以最大限度地增加独立子事件流的数量。换句话说，日志记录的顺序应该只保留在绝对必要的地方。如果任意两个日志记录在某种意义上没有合理的关联，那它们就不应该被绑定到同一个分区。这暗示你要使用不同的键值，因为 Kafka 将使用日志记录的键值作为一个散列源来派生其一致的分区映射。
2. 一个组里的 Consumers 数量。你可以增加 Consumer Group 里的 Consumer 数量来均衡入站的日志记录的负载，这个数量的上限是 Topic 的分区数量。(如果你愿意的话，你当然可以增加更多的 Consumers ，不过分区计数将会设置一个上限来确保每一个活跃的 Consumer 至少被指派到一个分区，多出来的 Consumers 将会一直保持在一个空闲的状态。) 请注意， Consumer 可以是进程或线程。依据 Consumer 执行的工作负载类型，你可以在线程池中使用多个独立的 Consumer 线程或进程记录。

### Kafka如何保证消息不丢失、不重复消费
#### 消息发送
Kafka消息发送有两种方式：同步（sync）和异步（async），默认是同步方式，可通过producer.type属性进行配置。Kafka通过配置request.required.acks属性来确认消息的生产：
0---表示不进行消息接收是否成功的确认；
1---表示当Leader接收成功时确认；
-1---表示Leader和Follower都接收成功时确认；
综上所述，有6种消息生产的情况，下面分情况来分析消息丢失的场景：
（1）acks=0，不和Kafka集群进行消息接收确认，则当网络异常、缓冲区满了等情况时，消息可能丢失；
（2）acks=1、同步模式下，只有Leader确认接收成功后但挂掉了，副本没有同步，数据可能丢失；

解决方案：
同步模式下，确认机制设置为-1，即让消息写入Leader和Follower之后再确认消息发送成功；
异步模式下，为防止缓冲区满，可以在配置文件设置不限制阻塞超时时间，当缓冲区满时让生产者一直处于阻塞状态；
#### 消息消费
Kafka消息消费有两个consumer接口，Low-level API和High-level API：
Low-level API：消费者自己维护offset等值，可以实现对Kafka的完全控制；
High-level API：封装了对parition和offset的管理，使用简单；

如果使用高级接口High-level API，可能存在一个问题就是当消息消费者从集群中把消息取出来、并提交了新的消息offset值后，还没来得及消费就挂掉了，那么下次再消费时之前没消费成功的消息就“诡异”的消失了；

解决方案：将消息的唯一标识保存到外部介质中（去重表），每次消费时判断是否处理过即可。

### Kafka不支持读写分离
在 Kafka 中，生产者写入消息、消费者读取消息的操作都是与 leader 副本进行交互的，从 而实现的是一种主写主读的生产消费模型。

Kafka 并不支持主写从读，因为主写从读有 2 个很明 显的缺点:
(1) 数据一致性问题。数据从主节点转到从节点必然会有一个延时的时间窗口，这个时间 窗口会导致主从节点之间的数据不一致。某一时刻，在主节点和从节点中 A 数据的值都为 X， 之后将主节点中 A 的值修改为 Y，那么在这个变更通知到从节点之前，应用读取从节点中的 A 数据的值并不为最新的 Y，由此便产生了数据不一致的问题。
(2) 延时问题。类似 Redis 这种组件，数据从写入主节点到同步至从节点中的过程需要经 历网络→主节点内存→网络→从节点内存这几个阶段，整个过程会耗费一定的时间。而在 Kafka 中，主从同步会比 Redis 更加耗时，它需要经历网络→主节点内存→主节点磁盘→网络→从节 点内存→从节点磁盘这几个阶段。对延时敏感的应用而言，主写从读的功能并不太适用。

### 事务消息
Transaction Coordinator 运行在 Kafka 服务端，下面简称 TC 服务。__transaction_state 是 TC 服务持久化事务信息的 topic 名称，下面简称事务 topic。Producer 向 TC 服务发送的 commit 消息，下面简称事务提交消息。TC 服务向分区发送的消息，下面简称事务结果消息。
1. 寻找 TC 服务地址  
Producer 会首先从 Kafka 集群中选择任意一台机器，然后向其发送请求，获取 TC 服务的地址。Kafka 有个特殊的事务 topic，名称为__transaction_state ，负责持久化事务消息。这个 topic 有多个分区，默认有50个，每个分区负责一部分事务。事务划分是根据 transaction id， 计算出该事务属于哪个分区。这个分区的 leader 所在的机器，负责这个事务的TC 服务地址。
2. 事务初始化
Producer 在使用事务功能，必须先自定义一个唯一的 transaction id。有了 transaction id，即使客户端挂掉了，它重启后也能继续处理未完成的事务。Kafka 实现事务需要依靠幂等性，而幂等性需要指定 producer id 。所以Producer在启动事务之前，需要向 TC 服务申请 producer id。TC 服务在分配 producer id 后，会将它持久化到事务 topic。
3. 发送消息
Producer 在接收到 producer id 后，就可以正常的发送消息了。不过发送消息之前，需要先将这些消息的分区地址，上传到 TC 服务。TC 服务会将这些分区地址持久化到事务 topic。然后 Producer 才会真正的发送消息，这些消息与普通消息不同，它们会有一个字段，表示自身是事务消息。这里需要注意下一种特殊的请求，提交消费位置请求，用于原子性的从某个 topic 读取消息，并且发送消息到另外一个 topic。我们知道一般是消费者使用消费组订阅 topic，才会发送提交消费位置的请求，而这里是由 Producer 发送的。Producer 首先会发送一条请求，里面会包含这个消费组对应的分区（每个消费组的消费位置都保存在 __consumer_offset topic 的一个分区里），TC 服务会将分区持久化之后，发送响应。Producer 收到响应后，就会直接发送消费位置请求给 GroupCoordinator。
4. 发送提交请求
Producer 发送完消息后，如果认为该事务可以提交了，就会发送提交请求到 TC 服务。Producer 的工作至此就完成了，接下来它只需要等待响应。这里需要强调下，Producer 会在发送事务提交请求之前，会等待之前所有的请求都已经发送并且响应成功。
5. 提交请求持久化
TC 服务收到事务提交请求后，会先将提交信息先持久化到事务 topic 。持久化成功后，服务端就立即发送成功响应给 Producer。然后找到该事务涉及到的所有分区，为每 个分区生成提交请求，存到队列里等待发送。读者可能有所疑问，在一般的二阶段提交中，协调者需要收到所有参与者的响应后，才能判断此事务是否成功，最后才将结果返回给客户。那如果 TC 服务在发送响应给 Producer 后，还没来及向分区发送请求就挂掉了，那么 Kafka 是如何保证事务完成。因为每次事务的信息都会持久化，所以 TC 服务挂掉重新启动后，会先从 事务 topic 加载事务信息，如果发现只有事务提交信息，却没有后来的事务完成信息，说明存在事务结果信息没有提交到分区。
6. 发送事务结果信息给分区
后台线程会不停的从队列里，拉取请求并且发送到分区。当一个分区收到事务结果消息后，会将结果保存到分区里，并且返回成功响应到 TC服务。当 TC 服务收到所有分区的成功响应后，会持久化一条事务完成的消息到事务 topic。

### 延迟队列
https://blog.csdn.net/yangbindxj/article/details/123295341

基于时间轮自定义了一个用于实现延迟功能的定时器（SystemTimer），可以将插入和删除操作的时间复杂度都降为O(1)。

底层使用数组实现，数组中的每个元素可以存放一个TimerTaskList对象。TimerTaskList是一个环形双向链表，在其中的链表项TimerTaskEntry中封装了真正的定时任务TimerTask.

Kafka中到底是怎么推进时间的呢？Kafka中的定时器借助了JDK中的DelayQueue来协助推进时间轮。具体做法是对于每个使用到的TimerTaskList都会加入到DelayQueue中。Kafka中的TimingWheel专门用来执行插入和删除TimerTaskEntry的操作，而DelayQueue专门负责时间推进的任务。再试想一下，DelayQueue中的第一个超时任务列表的expiration为200ms，第二个超时任务为840ms，这里获取DelayQueue的队头只需要O(1)的时间复杂度。如果采用每秒定时推进，那么获取到第一个超时的任务列表时执行的200次推进中有199次属于“空推进”，而获取到第二个超时任务时有需要执行639次“空推进”，这样会无故空耗机器的性能资源，这里采用DelayQueue来辅助以少量空间换时间，从而做到了“精准推进”。Kafka中的定时器真可谓是“知人善用”，用TimingWheel做最擅长的任务添加和删除操作，而用DelayQueue做最擅长的时间推进工作，相辅相成。