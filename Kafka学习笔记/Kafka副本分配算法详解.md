# 副本分配

> 为了起到备份的效果`Kafka`的副本分配应该满足如下规则
>
> - 在`Brokers`之间均分`Replicas`
> - `Replica`与所备份的节点不能再一台机器上
> - 如果`Broker`有`Rack`信息则`Partition`的`Replicas`尽量分配在不同`Rack`上面

为了了解该算法博主百度、Google了一番，发现大部分博客对该算法的解释如下

> - `将所有Broker和待分配的Partition排序`
>
> - `将第i个Partition分配到第(i mod n)个Broker上`
> - `将第i个Partition的第j个Replica分配到第（(i + j) mod n）个Broker上`

一眼看过去好像也没啥毛病，不过仔细想想会引发如下问题。参考了[[Kafka如何将分区放置到不同的Broker中]](https://www.iteblog.com/archives/2219.html)

> - 所有主题的第一个分区都是存放在第一个`Broker`上导致第一个`Broker`上的分区总数多于其他`Broker`
> - 如果主题的分区数多于`Broker`的个数，多于的分区都是倾向于将分区发放置在前几个`Broker`上

带着上诉的问题打开`Kafka`副本分配部分源码，主要分为无机架感知副本分配和机架感知副本分配

## 无机架感知副本分配

- `assignReplicasToBrokersRackUnaware`无机架感知算法

  ```scala
  private def assignReplicasToBrokersRackUnaware(nPartitions: Int, replicationFactor: Int, brokerList: Seq[Int], fixedStartIndex: Int, startPartitionId: Int): Map[Int, Seq[Int]] = {
    // 副本分配结果集
    val ret = mutable.Map[Int, Seq[Int]]()
    // 当前存在的Broker集合
    val brokerArray = brokerList.toArray
    // 选择起始Broker进行分配
    val startIndex = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
    // 选择起始Partition进行分配
    var currentPartitionId = math.max(0, startPartitionId)
    // 指定了副本间隔目的是为了更均匀的将副本分配到不同的Broker上
    var nextReplicaShift = if (fixedStartIndex >= 0) fixedStartIndex else rand.nextInt(brokerArray.length)
    for (_ <- 0 until nPartitions) {
      if (currentPartitionId > 0 && (currentPartitionId % brokerArray.length == 0))
        // 经过一轮分配递增nextReplicaShift
        nextReplicaShift += 1
      // 当前分区优先副本分配结果
      val firstReplicaIndex = (currentPartitionId + startIndex) % brokerArray.length
      val replicaBuffer = mutable.ArrayBuffer(brokerArray(firstReplicaIndex))
      for (j <- 0 until replicationFactor - 1)
        // 分配当前分区其他的副本
        replicaBuffer += brokerArray(replicaIndex(firstReplicaIndex, nextReplicaShift, j, brokerArray.length))
      ret.put(currentPartitionId, replicaBuffer)
      currentPartitionId += 1
    }
    ret
  }
  ```
  - 若未指定`fixedStartIndex`参数则使用`Round-Robin`指定`fixedStartIndex`，`fixedStartIndex`副作为第一个`Partition`的优先副本ID，后续`Partition`的优先副本ID在`fixedStartIndex`基础上递增
  - `Partition`的`Follower`副本在`fixedStartIndex`基础上递增【需关注`nextReplicaShift`参数】

即使在每行代码上都写了注释，但是其实看起来还是很抽象。所以我用一组测试数据来对上诉结论进行说明

【`nPartitions:13`、`replicationFactor:3`、`brokerRackMapping:{0,1,2,3}`、`fixedStartIndex:4`】

| 副本 |     分区      |
| :--: | :-----------: |
|  0   | 【3   0   1】 |
|  1   | 【0   1   2】 |
|  2   | 【1   2   3】 |
|  3   | 【2   3   0】 |
|  4   | 【3   1   2】 |
|  5   | 【0   2   3】 |
|  6   | 【1   3   0】 |
|  7   | 【2   0   1】 |
|  8   | 【3   2   0】 |
|  9   | 【0   3   1】 |
|  10  | 【1   0   2】 |
|  11  | 【2   1   3】 |
|  12  | 【3   0   1】 |

- 先看分区部分的第一列数据【3，0，1，2，3，0，1，2，3，0，1，2，3】从第一个分区开始数值递增
- 再看分区的第二列【分区第二列的值是根据第一列的值确定的 与`nextReplicaShift`的值有关】
  - 副本0~3分区第二列和第一列相差1
  - 副本4~7分区第二列和第一列相差2
  - 副本8-11分区第二列和第一列相差3
  - 副本11-12分区第二列和第一列相差4
- 再看分区的第三列【第三列数据是计算与第二列的数据差，计算方式和第二列一致】

## 机架感知副本分配

待完成...