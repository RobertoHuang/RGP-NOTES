- `handleCreateTopicsRequest`处理创建`Topic`请求
  - 判断`Controller`是否转移及授权认证校验【`controller.isActive`】
  - 将请求创建的`Topic`进行分类【`validTopics`、`duplicateTopics`】
  - 调用`AdminManager.createTopics`执行真正的创建`Topic`流程
    - 获取当前存活的`Broker`列表
    - 填充请求中带过来的`Topic`私有化配置并校验
    - 进行副本分配工作【手动指定分配/自动分配`AdminUtils.assignReplicasToBrokers`，副本自动分配涉及机架感知算法与无机架感知算法】可参考博主之前的文章:[Kafka副本分配算法详解](https://github.com/RobertoHuang/RGP-NOTES/blob/master/Kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/Kafka%E5%89%AF%E6%9C%AC%E5%88%86%E9%85%8D%E7%AE%97%E6%B3%95%E8%AF%A6%E8%A7%A3.md)
    - 校验创建的`Topic`是否符合要求【`CreateTopicPolicy.validate`】
    - 判断是否是只校验请求还是真正创建`Topic`请求【根据`validateOnly`字段】
    - 调用`adminZkClient.validateCreateOrUpdateTopic`校验创建`Topic`请求【如判断`Topic`名称是否合法、`Topic`是否已存在、`Topic`的分区副本分配是否合理等等】
    - 将`Topic`的私有化配置及将副本分配结果写入`Zookeeper`【`Controller`将监听该节点】
    - 判断请求能否立即返回【`timeout <= 0 || validateOnly || 创建请求全失败了`】
    - 若有成功创建`Topic`请求需要新建`DelayedCreatePartitions`放入延迟任务队列中【因为要等待`Controller`监听到节点变化执行真正的创建副本等操作，在这些操作完成后`Controller`将会发出`UpdateMetadataRequest`，在`handleUpdateMetadataRequest`中将处理延迟队列中的任务】

- `TopicCommand.deleteTopic`删除`Topic`功能

  - 筛选出需要删除的`Topic`【支持正则表达式】
  - 在`ZK`的`/admin/delete_topics`节点上写入`Topic`信息

- `TopicCommand.alterTopic`修改`Topic`的分区数量、副本的分配以及相关配置信息

  【` --alter --zookeeper x --topic x --partitions x --replica-assignment 0,0`】

  - 筛选出需要修改的`Topic`【支持正则表达式】
  - 获取待修改的`Topic`已有的分区副本分配情况
  - 如果已手动指定新增的副本分区分配则使用手动指定的副本分区分配，否则自动分配
  - 将已存在的副本分区分配集合与新增的副本分区分配集合合并写入`ZK`【`/brokers/topics/topic`】

- `ConfigCommand`修改配置/获取配置

  - 将要添加/删除的配置与`ZK`现有配置整合
  - 将整合好的配置持久化到`ZK`节点【`/config/{configType}/xxx`上】
  - 将发生变化的配置写入`changes`节点【`config/changes/config_change_xxx`有序节点】

- `ReassignPartitionsCommand`副本重新分配

  - `--generate`生成重新分配策略
    - 解析`topics-to-move-json-file`与`broker-list`
    - `AdminUtils.assignReplicasToBrokers`进行副本分配【不会修改`Partition`与`Replica`数】
  - `--execute`执行重新分配策略
    - 解析`reassignment-json-file`
    - 将副本重写分配方案写入**`/admin/reassign_partitions`**节点【副本转移与配额(略)】
  - `--verify`验证副本重新分配是否成功
    - 解析`reassignment-json-file`
    - 判断**`/admin/reassign_partitions`**节点是否存在【存在则说明正在执行副本分配】
    - 若**`/admin/reassign_partitions`**节点不存在则通过当前副本分配结果与预期分配是否一致来判断

- `PreferredReplicaLeaderElectionCommand`优先副本分配

  - 判断`path-to-json-file`是否有指定文件，解析成`TopicPartition`
  - 将解析后的`TopicPartition`数据写入**`/admin/preferred_replica_election`**节点
  - 即执行该命令的的本质是告知`Controller`哪些`TopicPartition`需要参与优先副本选举

**以上相关命令本质都是修改`ZK`节点的数据，并未真正执行有效的操作【有效业务逻辑都由`Controller`监听`ZK`节点变化完成，该部分将在`Controller`部分展开介绍】**



- `ConsumerGroupCommand`消费组相关操作
  - 获取消费组基础信息
    - 发送`FindCoordinatorRequest`请求查找`coordinator`所在`Broker`
    - 发送`DescribeGroupsRequest`请求查找消费组基础信息，封装为`ConsumerGroupSummary`返回【`ConsumerGroupSummary`包含组状态及策略、组成员信息、所在`Broker`等】
  - `--list`查询消费组列表
    - 往`Broker`发送`ListGroupsRequest`请求查询消费组列表
  - `--describe`获取指定组信息
    - `--offsets`获取消费组对应`offset`信息
      - 获取消费组基础信息
      - 往消费组所在`Broker`发送`OffsetFetchRequest`获取指定`Group`的`Offset`信息
      - 获取消费组所有`TopicPartition`对应的`offset`信息封装成`PartitionAssignmentState` 【`PartitionAssignmentState`包含`Offset`信息及`LEO`信息【可以算出`Lag`:滞后消费多少】
    - `--members`获取消费组对应成员信息
      - 获取消费组基础信息，封装成组成员信息`MemberAssignmentState`并返回
    - `--state`获取消费组对应状态信息
      - 获取消费组基础信息，封装成组状态信息`GroupState`并返回
  - `--reset-offsets`重置`Offset`
    - 获取消费组基础信息【仅能修改组状态为`Empty`和`Dead`的`Offset`】
    - 获取需要重置`Offset`的`TopicPartition`信息【可以手动指定】
    - 根据策略计算出需要将`Offset`重置为何值
      - ` --to-earliest` - 把位移调整到分区当前最小位移
      - `--to-latest` - 把位移调整到分区当前最新位移
      - ` --to-current` - 把位移调整到分区当前位移
      - `--to-offset` - 把位移调整到指定位移处
      - `--shift-by` - 把位移调整到当前位移 + N处，注意N可以是负数，表示向前移动
      - `--to-datetime` - 把位移调整到大于给定时间的最早位移处
      - `--by-duration` - 把位移调整到距离当前时间指定间隔的位移处
      - ` --from-file` - 从`CSV`文件中读取调整策略