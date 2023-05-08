# Apache Flink Stream and Batch Processing in a Single Engine
## 1. Introduction 
流处理系统和批处理系统在传统上通常被认为是两种非常不同类型的应用。它们使用不同的编程模型和 API 进行编程，并由不同的系统执行。下表列举了流处理和批处理的特征以及代表性的数据处理引擎。
| 数据处理类型 | 特征 | 数据处理引擎 | 
| --- | --- | --- |
| 批处理 | 输入是有限的数据集合；数据的收集和处理分阶段进行，数据收集完成后才进行数据处理；数据处理结果在整个分析处理阶段结束才能获取，延迟较高 | Apache Hadoop |
| 流处理 | 输入是连续不断的数据流；数据的收集和处理同时进行，输入的数据会被实时处理；数据处理结果立刻可用，并会随着新数据抵达继续更新，延迟较低 | Apache Storm、IBM Infosphere Streams、Apache Samza |

批处理系统忽略了数据生产的连续性和实时性，通常会将数据记录分批纳入静态的数据集（比如每小时、每天或每月的数据分块），然后以一种时间无关的方式进行数据处理。然而大量的数据实际上是随着时间不断产生的（比如网络日志、应用日志和传感器），建立连续的数据处理管道能够更好的体现数据的实时趋势。

Lambda 架构模式尝试在批处理系统旁边运行流处理系统，实现不同的计算路径：首先借助流处理系统及时获得低延迟、不准确的近似结果，批处理离线路径用于获取后期的精确结果，然后修正流处理系统的计算结果。这种融合的尝试没有解决批处理的高延迟、协调多个系统、合并计算结果等复杂的问题。

Apache Flink 在编程模型和执行引擎中都将数据流处理作为实时分析、连续流和批处理的统一模型。结合允许对数据流进行任意重放的消息队列，流处理程序在处理最新消息、在大窗口中定期连续聚合数据和处理历史数据时没有区别。这些不同类型的计算只是在数据流的不同起点开始进行处理，并在计算过程中保持不同形式的状态。同时，Flink 承认现在和将来都仍然需要通过批处理的方式处理静态数据。在实现上 Flink 将批处理作为流处理的一种特殊情况——处理的数据流是有限的，数据记录的顺序和时间并不重要。为了更好的支持批处理，Flink 还实现了定制的 API 来处理静态数据，并且为批处理定制了专门的调度策略。

本文的后续章节分别介绍以下内容：
- Flink 的软件架构和各个组件的职责（第2节）
- Flink 提出的统一流处理和批处理数据的结构（第3节）
- Flink 如何在数据流之上建立具有灵活窗口机制的成熟的流处理和批处理系统（第4、5节）

## 2. Flink  Architecture 
Figure 1 展示了 Flink 的软件栈。Flink 的核心是分布式数据流引擎，它负责执行数据流程序。在引擎层之上是两个核心API：用于处理有限数据集的 DataSet API 和用于处理无界的数据流的 DataStream API。在 API 层以上又衍生出特定领域的库和 API，其中 Flink ML 用于机器学习，Gelly用于图处理，Table API 用于类 SQL 的相关操作。
Flink 的运行时程序是一个有状态运算符的 DAG，DataSet API 和 DataStream API 都能创建出可由数据流引擎执行的运行时程序。
![image](https://user-images.githubusercontent.com/54345716/236875603-dc7ca575-d83e-4610-a180-d4f457c87e92.png)

Figure 2 展示的是在一个完整的 Flink 集群中，各个组件协调处理作业的流程。
- Flink Client：负责获取程序代码，将其转换成数据流图，将作业提交给 JobManager
- JobManager：负责协调数据流的分布式执行，跟踪每个作业的状态和进度，将作业调度给 TaskManager 执行，并负责协调作业的检查点和错误恢复。在高可用的设置下，JobManager 会将每个检查点的元数据持久化到一个容错存储中，这样备用的 JobManager 可以重建检查点并从哪里恢复数据流的执行
- TaskManager：负责执行一个或多个作业，并且向 JobManager 报告任务的状态。TaskManager 会维护 buffer pools 来缓冲或者物化数据流，通过网络连接在各个算子之间交换数据
![image](https://user-images.githubusercontent.com/54345716/236875636-2fb4bbb1-1c80-4994-8a2a-e432dc07a827.png)

## 3. The Common Fabric: Streaming Dataflows
尽管 Flink 支持丰富的上层 API，但所有的 Flink 程序最终都会被编译成通用的表示形式 Dataflow Graphs，这也是Flink 能够融合批处理和流处理的基础。

### 3.1 Dataflow Graphs 
DataFlow Graph 是一个有向无环图，由以下两个部分组成：
- DataStream 是数据的载体，数据流以点对点、广播、分区、扇出和合并等多种模式在生产操作和消费操作的算子之间传输数据
- Stateful Operator 是有状态的算子（特殊情况下也有可能是无状态的），负责实现所有的处理逻辑。常见的算子包括 filters、hash joins、stream window functions 等，其中许多算子是众所周知的教科书版本的标准算法实现，而 stream window function 是 Flink 中定义的一种算子，在第4节会详细介绍该算子的实现细节。
![image](https://user-images.githubusercontent.com/54345716/236875799-56fd4555-cd62-4f46-b72a-3327a5d0ea96.png)

如图 Figure 3 所示，Flink 在 DataFlow Graph 的基础上进行了以下抽象和设计。
- 数据流能够被拆分成多个分区， 图中的源数据流被拆分成了 SRC1 和 SRC2两个分区，相应的算子同时就被拆分为多个并行化的 subtasks，每个 subtask 负责处理数据流的一个分区
- 数据流从源数据 SRC 到目标数据 SNK 之间会通过 Intermediate Data Streams 交换数据，比如 IS1 和 IS2 是 pipelined 执行模式下的数据交换中间结果，IS3 是在 blocking 执行模式下的数据交换中间结果（第3.2节讨论）
- 数据流在经过算子运算的过程中会传递不同类型的 Control Event ，控制事件到达时会执行对应的操作进行响应（第3.2节讨论）
- Operator State 对应一个并行的算子实例。比如使用 Kafka Connector 时，每个并行的 Kafka 消费组里保存的 topic partition 和 offsets 作为 Operator State（第4.3节讨论）

### 3.2 Data Exchange through Intermediate Data Streams
Flink 的中间数据流是算子之间进行数据交换核心抽象。中间数据流表示的是由一个算子生成的可以被其他算子消费的中间数据的逻辑表示——他们是逻辑上的一种表示，可能不会被持久化到磁盘上。

**Pipelined and Blocking Data Exchange**
Pipelined stream 在同时运行的生产者和消费者之间交换数据，这就导致了数据流会从消费者向生产者传递 back pressure，通过中间缓冲池做一些调整来补偿短期的吞吐量波动。Flink 将 pipelined stream 用于连续流程序和批处理流的部分环节。
back pressure 是流处理系统中关于处理能力的动态反馈机制。当 DAG 中的某些算子无法按照接收数据记录的速度处理记录时就发发生 back pressure，多余的输入数据会先填充此慢算子的输入缓冲区。一旦输入缓冲区被填满，back pressure 就会传递到上游子任务的输出缓冲区。一旦上游任务的输出缓冲区被填满，上游也被迫减慢其处理的处理速度，依次类推直到源算子。
Flink 1.13 及以后，能够直接从Web UI中的作业图观察到 back pressure。
![image](https://user-images.githubusercontent.com/54345716/236875948-73875ee3-3b5e-4e23-a931-6dba2b61fe62.png)

Blocking stream 适用于有界数据流，它在被消费之前会将生产者的所有数据进行缓冲，从而将生产和消费算子分离到不同的阶段。blocking stream 自然需要更多的内存，经常需要溢出到二级存储，并且不传递 back pressure。它们可以用来隔离连续的算子。

**Balancing Latency and Throughput**
Flink 的数据交换机制是建立在交换 buffer records 的基础上。当数据记录在生产者一侧准备就绪时，会被序列化并分成一个或者多个buffer records，然后转发给下游的消费者。这样的设计使得 Flink 能够通过调整 buffer 的大小来调节吞吐量和延迟——将 buffer 设置为高值来实现高吞吐量，通过将 buffer 设置为低值来实现低延迟。
实际上 Flink 的算子在满足以下两个条件之一时就会向下游发送数据。
- 当输出 buffer 的空间占满时，将数据 flush 到下游
- 通过 buffer timeout 配置，每隔 buffer timeout 时间强制将 buffer 数据 flush 到下游
Figure 4 展示了buffer timeout 对30台机器上简单的流式处理作业中交付记录的吞吐量和延迟的影响。可以观察到当99分位延迟为20ms 的情况下，集群能够保证150万/秒个事件的吞吐量；当99分位延迟为50ms时，集群能够保证800多万/秒个事件的吞吐量。
![image](https://user-images.githubusercontent.com/54345716/236876003-1a4ce0a0-3dc4-4acc-871f-6a5e5b891634.png)

**Control Events**
Flink 中的流除了交换数据还可以传递不同类型的控制事件，这些控制事件是由算子在数据流中注入的特殊事件，它们会随着其他控制事件一起按顺序传递。Flink 使用很多特殊类型的控制事件，包括：
- Checkpoint barriers 将流划分为 pre-checkpointing 和 post-checkpointing 阶段来协调检查点（第3.3节讨论）
- Watermark 标志着流分区内事件的进展（第4.2节讨论）
- Iteration barriers 标志着流分区已经到达了步骤的终点（第5.3节讨论）

### 3.3 Fault Tolerance
Flink 通过 checkpointing 和 steam replay 来处理故障，提供了可靠的执行环境和 exactly-once 的一致性保证。为了有效提供这些保证，需要数据源是持久的和可重放的（比如消息队列）。在实践中，非持久的数据源可以通过在数据源算子的状态中保持一个 write-ahead 日志来提供重放的能力。
Flink 的检查点机制是建立在分布式一致性快照的概念上的。由于数据流可能是无界的，对于一个长期运行的作业来说完全重复计算需要重做所有的计算，显然这是不切实际的。Flink 会定期对算子的状态进行快照，包括每隔一段时间对输入流的当前位置进行检查。
引入 checkpoints 以后，故障中恢复只需要将所有算子的状态恢复到最后一次成功的快照中的各自状态，从有快照的最新屏障开始重新启动输入流。这样故障恢复的最大重新计算量必定小于两个结果障碍之间的数据输入量。如下图所示，定期做 checkpoints 以后在故障恢复时 Flink 直接从 checkpoint n 开始重放，不再需要关注更老的数据记录。  
![image](https://user-images.githubusercontent.com/54345716/236876603-f95decb0-a72c-4164-b42e-43549cd40b99.png)
对所有并行运行的算子进行一致性快照的核心挑战在于不能停止程序的执行。从本质上来讲，所有算子的快照应该是指计算中的同一逻辑时间，Flink中使用的机制被称为异步障碍物快照（Asynchronous Barrier Snapshotting）。
![image](https://user-images.githubusercontent.com/54345716/236876906-ecf4cbf5-f485-41fa-9a1d-62677cc7bb4a.png)

3.4 Iterative Dataflows 
Flink 的 Dataflow Graph 模型同样能够支持迭代计算。在数据并行处理系统中对迭代的支持通常是通过为每个迭代提交一个新的作业或者向运行中的DAG添加额外的节点或者反馈边，重复地执行执行函数直到达到某个终止条件。Flink 中的迭代实现是 iteration steps——这些特殊的算子本身包含一个 DAG 执行图，如 Figure 6 所示。在第4.5节和第5.3节会分别介绍流处理系统和批处理系统分别是如何实现迭代计算的。
![image](https://user-images.githubusercontent.com/54345716/236876934-369f3fac-cd28-492a-a681-7424317a20cd.png)

迭代计算中比较重要的概念包括：
Iteration Head：获取数据源或者算子上一次迭代的输出结果
Iteration Step：在每次迭代过程中执行，是包含了各种算子的数据流
Feedback stream：在每一次迭代中，输出都会反馈到下一次迭代中
Iteration Tail：单次迭代的输出结果
有多个条件可以指定迭代的终止条件：
- 最大迭代次数：如果没有任何进一步的条件，迭代计算达到最大迭代次数后停止
- 自定义聚合器收敛：允许自定义聚合器和收敛标准，如 sum 聚合器可以定义参与计算的记录数量或者累加等于0的时候退出终止迭代

## 4. Stream Analytics on Top of Dataflows 
Flink 的 DataStream API 实现了完整的流处理分析框架，包括时间的定义、无序事件的处理、定义窗口、维护和更新用户定义的状态等特性。
### 4.1 The Notion of Time
Flink 中定义了多种时间的类型，时间类型的定义和特点参见下表：
| 时间类型 | 具体描述 | 特点 |
| --- | --- | --- |
| Event Time |指的是事件发生的时间，一旦确定以后再也不会改变。例如，事件被记录在日志文件中，日志中记录的时间戳就是时间事件| 不依赖操作系统的时钟，无论执行多少次可以保证计算结果相同 |
| Processing Time | 指的是事件被计算流引擎处理的时间，以计算节点的本地时间为准 | 依赖操作系统的时钟，重复执行结果可能是不同的 |
| Ingestion Time |指的是事件进入流处理引擎的时间 | 处理机制上类似于 Event Time |

Flink 中实际使用的比较多的是 Event Time 和 Processing Time，Ingestion Time 一般使用的比较少，当数据记录中没有时间记录，又想使用 Event Time 的机制来处理数据，可以考虑选择使用 Ingestion Time 作为替代。
从流处理的原始设备产生事件开始，到 Flink 读取到数据，再到 Flink 多个算子处理数据，这个过程会导致事件发生的时间 Event Time 到时间被流引擎处理的时间 Processing Time 之间存在一定程度的延迟。实际情况中流处理过程中还会受到网络延迟、数据乱序、反压、Failover等多种情况的影响，输入数据是乱序的。考虑此类异常情况，为了保证计算结果的正确性就需要等待数据，而这样势必会带来计算的延迟进一步增大。对于延迟太久的数据，我们又不能无限期的等下去，所以 Flink 引入了 watermark 的机制来保证特定时间后一定会触发窗口进行计算。

### 4.2 Watermark
Flink 作为流处理引擎并不感知数据的生产情况，通过 watermark 来断言所有的数据已经到达，不再等待更早的数据。以时间进度为例，具有时间属性 t 的 watermark，表示所有低于 t 的事件都已经进入了算子，生成 watermark 时引擎不再等待数据并触发一次计算。
如果数据流都是有序的，watermark 就是一个简单的周期性标记，并不能发挥额外的作用。
![image](https://user-images.githubusercontent.com/54345716/236877336-8caeee5f-c657-4114-a87d-cb4f4c35625b.png)
如果数据流是无序的，watermark 告诉算子比 watermark 更早的事件都已经到达，算子可以将内部事件提前到watermark 的时间戳。比如下图的例子中数据 19 出现在 w(20) 后才出现，可以根据引擎的策略决定对数据的具体处理方式 。
![image](https://user-images.githubusercontent.com/54345716/236877350-d6925590-b07d-4a47-aad4-7366ed74ed98.png)
比 watermark 更晚的出现的数据一般有以下三种处理方式：
- 直接将延迟的数据丢弃（Flink 的默认处理方式）
- 通过 allowedLateness 指定允许数据的最大延迟时间，Fllink 会在窗口关闭后一直保存窗口的状态，期间迟到的事件不会被丢弃，而是默认会触发窗口的重新计算。
- 通过 sideOutputLateData 收集延迟的数据，统一存储方便后期排查问题 

**When Generating Watermark ?**
DataStream 和 Flink Table & SQL 模块中，使用了各自的 watermark 生成体系。
DataStream 中生成 watermark 相对更加灵活，通常在 Source Function 中生成 watermark，直接为数据元素分配时间戳，同时向下游发送 watermark。Flink 提供了额外的机制，允许用户调用 DataStream API 根据业务逻辑的需要，使用时间戳和 watermark 生成器修改数据记录的时间戳和 watermark。
Flink Table & SQL 中的 watermark 的生成策略包括以下三类：
策略类型
详细描述
周期性 watermark 策略
周期性地产生一个 watermark，间隔一定的时间或者达到一定的记录条数
每事件 watermark 策略
为每一个递增的 EventTime 产生一个 watermark，使用这种方式会生成大量的 watermark，只有在实时性要求非常高的场景才会使用
无为策略
Flink SQL 中不设定 watermark 策略，使用底层的 DataStream 中的 watermark 策略
### 4.3 Stateful Stream Processing
**What is State ?**
对于流处理系统而言，事件持续不断地产生，如果每次计算都是相互独立的，不依赖于上下游的事件，则是无状态的计算。如果计算需要依赖之前或后续的事件，则是有状态的计算。Flink 的算子都提供了对状态 State 的支持。
举几个例子方便理解 Flink 中状态的作用，比如涉及到求和的场景如果重复计算需要依赖所有的事件，如果记录当前所有值的总和，就只需要在中间结果的基础上直接求和即可。 比如在上游数据源是消息队列的场景中，通过记录队列的消费 offset 作为状态能够去掉数据流中的重复数据，避免重复处理。
按照数据结构的不同，Flink中定义了多种类型的 State 用于不同的场景，具体如下：
按是否有 Key 划分
具体类型
详细描述



Keyed State
ValueState
类型为 T 的单值状态，这个状态与对应的 Key 绑定

ListState
类型为列表的状态

ReducingState
通过用户传入的reduceFunction，对添加的数据进行计算，得到一个单一的状态值

AggregatingState
类型为聚合状态

MapState
类型为 Map 的状态

FoldingState
跟ReducingState有点类似，不过它的状态值类型可以与传入的元素类型不同 
Operator State
ListState
类型为列表的状态
Keyed State 在 KeyedStream 中使用，状态和特定的 Key 进行绑定，即 KeyedStream 流上的每一个 Key 对应一个 State 对象；Operator State 跟一个特定算子的一个实例绑定，整个算子只对应一个 State。

Using State
在 Flink 中使用状态，有两种典型的场景：
- 状态操作：使用状态对象本身存储、写入和更新数据
- 状态访问：从状态存储中获取状态对象本身
状态操作面向应用开发者只提供了对 State 的数据进行添加、更新和删除等基本的操作接口，面向 Flink 框架除了能够对 State 的数据进行操作，还提供了内部的运行时信息，比如 State 中数据的序列化器、命名空间合并等接口。
状态访问面向应用开发者可以从自定义的 UDF 中访问到相关的状态信息，面向 Flink 框架能够读取对应的状态信息在故障恢复时发挥重要的作用。

State Backends
State 需要被持久化到可靠存储中，才能让服务具备应用级的容错能力，Flink 的状态都存储在 StateBackend 中。根据使用场景的不同，Flink 内置了以下三种 StateBackend。
[图片]
名称
Working State
状态备份
快照
特征
RocksDBStateBackend
RocksDB 磁盘
分布式文件系统
全量/增量
- 支持大于内存大小的状态
- 比基于JVM Heap的StateBackend慢
FsStateBackend
JVM Heap
分布式文件系统
全量
- 快速，需要大的堆内存
- 受限于 GC
MemoryStateBackend
JVM Heap
JobManager JVM Heap
全量
- 适用于小状态的（本地）测试和实验
当使用基于 JVM Heap 的 StateBackend 访问和更新对象直接堆上进行，而对于文件型的 RocksDBStateBackend 访问和更新涉及到序列化和反序列化，所以会有更大的开销。但使用 RocksDBStateBackend 能够克服状态存储受TaskManager 堆内存大小限制的问题，同时又能够支持增量的持久化数据，比较适合再生产中使用。
### 4.4 Stream Windows
Stream Windows 是 Flink 定义的一个有状态的算子。它是处理无界流数据的核心抽象，通过将无限的数据流分解成若干个窗口，然后在这些有界的窗口上进行计算。
Windows 由以下三个核心功能组成：
组件名称
功能描述
分配器（window assigner）
决定将数据记录分配给哪个逻辑窗口，根据分配策略的不同将窗口分为CountWindow、TimeWindow、SessionWindow
触发器（trigger）
决定何时执行窗口上定义的关联操作，当有数据加入时调用触发器判断是否需要触发计算，触发的结果包括CONTINUE、FIRE、PUGE、FIRE_AND_PURGE
驱逐器（evictor）
负责决定每个窗口中保留哪些记录，包括 CountEvictor、DeltaEvictor、TimeEvictor
下面这个例子定义了范围为6秒的时间窗口，每隔2秒滑动一次（即分配器），一旦 watermark 通过窗口的末端（即触发器）就会触发窗口计算，执行窗口函数。
```SQL
stream
 .window(SlidingTimeWindows.of(Time.of(6, SECONDS), Time.of(2, SECONDS))
 .trigger(EventTimeTrigger.create()) 
```
下面这个例子定义了一个全局窗口（即分配器），每次计数到1000个事件触发窗口计算（即触发器），计算时从窗口头部开始丢弃数据仅保留最后100个元素（即驱逐器）。
```SQL
stream 
 .window(GlobalWindow.create()) 
 .trigger(Count.of(1000)) .evict(Count.of(100)) 
```
对窗口中的数据进行计算的函数称为 window 函数，窗口函数通常是聚合、累加等常规的计算函数。Flink 对一些聚合类（如 sum 和 min）的窗口计算做了优化，通过在内存中保存中间结果值，每次有新元素进入窗口时直接在中间结果值的基础上进行修改，但如果用户定义了驱逐器 Evictor，则不会启用对聚合窗口的优化。
### 4.5 Asynchronous Stream Iterations 
数据流的循环对一些应用来说是必不可少的，如构建机器学习模型、强化学习等。在大部分这种情况下，反馈的循环不需要额外的协调，少部分情况需要异步迭代的机制来处理并行优化问题。      
Flink 的执行模型目前已经支持了异步流迭代，DataStream API 允许明确地定义反馈流，并且能够定制化地对流的循环提供支持以及进度跟踪。
## 5. Batch Analytics on Top of Dataflows 
Flink 支持批处理的方式是将批处理输入的有界数据集作为无界数据流的一个特例——将有界的数据集视为一个窗口内输入的所有数据流构建处理程序，批处理可以完全被上文介绍的功能所覆盖。Flink 进行了以下适配来执行批处理：
- 提供 DataSet API 专门处理批处理，并且提供查询优化层将 DataSet 程序转化成高效的可执行程序
- 批量计算是与流计算相同的运行时执行的。运行时通过 Blocking Data Exchange 的方式将计算分解为相对独立的几个阶段
- 批处理的故障恢复不使用 checkpointing，由于输入流是有界的数据集当出现故障时可以直接从物化的数据流或者最新的流分区进行重放。这样适配会让整个计算过程在出现异常在流重放上消耗更多，但同时会让计算在正常运行时花费更少（不进行周期性的checkpointing）
- Blocking Operators 只是算子层面的实现，他们会保持阻塞状态直到消费所有的输入数据，程序运行时并不知道算子是否被阻塞了。这些算子能够利用 Flink 提供的管理内存缓冲数据，当输入超过内存界限时会溢出到磁盘
### 5.1 Query Optimization
Flink 的批处理优化器建立在并行数据库系统的技术之上，比如 plan equivalence, cost modeling and interesting-property propagation。由于构建 Flink 数据流程序中包含用户定义的函数（UDF），导致算子的运算复杂度难以评估，因此 Flink 的查询优化无法直接应用传统的优化技术。
目前，Flink 的运行时支持多种执行策略，包括重新分区和广播数据传输，以及基于排序的分组和哈希链接的实现。Flink 的优化器根据 interesting properties propagation 的概念枚举多个物理执行计划，基于 CBO 的方法在多个执行计划中进行选择。考量的执行成本包括网络、磁盘 I/O 和 CPU 的使用成本。为了克服 UDF 引起的基数耗时难以预估准确的问题，Flink 优化器还可以支持通过 hint 的方式执行程序员指定的执行方式。
### 5.2 Memory Management
Flink 的内存管理构建在数据库技术的基础上，它通过将数据序列化为内存段，而不是在JVM堆上表示 buffer records。诸如 sort 或 join 算子可以直接在二进制数据上进行操作，从而降低序列化和反序列化的开销。
为了处理任意类型的对象，Flink 引入了类型推断和自定义序列化的机制。通过使用二进制数据、堆外存储尝试减少GC带来的开销。
### 5.3 Batch Iterations
迭代图分析、并行梯度下降和优化技术过去一直是在整体同步并行模型（Bulk Synchronous Parallel ）和同步并行模型（Stale Synchronous Parallel ）上实现的。Flink 通过实现 iteration-control events 能够在上边任意一种类型的结构上迭代逻辑，比如在 BSP 执行模型中 iteration-control events 标志着迭代计算的 Super Step 步骤开始和结束。另外，Flink 还引入了更多新的优化技术，比如可以利用稀疏的计算的delta iterations概念——目前已经被应用在了Flink’s Graph API上。
整体同步并行模型：引入Super Step的概念，每一个Super Step均代表 BSP 模型中一次完整的并行计算过程，整个运算过程包含若干个串行超步，超步包含本地计算过程、运算节点间通讯过程和同步过程。超步中的三个阶段是严格串行的，即所有处理机本地计算结束后统一进行通讯过程，最后执行同步阶段。除此之外，每个运算单元在一个超步内只能传递或接收一次数据。
[图片]

## Reference
* https://nightlies.apache.org/flink/flink-docs-release-1.16/zh/docs/concepts/stateful-stream-processing/
* https://nightlies.apache.org/flink/flink-docs-master/docs/ops/state/checkpoints/
* https://nightlies.apache.org/flink/flink-docs-release-1.16/zh/docs/dev/datastream/fault-tolerance/checkpointing/
* https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/windows/
* https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/concepts/time_attributes/
* https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/event-time/generating_watermarks/
* https://nightlies.apache.org/flink/flink-docs-master/docs/dev/dataset/iterations/
* https://nightlies.apache.org/flink/flink-docs-master/docs/dev/dataset/overview/
* https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/
* 《Flink 内核原理与实现》
