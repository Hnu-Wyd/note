- [Flink简介](#flink简介)
- [Flink的特点](#flink的特点)
- [Yarn模式](#yarn模式)
  - [Session-Cluster模式](#session-cluster模式)
  - [Pre-Job-Cluster模式](#pre-job-cluster模式)
- [Flink的运行架构](#flink的运行架构)
  - [作业管理器](#作业管理器)
  - [任务管理器](#任务管理器)
  - [资源管理器](#资源管理器)
  - [分发器](#分发器)
- [程序与数据流](#程序与数据流)
- [执行图](#执行图)
- [任务链](#任务链)
- [Window](#window)
  - [窗口的类型](#窗口的类型)
  - [滚动窗口（Tumbling Windows）](#滚动窗口tumbling-windows)
  - [滑动窗口（Sliding Windows）](#滑动窗口sliding-windows)
  - [会话窗口（Session Windows）](#会话窗口session-windows)
  - [Window API](#window-api)
  - [创建不同的时间窗口API](#创建不同的时间窗口api)
  - [创建不同的计数窗口API](#创建不同的计数窗口api)
  - [窗口函数](#窗口函数)
- [时间语义](#时间语义)
  - [使用哪种时间语义](#使用哪种时间语义)
  - [设置时间语义](#设置时间语义)
- [水位线](#水位线)
  - [数据乱序](#数据乱序)
  - [水位线](#水位线-1)
  - [水位线的特点](#水位线的特点)
- [Flink中的状态](#flink中的状态)
  - [算子状态（Operator State）](#算子状态operator-state)
  - [键控状态（Keyed State）](#键控状态keyed-state)
  - [状态后端（State Backends）](#状态后端state-backends)

### Flink简介
Flink是为分布式、高性能，随时可用以及准确的流处理应用打造的开源流处理框架。是一个框架和分布式的流处理引擎，对于有界和无界的数据流进行状态计算。


### Flink的特点
- 事件驱动型

    事件驱动型是一类具有状态的应用，它聪一个或多个事件流程中提取数据，并根据事件触发计算、状态更新或其他外部动作。比较典型的就是以Kafka为代表的消息队列几乎都是事件驱动型应用。与之不同的就是SparkStreaming微批次，如下图所示。
    ![picture 9](img/Flink/Spark_Streaming.png) 

    事件驱动型

    ![picture 10](img/Flink/Event_Driven.png)  

 - 流与批的世界观

    批处理的特点就是有界、持久、大量，非常适合访问全套记录才能完成计算工作，一般用于离线计算。

    流处理的特点是无解、实时无需针对整个数据集执行操作，而是对通过系统传输的每个数据项执行操作，一般用于实时计算。

    在Spark的世界观中，一切都是有批次组成的，离线数据就是一个大批次，而实时数据是由一个个无线的小批次组成的。

    在Flink的世界观中，一切都是由流组成的，离线数据是有界的流，而实时数据是一个没有界限的流，这就是所谓的有界流和无界流。

- 分层API

    Flink提供的最高层级的抽象是 SQL 。这一层抽象在语法与表达能力上与 Table API 类似，但是是以SQL查询表达式的形式表现程序。SQL抽象与Table API交互密切，同时SQL查询可以直接在Table API定义的表上执行。

    ![picture 11](img/Flink/Level_API.png)  


- 支持事件时间（event-time）和处理时间（processing-time）语义
- 精确一次（exactly-once）的状态一致性保证
- 低延迟，每秒处理数百万个事件，毫秒级延迟
- 与众多常用存储系统的连接
- 高可用，动态扩展，实现7*24小时全天候运行


### Yarn模式
Flink提供了两种在yarn上运行的模式，分别为Session-Cluster模式和Pre-Job-Cluster模式


#### Session-Cluster模式
Sesssion-Cluster模式需要先启动集群，然后再提交作业，接着会向yarn申请一块空间后，资源永远保持不变，如果资源满了，下一个作业就无法提交，只能等到yarn中的其中一个作业执行完成后，释放资源，下个作业才会正常提交。所有作业共享Dispatcher和ResourceManager。共享资源，适合小规模执行时间短的作业。在yarn中初始化一个flink集群，开辟指定的资源，以后提交任务都向这里提交。这个flink集群会常驻在yarn集群中，除非手工停止。
![picture 12](img/Flink/Session_Cluster.png)  


#### Pre-Job-Cluster模式
一个Job会对应一个集群，每提交一个作业会根据自身的情况，都会单独向yarn申请资源，直到作业执行完成，一个作业的失败与否并不会影响下一个作业的正常提交和运行。独享Dispatcher和ResourceManager，按需接受资源申请；适合规模大长时间运行的作业。每次提交都会创建一个新的flink集群，任务之间互相独立，互不影响，方便管理。任务执行完成之后创建的集群也会消失。
![picture 13](img/Flink/Pre_Job_Cluster.png)  



### Flink的运行架构

![picture 14](img/Flink/Flink_Struct.png)

任务提交的流程

![picture 15](img/Flink/Job_Sumbit_Process.png)  

当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。

Client 为提交 Job 的客户端，可以是运行在任何机器上（与 JobManager 环境连通即可）。提交 Job 后，Client 可以结束进程（Streaming的任务），也可以不结束并等待结果返回。

JobManager 主要负责调度 Job 并协调 Task 做 checkpoint，职责上很像 Storm 的 Nimbus。从 Client 处接收到 Job 和 JAR 包等资源后，会生成优化后的执行计划，并以 Task 的单元调度到各个 TaskManager 去执行。

TaskManager 在启动的时候就设置好了槽位数（Slot），每个 slot 能启动一个 Task，Task 为线程。从 JobManager 处接收需要部署的 Task，部署启动后，与自己的上游建立 Netty 连接，接收数据并处理。



#### 作业管理器
- 控制一个应用程序执行的主进程，也就是说，每个应用程序都会被一个不同的JobManger所控制。
- JobManager会先接受到要执行的应用程序，这个应用程序包含：作业图（JobGraph）、逻辑数据流图（logic dataflow graph）和打包了所有的类库以及其他的资源的jar包。
- JobManager会把JobGraph转成一个物理层面的数据流图，这个图叫做：执行图（ExecutionGraph），包含了所有可以并发执行的任务。
- JobManager会向资源管理器ResourceManger申请执行任务必要的资源，也就是任务管理器TaskManager上的插槽slot，一旦它获取到足够的资源，就会将执行图分发到真正运行它们的TaskManager上，在运行过程中，JobManager会负责所有需要的重要协调工作，比如检查点checkPoints的协调。


#### 任务管理器
- Flink中的所有工作进行，都在Flink中会有多个TaskManger运行，每一个TaskManger都会包含一定数量的插槽slots。插槽的数量限制了TaskManger能够执行的任务数量。
- 启动之后TaskManager会向资源管理注册它的插槽，收到资源管理器的指令后，TaskManager会将一个或者多个插槽提供给JobManager调用。JobManager就好像插槽分配任务执行。
- 在执行过程中，一个任务管理器可以跟其他运行在同一个应用程序的TaskManager交换数据。

#### 资源管理器
- 主要负责管理任务管理器的插槽资源。
- 当JobManage申请插槽资源时，资源管理器将有空闲的插槽分配给ResourceManager，如果没有足够的插槽来满足JobManager的请求，它会向资源提供平台发起会话，以提供启动TaskManager进程的容器。

#### 分发器
- 可以跨作业运行，它为应用提供了Rest接口。
- 当一个应用被提交执行时，分发器就会启动并将应用移交给一个JobManager。
- Dispatcher也会启动一个Web UI，用来方便地展示和监控作业执行的信息。
- Dispatcher在架构中可能并不是必需的，这取决于应用提交运行的方式。

### 程序与数据流

![picture 16](img/Flink/Data_Stream.png)  

- 所有的Flink程序都是由三部分组成的：  Source 、Transformation 和 Sink。
- Source 负责读取数据源，Transformation 利用各种算子进行处理加工，Sink 负责输出
- 在运行时，Flink上运行的程序会被映射成“逻辑数据流”（dataflows），它包含了这三部分。每一个dataflow以一个或多个sources开始以一个或多个sinks结束。dataflow类似于任意的有向无环图（DAG）。

### 执行图
![picture 17](img/Flink/Execution_Graph.png)  


Flink 中的执行图可以分成四层：StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图

- StreamGraph：是根据用户通过 Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。
- JobGraph：StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点
- ExecutionGraph：JobManager 根据 JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。
- 物理执行图：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。

### 任务链
![picture 18](img/Flink/Operator_Chain.png)  


Flink 采用了一种称为任务链的优化技术，可以在特定条件下减少本地通信的开销。为了满足任务链的要求，必须将两个或多个算子设为相同的并行度，并通过本地转发（local forward）的方式进行连接。

相同并行度的 one-to-one 操作，Flink 这样相连的算子链接在一起形成一个 task，原来的算子成为里面的 subtask。

并行度相同、并且是 one-to-one 操作，两个条件缺一不可。


### Window
![picture 19](img/Flink/Stream.png)  

一般真是的流都是无界的，可以把无界的数据流进行切分，得到有限的数据集进行处理，也就是得到有界流。窗口将无界的流切分为有限的流，将流数据分发到有限大小的桶bucket进行分析。

#### 窗口的类型
- 时间窗口（time window）
  - 滚动时间窗口
  - 滑动时间窗口
  - 会话窗口

- 计数窗口（count window）
  - 滚动计数窗口
  - 滑动计数窗口

#### 滚动窗口（Tumbling Windows）
![picture 20](img/Flink/Tumbing_Windows.png)  


- 将数据依据固定的窗口长度对数据进行切分
- 时间对齐，窗口长度固定，没有重叠。

#### 滑动窗口（Sliding Windows）
![picture 21](img/Flink/Sliding_Windows.png)  

- 滑动窗口是固定窗口的更广义的一种形式，滑动窗口由固定的窗口长度和滑动间隔组成
- 窗口长度固定，可以有重叠。

#### 会话窗口（Session Windows）
![picture 22](img/Flink/Session_Windows.png)  

- 由一系列事件组合一个指定时间长度的 timeout 间隙组成，也就是一段时间没有接收到新数据就会生成新的窗口。
- 时间无对齐

#### Window API

- 我们可以用 .window() 来定义一个窗口，然后基于这个 window 去做一些聚合或者其它处理操作。注意 window () 方法必须在 keyBy 之后才能用。
- Flink 提供了更加简单的 .timeWindow 和 .countWindow 方法，用于定义时间窗口和计数窗口。


#### 创建不同的时间窗口API
- 滚动时间窗口（tumbling time window）

    ```JAVA
    .timeWindow(Time.seconds(15))
    ```

- 滑动时间窗口（sliding time window）

    ```java
    .timeWindow(Time.seconds(15),Time.seconds(5))
    ```

- 会话窗口

    ```java
    .window(WventTimeSessionWindows.withGap(Time.minutes(10)))
    ```

#### 创建不同的计数窗口API    

- 滚动计数窗口（tumbling count window）

    ```JAVA
    .countWindow(5)
    ```

- 滑动计数窗口（sliding count window）

    ```java
    .countWindow(10,2)
    ```

#### 窗口函数
window function 定义了要对窗口中收集的数据做的计算操作，可以分为下面两类：
- 增量聚合函数（incremental aggregation functions）
  - 每条数据到来就进行计算，保持一个简单的状态
  - ReduceFunction, AggregateFunction
- 全窗口函数（full window functions）
  - 先把窗口所有数据收集起来，等到计算的时候会遍历所有数据
  - ProcessWindowFunction


### 时间语义
![picture 23](img/Flink/Time.png)  

- Event Time：事件创建的时间,通常记录在数据中。
- Ingestion Time：数据进入Flink的时间
- Processing Time：执行操作算子的本地系统时间，与机器相关

#### 使用哪种时间语义

![picture 24](img/Flink/Event_Time_And_Processing_Time.png)  

- 不同的时间语义有不同的应用场合
- 我们往往更关心事件时间（Event Time）

#### 设置时间语义
我们可以直接在代码中，对执行环境调用 setStreamTimeCharacteristic 方法，设置流的时间特性具体的时间，还需要从数据中提取时间戳（timestamp）
![picture 25](img/Flink/Set_Time.png)  


### 水位线

#### 数据乱序
![picture 26](img/Flink/No_Order.png)  

- 当 Flink 以 Event Time 模式处理数据流时，它会根据数据里的时间戳来处理基于时间的算子
- 由于网络、分布式等原因，会导致乱序数据的产生
- 乱序数据会让窗口计算不准确

#### 水位线
怎样避免乱序数据带来计算不正确？

- 遇到一个时间戳达到了窗口关闭时间，不应该立刻触发窗口计算，而是等待一段时间，等迟到的数据来了再关闭窗口
- Watermark 是一种衡量 Event Time 进展的机制，可以设定延迟触发
- Watermark 是用于处理乱序事件的，而正确的处理乱序事件，通常用Watermark 机制结合 window 来实现；
- 数据流中的 Watermark 用于表示 timestamp 小于 Watermark 的数据，都已经到达了，因此，window 的执行也是由 Watermark 触发的。
- watermark 用来让程序自己平衡延迟和结果正确性

#### 水位线的特点
- watermark 是一条特殊的数据记录
- watermark 必须单调递增，以确保任务的事件时间时钟在向前推进，而不是在后退
- watermark 与数据的时间戳相关



### Flink中的状态
![picture 27](img/Flink/Status.png)  

- 由一个任务维护，并且用来计算某个结果的所有数据，都属于这个任务的状态
- 可以认为状态就是一个本地变量，可以被任务的业务逻辑访问
- Flink 会进行状态管理，包括状态一致性、故障处理以及高效存储和访问，以便开发人员可以专注于应用程序的逻辑

总的说来，有两种类型的状态：算子状态和键控状态。


#### 算子状态（Operator State）
![picture 28](img/Flink/Operator_State.png) 

算子状态的作用范围限定为算子任务，算子状态的作用范围限定为算子任务，由同一并行任务所处理的所有数据都可以访问到相同的状态，状态对于同一子任务而言是共享的，算子状态不能由相同或不同算子的另一个子任务访问。

算子状态的数据结构
- 列表状态（List state）将状态表示为一组数据的列表
- 联合列表状态（Union list state）也将状态表示为数据的列表。它与常规列表状态的区别在于，在发生故障时，或者从保存点（savepoint）启动应用程序时如何恢复
- 广播状态（Broadcast state）如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合应用广播状态。




#### 键控状态（Keyed State）
![picture 29](img/Flink/Keyed_State.png)  

根据输入数据流中定义的键（key）来维护和访问，Flink 为每个 key 维护一个状态实例，并将具有相同键的所有数据，都分区到同一个算子任务中，这个任务会维护和处理这个 key 对应的状态，当任务处理一条数据时，它会自动将状态的访问范围限定为当前数据的 key。

键控状态的数据结构
- 值状态（Value state）将状态表示为单个的值
- 列表状态（List state）将状态表示为一组数据的列表
- 映射状态（Map state） 将状态表示为一组 Key-Value 对
- 聚合状态（Reducing state & Aggregating State）将状态表示为一个用于聚合操作的列表

#### 状态后端（State Backends）
- 每传入一条数据，有状态的算子任务都会读取和更新状态
- 由于有效的状态访问对于处理数据的低延迟至关重要，因此每个并行任务都会在本地维护其状态，以确保快速的状态访问
- 状态的存储、访问以及维护，由一个可插入的组件决定，这个组件就叫做状态后端（state backend）
- 状态后端主要负责两件事：本地的状态管理，以及将检查点（checkpoint）状态写入远程存储

状态后端的类型：
1. MemoryStateBackend

    内存级的状态后端，会将键控状态作为内存中的对象进行管理，将它们存储在 TaskManager 的 JVM 堆上，而将 checkpoint 存储在 JobManager 的内存中特点：快速、低延迟，但不稳定
2. FsStateBackend
    
    将 checkpoint 存到远程的持久化文件系统（FileSystem）上，而对于本地状态，跟 MemoryStateBackend 一样，也会存在 TaskManager 的 JVM 堆上同时拥有内存级的本地访问速度，和更好的容错保证

3. RocksDBStateBackend

    将所有状态序列化后，存入本地的 RocksDB 中存储。

























