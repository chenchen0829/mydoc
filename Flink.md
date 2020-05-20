## 架构

网上找了一个flink工作原理的例子。我们是在yarn上运行的。官方给了两种运行的模式,我们这边用的是Flink Run(yarn Session的运行规则决定了集群一次只能跑一个任务，单次任务比较合适，不适合批量长期任务运行)。

```
客户端提交flink程序，并且向yarn申请资源，包含一个jobManager和若干个TaskManager, 每一个都是一个jvm进程。jobManager通过yarn重启失败jobManager以及zk来保证其HA(官方推荐yarn版本在2.5.0以上),
	具体的逻辑：
		当jobmanager失败，yarn会尝试重启jobManager, 重启后，会重新启动Flink的job, 如果出现问题，zk作为协调锁，做HA的自动故障的切换。
	对应的参数：
		high-availability: zookeeper
    high-availability.zookeeper.quorum: localhost:2181
    high-availability.storageDir: hdfs:///flink/recovery
    high-availability.zookeeper.path.root: /flink
    yarn.application-attempts: 10 //应用重启有个上线，不应该超过yarn资源管理器的最大尝试次数。
		
```

![64AE5A70-30E4-4167-A554-5A012BB3F195](/var/folders/cb/dhbmn2ds2sl104111vsrzcmw0000gn/T/64AE5A70-30E4-4167-A554-5A012BB3F195.png )



## 启动参数

```
-yjm=2048 
	指定jobManager的内存占用，我一般设置为1024或者更小.
	
-ytm=5120 
	指定taskManager的内存占用
	
-ys=2
	指定一个taskManager中的slot数目。
	
-yD yarn.containers.vcores=1 
	指定taskManager的vcore数目，需要说明的是vcore在一个taskManager-slot间是共享的，但是内存是不共享的。
	
-yp 10	
		并发度

	需要说明的是：
	1. taskManager的数量是由提交任务的并发度来控制的，并发度的优先级(代码内部>提交参数->flink-conf.yaml)
	2. taskManager vcore是逻辑core, 一个taskManager的vcore在slot之间是共享的，但是内存是隔离的。
	3. 这里就会有一个问题，一个taskManager到底要设置多少个slot，多少个vcore？ 
		1. 我们在使用storm的时候，为了线程之间的隔离，我们每一个excutor只有一个task, 但是storm是没办法管理一个excutor可以包含多少个core的。在flink里面是可以的。
		2. 在storm中一个exector里面多个task是共享内存的，所以在读写共享变量的时候，会有多线程的问题。我们在开发过程中就出现过。在flink里面相同subTask是不会在一个taskSlot里面的，这个是由并行度决定的。flink中如果一个taskManger一个slot，带来的好处是多个任务可以共享vcore的资源，在任务类型不同的情况下，有些是IO性，有些计算型，并且一个链下的任务，数据交换，tcp等，可以大大减少任务的开销。
		
	
	我的一些经验：
		正常情况在一个slot里面可能运行多种不同类型的任务，source, map， sink等。一般的推荐配置为 多个小taskManager, 一个taskManger, 2~4个slot，1~2个vcore。
	
```

![703D9D4F-5A2C-4DF1-B6B3-C2DDD7A81A3B](/var/folders/cb/dhbmn2ds2sl104111vsrzcmw0000gn/T/703D9D4F-5A2C-4DF1-B6B3-C2DDD7A81A3B.png)



## 窗口

```
窗口一般使用的是固定时间窗口，滑动窗口。其他的窗口(session, global)在我们的业务中基本上不会使用。一般固定时间窗口和滑动窗口比较多。

说两点的是，
	1. 窗口的开始和截止，是以EventTime来计算的。和窗口大小求余。并不是24小时计算。不过我们一般用的场景都是24小时制的，所以对这个没有概念。
	2. flink窗口是基于纪元时间的。需要+8hours
```



## Watermark

> ​	watermark出现的目的是为了解决日志乱序问题和日志延迟到达问题。

​	

```

flink提供了三种时间类型，eventTime(事件发生的时间)， ProcessingTime(事件处理的时间)， IngestionTime(事件处理的时间)， 由于ProcessingTime. IngestionTime不能准确反映数据发生的时间序列，只能在一部分场景使用，在大部分场景下我们多选择EventTime.

目前只有eventTime才有waterMark. waterMark是eventTime的标识，一个带有时间戳的waterMark到达，说明任务EventTime小于waterMark的数据都已经到达。我们在storm中一般没有办法处理这种场景，需要特殊处理。

两种waterMark方式，
	定期水位线(periodic Watermark)
		以固定时间间隔生成新的水位线，不管是否有消息抵达，用户可以定义时间间隔，当前批次时间戳的间为基准(一般设置当前批次最大的事件时间)。
	
	标点水位线(punctuated watermark)
		通过数据流一些特殊标记触发新水位线。这种水位线与窗口触发和时间无关。一般很少用。
	
数据乱序/延迟的处理方案：	
  1. 数据乱序，但是我们不关心数据乱序，数据需要写入到对应的窗口中。
  2. 数据乱序，但是已经乱了，但是最多容忍N mins的数据乱序。在N mins内数据乱序，重新激活窗口，修正结果，或者乱序的数据，记录下来特殊处理。
  3. 数据延迟到达，我们还是需要这部分数据写入对应的时间窗口。
  4. 数据延迟到达，我们最多能容忍Nmins的延迟，激活窗口，修正数据，超过Nmins直接忽略。或者乱序的数据，记录下来特殊处理。

flink的做法：
	1. 默认是直接丢弃。
		大部分场景下我们直接根据窗口，设置watermark延迟时间，比如5mins，超过5mins的直接丢弃。但是很多场景下，如果实时数据与离线数据不一致，这种直接丢弃导致我们没有办法排查问题。所以我的应用里面，一般不会选择这种方案。
	2. sideOut
		把迟到的时间单独放入一个数据流分支，
		
	3. AllowedLateness
		设置一个允许的最大的迟到时长，flink再关闭窗口后一直保存这个窗口的状态，指导超过允许迟到的时长。这期间迟到时间不会被丢弃。而是触发窗口重新计算。
		
我这边的做法。
	1. 在生成waterMark的时候，我继承抽象类：BoundedOutOfOrdernessTimestampExtractor， 并且 设置maxOutOfOrderness(最大数据乱序时间)。
	2. 设置窗口的销毁时间allowedLatedness  = 设置maxOutOfOrderness + 10000(窗口的销毁时间为截止水位线往后面推10s,10s之内肯定能处理完数据), 
  3. 设置sideOut.
  	把所有超出maxOutOfOrderness乱序的数据通过sideOut进行输出，打印出来，并且添加sentry报警。记录个数。
  	
注意一点的是maxOutOfOrderness是水位的延迟时间，  allowedLatedness是窗口的销毁时间。

其他一些经验：
	1.设置水位/窗口的时候注意source消息报分区数据是否均衡，因为水位线是以每个分区最小值来算的。如果某一个source分区的数据超级延迟，窗口会会很长时间不会销毁。水位不会提交。
	2. 注意异常eventTime, 有些eventTime为0或者1970.。。会导致水位一直在很低的状态，窗口不会销毁。
	
```



## checkpoint/savepoint

```
两种state
Operate State
	task中每一个operate对应的state. 如果并发度>1 每一个并发度代表一个State, 自定义的Operate State需要实现CheckpointedFunction或者ListCheckpointed<T>. 区别在于后者不需要进行初始化,Flink内部自己解决。
	
KeyedState
	主要应用在keyedStream中，task中每一个key对应一个state。
	
checkpoint
		是从source触发到下游所有节点完成的一次全局操作。应用定时触发。用户保存state快照。
		一般用在系统崩溃重启的时候，从最近的checkpoint重启。
    当然这个要确保source一下下游的操作都已经保存状态的情况下(包含各种source的offset state, 以及下游各种中间的operateState, KeyState),关于source的offset怎么设置，下面再说。
	
savepoint 
	用户手动执行，不会过期，相当于备份。其工作原理和checkpoint是一致的。
	主要用在应用重启从当前的savepoint启动(包含修改并行度)，或者在kill任务的时候保存savepoint.
	

Exactly Once/At Least Once

exactly Once 就是在出发checkpoint的时候，从SouceTask生成barrier, 在发送给下游的同时，后面的消息全部流入输入缓冲区，暂停消费。直到下游所有task出发barrier,然后出发快照生成state. barrier的作用就是把快照前后的数据区分开。
At Least Once 就是barrier发送下游的同时，后面的消息还能进行消费。导致source的offset和真实消费的offset有一定的差异。

实际经验： 
	一般场景下exactly Once就已经足够了。但是对于大流量的场景下，一次checkpoint可能消耗很长的时间。业务是否能否容忍这种数据延迟。如果容忍不了就需要用at Least Once. 当然在存储的时候选择RocksDBStateBackend做增量也是很好的选择。
	
	
	
```



```

我的使用经验：	
	之前在使用storm的时候，我们使用了很粗暴的方法，每次重启的时候我们都会回刷数据都0点，因为内存里面的数据都没有了需要重新建立。但是有了flink之后一切变得都不一样了。
	
	理想状态下，checkpoint可以代替savepoint. 比如5mins执行一次checkpoint, 当系统重启的时候，我从最近一次checkpoint来重启和系统关闭的时候执行savepoint的点重启是没有太大差别的。但是前提是你的系统支持幂等。(比如sink阶段插入是支持幂等的。)
	
	现实中的情况下，我们一般还是愿意在系统kill的时候先执行一次savepoint,重启的时候直接从savepoint的位点进行重启。这个时候不需要考虑下游是否幂等。因为重启之后系统的数据还是重启之前的，source的offset还是重启之前的。(需要关心一下如果修改并发度的情况)
	
	但是当系统出现故障的时候，我们没办法savepoint, 只能通过最近一次的checkpoint恢复。这个时候就需要考虑幂等了(包括sink和中间读写缓存部分)。这个地方要注意一下。
	
	flink的配置：
	env.enableCheckpointing(2 * 60 * 1000L, CheckpointingMode.EXACTLY_ONCE); 
	//一般场景下都是at_least_once，做好幂等。checkpoint根据你你的job大小。如果job很大，并且很重要，你可以配置的间隔时间相对短一点，
	env.getCheckpointConfig().setCheckpointTimeout(2 * 60 * 1000L);
	env.getCheckpointConfig().setMinPauseBetweenCheckpoints(60 * 1000L);
	minPauseBetweenCheckpoints用于指定checkpoint coordinator上一个checkpoint完成之后最小等多久可以出发另一个checkpoint，当指定这个参数时，maxConcurrentCheckpoints的值为1

	存储可以使用RocksDBStateBackend(支持增量，针对大状态，大窗口使用)， FsStateBackend

```



## 异步IO

```
与第三方数据(rpc, rest, dataSource)打交道，是实时任务经常需要做的事情。
与第三方数据交互，一直整个实时任务的瓶颈点。在之前storm的时候我们经常做的比较简单，
	1. 直接访问rpc,datasource
	2. 做的好点的在rpc，dataSource加上限流
	3. 对于io延迟敏感和第三方数据量较大的应用，一般还要做三级缓存(数据写入redis, 本地做localLruCache))
	4. 对于一些数据量不大的，可以直接数据预加载到本地缓存，然后起一个定时任务定时更新。
不过以上的查询基本还是block-IO, flink的异步io的出现把之前的单线程改成多线程异步访问。
异步IO要求访问第三方应用可以异步，需要第三方应用支持异步IO，我们使用的场景是使用线程池，异步回调方式来做。
异步IO的两种形式，
	orderedWait 消息的发送顺序和watermark的顺序，底层把异步事件的promise存储到有序/无序的队列中，并且保存到ListState,可以从checkpoint恢复，继续与第三方系统交互。
	unorderedWait，在eventTime下的无序，只是相对于同一个watermark之间无序，两个先后的watermark还是有序的。

```



## Exactly Once & At Least Once

```
flink内存的exactly Once
	flink通过checkpoint保证了flink内部的exactly-once. checkpoint保存了flink内部状态和数据输入流的位置。并且可以异步存储到hdfs中。
	
flink端到端的exactly Once
	TwoPhaseCommitSinkFunction, 需要外部数据存储支持两阶段提交的预提交和回滚。

```



#### FlinkKafkaConsumer010

```
接到上面的checkpoint部分，这里说一下这个类FlinkKafkaConsumer010，因为我们使用的是kafka，所以避免不了使用这个类。


前提是你使用了checkpoint保存状态，如果不使用，只当我没说。

source部分的状态保存是一个挺麻烦的事情。对于kafka来讲，offset机制有两种1：自动提交，2:手动提交。 如果我们使用自动提交，一旦涉及到系统重启，如果不进行改动，使用savepoint是完全没有问题的，但是如果有改动，或者系统出现故障重启的时候，offset已经提交，我们必须手动的回溯offset, 但是回溯到哪个点，这个时候就搞不定了(除非不使用offset，直接暴力设置offset到0点)。
在这种情况下只能使用手动提交offset的方式， 手动提交offset 又需要保存state,这个时候就需要FlinkKafkaConsumer010， 使用这个类了之后，也不需要回溯offset了(除非你重启的时候不指定offset)。

所以我的经验是：

kafka的配置：
	"auto.offset.reset" : "latest"
	enable.auto.commit  : false  //只要指定checkpoint这个默认是false
	而且不需要指定key，value的序列化方式，系统会使用默认的 ByteArrayDeserializer
	kafka的offset在重启的时候从state获取，每次checkpoint的时候都会把当前的offset提交到kafka。并且更新state.
		
		
		
```



#### FlinkKafkaProducer011

```
flinkKafkaProducer011提供了exactly-once 两阶段提交的具体实现。11版本之前提供的是at-least-once. 
flinkKafkaProducer011 实现了TwoPhaseCommitSinkFunction。主要借助了kafka-client的事务机制来实现。
FlinkKafkaConsumer010提供了at-least-once的具体实现，
setLogFailuresOnly(false) // 设置为false，如果数据写入报错，直接往外抛出错误，而不是记录/捕获。
setFlushOnCheckpoint(true) // checkpoint会等待在checkpoint成功之前被Kafka识别的时间内传输的记录，这就保证了所有checkpoint之前的记录都被写入Kafka 中

两阶段提交的四个步骤。
beginTransaction
	在开始事务之前，我们创建一个临时文件，将记录写入到临时文件中。
	通过initializeState来调用开始事务。
preCommit
	在预提交阶段，我们刷新之前的文件，关闭文件，不再有新的写入，并且增加新的文件来来启动新的事务。FlinkKafakProducer011是调用snapshotState的时候。当所有的barrier事件进入sink的时机。
commit
	将预提交的文件原子的转移到目标目录。(有一定的延迟行), 所以需要增量进行checkpoint,并且根据业务的需要，确定两次checkpoint之间的间隔。
	
abort
	删除临时文件。



```



## 内存管理/序列化

```
我这边都是以flink-on-yarn上来说明：
在内存管理上1.10版本和1.10之前的版本有较大的差异。具体差异点可以在我的另外一篇文章中说明：
Flink总内存大致分为三块(因为这三块内存是最主要的，
1. jvm堆内存(Heap Memory)
	用户代码和taskManager的数据结构来使用。
2. 托管内存(Managed Memory)
	MemorySegment组成的超大集合，Flink 中的算法（如 sort/shuffle/join）会向这个内存池申请 MemorySegment，将序列化后的数据存于其中。
	在stream流处理的时候managed Memory内存一般是不需要的)
3. 网络缓存(net buffers) 
	一定数据量的32kb的buffer，主要用于网络传输，如果显示配置：taskmanager.network.numberOfBuffers 默认2048个(64MB)。 否则需要 min[1024, max(64, maxTotalMomery * 0.1)] 

flink定制了自己的序列化框架。flink处理的数据流通常为一种类型，所以可以只保存一份对象schema，具体数据可以通过偏移量存取。


1. flink使用了更多的堆外内存存储数据。堆外内存可以存储更多的数据，进行高效的IO操作，避免频繁GC,堆外内存在写磁盘和网络传输的时候可以实现零复制，不通过cpu计算。
2. 
```



## 监控调优



```
1. 上游延迟情况
    source阶段：可以配置kafka的堆积量，以及kafka日志的事件事件作为监控指标。
    process阶段，可以根据每个operator输入输出量来判断。
    警惕kafka某一个partition分区延迟过多的问题，需要上游进行修改。
2. 数据倾斜问题。
	 从业务判断是否存储热点key。如果需要就需要进行打散。
3. 判断压力的瓶颈点
	结合operator输入输出用量，以及输入输出队列长度来判断。
	处理逻辑是否过于复杂，判断逻辑是否有外部依赖。就要对并发度进行调整。
	case1: 遇到过在处理逻辑时候用到aviator表达式来做匹配的时候由于没有加缓存导致。
	case2: 调度外部存储，rt过高，这个时候考虑是否做二级或者三级缓存。
4. 任务jvm使用，以及fgc的情况
	1. 一般都是在内存中存储中间数据导致的。flink可以考虑在存储state的时候存储中间结果数据，不要存储原始数据。
	2. 如果需要存储大量中间数据的场景，考虑调整jvm的堆内存大小。
	
```