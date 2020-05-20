## V1.10之前的版本

```
整个内存包含四个部分：
1. container cut-off 
		RocksDB、JVM 内部开销
2. task能用的memory
	  jvm-heap
	  	usercode, functions
	  managed memory on-heap
	  	内部算子的逻辑内存分配。可以在堆内，堆外同事分配。
	  managed momory off-heap
	  	排序，hashing分区，缓存等，
	  network buffers
	  	用户网络传输及其(shuffle, 广播等)。分布在堆外内存-off-heap
	  	
	  参考链接：https://ci.apache.org/projects/flink/flink-docs-release-1.9/ops/mem_setup.html#total-memory
```

在1.10之前的版本 我们一般只需要设置taskManager的总内存大小(-ytm/--yarntaskManagerMemory),也就是总的container大小就可以了。其他的配置都可以根据默认值进行推导。

![image-20200519182631743](/Users/beifang/Library/Application Support/typora-user-images/image-20200519182631743.png)

```
ag:举一个例子，我们设置-ytm=2048M. 

tm实际能用的内存
    tm实际能用的内存(包含堆内，堆外) = taskmanager.heap.size - max[containerized.heap-cutoff-min, taskmanager.heap.size * containerized.heap-cutoff-ratio] 
    2048 - max[600, 2048 * 0.25] = 1448M

网络缓存的大小 
	如果在配置文件中不显示配置（taskmanager.network.numberOfBuffers）：
	network_buffer_memory = min[taskmanager.network.memory.max, max(taskmanager.network.memory.min, tm_total_memory * taskmanager.network.memory.fraction)]
	network_buffer_memory = min[1024, max(64, 1448 * 0.1)] = 144.8M
  如果显示配置(taskmanager.network.numberOfBuffers)：
  network_buffer_memory = taskmanager.network.numberOfBuffers * taskmanager.memory.segment-size 
  network_buffer_memory = 2048 * 32KB = 64MB


所以如果网络缓存显示配置， taskManager使用的堆内存(-Xms/-Xmx)= 1448 - 144.8 = 1304M.（所以在taskManager下的配置参数如下：-Xms1304m -Xmx1304m -XX:MaxDirectMemorySize=744m）

堆内托管内存(flink-ui上面显示的大小)：flink_managed_memory = tm_heap_memory * taskmanager.memory.fraction = 1304 * 0.7 = 912M



```



## V1.10+版本

1. 新版本已经没有了堆上的（On-Heap）*托管内存*, 托管内存只在off-heap. 不再使用直接内存。不再受jvm的管理。
2. 不在支持托管内存的预分配，托管内存都是惰性分配的。
3. cut-off容器内存已经失效。流处理中的rockDB部分已经归为托管内存(managed momory)中。



![image-20200519191614356](/Users/beifang/Library/Application Support/typora-user-images/image-20200519191614356.png)

```
设置的方式： 
1 taskmanager.memory.process.size = container总大小（-ytm），可以继承之前的版本。
2. 如果设置了taskmanager.memory.flink.size 就会少了jvm metaspace和jvm overhead的部分。
3. 指定 taskmanager.memory.task.heap.size 和 taskmanager.memory.managed.size，即直接指定了堆内存和Flink Managed内存的大小
三者可以选择其中之一。

如果设置按照1来进行

ag: -ytm = 2048M.
jvmMetaspaceSize = taskmanager.memory.jvm-metaspace.size 256m(默认值)
jvmOverheadSize = max[192M, mix[1g, 2048 * 0.1 / 1 - 0.1]] = 227M
totalFlinkMemorySize =  -ytm  - jvmMetaspaceSize - jvmOverheadSize = 2048 - 256 - 227 = 1562M
1. 如果配置 taskmanager.memory.task.heap.size, networkMemorySize的大小就通过totalFlinkMemorySize - managedMemorySize - taskOffHeapMemorySize得到
2. 如果不配置 taskmanager.memory.task.heap.size 就使用networkMemorySize的默认值64M,来计算taskmanager.memory.task.heap.size。

如果设置按照2来进行。
总的container的大小根据 jvmMetaspaceSize + jvmOverheadSize + taskmanager.memory.flink.size的大小来确定。


启动参数：
1. -Xmx -Xms    framework heap + task heap
2. -XX:MaxDirectMemorySize 框架堆外内存 + 任务堆外内存+ 网络内存
3. -XX:MaxMetaspaceSize JVM Metaspace

我的一些建议：
如果是针对stream任务，flinkManaged并没有用，可以设置最小值(这个没有尝试过，文档上可以设置为0)。
如果是旧版本升级过来的，可以考虑设置taskmanager.memory.process.size(-ytm), 并且设置jvmMetaspaceSize + jvmOverheadSize+taskmanager.memory.task.heap.size的大小。
新版本的可以直接考虑taskmanager.memory.flink.size+jvmMetaspaceSize + jvmOverheadSize+ taskmanager.memory.task.heap.size(更加清晰，方便内存计算和管理)



参考链接： https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/ops/memory/mem_detail.html
```

