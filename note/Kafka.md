## Kafka 基本介绍

Kafka 是一个基于发布/订阅的分布式消息系统。目前 Kafka定位为一个分布式流式处理平台，它以高吞吐、可持久化、可水平扩展、支持流数据处理等多种特性而被广泛使用

消息系统的基本功能

1. 消息的发送和接收；需要涉及到网络通信就一定会涉及到NIO
2. 消息中心的消息存储（持久化/非持久化）
3. 消息的序列化和反序列化
4. 是否跨语言
5. 消息的确认机制，如何避免消息重发

高级功能

1. 消息的有序性
2. 是否支持事务消息
3. 消息收发的性能，对高并发大数据量的支持
4. 是否支持集群
5. 消息的可靠性存储
6. 是否支持多协议

### **三大角色**

* 消息系统：消息系统的三大功能：解耦、异步、削峰；同时， Kafka提供了消息顺序性保障及回溯消费的功能
* 存储系统：Kafka把消息持久化到磁盘，相比其他基于内存存储的系统而言，有效地降低了数据丢失的风险
* 流式处理平台：Kafka不仅为每个流行的流式处理框架提供了可靠的数据来源，还提供了一个完整的流式处理类库，比如窗口、连接、变换和聚合等各类操作 



####   Kafka的特性

- 高吞吐量、低延迟：kafka每秒可以处理几十万条消息，它的延迟最低只有几毫秒，每个topic可以分多个partition, consumer group 对partition进行consume操作。
- 可扩展性：kafka集群支持热扩展
- 持久性、可靠性：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失
- 容错性：允许集群中节点失败（若副本数量为n,则允许n-1个节点失败）
- 高并发：支持数千个客户端同时读写

Kafka的使用场景
- 日志收集：一个公司可以用Kafka可以收集各种服务的log，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等。
- 消息系统：解耦和生产者和消费者、缓存消息等。
- 用户活动跟踪：Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到hadoop、数据仓库中做离线分析和挖掘。
- 运营指标：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。
- 流式处理：比如spark streaming和storm


应用场景

1. 活动跟踪：Kafka最初的使用场景是跟踪用户的活动。网站用户与前端应用程序发生交互，前端应用程序生成用户活动相关的消息。
2. 传递消息：Kafka的另一个基本用途是传递消息。应用程序向用户发送通知（比如邮件）就是通过传递消息来实现的。这些应用程序组件可以生成消息，而不需要关心消息的格式，也不需要关心消息是如何被发送的。
3. 度量指标和日志记录：Kafka也可以用于收集应用程序和系统度量指标以及日志
4. 提交日志：Kafka的基本概念来源于提交日志，所以使用Kafka作为提交日志是件顺理成章的事。
5. 流处理：流处理是又一个能提供多种类型应用程序的领域。



### 基本概念

**Producer**

生产者，生产消息的一方，负责生产消息，然后发送到Kafka中

**Consumer**

消费者，接收消息的一方，连接到Kafka并接收消息，然后对其进行业务处理

**Broker**

服务代理节点，Kafka集群包含一个或多个服务器，这种服务器被称为broker，负责消息的存储和转发

**Topic**

消息类别，Kafka 按照topic 来分类消息，topic 是一个逻辑上的概念，由多个partition组成

**Partition**

topic 的分区，一个topic 可以包含多个partition，topic 消息保存在各个partition 上

**offset**

消息在日志中的位置，可以理解是消息在partition 上的偏移量，也是代表该消息的唯一序号

offset是消息在分区中的唯一标识， Kafka通过它来保证消息在分区内的顺序性，不过offset并不跨越分区，所以Kafka保证的是分区有序而不是主题有序

**Consumer Group**

消费者分组，每个Consumer 必须属于一个group

### 架构

![Kafka架构](img/Kafka架构.png)

#### 副本

为了提高容灾能力，Kafka引入了多副本机制，每个分区有多个副本，副本之间是一主多从的关系

生产者和消费者只与 leader副本进行交互，而 follower副本只负责消息的同步，很多时候 follower副本中的消息相对 leader副本而言会有一定的滞后

**AR**（ Assigned Replicas）: 分区中的所有副本统称为AR

**ISR**（Im- Sync Replicas）: 与leader副本保持一定程度的同步的副本集合（包含leader副本）

**OSR**（Out- of-Sync Replicas）: 与 leader副本同步滞后过多的副本集合

AR=ISR+OSR，在正常情况下，所有的 follower副本都应该与 leader副本保持一定程度的同步，即AR=ISR，OSR集合为空

![kafka-offset](img/kafka-offset.png)

**HW**：高水位，正在进行副本同步的offset，该offset之前的offset已经同步完成，可以被Consumer消费

**LEO**：低水位，Leader最大的offset+1，下一条待写入的消息的位置

## 生产者

~~~java
public class KafkaProducerTest { 
    // 地址
    public static final String brokerList = "192.168.7.101:9092,192.168.7.102:9092,192.168.7.103:9092";
    // 生产 topic
    public static final String topic = "topic1";
	// 初始化 配置
    public static Properties initConfig() {
        Properties properties = new Properties();
        // 生产者连接Kafka集群所需的 broker 地址清单
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);
        // 对 key 进行序列化
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        // 对 value 进行序列化
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        // 生产者id
        properties.put(ProducerConfig.CLIENT_ID_CONFIG,"producer1");
        return properties;
    } 
    public static void main(String[] args) {
        Properties properties = initConfig();
        // 根据 properties 创建生产者
        KafkaProducer<String, String> producer = new KafkaProducer<>(properties);
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, "hello, Kafka1 ");
        try {
            producer.send(record);
            TimeUnit.SECONDS.sleep(2);
        } catch(Exception e){
            e.printStackTrace(); 
        }
    }
}
~~~



### 消息发送

#### **ProducerRecord**

~~~java
public class ProducerRecord<K, V> { 
    private final String topic; // 主题
    private final Integer partition; // 分区，可以指定分区 发送消息
    private final Headers headers; // 头信息，可以实现自定义的某些功能，默认未序列化
    private final K key; // key
    private final V value; // value 发送的信息
    private final Long timestamp; // 时间戳

    public ProducerRecord(String topic, Integer partition, Long timestamp, K key, V value, 
                          Iterable<Header> headers);
    public ProducerRecord(String topic, Integer partition, Long timestamp, K key, V value);
    public ProducerRecord(String topic, Integer partition, K key, V value, Iterable<Header> headers);
    public ProducerRecord(String topic, Integer partition, K key, V value);
    public ProducerRecord(String topic, K key, V value);
    public ProducerRecord(String topic, V value);
}
~~~

#### send

创建生产者实例和构建消息之后，就可以开始发送消息了。发送消息主要有三种模式

发后即忘(fire-and-forget)、同步(sync)及异步Casync)

~~~java
// 发送消息的
ProducerRecord<Integer, String> record = new ProducerRecord<>(topic, 100,"hello, Kafka1 "); 
// 1.发后即忘,本身是异步的,不能保证每次都发送成功
producer.send(record);
// 2.同步,Future对象的get()方法如果没执行完会阻塞
Future<RecordMetadata> future = producer.send(record);
RecordMetadata data = future.get(); 
// 3.异步,通过Callback进行回调
producer.send(record, new Callback() {
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        // 处理回调数据
    }
});
// lambda 表达式
producer.send(record,(metadata, exception)-> { }) ;
~~~

虽然send方法本身是异步的，但由于Future的get()方法在执行完之前阻塞的，所以get()方法何时调用就会存在问题

Callback的方式非常简洁明了，Kafka有响应时就会回调，要么发送成功，要么抛出异常

### 序列化

生产者需要用序列化器(Serializer)把对象转换成字节数组才能通过网络发送给Kafka。消费者需要用反序列化器(Deserializer)把从Kafka中收到的字节数组转换成相应的对象

~~~java
// 序列化接口
public interface Serializer<T> extends Closeable {
    // 读取配置信息
    default void configure(Map<String, ?> configs, boolean isKey) {
    } 
    // 序列化方法，将对象转换成字节数组
    byte[] serialize(String var1, T var2); 
    
    default byte[] serialize(String topic, Headers headers, T data) {
        return this.serialize(topic, data);
    } 
    default void close() {
    }
}
~~~

### 分区器

消息在通过send()方法发往broker 的过程中，有可能需要经过拦截器(Interceptor)、序列化器(Serializer)和分区器(Partitioner)的一系列作用之后才能被真正地发往broker

分区器(Partitioner)的作用是通过指定的算法来控制消息发送到broker 的哪个分区

~~~java
public interface Partitioner extends Configurable, Closeable {
    // 确定分区号
    int partition(String var1, Object var2, byte[] var3, Object var4, byte[] var5, Cluster var6);
    void close();
}

// 默认的分区器
public class DefaultPartitioner implements Partitioner {
    private final ConcurrentMap<String, AtomicInteger> topicCounterMap = new ConcurrentHashMap(); 
    // 确定分区号
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, 
                         Cluster cluster) {
        // 获取集群中当前topic 的所以分区
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        // 分区数量
        int numPartitions = partitions.size();
        if (keyBytes == null) {
            // 每次自增，实现轮询
            int nextValue = this.nextValue(topic);
            // 获取可用分区数量
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                // nextValue%可用分区数量 获取分区编号
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return ((PartitionInfo)availablePartitions.get(part)).partition();
            } else {
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
            // 如果key不为空，通过计算hash然后取模获取分区
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
	// 通过原子类试下自增 
    private int nextValue(String topic) {
        AtomicInteger counter = (AtomicInteger)this.topicCounterMap.get(topic);
        if (null == counter) {
            counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
            AtomicInteger currentCounter = (AtomicInteger)this.topicCounterMap.putIfAbsent(topic, counter);
            if (currentCounter != null) {
                counter = currentCounter;
            }
        } 
        return counter.getAndIncrement();
    } 
}
~~~



### 生产者拦截器

生产者拦截器既可以用来在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的消息、修改消息的内容等，也可以用来在发送回调逻辑前做一些定制化的需求，比如统计类工作。

~~~java
public interface ProducerInterceptor<K, V> extends Configurable {
    // 通过onSend方法对消息进行处理
    ProducerRecord<K, V> onSend(ProducerRecord<K, V> var1);
	// 在消息被应答之前或消息发送失败时调用 onAcknowledgement方法， 优先于用户设定的Callback
    void onAcknowledgement(RecordMetadata var1, Exception var2);

    void close();
}
~~~



![kafka-producerwork](img/kafka-producerwork.png)

整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和 Sender线程（发送线程）。在主线程中由 KafkaProducer创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器（ RecordAccumulator，也称为消息收集器）中。 Sender线程负责从RecordAccumulator中获取消息并将其发送到 Kafka中。





## 消费者

**消费者(Consumer)**：负责订阅Kafka 中的主题(Topic), 并且从订阅的主题上拉取消息。

**消费组(Consumer Group)**：每个消费者都有一个消费组，当消息发布到Kafka的Topic后，只会被投递给订阅它的每个消费组中的一个消费者，而不是全部消费者

如果所有的消费者都隶属于同一个消费组，那么所有的消息都会被均衡地投递给每一个消费者，即每条消息只会被一个消费者处理，这就相当于点对点模式的应用。

如果所有的消费者都隶属于不同的消费组，那么所有的消息都会被广播给所有的消费者，即每条消息会被所有的消费者处理，这就相当于发布/订阅模式的应用。

~~~java
public class KafkaConsumerTest {
	// 地址
    public static final String brokerList = "192.168.7.101:9092,192.168.7.102:9092,192.168.7.103:9092";
    public static final String topic = "topic1"; // 消费topic
    public static final String groupid = "group1"; // 消费group
    public static Properties initConfig() {
        Properties properties = new Properties();
        // key - value反序列化
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
		// 消费者连接Kafka集群所需的 broker 地址清单
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList); 
        properties.put(ConsumerConfig.CLIENT_ID_CONFIG,"consumer1");
        // 消费者group
        properties.put(ConsumerConfig.GROUP_ID_CONFIG,groupid);
        return properties;
    }
    public static void main(String[] args) {
        Properties properties = initConfig();
        KafkaConsumer<Integer, String> consumer= new KafkaConsumer<>(properties);
        consumer.subscribe(Collections.singleton(topic));// 添加topic，可以添加多个
        try {
            while (true) { 
                ConsumerRecords<Integer,String> consumerRecords=consumer.poll(Duration.ofSeconds(1));
                consumerRecords.forEach(record->{ 
                    System.out.println(record.key()+"->"+record.value()+"->"+record.offset());
                });
            }
        } catch(Exception e) {
            log.error(e.getMessage(), e);
        } finally{
            consumer.close(); 
        }
    }
}
~~~

### 订阅主题

~~~java
// 使用Collection集合订阅主题
public void subscribe(Collection<String> topics) {
    this.subscribe((Collection)topics, new NoOpConsumerRebalanceListener());
}
// 使用正则表达式 Pattern 订阅主题
public void subscribe(Pattern pattern) {
    this.subscribe((Pattern)pattern, new NoOpConsumerRebalanceListener());
} 
// new NoOpConsumerRebalanceListener() 设置相应的再均衡监听器 
~~~

### 反序列化

~~~java
public interface Deserializer<T> extends Closeable {
     // 读取配置信息
    default void configure(Map<String, ?> configs, boolean isKey) {
    }
     // 反序列化方法，将字节数组转换成对象
    T deserialize(String var1, byte[] var2); 
    default T deserialize(String topic, Headers headers, byte[] data) {
        return this.deserialize(topic, data);
    }
    default void close() {
    }
}
~~~



### 消息消费

Kafka中的消费是基于poll模式的，Kafka中的消息消费是一个不断轮询的过程，消费者需要不断调用poll()方法，而poll()方法返回的是所订阅的主题（分区）上的一组消息

~~~java
// 1s 轮询一次
ConsumerRecords<Integer,String> consumerRecords=consumer.poll(Duration.ofSeconds(1));
consumerRecords.forEach(record->{ 
    System.out.println(record.key()+"->"+record.value()+"->"+record.offset());
});
~~~

消费端信息ConsumerRecord

~~~java
public class ConsumerRecord<K, V> {  
    private final String topic; // 主题
    private final int partition; // 分区
    private final long offset; // 偏移量
    private final long timestamp;// 时间戳
    private final TimestampType timestampType;
    private final int serializedKeySize;
    private final int serializedValueSize;
    private final Headers headers;
    private final K key;
    private final V value;
    private final Optional<Integer> leaderEpoch;
    private volatile Long checksum;
}
~~~

#### 位移提交

Kafka的每个分区的每条消息都有唯一的offset

1. poll()返回没有被消费过的消息集合
2. 消费位移存储在Kafka内部主题__consumer_offsets中
3. 把消费位移存储起来的过程称为提交
4. 消费位移提交的offset是当前消费的offset+1，表示下一条需要拉取的消息的位置

自动提交会再次重复消费和消费丢失

poll() ->[x+1,x+7] 当前处理的消息x+5

* 拉取到消息立刻提交x+8，如果x+5异常，重启后从x+8开始消费，x+5到x+7的消息丢失
* 消费完成后提交x+8，如果x+5异常，重启后从x+1开始消费，x+1到x+4的消息重复消费

### 消费者拦截器

~~~java
public interface ConsumerInterceptor<K, V> extends Configurable, AutoCloseable {
    ConsumerRecords<K, V> onConsume(ConsumerRecords<K, V> var1); 
    void onCommit(Map<TopicPartition, OffsetAndMetadata> var1); 
    void close();
}
~~~

### 再均衡

再均衡是指分区的所属权从一个消费者转移到另一消费者的行为，它为消费组具备高可用性和伸缩性提供保障，使我们可以方便又安全地删除或添加消费者；再均衡发生期间，消费组内的消费者是无法读取消息的，在再均衡发生期间的这一小段时间内，消费组会变得不可用 

## 主题和分区

**Topic**: 消息类别，Kafka 按照topic 来分类消息

**Partition**: topic 的分区，一个topic 可以包含多个partition，topic 消息保存在各个partition 上

**replication-factor**: 副本因子，用来设置Topic的副本数。每个Topic可以有多个副本，副本位于集群中不同的broker上，保证有效的数据冗余，防止宕机造成的数据丢失；同时副本的数量不能超过broker的数量，否则topic创建失败。

topic和partition都是提供给上层用户的抽象，而在副本层面或更加确切地说是Log层面才有实际物理上的存在

![topic-partition](img/topic-partition.png)

### 优先副本

分区使用多副本机制来提升可靠性，但只有`leader`副本对外提供读写服务，而`follower`副本只负责在内部进行消息的同步。

* 如果一个分区的 leader副本不可用，那么就意味着整个分区变得不可用
* 此时就需要 Kafka从剩余的 follower副本中挑选一个新的 leader副本来继续对外提供服务



### 分区重分配

当Kafka集群中的broker节点数发生变化时，会出现以下问题

1、当集群中的一个节点突然宕机下线时

* 如果节点上的分区是单副本的，那么这些分区就不可用，在节点恢复前，相应的数据也就处于丢失状态；
* 如果节点上的分区是多副本的，那么该节点上的 leader副本的角色会转交到其他 follower副本中

2、当集群中新增broker节点时，只要新建topic的分区会分配到该节点，之前的topic分区不会分配到新的broker节点，会导致之前的topic负载不均衡

为了解决上述的问题，使分区副本合理的分配，Kafka提供了分区冲分配的功能

### Leader选举

当topic的分区数发生变化（新分区上线或者leader副本宕机）时，分区需要选举一个新的leader对外提供服务，对应的选举策略为OfflinePartition Leader Election Strategy

基本思路是按照AR集合中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中

* 注：这里是根据AR的顺序，而不是ISR的顺序进行选举





## Kafka 存储原理

### 存储结构

![文件结构](img/文件结构.png)

为了便于消息的检索,每个 LogSegment中的日志文件(以".log"为文件后缀)都有对应的两个索引文件

* 偏移量索引文件(以". index"为文件后缀)
* 时间戳索引文件(以".timeindex"为文件后缀)

每个LogSegment都有一个基准偏移量 baseOffset，用来表示当前LogSegment中第一条消息的 offset；偏移量是一个64位的长整型数，日志文件和两个索引文件都是根据基准偏移量( baseOffset)命名的，名称固定为20位数字，没有达到的位数则用0填充

* 例如第一个LogSegment的基准偏移量为0，对应的日志文件为00000000000000000000.log
* `00000000000000000133.1og`文件表示当前LogSegment对应的基准位移是133，该 LogSegment中的第一条消息的偏移量为133，同时反映出第一个 LogSegment中共有133条消息(偏移量从0至132的消息)

### 日志格式

Kafka查看日志信息命令

~~~sh
$ bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/topic-create-3/00000000000000000000.log  --print-data-log

Dumping /tmp/kafka-logs/topic-create-3/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 0 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 0 CreateTime: 1582795316559 size: 81 magic: 2 compresscodec: NONE crc: 1858205989 isvalid: true
| offset: 0 CreateTime: 1582795316559 keysize: 4 valuesize: 9 sequence: -1 headerKeys: [] key:  payload: msg-test2
baseOffset: 1 lastOffset: 1 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false isControl: false position: 81 CreateTime: 1582795317061 size: 81 magic: 2 compresscodec: NONE crc: 1682208138 isvalid: true
| offset: 1 CreateTime: 1582795317061 keysize: 4 valuesize: 9 sequence: -1 headerKeys: [] key:  payload: msg-test3
~~~

#### v0版本

Kafka消息格式的第一个版本为v0版本，在 Kafka0.10.0之前都采用的这个消息格式

![v0格式](img/v0格式.png)

**crc32**(4B)：crc32校验值，校验范围为 magic到value之间

**magic**(1B)：消息格式版本号，此版本的 magic值为0

**attributes**(1B)：消息的属性；总共占1个字节，低3位表示压缩类型：0表示NONE、1表示GZIP、2表示 SNAPPY、3表示LZ4

**key length**(4B)：表示消息的key的长度；如果为-1，则表示没有设置key，即key==null 

**key**：消息的key，可以为null 

**value length**(4B)：实际消息体的长度；如果为-1，则表示消息为空

**value**：消息体，可以为null 

v0版本中一个消息的最小长度为crc32 + magic + attributes + key length + value length = 4B+1B+B+4B+4B=14B；v0版本中一条消息的最小长度为14B

#### v1版本

Kafka从0.10.0~0.11.0版本所使用的消息格式版本为v1，相比v0版本，仅仅多了一个 timestamp字段，表示消息的时间戳

![v1格式](img/v1格式.png)

#### v2版本

v2版本中消息集称为 Record Batch，而不是之前的 Message Set，内部也包含了一条或多条消息

### 日志清理

Kafka将消息存储在磁盘中，随着磁盘占用空间的不断增加，我们需要对消息做一定的清理操作。Kafka提供了两种日志清理策略

1. **日志删除**( Log retention)：按照一定的保留策略直接删除不符合条件的日志分段
2. **日志压缩**( Log Compaction)：针对每个消息的key进行整合，对于有相同key的不同value的消息，只保留最后一个版本.

#### 日志删除

在 Kafka的日志管理器中会有一个专门的日志删除任务，用来周期性地检测和删除不符合保留条件的日志分段文件

##### 基于时间

日志删除任务会检查当前日志文件中是否存在保留时间超过设定的阈值，以此寻找可删除的日志分段文件集合

査找过期的日志分段文件，并不是简单地根据日志分段的最近修改时间 lastModifiedTime来计算的，而是根据日志分段中最大的时间戳 largestTimeStamp来计算的

##### 基于日志大小

检查当前日志的大小是否超过设定的阈值，以此寻找可删除的日志分段的文件集合

##### 基于起始偏移量

基于日志起始偏移量的保留策略的判断依据是某日志分段的下一个日志分段的起始偏移量baseOffset是否小于等于 logStartOffset，若是，则可以删除此日志分段

#### 日志压缩

对于有相同key的不同 value值的消息，只保留最后一个版本；如果应用只关心key对应的最新value值，可以开启Kafka的日志清理功能，Kafka会定期将相同key的消息进行合并，只保留最新的 value值

### 磁盘存储

Kafka依赖于文件系统(磁盘)来存储和缓存消息，利用磁盘的顺序IO来进行快速的读写，同时减少随机IO

#### 页缓存

页缓存是操作系统实现的一种主要的磁盘缓存，用来减少对磁盘IO的操作；通过把磁盘中的数据缓存到内存中，把对磁盘的访问变为对内存的访问。

当一个进程准备读取磁盘上的文件内容时，操作系统会先查看待读取的数据所在的页(page)是否在页缓存(pagecache)中，如果存在(命中)则直接返回数据，从而避免了对物理磁盘的IO操作；如果没有命中，则操作系统会向磁盘发起读取请求并将读取的数据页存入页缓存，之后再将数据返回给进程

#### **顺序写入**

对大多数硬盘，尤其是机械硬盘，顺序IO速度要远大于随机IO速度，Kafka使用顺序IO写入数据，把数据写入每个Partition文件的尾部，但同时导致数据无法删除，所以Kafka无法删除消息

##### Memory Mapped Files

即便是顺序写入硬盘，硬盘的访问速度还是不可能追上内存。所以Kafka的数据并不是实时的写入硬盘，它充分利用了现代操作系统分页存储来利用内存提高I/O效率

#### 零拷贝

零拷贝（Zero-copy）：指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域，这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽

传统模式下，对一个文件进行传输的时候，流程如下：

- 调用 Read 函数，文件数据被 Copy 到内核缓冲区
- Read 函数返回，文件数据从内核缓冲区 Copy 到用户缓冲区
- Write 函数调用，将文件数据从用户缓冲区 Copy 到内核与 Socket 相关的缓冲区
- 数据从 Socket 缓冲区 Copy 到相关协议引擎

文件数据实际上是经过了四次 Copy 操作：

**硬盘—>内核 buf—>用户 buf—>Socket 相关缓冲区—>协议引擎**

基于 Sendfile 实现零拷贝，可以减少Copy次数，同时减少上下文切换

- Sendfile 系统调用，文件数据被 Copy 至内核缓冲区。
- 再从内核缓冲区 Copy 至内核中 Socket 相关的缓冲区。
- 最后再 Socket 相关的缓冲区 Copy 到协议引擎。

