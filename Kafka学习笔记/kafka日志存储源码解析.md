# ��־����

> ��������:��־�Ķ�д���ֶΡ�����͹���
>
> `Log`������ֱ�Ӷ�Ӧ�ڴ����ϵ�һ����־�ļ����Ƕ�Ӧ�����ϵ�һ��Ŀ¼`<topic_name>_<partition_id>`
>
> `Log`��Ϊ���`LogSegment`��һ��`LogSegment`��Ӧ������һ����־�ļ��������ļ���
>
> ��־���ļ�����������Ϊ`[baseOffset].log`��`baseOffset`����־�е�һ����Ϣ��`offset`

## `ByteBufferMessageSet`

> �ײ�ʹ��`ByteBuffer`�������ݣ���Ҫ�ṩ���¹���

- `create`����Ϣ����ָ����ѹ�����ͽ���ѹ��

- 


- ��Ϣѹ��
- ����ѹ����Ϣ

## `MemoryRecordsBuilder`



## `OffsetIndex`

> Ϊ����߲�����Ϣ������`Kafka`Ϊÿһ����־�ļ������2�������ļ�`OffsetIndex`��`TimeIndex`

`OffsetIndex`�����ļ���ʽ:ÿһ��������Ϊ8�ֽڡ�`offset`ռ��4���ֽڣ�`position`ռ��4���ֽڡ�

- `append`���������¼
- `truncateTo`�ض������ļ�
- `lookup`����`offset`С��`targetOffset`������������ַ
- `indexSlotRangeFor`����������ֵ������ӽ�`target`��
- `mmap`�������������ļ���`MappedByteBuffer`

## `LogSegment`

> Ϊ�˷�ֹ`Log`�ļ�����`Log`�зֳɶ����־�ļ���ÿ����־�ļ���Ӧһ��`LogSegment`

- `append`д����־
  - д����Ϣ��д�뵽`FileRecords`�С�
  - �������Ҫ�Ļ�����`OffsetIndex`��`TimeIndex`
- `read`��ȡ��Ϣ
  - ���㿪ʼ��ȡ��Ϣλ��`startPosition`
  - �����ȡ���ȡ���`maxOffset`��`maxPosition`��`maxSize`��ͬ������
- `recover`������־�ļ���д�����ļ�����֤��Ϣ�Ϸ���

## `Log`

ÿ��`Replica`��Ӧһ��`Log`���������������������`segment`�ļ���`segments`:`LogSegment`��`baseOffset`��Ϊ`key`��`LogSegment`�Ķ�����Ϊ`value`���������ļ��������Ǽ�����Ҫ����

```reStructuredText
- logEndOffset��һ����Ϣ��ƫ����

- nextOffsetMetadata��һ��ƫ��Ԫ����
  - ��һ����Ϣ��ƫ����
  - ��־�ֶεĻ�׼ƫ����
  - ����־�ֶ��ϵ�position
  
- activeSegment�����־�ֶΡ���֤˳��д�롿
```

`append`��`Log`��һ������Ҫ�ķ���������־��д�롾׷����־��

- `analyzeAndValidateRecords`У����Ϣ�����㲿������
- `trimInvalidBytes`ɾ��������Ϣ����Ч����Ϣ

- `validateMessagesAndAssignOffsets`Ϊÿ����Ϣ���þ���ƫ������ʱ���
- `maybeRoll`�Ƿ���Ҫ�����µ�`activeSegment`
  - ���δ�׷�ӵ���Ϣ���ϴ�С��������`LogSegment`����󳤶�
  - �����ϴ���־�ֶε�ʱ���Ƿ�ﵽ�����õ���ֵ
  - �����ļ�����/��Ϣ���λ�ƺͻ���λ�ƴ�С��ֵ����`int`���ֵ
- `segment.append`��`segment`��׷������
- ����`logEndOffset`���ж��Ƿ���Ҫˢ�´��̡������Ҫ�Ļ�����`flush()`����ˢ�����̡�

`roll`�������ض������󽫻ᴴ���µ�`segment`���������������ݡ�������־��

- �����µ���־�ļ��������ļ�
- ������Ӧ��`segment`������ӵ�������
- ����`activeSegment.baseOffset`��`activeSegment.size`
- ʹ��`KafkaScheduler`ִ��`flush-log`����`KafkaScheduler`�����ǽ�`fun`��װ��`Runable`��

`flush`��`recoverPoint~LEO`֮�����Ϣ����ˢ�µ����̲��޸�`recoverPoint`��ֵ

- ͨ����`segments`�����������`recoverPoint`��`offset`֮���`LogSegment`����
- ����`segment.flush`������ˢ�µ����̡���֤���ݳ־��ԡ�

`read`��־�Ķ�ȡ����`nextOffsetMetadata`����ɷ����ֲ�������֤�̰߳�ȫ��

- ��־�߽���

- �����Ƿ���`activeSegment`����`maxPosition`

  ```reStructuredText
  ���Fetch����պ÷�����activeSegment�ϣ������Fetch����ͬʱ�������nextOffsetMetadata���²���ʱ,���ܻᵼ�·���OffsetOutOfRangeException�쳣��Ϊ�˽��������������ܶ�ȡ�����λ���Ƕ�Ӧ������λ��exposedPos������activeSegment��LEO
  ```

- ����`segment.read`������ȡ���ݣ����û��ȡ���������ȡ��һ��`LogSegment`

##  `LogManager`

- ��ʱ����
  - `cleanupLogs`����`LogSegment`������`LogSegment`�Ĵ��ʱ��/����`Log`�Ĵ�С��
    - ������־����ʱ�䡾��ǰʱ��-����޸�ʱ��<`retention.ms`��
    - ����`Log`��С��`diff - segment.size >= 0` `diff`Ϊ������־�洢���ֵ���ֵ����ݡ�
    - һ�¸�`Segment`��`baseOffset`����С��`logStartOffset`
  - `flushDirtyLogs`ˢ����־�ļ�����־δˢ��ʱ��>��`Log`��`flush.ms`������ָ����ʱ����
  - `checkpointLogRecoveryOffsets`����`RecoveryPointCheckPoint`�ļ�
  - `checkpointLogStartOffsets`����`checkpoint`��`start offset`�������ȡ����ɾ������
  - `deleteLogs`����ɾ��`logsToBeDeleted`�б��е�����
    - `LogManager`����ʱ��ɨ�����з���Ŀ¼����β��`-delete`�ķ������뵽`logsToBeDeleted`�С�����Ҫ�Ȱѷ���Ŀ¼���һ���ں�׺����`-delete`��ʾ�÷���׼��ɾ���ˣ����������Է�ֹ���ɾ��ʱ��û����崻��´�����ʱ����ɨ��`-delete`��β�ķ�����ɾ����
    - ������ɾ����ʱ���ߵĶ����첽ɾ�����Ի��ȱ����뵽`logsToBeDeleted`�еȴ�ɾ��
- ��ʼ������
  - `createAndValidateLogDirs`�����־Ŀ¼
    - ȷ������Ŀ¼��û���ظ�������Ŀ¼
    - ȷ��������Ŀ¼���������ѻ�Ŀ¼��
    - ����Ŀ¼�����ڵĻ��ʹ�����Ӧ��Ŀ¼
    - ���ÿ��Ŀ¼·���Ƿ��ǿɶ���
  - `loadLogs`����������־����
    - Ϊÿ��`Log`Ŀ¼����һ����`recoveryThreadsPerDataDir`���߳������̳߳�
    - ���`Broker`�ϴιر��Ƿ�����������`Broker`״̬����`Broker`�����ر�ʱ�ᴴ��һ��`.kafka_cleanshutdown`�ļ����������ͨ�����ļ������жϵġ�
    - ����ÿ��`Log`��`recoveryPoints`�����ڻָ���
    - Ϊÿ��`Log`����һ���ָ����񽻸��̳߳ش���
    - ���߳������ȴ������ָ̻߳�������ɲ��ر�֮ǰ�������̳߳�

## �����ļ�

- `Kafka`����ʱ��ȡ�����ļ���ÿ��������Ӧ�ļ���`CheckPoint`��Ϊ��־�Ļָ���`RecoveryPoint`
- ��Ϣ׷�ӵ�������Ӧ����־����ˢ����־ʱ�����µ�ƫ������Ϊ��־�ļ���(ˢ����־ʱ����¼���λ��)
- `LogManager`��ʱ��ȡ������־���㣬��д��ȫ�ּ����ļ�(��ʱ�������λ�ø��µ������ļ���)



## `LogCompact`

> ��־ѹ��:�˴�����־ѹ������ָͨ��ѹ���㷨����־����ѹ�������Ƕ��ظ���־�����������ﵽĿ�ġ�����־��������л������ظ���`key`��������`Redis`��־��д�����������һЩ`segment`�ļ���С�ͻ��С����ʱ��`kafka`�Ὣ��ЩС���ļ��ٺϲ���һ�����`segment`�ļ�

![��־ѹ��ʾ��ͼ](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/Kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E6%97%A5%E5%BF%97%E5%8E%8B%E7%BC%A9%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

> - `cleaner-offset-checkpoint`��¼��`cleanerCheckPoint`(��ǰ����������)
>
> - ��δ�����`segment`���ҳ�����������ǲ���`segment`
>   - `activeSegment`����������
>   - `min.compaction.lag.ms`����`segment`����һ����¼�Ĳ���ʱ���Ƿ��Ѿ�������С����ʱ��
> - ���ݿ��������`segment`����`SkimpyOffsetMap`��������һ��`key`��`offset`��ӳ���ϵ�Ĺ�ϣ��
> - �ٱ����������ֺͿ��������ֵ�`segment`��ÿһ����־������`SkimpyOffsetMap`���ж��Ƿ���
> - `log.cleaner.delete.retention.ms`����`value`Ϊ`null`����־`kafka`��������־ΪĹ����Ϣ����ִ����־����ʱ��ɾ�����ڵ�Ĺ����Ϣ��Ĺ����Ϣ�ı���ʱ����������ֵ����һ��`segment`�й�ϵ

![��־�ϲ�ʾ��ͼ](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/Kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E6%97%A5%E5%BF%97%E5%90%88%E5%B9%B6%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

- `Kafka`����־�ļ�����ѹ����Ҫ����������������
  - `log.segments.bytes(Ĭ��ֵ��1G)`�ϲ����`segment`��С������`segmentSize`
  - `log.index.interval.bytes(Ĭ��ֵΪ10MB)`�ϲ��������ļ�ռ�ô�С֮�Ͳ�����`maxIndexSize`

### `LogToClean`

`LogToClean`������ʾҪ�������`Log`��������������Ϣ

- `firstDirtyOffset`��ʾ����������ʼ�㣬��ǰ�ߵ�`offset`������������������`message`��`key`�ĺϲ�

- `cleanBytes` `[0, firstDirtyOffset)`���ֽ������������ֽ�����
- `cleanableBytes` `[firstDirtyOffset, firstUncleanableOffset)`���ֽ�������Ҫ�������ֽ�����
- `totalBytes`  �ܵ��ֽ�������Ҫ����Ĳ���+����Ҫ����Ĳ��֡�
- `cleanableRatio` ��Ҫ�����`log`�ı���,���ֵԽ��Խ���ܱ����ѡ��������

### `CleanerThread`

> `CleanerThread`�̳���`ShutdownableThread`�����ĺ�����������`doWork`����ɵ�

- `cleanOrSleep`ִ������������ӳ�ִ����������
  - ͨ��`LogCleanerManager.grabFilthiestCompactedLog`�����Ƿ�����Ҫ������־ѹ����`Log`
  - ����������`cleaner.clean`ִ���������񣬷����˱�`15S`�����ִ�и÷���

### `LogCleaner`

![��־ѹ���ϲ�����](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/Kafka%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E6%97%A5%E5%BF%97%E5%8E%8B%E7%BC%A9%E5%90%88%E5%B9%B6%E8%BF%87%E7%A8%8B.png)

- `doClean`������־������
  - `buildOffsetMap`��`firstDirtyOffset`��ʼ����`LogSegment`���`OffsetMap`
    - ��־���е���Ϣ��������`key`�ģ���Ϊ��־ѹ�����Ǹ���`key`��ʵ�ֵ�
    - ���һ����Ϣ�Ĵ�С�ȶ���������СҪ��Ļ���ô��Ҫ����`growBuffers`�����ݶ������������ɴ���Ϣ�������ڹ�����`mapping`֮��Ҫ�ָ����������Ĵ�С����
  - `groupSegmentsBySize`��0��`endOffset`����Ϣ���з��顾����ĵ�λ��`LogSegment`��
    - `log.segments.bytes(Ĭ��ֵ��1G)`�ϲ����`segment`��С������`segmentSize`
    - `log.index.interval.bytes(Ĭ��ֵΪ10MB)`�ϲ��������ļ�ռ�ô�С֮�Ͳ�����`maxIndexSize`
  - `cleanSegments`��������`LogSegment`���Ͻ�����־ѹ��
    - ��ȡ����
    - �ж��Ƿ���Ҫ��������
      - ����Ϣ���Ƿ���`Key`
      - `OffsetMap`���Ƿ�����ͬ`Key`��`Offset`�������Ϣ
      - ����Ϣ��"ɾ�����"�Ҵ�`LogSegment`�е�ɾ����ǿ��԰�ȫɾ��
    - ���µ�`Segment`�滻�ϵ�`Segment`
      - ���µ�`segment`�ļ���׺��`.cleaned`��������`.swap`�ļ�
      - ���µ�`segment`���뵽`log.segments`��, ���ϵ�һ�����Ƴ�
      - Ȼ���ϵ��ļ�������Ϊ`.deleted`�ļ�, �첽�߳���ɾ���ϵ�`segment`�ļ�
      - ����µ�`segment`������ȥ��`swap`��׺���`clean`

### `LogCleanerManager`

- ״̬ת��

  > ����ʼ������־ѹ������ʱ���Ƚ���`LogCleaningInProgress`��״̬��ѹ��������Ա���ͣ�����ʱ����`LogCleaningPaused`״̬��ѹ�����������ж����Ƚ���`LogCleaningAborted`״̬�ȴ�`Cleaner`�߳̽�����������ֹ��Ȼ�����`LogCleaningPaused`״̬������`LogCleaningPaused`״̬����־�����ٱ�ѹ����ֱ���������ָ̻߳���ѹ��״̬

- `grabFilthiestCompactedLog`ѡȡ��һ����Ҫ������־ѹ����`Log`

  - �����е�`Log`�в�����`LogToClean`�����б�
    - ���˵�`cleanup.policy`������Ϊ`delete`��`Log`
    - ���˵�`inProgress`����״̬��`Log`��������������
  - �ӻ�ȡ��`LogToClean`�б��й��˳�`cleanableRatio`����`config`�����õ�������ʵ�`LogToClean`
  - �ӹ��˺��`LogToClean`�б��л�ȡ`cleanableRatio`���ļ�Ϊ��ǰ����Ҫ�������`LogToClean`

- `updateCheckpoints`ÿ��������Ҫ���µ�ǰ�Ѿ�������λ��, ��¼��`cleaner-offset-checkpoint`�ļ�