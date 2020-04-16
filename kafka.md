# KAFKA

1. Kafka中的ISR(InSyncRepli)、OSR(OutSyncRepli)、AR(AllRepli)又代表什么？

    ```text
    为保证 producer 发送的数据，能可靠的发送到指定的 topic，topic 的每个 partition 收到producer 发送的数据后，都需要向 producer 发送 ack（acknowledgement 确认收到），如果producer 收到 ack，就会进行下一轮的发送，否则重新发送数据。
    ```

    | 方案                      | 优点                                               | 缺点                                                |
    | ------------------------- | -------------------------------------------------- | --------------------------------------------------- |
    | 半数以上完成同步，发送ack | 延迟低                                             | 选举新的leader时，容忍n台节点的故障，需要2n+1个副本 |
    | 全部完成同步，发送ack     | 选举新的leader时，容忍n台节点的故障，需要n+1个副本 | 延迟高                                              |

    ```text
    Kafka 选择了第二种方案，原因如下：
    1.同样为了容忍 n 台节点的故障，第一种方案需要 2n+1 个副本，而第二种方案只需要 n+1
    个副本，而 Kafka 的每个分区都有大量的数据，第一种方案会造成大量数据的冗余。
    2.虽然第二种方案的网络延迟会比较高，但网络延迟对 Kafka 的影响较小。
    
    采用第二种方案之后，设想以下情景：leader 收到数据，所有 follower 都开始同步数据，
    但有一个 follower，因为某种故障，迟迟不能与 leader 进行同步，那 leader 就要一直等下去，
    直到它完成同步，才能发送 ack。这个问题怎么解决呢？
    Leader 维护了一个动态的 in-sync replica set (ISR)，意为和 leader 保持同步的 follower 集
    合。当 ISR 中的 follower 完成数据的同步之后，leader 就会给 follower 发送 ack。如果 follower
    长时间 未 向 leader 同 步 数 据 ， 则 该 follower 将 被 踢 出 ISR ， 该 时 间 阈 值 由replica.lag.time.max.ms 参数设定。
   Leader 发生故障之后，就会从 ISR 中选举新的 leader。
    ```

2. Kafka中的HW、LEO等分别代表什么？
```text
LEO(Log End Offset):每个副本的最后一个offset
HW(Hgh Watermark):所有副本里最小的LEO

（1）follower 故障
follower 发生故障后会被临时踢出 ISR，待该 follower 恢复后，follower 会读取本地磁盘
记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 leader 进行同步。
等该 follower 的 LEO 大于等于该 Partition 的 HW，即 follower 追上 leader 之后，就可以重
新加入 ISR 了。
（2）leader 故障
leader 发生故障之后，会从 ISR 中选出一个新的 leader，之后，为保证多个副本之间的数据一致性，其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader
同步数据。
注意：这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。
```
3. Kafka中是怎么体现消息顺序性的？

4.Kafka中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？

5.Kafka生产者客户端的整体结构是什么样子的？使用了几个线程来处理？分别是什么？

6.“消费组中的消费者个数如果超过topic的分区，那么就会有消费者消费不到数据”这句话是否正确？

7.消费者提交消费位移时提交的是当前消费到的最新消息的offset还是offset+1？

8.有哪些情形会造成重复消费？

9.那些情景会造成消息漏消费？

10.当你使用kafka-topics.sh创建（删除）了一个topic之后，Kafka背后会执行什么逻辑？

    1）会在zookeeper中的/brokers/topics节点下创建一个新的topic节点，如：/brokers/topics/first
    
    2）触发Controller的监听程序
    
    3）kafka Controller 负责topic的创建工作，并更新metadata cache

11.topic的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？

12.topic的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？

13.Kafka有内部的topic吗？如果有是什么？有什么所用？

14.Kafka分区分配的概念？

15.简述Kafka的日志目录结构？

16.如果我指定了一个offset，Kafka Controller怎么查找到对应的消息？

17.聊一聊Kafka Controller的作用？

18.Kafka中有那些地方需要选举？这些地方的选举策略又有哪些？

19.失效副本是指什么？有那些应对措施？

20.Kafka的哪些设计让它有如此高的性能？