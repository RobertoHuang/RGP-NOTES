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

- `PreferredReplicaLeaderElectionCommand`优先副本分配

  - 判断`path-to-json-file`是否有指定文件，解析成`TopicPartition`
  - 将解析后的`TopicPartition`数据写入`/admin/preferred_replica_election`节点
  - 真正的副本分配工作是由`Controoler`监听`/admin/preferred_replica_election`节点变化完成的，即此处执行的代码作用是告知`Controller`哪些`TopicPartition`需要执行优先副本选举
