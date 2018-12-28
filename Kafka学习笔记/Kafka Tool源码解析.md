## `kafka-topics.sh`主题相关操作

- `--create`创建`Topic`
  - 支持手动副本分配和自动副本分配
  - 校验创建`Topic`请求【名称合法性、副本分配是否合法等】
  - 将`Topic`配置写入`/config/topics/xxx`节点、`Topic`分区分配写入`/brokers/topics/xxx`节点
- `--delete`删除`Topic`功能
  - 筛选出需要删除的`Topic`【支持正则表达式】
  - 在`ZK`的`/admin/delete_topics`节点上写入需要删除的`Topic`信息
- `--alter`修改`Topic`的分区数量、副本的分配以及相关配置信息
  - 【` --alter --zookeeper x --topic x --partitions x --replica-assignment 0,0`】
  - 筛选出需要修改的`Topic`【支持正则表达式】
  - 获取待修改的`Topic`已有的分区副本分配情况
  - 如果已手动指定新增的副本分区分配则使用手动指定的副本分区分配，否则自动分配
  - 将已存在的副本分区分配集合与新增的副本分区分配集合合并写入`ZK`【`/brokers/topics/topic`】

## `kafka-configs.sh`配置相关操作

- 将要添加/删除的配置与`ZK`现有配置整合
- 将整合好的配置持久化到`ZK`节点【`/config/{configType}/xxx`上】
- 将发生变化的配置写入`changes`节点【`config/changes/config_change_xxx`有序节点】

## `kafka-reassign-partitions.sh`副本重新分配

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

## `kafka-preferred-replica-election.sh`优先副本分配

- 判断`path-to-json-file`是否有指定文件，解析成`TopicPartition`
- 将解析后的`TopicPartition`数据写入**`/admin/preferred_replica_election`**节点
- 即执行该命令的的本质是告知`Controller`哪些`TopicPartition`需要参与优先副本选举

> 以上相关命令本质都是修改`ZK`节点的数据，并未真正执行有效的操作【有效业务逻辑都由`Controller`监听`ZK`节点变化完成，该部分将在`Controller`部分展开介绍】

## `kafka-consumer-groups.sh`消费组相关操作

消费组实际操作委托给了`ConsumerGroupCommand`完成，`ConsumerGroupCommand.describeConsumerGroup`方法为消费组相关操作提供了基础，它用于查询消费组的基础信息，流程如下

> 发送`FindCoordinatorRequest`查找`coordinator`所在`Broker`
>
> 发送`DescribeGroupsRequest`请求查找消费组基础信息，封装为`ConsumerGroupSummary`返回
>
> 【`ConsumerGroupSummary`包含组消费组状态、分配策略、消费组成员、消费组所在`Broker`等信息】

- `--list`查询消费组列表

  - 往`Broker`发送`ListGroupsRequest`请求查询消费组列表

- `--describe`获取指定组信息

  - `--state`获取消费组对应状态信息
    - 获取消费组基础信息，封装成组状态信息`GroupState`并返回
  - `--members`获取消费组对应成员信息
    - 获取消费组基础信息，封装成组成员信息为`MemberAssignmentState`集合并返回
  - `--offsets`获取消费组对应`offset`信息
    - 获取消费组基础信息【主要是为了获取消费组所在`Broker`及组成员信息】
    - 往消费组所在`Broker`发送`OffsetFetchRequest`获取指定`Group`的`Offset`信息
    - 按消费者分组获取`TopicPartition`对应的`offset`信息封装成`PartitionAssignmentState` 集合
    - 【`PartitionAssignmentState`包含`Offset`信息及`LEO`信息【可以算出`Lag`:当前滞后消费多少】

- `--reset-offsets`重置`Offset`信息

  - 获取消费组基础信息【仅能修改组状态为`Empty`和`Dead`的`Offset`】

  - 获取需要重置`Offset`的`TopicPartition`信息【`--all-topics`或`--topic test-topic:1,2,3`】

  - 根据策略计算出需要将`Offset`重置为何值【用于最终提交`Offset`使用】，以下是`Kafka`提供的策略

    |    指令     |                           含义                            |
    | :---------: | :-------------------------------------------------------: |
    | to-earliest |               把位移调整到分区当前最小位移                |
    |  to-latest  |               把位移调整到分区当前最新位移                |
    | to-current  |                 把位移调整到分区当前位移                  |
    |  to-offset  |                  把位移调整到指定位移处                   |
    |  shift-by   | 把位移调整到当前位移 + N处，注意N可以是负数，表示向前移动 |
    | to-datetime |           把位移调整到大于给定时间的最早位移处            |
    | by-duration |         把位移调整到距离当前时间指定间隔的位移处          |
    |  from-file  |                  从CSV文件中读取调整策略                  |

  - 确定方案【`--dry-run`，`--execute`判断是否真正执行重置`Offset`操作，`--export`导出执行结果】