# 副本备份机制

## `Replica`

 `Kafka`使用一个`Replica`对象表示一个分区的副本，主要用于管理`HW`与`LEO`

## `Partition`

- `getOrCreateReplica`获取/创建副本

  > 如果创建的是本地副本还会创建/恢复对应的`Log`并初始化/恢复`HW`

- `makerLeader/makeFollower`更新副本为`Leader/Follow`副本

  > `Broker`会根据`KafkaController`发出的`LeaderAndISRRequest`请求控制副本的角色切换【按照`PartitionState`指定的信息将`leader/follow`字段指定的副本转换成`Leader/Follow`副本】

- `maybeIncrementLeaderHW`尝试更新`Leader`副本的`HW`

  > 当`ISR`集合发生增减或是`ISR`集合中任一副本的`LEO`发生变化时都可能导致`ISR`集合中最小的`LEO`变大，所以这些情况都要调用`maybeIncrementLeaderHW`方法进行检测

- `maybeExpandIsr`管理`ISR`集合【添加】

  > 随着`Follower`副本不断与`Leader`副本进行消息同步，`Follower`副本的`LEO`会逐渐后移并最终能赶上`Leader`副本的`HW`，此时该`Follower`副本就有资格进入`ISR`集合

- `maybeShrinkIsr`管理`ISR`集合【删减】

  > 通过检测`Follower`副本的`lastCaughtUpTimeMs`字段找出已滞后的`Follower`集合并从`ISR`集合中移除【无论是长时间没有与`Leader`副本进行同步还是其`LEO`与`HW`相差太大都可以从该字段反应出来】

- `appendRecordsToLeader`向`Leader`副本对应的`Log`中追加消息

- `checkEnoughReplicasReachOffset`检查是否足够多的`Follower`副本已同步消息

## `ReplicaManager`

- `becomeLeaderOrFollower`更新副本为`Leader/Follow`副本
- `makeLeaders`将指定分区的`Local Replica`切换为`Leader`
- `makeFollowers`将指定分区的`Local Replica`切换为`Follower`【会启动同步消息线程】
- `appendToLocalLog`追加消息
  - 检测目标`Topic`是否是`Kafka`的内部`Topic`及是否允许向内部`Topic`追加数据
  - 根据`topicPartition`获取对应的`Partition`【`Partition`是否存在及是否是`OfflinePartition`】
  - 调用`Partition`的`appendRecordsToLeader`方法往`Log`中追加消息
- `readFromLocalLog`读取消息
  - `minOneMessage`保证即使在有单次读取大小限制的情况下，至少读取到一条数据
  - 根据`TopicPartition`获取`Replica`信息【`fetchOnlyFromLeader`来判断是否必须为`Leader`副本】
  - 根据`readOnlyCommitted`决定`maxOffsetOpt`【`lastStableOffset`是与事务有关的参数】
  - 从`Log`中读取数据并返回【限速处理`shouldLeaderThrottle`】
- `updateFollowerLogReadResults` `Follower`副本读取消息结束后操作
  - 更新`Leader`副本上维护的`Follower`副本的各项状态
  - 随着`Follower`不断`fetch`消息最终追上`Leader`副本，可能对`ISR`集合进行扩张【尝试更新`HW`】
- `stopReplicas`关闭副本
  - 停止对指定分区的`Fetch`操作
  - 调用`stopReplica()`关闭指定的分区副本【判断是否需要删除`Log`】
- 定时任务
  - `checkpointHighWatermarks`将`HW`的值刷新到`replication-offset-checkpoint`中
  - `maybePropagateIsrChanges`定期将`ISR`集合发送变化的分区记录到`Zookeeper`中
    - `isrChangeSet`集合不为空
    -  最后一次`ISR`集合发生变化的时间距今已超过5秒
    - 上次写入`Zookeeper`时间距今已超过60秒
- `maybeUpdateMetadataCache`更新集群缓存

## `ReplicaFetcherManager`

- `addFetcherForPartitions`添加副本同步任务
  - 根据`Leader Broker`和分区编号进行分组【同个`Leader Broker`下的几个`topicAndPartition`一组】
  - 对于每个`BrokerAndFetcherId`分配一个`FetcherThread`【如果不存在则创建】
  - 调用`FetcherThread.addPartitions`给该线程添加`Fetch`任务
- `removeFetcherForPartitions`停止指定副本同步任务
  - 调用`FetcherThread.removePartitions`删除指定的`Fetch`任务
  - 【若`Fetcher`线程不再为任何`Follower`副本进行同步将在`shutdownIdleFetcherThreads`中被停止】

## `ReplicaFetcherThread`

- `addPartitions`添加副本同步任务

  - 添加需要同步的`TopicPartition`信息及开始同步的`offset`
  - 如果提供的`offset`则通过`handleOffsetOutOfRange`来获取有效的初始`offset`

- `removePartitions`删除副本同步任务

  - 删除需要同步的`TopicPartition`信息

- `handleOffsetOutOfRange`处理`Follower`副本请求的`offset`超过`leader`副本的`offset`范围

  > 可能情况:【`leader的LEO`<`Follow`副本请求的`offset`<`leader`最小`offset`】

  - `Leader LEO`<`Follow LEO`

    ```reStructuredText
    1.一个Follower副本发生宕机而Leader副本不断接收来自生产者的消息并追加到Log中，此时Follower副本因为宕机并没有与Leader副本进行同步【Follower副本数据远落后于Leader副本】
    2.此Follower副本重新上线，在它与Leader副本完全同步之前它还没有资格进入ISR集合，假设ISR集合中的Follower副本在此时全部宕机，只能选举此Follower副本为新的Leader副本
    3.Leader副本重新上线成为Follower副本，此时就会出现Follower副本的LEO超越了Leader副本的LEO
    ```

    - 根据配置决定是否需要停机【`unclean.leader.election.enable`】
    - 将分区对应的`Log`截断到`Leader`副本的`LEO`位置

  - `Follow LEO`<`Leader startOffset`

    ```reStructuredText
    如果Follower副本宕机后过了很长时间才重新上线，Leader副本在此期间可能执行了多次log retention任务来删除陈旧的日志，这就可能导致Leader副本中的StartOffset大于Follow副本的LEO
    ```

    - 将`Log`全部截断并创建新的`activeSegment`

- `handlePartitionsWithErrors`将对应分区的同步操作暂停【`PartitionFetchState`置为`delay`】

- `doWork`线程执行体【同步任务】

  > `ReplicaFetcherThread`继承自`AbstractFetcherThread`继承自`ShutdownableThread`，其核心业务代码在`doWork()`方法中【创建`FetchRequest`进行副本同步，如果没有需要进行同步的任务则退避】

  - `buildFetchRequest`创建同步请求
  - `processFetchRequest`发送并处理请求响应
    - 同步发送请求获取响应
    - 结果返回期间可能发生日志截断或者分区被删除重加等操作，因此这里只对`offset`与请求`offset`一致并且`PartitionFetchState`的状态为`isReadyForFetch`的情况下进行处理
    - 调用子类实现的`processPartitionData()`方法处理返回结果【将获取的消息集合追加到`Log`中】
      - `processPartitionData`处理返回结果
        - 将获取的消息集合追加到`Log`中
        - 更新本地副本`highWatermark`【本地副本的`logEndOffset.messageOffset`和返回结果`partitionData.highWatermark`中取较小值作为本地副本`highWatermark`】
        - 更新本地副本`logStartOffset`【根据`leaderLogStartOffset`来进行判断】
    - 更新`TopicPartition`对应的`PartitionFetchState`信息

## `NetworkClientUtils`

- `awaitReady`阻塞等待直到指定`Node`处于`Ready`状态
- `sendAndReceive`发送请求后阻塞等待数据响应

## `MetadataCache`

用来缓存整个集群中全部分区状态的组件

- `updateCache`更新集群元数据

- `getTopicMetadata`获取元数据

## 关键概念【附录】

`baseOffset`该副本当前所含第一条消息的`offset`

`logEndOffset LEO`日志末端位移，记录了该副本对象底层日志文件的下一条消息的`offset`【若`LEO=10`那么表示在该副本日志上已保存了10条消息，位移范围是`[0-9]`】以下是`LEO`更新机制

- `Follower`副本端的`Follower`副本的`LEO`

  发送`Fetch`请求后收到`Leader`响应后，`Follower`开始向底层`LOG`写数据从而自动更新`LEO`

- `Leader`副本端的`Follower`副本的`LEO`

  `Leader`副本端的`Follower`副本`LEO`更新发生在`Leader`处理`Follower Fetch`请求时【返回响应前】

`highWatermark HW`高水印(已提交的)【保存最新一条已提交消息的`offset`，任何一个副本的`HW`值一定不大于其`LEO`值，`Leader`副本`HW`决定消费者能看到消息数量】以下是`HW`更新机制

- 副本成为`Leader`副本时
- `Broker`出现奔溃导致副本被踢出`ISR`时
- `Leader`处理`Follower Fetch`时

```reStructuredText
- Follower为什么要维护HW
  【保护机制】假设只有Leader维护了HW信息，一旦Leader宕机就没有其它Broker知道现在HW是多少
  
- 影响HW的值与最小同步副本数有关，与ACK机制没有直接关系

- Leader维护了ISR(同步副本集合)，每个Partition当前的Leader和ISR信息会记录在Zookeeper中
  1.所有Follower都会和Leader通信获取最新消息，所以由Leader来管理ISR最合适
  2.Leader副本总是包含在ISR中，只有ISR中的副本才有资格被选举为Leader
```
