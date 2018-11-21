# 日志管理

> 包含内容:日志的读写、分段、清理和管理
>
> `Log`并不是直接对应于磁盘上的一个日志文件而是对应磁盘上的一个目录`<topic_name>_<partition_id>`
>
> `Log`分为多个`LogSegment`【一个`LogSegment`对应磁盘上一个日志文件和索引文件】
>
> 日志的文件名命名规则为`[baseOffset].log`，`baseOffset`是日志中第一条消息的`offset`

## `FileRecords`

> 从文件中读取消息，并提供遍历消息功能

关于`FileRecords`详细解析可参考:[FileRecords详解](http://www.voidcn.com/article/p-xagajpcq-brh.html)

## `OffsetIndex`

> 为了提高查找消息的性能`Kafka`为每一个日志文件添加了2个索引文件`OffsetIndex`和`TimeIndex`

`OffsetIndex`索引文件格式:每一个索引项为8字节【`offset`占用4个字节，`position`占用4个字节】

- `append`添加索引记录
- `truncateTo`截断索引文件
- `lookup`查找`offset`小于`targetOffset`的最大项物理地址
- `indexSlotRangeFor`返回两个数值都是最接近`target`的
- `mmap`用来操作索引文件的`MappedByteBuffer`

## `LogSegment`

> 为了防止`Log`文件过大将`Log`切分成多个日志文件，每个日志文件对应一个`LogSegment`

- `append`写入日志
  - 写入消息【写入到`FileRecords`中】
  - 如果有需要的话更新`OffsetIndex`和`TimeIndex`
- `read`读取消息
  - 计算开始读取消息位置`startPosition`
  - 计算读取长度【由`maxOffset`、`maxPosition`、`maxSize`共同决定】
- `recover`根据日志文件重写索引文件并验证消息合法性

## `Log`

每个`Replica`对应一个`Log`对象它包含这个分区所有`segment`文件【`segments`:`LogSegment`的`baseOffset`作为`key`，`LogSegment`的对象作为`value`】及索引文件。以下是几个重要参数

```reStructuredText
- logEndOffset下一条消息的偏移量

- nextOffsetMetadata下一个偏移元数据
  - 下一条消息的偏移量
  - 日志分段的基准偏移量
  - 在日志分段上的position
  
- activeSegment活动的日志分段【保证顺序写入】
```

- `loadSegments`从磁盘`Log`文件中载入`Segment`

  - 清理临时文件【`.delete`和`.cleaned`结尾的文件】，收集`Swap`文件
  - 加载`Segment`和`Index`文件，将`Segment`依次加入`cache`中
  - 完成被中断的`swap`操作【载入`SwapSegment`并替换对应`Segment`，为了保持不`crash`系统旧的`log`文件添加`.deleted`后缀，后面的定时任务或者下次的系统重启会删除】
  - 对于空的`Log`需要创建`activeSegment`保证`Log`中至少有一个`LogSegment`，否则执行恢复操作

- `append`在`Log`中一个最重要的方法就是日志的写入【追加日志】
  - `analyzeAndValidateRecords`校验消息及计算部分数据
  - `trimInvalidBytes`删除这批消息中无效的消息

  - `validateMessagesAndAssignOffsets`为每条消息设置绝对偏移量和时间戳
  - `maybeRoll`是否需要创建新的`activeSegment`
    - 本次待追加的消息集合大小超过配置`LogSegment`的最大长度
    - 距离上次日志分段的时间是否达到了设置的阈值
    - 索引文件满了/消息最大位移和基础位移大小差值超过`int`最大值
  - `segment.append`往`segment`中追加数据
  - 更新`logEndOffset`并判断是否需要刷新磁盘【如果需要的话调用`flush()`方法刷到磁盘】

- `roll`当满足特定条件后将会创建新的`segment`对象用来保存数据【滚动日志】
  - 创建新的日志文件和索引文件
  - 创建对应的`segment`对象并添加到跳表中
  - 更新`activeSegment.baseOffset`和`activeSegment.size`
  - 使用`KafkaScheduler`执行`flush-log`任务【`KafkaScheduler`本质是将`fun`封装成`Runable`】

- `flush`将`recoverPoint~LEO`之间的消息数据刷新到磁盘并修改`recoverPoint`的值
  - 通过对`segments`调表操作查找`recoverPoint`和`offset`之间的`LogSegment`对象
  - 调用`segment.flush`将数据刷新到磁盘【保证数据持久性】

- `read`日志的读取【将`nextOffsetMetadata`保存成方法局部变量保证线程安全】
  - 日志边界检查

  - 根据是否是`activeSegment`处理`maxPosition`

    ```reStructuredText
    如果Fetch请求刚好发生在activeSegment上，当多个Fetch请求同时处理如果nextOffsetMetadata更新不及时,可能会导致发生OffsetOutOfRangeException异常。为了解决这个问题这里能读取的最大位置是对应的物理位置exposedPos而不是activeSegment的LEO
    ```

  - 调用`segment.read`方法读取数据，如果没读取到数据则读取下一个`LogSegment`


##  `LogManager`

> `LogManager`初始化过程主要完成如下功能
>
> - 启动定时任务
> - 调用`createAndValidateLogDirs`和`loadLogs`
> - 启动`Cleaner`后台线程用于日志的压缩清理工作【根据配置决定】

- 初始化过程
  - `createAndValidateLogDirs`检查日志目录

    - 确保数据目录中没有重复的数据目录
    - 确保`liveLogDirs`与`offlineLogDirs`日志目录不能有重叠
    - 如果数据目录不存在的话就创建相应的目录
    - 检查每个目录路径是否是可读的
  - `loadLogs`加载所有日志分区

    - 为每个`Log`目录分配一个有`numRecoveryThreadsPerDataDir`条线程数的线程池
    - 检测`Broker`上次是否正常关闭并设置`Broker`状态【通过`.kafka_cleanshutdown`文件判断】
    - 从检查点文件读取`Topic`对应的恢复点`offset`信息【`recovery-point-offset-checkpoint`】
    - 从检查点文件读取`Topic`对应的`startoffset`信息【`log-start-offset-checkpoint`】
    - 为每个日志子目录创建一个任务(`loadLog`【`TopicPartition`生成`Log`对象】)并交给线程池处理
    - 主线程阻塞等待所有线程任务完成，删除对应的`cleanShutdownFile`并关闭之前创建的线程池
  - 定时任务【所有日志分区加载完毕后执行】
    - `cleanupLogs`清理`LogSegment`工作【`LogSegment`的存活时长/整合`Log`的大小】
      - 根据日志保留时间【当前时间-最后修改时间<`retention.ms`】
      - 根据`Log`大小【`diff - segment.size >= 0` `diff`为超过日志存储最大值部分的数据】
      - 下个`Segment`的`baseOffset`依旧小于`logStartOffset`
    - `flushDirtyLogs`刷新日志文件【日志未刷新时间>此`Log`的`flush.ms`配置项指定的时长】
    - `checkpointLogRecoveryOffsets`更新`recovery-point-offset-checkpoint`文件
    - `checkpointLogStartOffsets`更新`log-start-offset-checkpoint`文件避免读取到已删除数据
    - `deleteLogs`定期删除`logsToBeDeleted`列表中的数据
      - `LogManager`启动时会扫描所有分区目录名结尾是`-delete`的分区加入到`logsToBeDeleted`中【把需要先把分区目录标记一下在后缀加上`-delete`表示该分区准备删除了，这样做可以防止如果删除时间没到就宕机下次重启时可以扫描`-delete`结尾的分区再删除，保障可靠性】
      - 分区被删除的时候走的都是异步删除策略会先被加入到`logsToBeDeleted`中等待删除
- `LogManager`中还提供三个重要方法【`createLog`、`deleteLog()`、`getLog()`】

## `LogCompact`

> 日志压缩:此处的日志压缩不是指通过压缩算法对日志进行压缩，而是对重复日志进行清理来达到目的。在日志清理过程中会清理重复的`key`【类似于`Redis`日志重写】在清理完后一些`segment`文件大小就会变小，这时候`kafka`会将那些小的文件再合并成一个大的`segment`文件

![日志压缩示意图](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/Kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E6%97%A5%E5%BF%97%E5%8E%8B%E7%BC%A9%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

> - `cleaner-offset-checkpoint`记录着`cleanerCheckPoint`(当前清理到哪里了)
>
> - 在未清理的`segment`中找出可以清理的那部分`segment`
>   - `activeSegment`不允许清理
>   - `min.compaction.lag.ms`根据`segment`最后的一条记录的插入时间是否已经超过最小保留时间
> - 根据可以清理的`segment`构建`SkimpyOffsetMap`对象，这是一个`key`与`offset`的映射关系的哈希表
> - 再遍历已清理部分和可以清理部分的`segment`的每一条日志，根据`SkimpyOffsetMap`来判断是否保留
> - `log.cleaner.delete.retention.ms`对于`value`为`null`的日志`kafka`称这种日志为墓碑消息，在执行日志清理时会删除到期的墓碑消息。墓碑消息的保留时间和已清理部分的最后一个`segment`有关系

![日志合并示意图](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/Kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E6%97%A5%E5%BF%97%E5%90%88%E5%B9%B6%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

- `Kafka`对日志文件进行压缩需要符合以下两个条件
  - `log.segments.bytes(默认值是1G)`合并后的`segment`大小不超过`segmentSize`
  - `log.index.interval.bytes(默认值为10MB)`合并后索引文件占用大小之和不超过`maxIndexSize`

### `LogToClean`

`LogToClean`类来表示要被清理的`Log`，保存了如下信息

- `firstDirtyOffset`表示本次清理起始点，其前边的`offset`将被作清理与在其后的`message`作`key`的合并

- `cleanBytes` `[0, firstDirtyOffset)`总字节数【已清理字节数】
- `cleanableBytes` `[firstDirtyOffset, firstUncleanableOffset)`总字节数【需要清理部分字节数】
- `totalBytes`  总的字节数【需要清理的部分+不需要清理的部分】
- `cleanableRatio` 需要清理的`log`的比例,这个值越大越可能被最后选中作清理

### `CleanerThread`

> `CleanerThread`继承自`ShutdownableThread`，它的核心任务是在`doWork`中完成的

- `cleanOrSleep`执行清理任务或延迟执行清理任务
  - 通过`LogCleanerManager.grabFilthiestCompactedLog`查找是否有需要进行日志压缩的`Log`
  - 如果有则调用`cleaner.clean`执行清理任务，否则退避`15S`后继续执行该方法

### `LogCleaner`

![日志压缩合并过程](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/Kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E6%97%A5%E5%BF%97%E5%8E%8B%E7%BC%A9%E5%90%88%E5%B9%B6%E8%BF%87%E7%A8%8B.png)

- `doClean`进行日志的清理
  - `buildOffsetMap`从`firstDirtyOffset`开始遍历`LogSegment`填充`OffsetMap`
    - 日志段中的消息必须是有`key`的，因为日志压缩就是根据`key`来实现的
    - 如果一条消息的大小比读缓冲区大小要大的话那么需要调用`growBuffers`来扩容读缓冲区以容纳大消息，并且在构建完`mapping`之后要恢复读缓冲区的大小长度
  - `groupSegmentsBySize`对0到`endOffset`的消息进行分组【分组的单位是`LogSegment`】
    - `log.segments.bytes(默认值是1G)`合并后的`segment`大小不超过`segmentSize`
    - `log.index.interval.bytes(默认值为10MB)`合并后索引文件占用大小之和不超过`maxIndexSize`
  - `cleanSegments`将分组后的`LogSegment`集合进行日志压缩
    - 读取数据
    - 判断是否需要保留数据
      - 此消息中是否含有`Key`
      - `OffsetMap`中是否有相同`Key`且`Offset`更大的消息
      - 此消息是"删除标记"且此`LogSegment`中的删除标记可以安全删除
    - 用新的`Segment`替换老的`Segment`
      - 将新的`segment`文件后缀从`.cleaned`重命名到`.swap`文件
      - 将新的`segment`放入到`log.segments`中, 将老的一个个移除
      - 然后将老的文件重命名为`.deleted`文件, 异步线程中删除老的`segment`文件
      - 最后将新的`segment`重命名去掉`swap`后缀完成`clean`

### `LogCleanerManager`

- 状态转换

  > 当开始进行日志压缩任务时会先进入`LogCleaningInProgress`的状态，压缩任务可以被暂停进入此时进入`LogCleaningPaused`状态，压缩任务若被中断则先进入`LogCleaningAborted`状态等待`Cleaner`线程将其中任务中止，然后进入`LogCleaningPaused`状态。处于`LogCleaningPaused`状态的日志不会再被压缩，直到有其它线程恢复其压缩状态

- `grabFilthiestCompactedLog`选取下一个需要进行日志压缩的`Log`

  - 从所有的`Log`中产生出`LogToClean`对象列表
    - 过滤掉`cleanup.policy`配置项为`delete`的`Log`
    - 过滤掉`inProgress`包含状态的`Log`【已有清理任务】
  - 从获取的`LogToClean`列表中过滤出`cleanableRatio`大于`config`中配置的清理比率的`LogToClean`
  - 从过滤后的`LogToClean`列表中获取`cleanableRatio`最大的即为当前最需要被清理的`LogToClean`

- `updateCheckpoints`每次清理完要更新当前已经清理到的位置, 记录在`cleaner-offset-checkpoint`文件

