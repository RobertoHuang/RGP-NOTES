# 副本机制

> `Kafka`从`0.8`版本开始引入副本`Replica`的机制，其目的是为了增加`Kafka`集群的高可用性
>
> 其中`Leader`副本负责读写，`Follower`副本负责从`Leader`拉取数据做热备，副本分布在不同的`Broker`上
>
> 
>
> `AR(Assigned Replica)`:副本集合(`Leader+Follower`的总和)
>
> `ISR(IN-SYNC Replica)`:同步副本集表示目前可用`Alive`且消息量与`Leader`相差不多的副本集合
>
> - 副本所在节点必须维持着与`Zookeeper`连接
> - 副本最后一条消息的`Offset`与`Leader`副本的最后一条消息的`Offset`之间差值不能超过指定阈值

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

## ReplicaManager副本管理器

- `becomeLeaderOrFollower`更新副本为`Leader/Follow`副本
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
- 定时任务
  - `checkpointHighWatermarks`将`HW`的值刷新到`replication-offset-checkpoint`中
  - `maybePropagateIsrChanges`定期将`ISR`集合发送变化的分区记录到`Zookeeper`中
    - `isrChangeSet`集合不为空
    -  最后一次`ISR`集合发生变化的时间距今已超过5秒
    - 上次写入`Zookeeper`时间距今已超过60秒
- `maybeUpdateMetadataCache`更新集群缓存

`OffsetCheckPoint`:用于管理`Offset`相关文件

- `replication-offset-checkpoint`:以`TopicPartion`为`Key`记录`HW`的值

  `HW`为已经复制到所有`Replica`的最后提交的`Offset`，`replication-offset-checkpoint`文件结构如下

  >第一行: 版本号
  >
  >第二行:当前写入`TopicPartition`的记录个数
  >
  >其他每行格式:`Topic Partition Offset`，如:`topic-test 0 0`

- `recovery-point-offset-checkpoint`:

##  数据同步

### 数据同步线程

- `AbstractFetcherThread`数据同步线程抽象类

  - `addPartitions(Map[TopicAndPartition, Long])`添加需要抓取的`TopicPartition`信息

  - `removePartitions(Set[TopicPartition])`删除指定`TopicPartition`副本的抓取任务

  - `delayPartitions(Iterable[TopicPartition], Long)`延迟抓取

  - `doWork()`:线程执行体，构造并同步发送`FetchRequest`，接收并处理`FetchResponse`

    ```reStructuredText
    1.buildFetchRequest创建数据抓取请求
    
    2.processFetchRequest发送并处理请求响应，最终写入副本的Log实例中
        2.1.同步发送请求获取响应【发送fetch请求失败则会退避replica.fetch.backoff.ms时间】
        2.2.结果返回期间可能发生日志截断或分区被删除重加等操作，因此这里只对offset与请求offset一致并且PartitionFetchState的状态为isReadyForFetch进行处理。调用processPartitionData()方法将拉取到的消息追加到本地副本的日志文件中，如果返回结果有错误消息就对相应错误进行相应的处理
        
    3.更新PartitionStates【Follower会根据自身拥有多少个需要同步的TopicPartition来创建对应的PartitionFetchState，这个东西记录了从Leader的哪个Offset开始获取数据】
    ```

- `ReplicaFetcherThread`副本数据同步线程

  - `processPartitionData`处理抓取线程返回的数据

    ```reStructuredText
    1.将获取到的消息集合追加到Log中
    
    2.更新本地副本的HW【本地副本的LEO和返回结果中Leader的HW值取较小值作为本地副本HW】
    
    3.更新本地副本的LSO【若返回结果中Leader的LSO值大于本地副本的LSO则更新本地副本的LSO值】
    ```

  - `handleOffsetOutOfRange`处理`Follower`副本请求的`offset`超过`leader`副本的`offset`范围

    ```reStructuredText
    handleOffsetOutOfRange方法主要处理如下两种情况:
        1.假如当前本地id=1的副本现在是Leader其LEO假设为1000，而另一个在ISR的副本id=2其LEO为800，此时出现网络抖动id=1的机器掉线后又上线，但此时副本的Leader实际上已经变成了id=2的机器，而id=2的机器LEO为800，这时候id=1的机器启动副本同步线程去id=2上机器拉取数据，希望从offset=1000的地方开始拉取，但是id=2的机器最大的offset才是800
    
        2.假设一个Replica(id=1)其LEO是10，它已经掉线好几天了，这个Partition的Leader的Offset范围是[100~800]，那么当id=1的机器重新启动时，它希望从offset=10的地方开始拉取数据，这时候就发生了OutOfRange，不过跟上面不同的是这里是小于Leader的Offset范围
    
    handleOffsetOutOfRange方法针对两种情况提供的解决方案:
        1.如果Leader的LEO小于当前的LEO，则对本地log进行截断(截断到Leader的LEO)
    
        2.如果Leader的LEO大于当前的LEO则说明有有效的数据可以同步，接下来要判断从哪里开始同步。如果Follow宕机比较久后再启动，可能Leader已经对部分日志进行清理，当前副本的LEO小于Leader副本的LSO的情况，当前副本需要截断所有日志并滚动新日志与Leader进行同步
    ```

  - `handlePartitionsWithErrors`将对应分区的同步操作暂停【`PartitionFetchState`置为`delay`】

### 数据同步线程管理器

- `AbstractFetcherManager`数据同步线程管理器抽象类

  `fetcherThreadMap KEY:BrokerIdAndFetcherId VALUE:AbstractFetcherThread`:实现消息的拉取由`AbstractFetcherThread`负责，`fetcherThreadMap `保存了`BrokerIdAndFetcherId`与同步线程关系

  - `addFetcherForPartitions(Map[TopicPartition, BrokerAndInitialOffset])`添加副本同步任务

    ```reStructuredText
    1.计算出Topic-Partition对应的fetcher id
    
    2.根据BrokerIdAndFetcherId获取对应的replica fetcher线程(若无调用createFetcherThread创建新的fetcher线程)，将Topic-Partition记录到FetcherTheadMap中(这个变量记录了每个Replica Fetcher线程需要同步的Topic-Partition列表
    ```

  - `removeFetcherForPartitions(Set[TopicPartition])`停止指定副本同步任务

    ```reStructuredText
    1.调用FetcherThread.removePartitions删除指定的Fetch任务
    
    2.若Fetcher线程不再为任何Follower副本进行同步将在shutdownIdleFetcherThreads中被停止
    ```

  - `shutdownIdleFetcherThreads()`某些同步线程负责同步的`TopicPartition`数量为0则停止该线程

  - `closeAllFetchers()`停止所有抓取线程

  - `createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint)`创建同步线程

- `ReplicaFetcherManager`副本数据同步线程管理器实现类 - > `ReplicaFetcherThread`

  `ReplicaFetcherManager`继承`AbstractFetcherManager`实现了抽象方法`createFetcherThread`


##  ReplicaManager副本管理器

`ReplicaManager`通过对`Partition`对象的管理【`ReplicaManager.allPartitions`】来控制着` Partition `对应的`Replica`实例【`Partition.allReplicasMap`】，`Replica`实例通过`Log`对象实例管理着其底层的存储内容

- `stopReplicas`关闭副本

  ```reStructuredText
  1.停止对指定分区的Fetch操作
  
  2.调用stopReplica()关闭指定的分区副本【判断是否需要删除Log】
  ```

- `makeFollower`将指定分区的`Local Replica`切换为`Follower`

  ```reStructuredText
  1.调用partition的makeFollower将Local Replica切换为Follower，后面就不会接受这个partition的Produce请求了，如果有Client再向这台Broker发送数据会返回相应的错误
  
  2.停止对这些partition的副本同步(如果本地副本之前是Follower现在还是Follower，先关闭的原因是:这些Partition的Leader发生了变化)，这样可以保证本地副本将不会有新的数据追加
  
  3.清空时间轮中的produce和fetch请求【tryCompleteDelayedProduce、tryCompleteDelayedFetch】
  
  4.若Borker没有掉线则向这些Partition的新Leader启动副本同步线程【不一定每个Partition数据同步都会启动一个Fetch线程，对于一个Broker只会启动num.replica.fetchers个线程。这个Topic-Partition会分配到哪个fetcher线程上是根据topic名和partition id进行计算得到的】
  ```

- `makeLeaders`将指定分区的`Local Replica`切换为`Leader`

  ```reStructuredText
  1.先停止对这些Partition的副本同步流程，因为这些Partition的本地副本以及被选举为Leader
  
  2.将这些Partition本地副本设置为Leader，并且开始更新对应的Meta信息(记录与其他Follower相关信息)
  ```

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


