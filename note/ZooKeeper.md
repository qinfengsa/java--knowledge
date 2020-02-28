# ZooKeeper 

ZooKeeper 是一个开源的分布式协调服务，ZooKeeper 的设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用。ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

ZooKeeper 将数据保存在内存中，这也就保证了 高吞吐量和低延迟（但是内存限制了能够存储的容量不太大，此限制也是保持ZNode中存储的数据量较小的进一步原因）。

ZooKeeper 底层其实只提供了两个功能：

* 管理（存储、读取）用户程序提交的数据；
* 为用户程序提供数据节点监听服务

## ZooKeeper 重要概念

### 集群角色

ZooKeeper 本身就是一个分布式程序，为了保证高可用，需要以以集群形态来部署 ZooKeeper；ZooKeeper 集群是一个基于主从复制的高可用集群；有以下3种角色

| 角色     | 工作描述                                                     |
| -------- | ------------------------------------------------------------ |
| Leader   | 1. 事务请求的唯一调度和处理者，保证集群事务处理的顺序性;<br/>2. 集群内部各服务器的调度者; |
| Follower | 1. 处理客户端非事务请求，转发事务请求给 Leader服务器；<br/>2. 参与事务请求 Proposal的投票；<br/>3. 参与 Leader选举投票. |
| Observer | 1. Observer和Follower类似，唯一的区别在于Observer不参与投票和选举；<br/>2. 增加Observer可以在不影响写性能的情况下提高读性能 |



### 会话(Session)

- session是客户端与ZooKeeper 服务端之间建立的长链接；
- ZooKeeper 在一个会话中进行心跳检测来感知客户端链接的存活；
- ZooKeeper 客户端在一个会话中接收来自服务端的watch事件通知；
- ZooKeeper 可以给会话设置超时时间；

### 数据节点(ZNode)

- ZNode是ZooKeeper 树形结构中的数据节点，用于存储数据；
- ZNode分为持久节点和临时节点两种类型：
  - **持久节点(PERSISTENT)**：一旦创建，除非主动调用删除操作，否则一直存储在ZooKeeper 上；
  - **持久有序节点(PERSISTENT_SEQUENTIAL)**：每个节点都会为它的一级子节点维护一个顺序；
  - **临时节点(EPHEMERAL)**：临时节点的生命周期和客户端的会话绑定在一起，当客户端会话失效该节点自动清理；
  - **临时有序节点(EPHEMERAL_SEQUENTIAL)**：在临时节点的基础上多了一个顺序性
  - **CONTAINER**：当子节点都被删除后，Container 也随即删除
  - **PERSISTENT_WITH_TTL和PERSISTENT_SEQUENTIAL_WITH_TTL**：带TTL（time-to-live，存活时间）的永久节点，客户端断开连接后，节点在TTL时间之内没有得到更新并且没有孩子节点，就会被自动删除。

### 版本

- Version：代表当前ZNode的版本；
- CVersion：代表当前ZNode的子节点的版本，子节点发生变化时会增加该版本号的值；
- AVersion：代表当前ZNode的ACL(访问控制)的版本，修改节点的访问控制权限时会增加该版本号的值；



### Watcher

- watcher监听在ZNode节点上；
- 当节点的数据更新或子节点的状态发生变化都会使客户端的watcher得到通知；

### ACL(访问控制)

有以下几种访问控制权限：

- CREATE：创建子节点的权限；
- READ：获取节点数据和子节点列表的权限；
- WRITE：更新节点数据的权限；
- DELETE: 删除子节点的权限；
- ADMIN：设置节点ACL的权限；

## ZooKeeper 特点

- **顺序一致性：** 从同一客户端发起的事务请求，最终将会严格地按照顺序被应用到 ZooKeeper 中去。
- **原子性：** 所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，也就是说，要么整个集群中所有的机器都成功应用了某一个事务，要么都没有应用。
- **单一系统映像 ：** 无论客户端连到哪一个 ZooKeeper 服务器上，其看到的服务端数据模型都是一致的。
- **可靠性：** 一旦一次更改请求被应用，更改的结果就会被持久化，直到被下一次更改覆盖。

## 一致性协议

### 2PC

**2PC**：是Two- Phase commit的缩写，即二阶段提交，是计算机网络尤其是在数据库领域内，为了使基于分布式系统架构下的所有节点在进行事务处理过程中能够保持原子性和一致性而设计的一种算法；

顾名思义，二阶段提交协议是将事务的提交过程分成了两个阶段来进行处理，其执行流程如下

**阶段一：提交事务请求**

类似投票阶段，而且是一票否决

1. **事务询问**

   协调者向所有的参与者发送事务内容，询问是否可以执行事务提交操作，并开始等待各参与者的响应 

2. **执行事务**

   各参与者节点执行事务操作，并将Undo和Redo信息记入事务日志中

3. **各参与者向协调者反馈事务响应**

   如果参与者成功执行了事务，反馈Yes响应，表示事务可以执行；

   如果参与者没有成功执行事务，反馈No响应，表示事务不可以执行。

**阶段二：执行事务提交**

在阶段二中，协调者会根据各参与者的反馈情况来决定最终是否可以进行事务提交操作，包含以下两种可能
**执行事务提交**：所有的反馈都是Yes响应

1. **发送提交请求**

   协调者向所有参与者节点发出 Commit请求

2. **事务提交**

   参与者接收到 Commit请求后，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行期间占用的事务资源

3. **反馈事务提交结果**

   参与者在完成事务提交之后,向协调者发送Ack消息

4. **完成事务**

   协调者接收到所有参与者反馈的Ack消息后，完成事务

**中断事务：**任何一个参与者反馈了No响应，或者等待超时

1. **发送回滚请求**

   协调者向所有参与者节点发出 Rollback请求

2. **事务回滚**

   参与者接收到 Rollback请求后，会利用其在阶段一中记录的Undo信息来执行事务回滚操作，并在完成回滚之后释放在整个事务执行期间占用的资源

3. **反馈事务回滚结果**

   参与者在完成事务回滚之后，向协调者发送Ack消息

4. **中断事务**

   协调者接收到所有参与者反馈的Ack消息后，完成事务中断.

二阶段提交将一个事务的处理过程分为了投票和执行两个阶段，其核心是对每个事务都采用先尝试后提交的处理方式，因此也可以将二阶段提交看作一个强一致性的算法；

**优点：**原理简单，实现方便

**缺点：**同步阻塞、单点问题、数据不一致、容错性

* 同步阻塞：在二阶段提交的执行过程中，所有参与的事务都处于阻塞状态，会导致无法进行其他操作
* 单点问题：协调者在2PC过程中作用非常重要，职责过重，如果协调者出现问题，会导致整个事务操作不可用
* 数据不一致：如果阶段二提交过程中，Commit请求中发生网络异常等问题，会导致只有部分参与者进行事务提交，使得数据和其它未提交的参与者不一致
* 容错性：由于投票阶段使用一票否决制，导致在复杂的网络环境下容错性非常差，只要有一台参与者发生故障或网络问题，整个事务一直失败

### 3PC

**3PC**：是 Three-Phase Commit的缩写，即三阶段提交，是2PC的改进版，其将二阶段提交协议的"提交事务请求"过程一分为二，形成了由 Can Commit、 PreCommit和 do Commit三个阶段组成的事务处理协议；

**阶段一: Can Commit**

1. **事务询问**

   协调者向所有的参与者发送一个包含事务内容的can Commit请求，询问是否可以执行事务提交操作，并等待各参与者的响应. 

2. **各参与者向协调者反馈响应.**

   参与者在接收到来自协调者的 can Commit请求后，如果其自身认为可以顺利执行事务，反馈Yes响应，并进入预备状态，

   否则反馈No响应

**阶段二: Pre Commit**

在阶段二中，协调者会根据参与者的反馈情况来决定是否可以进行事务的 PreCommit操作，包含两种可能

**执行事务预提交**：所有的反馈都是Yes响应 

1. **发送预提交请求**

   协调者向所有参与者节点发出 pre Commit的请求，并进入 Prepared阶段.

2. **事务预提交**

   参与者接收到 pre Commit请求后，会执行事务操作，并将Undo和Redo信息记录到事务日志中 

3. **各参与者向协调者反馈事务执行的响应**

   如果参与者成功执行了事务操作，那么就会反馈给协调者Ack响应，同时等待最终的指令:提交( commit)或中止( abort)

**中断事务：**任何一个参与者反馈了No响应，或者等待超时

1. **发送中断请求**

   协调者向所有参与者节点发出 abort请求.

2. **中断事务**

   无论是收到来自协调者的abort请求，或者是在等待协调者请求过程中出现超时，参与者都会中断事务.

**阶段三: do Commit**

真正的事务提交，会存在以下两种可能的情况

**执行提交**

1. **发送提交请求**

   进入这一阶段，假设协调者处于正常工作状态，并且它接收到了来自所有参与者的Ack响应，那么它将从"预提交"状态转换到"提交"状态，并向所有的参与者发送 do Commit请求.

2. **事务提交**

   参与者接收到 do Commit请求后，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行期间占用的事务资源

3. **反馈事务提交结果**

   参与者在完成事务提交之后，向协调者发送Ack消息

4. **完成事务**

   协调者接收到所有参与者反馈的Ack消息后，完成事务

**中断事务：**任何一个参与者反馈了No响应，或者等待超时

1. **发送中断请求**

   协调者向所有参与者节点发出 abort请求

2. **事务回滚**

   参与者接收到 abort请求后，会利用其在阶段二中记录的Undo信息来执行事务回滚操作，并在完成回滚之后释放在整个事务执行期间占用的资源

3. **反馈事务回滚结果**

   参与者在完成事务回滚之后，向协调者发送Ack消息

4. **中断事务**

   协调者接收到所有参与者反馈的Ack消息后，完成事务中断.

**优点：**相较于二阶段提交协议，三阶段提交协议最大的优点就是降低了参与者的阻塞范围，并且能够在出现单点故障后继续达成

**缺点：**三阶段提交协议在去除阻塞的同时也引入了新的问题，那就是在参与者接收到 pre Commit消息后，如果网络出现分区，此时协调者所在的节点和参与者无法进行正常的网络通信，在这种情况下，该参与者依然会进行事务的提交，这必然出现数据的不一致性.

### PAXOS协议

Paxos三种角色：Proposer, Acceptor, Learner 

* **Proposer**：只要 Proposer发的提案被半数以上 Acceptor接受，Proposer就认为该提案里的value被选定
* **Acceptor**：只要 Acceptor接受了某个提案，Acceptor 就认为该提案里的 value被选定了
* **Learner**: Acceptor告诉 Learner哪个 value被选定，Learner就认为那个value被选定

Paxos算法分为两个阶段：

**阶段一(准 leader 确定)**

1. Proposer选择一个提案编号N，然后向半数以上的 Acceptor发送编号为N的 Prepare请求
2. 如果一个 Acceptor收到一个编号为N的 Prepare请求，且N大于该 Acceptor已经响应过的所有 Prepare请求的编号，那么它就会将它已经接受过的编号最大的提案(如果有的话)作为响应反馈给 Proposer，同时该 Acceptor承诺不再接受任何编号小于N的提案

**阶段二( Leader确认)**

1. 如果 Proposer收到半数以上 Acceptor对其发出的编号为N的 Prepare请求的响应，那么它就会发送一个针对[N，V]提案的 Accept请求给半数以上的 Acceptor（注意：V就是收到的响应中编号最大的提案的value，如果响应中不包含任何提案，那么V就由 Proposer自己决定）
2. 如果 Acceptor收到一个针对编号为N的提案的 Accept请求，只要该 Acceptor没有对编号大于N的 Prepare请求做出过响应，它就接受该提案.

### ZAB 协议 

> ZAB协议的核心是定义了对于那些会改变ZooKeeper服务器数据状态的事务请求的处理方式，即：
>
> 所有事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为 Leader服务器，而余下的其他服务器则成为 Follower服务器。Leader服务器负责将一个客户端事务请求转换成一个事务 Proposal(提议)，并将该 Proposal分发给集群中所有的Follower服务器。之后 Leader服务器需要等待所有 Follower服务器的反馈，一旦超过半数的 Follower服务器进行了正确的反馈后，那么 Leader就会再次向所有的 Follower服务器分发 Commit消息，要求其将前一个 Proposal进行提交。

ZAB协议包括两种基本的模式，分别是 **崩溃恢复和消息广播**。

**消息广播：**广播的过程实际上是一个简化的2PC过程：

1. Leader 接收到消息请求后，将消息赋予一个全局唯一的 64 位自增 id(事务ID)，叫做：zxid，通过 zxid 的大小比较即可实现因果有序这一特性。
2. Leader 通过FIFO先进先出队列（通过 TCP 协议来实现，以此实现了全局有序这一特性）将带有 zxid 的消息作为一个提案（proposal）分发给所有 follower。
3. 当 follower 接收到 proposal，先将 proposal 写到硬盘，写硬盘成功后再向 leader 回一个 ACK。
4. 当 leader 接收到合法数量（超过半数节点）的 ACKs 后，leader 就向所有 follower 发送 COMMIT 命令，同事会在本地执行该消息。
5. 当 follower 收到消息的 COMMIT 命令时，就会执行该消息

在这种简化了的二阶段提交模型下，是无法处理 Leader服务器崩溃退出而带来的数据不一致问题的，因此在ZAB协议中添加了另一个模式，即釆用崩溃恢复模式来解决这个问题。另外，整个消息广播协议是基于具有FIFO特性的TCP协议来进行网络通信的，因此能够很容易地保证消息广播过程中消息接收与发送的顺序性。

**崩溃恢复：** ZAB 协议的广播部分不能处理 leader 挂掉的情况，ZAB 协议引入了恢复模式来处理这一问题。为了使 leader 挂了后系统能正常工作，需要解决以下两个问题：

**已经被处理的消息不能丢**

为了实现已经被处理的消息不能丢这个目的，ZAB 的恢复模式使用了以下的策略：

1. 选举拥有 proposal 最大值（即 zxid 最大） 的节点作为新的 leader：由于所有提案被 COMMIT 之前必须有合法数量的 follower ACK，即必须有合法数量的服务器的事务日志上有该提案的 proposal，因此，只要有合法数量的节点正常工作，就必然有一个节点保存了所有被 COMMIT 消息的 proposal 状态。
2. 新的 leader 将自己事务日志中 proposal 但未 COMMIT 的消息处理。
3. 新的 leader 与 follower 建立先进先出的队列， 先将自身有而 follower 没有的 proposal 发送给 follower，再将这些 proposal 的 COMMIT 命令发送给 follower，以保证所有的 follower 都保存了所有的 proposal、所有的 follower 都处理了所有的消息。
   通过以上策略，能保证已经被处理的消息不会丢

**被丢弃的消息不能再次出现**

ZAB 通过巧妙的设计 zxid 来实现这一目的。一个 zxid 是64位，高 32 是朝代（epoch）编号，每经过一次 leader 选举产生一个新的 leader，新 leader 会将 epoch 号 +1。低 32 位是消息计数器，每接收到一条消息这个值 +1，新 leader 选举后这个值重置为 0。这样设计的好处是旧的 leader 挂了后重启，它不会被选举为 leader，因为此时它的 zxid 肯定小于当前的新 leader。当旧的 leader 作为 follower 接入新的 leader 后，新的 leader 会让它将所有的拥有旧的 epoch 号的未被 COMMIT 的 proposal 清除。

ZAB 的核心：少数服从多数；资历老（处理的事务最多，zxid最大）的节点优先继位

## ZooKeeper 数据模型

在ZooKeeper中，节点也称为ZNode。由于对于程序员来说，对ZooKeeper 的操作主要是对ZNode的操作;

ZooKeeper采用了类似文件系统的的数据模型，其节点构成了一个具有层级关系的树状结构

<img src="img/zookeeper-node.png" alt="img" style="zoom:80%;" />

ZNode是ZooKeeper 树形结构中的数据节点，用于存储数据；

ZNode分为持久节点和临时节点两种类型：

- **持久节点(PERSISTENT)**：一旦创建，除非主动调用删除操作，否则一直存储在ZooKeeper 上；
- **持久有序节点(PERSISTENT_SEQUENTIAL)**：每个节点都会为它的一级子节点维护一个顺序；
- **临时节点(EPHEMERAL)**：临时节点的生命周期和客户端的会话绑定在一起，当客户端会话失效该节点自动清理；
- **临时有序节点(EPHEMERAL_SEQUENTIAL)**：在临时节点的基础上多了一个顺序性
- **CONTAINER**：当子节点都被删除后，Container 也随即删除
- **PERSISTENT_WITH_TTL和PERSISTENT_SEQUENTIAL_WITH_TTL**：带TTL（time-to-live，存活时间）的永久节点，客户端断开连接后，节点在TTL时间之内没有得到更新并且没有孩子节点，就会被自动删除。

**事务ID**：在ZooKeeper中，事务是指能够改变 ZooKeeper 服务器状态的操作，我们也称之为事务操作或更新操作，一般包括数据节点创建与删除、数据节点内容更新和客户端会话创建与失效等操作。

**对于每一个事务请求，ZooKeeper 都会为其分配一个全局唯一的事务ID,用 ZXID 来表示**，通常是一个64位的数字。每一个ZXID对应一次更新操作，**从这些 ZXID 中可以间接地识别出ZooKeeper处理这些更新操作请求的全局顺序**。

实现中zxid 是一个64 位的数字，它高32 位是epoch（ZAB 协议通过epoch 编号来区分Leader 周期变化的策略）用来标识leader 关系是否改变，每次一个leader 被选出来，它都会有一个新的epoch=（原来的epoch+1），标识当前属于那个leader 的统治时期。低32 位用于递增计数。

### ZNode状态属性

|    状态属性    | 说明                                                         |
| :------------: | :----------------------------------------------------------- |
|     czxid      | Created ZXID，表示该数据节点被创建时的事务ID                 |
|     mzxid      | Modified ZXID，表示该节点最后一次被更新时的事务ID            |
|     ctime      | Created Time，表示节点被创建的时间                           |
|     mtime      | Modified Time，表示该节点最后一次被更新的时间                |
|    version     | 数据节点的版本号                                             |
|    cversion    | 子节点的版本号                                               |
|    aversion    | 节点的ACL版本号                                              |
| ephemeralOwner | 创建该临时节点的会话的 session；如果该节点是持久节点，那么这个属性值为0 |
|   dataLength   | 数据内容的长度                                               |
|  numChildren   | 当前节点的子节点个数                                         |
|     pzxid      | 表示该节点的子节点列表最后一次被修改时的事务ID；<br/>注意：只有子节点列表变更了才会变更 pzxid，子节点内容变更不会影响 pzxid |

### 版本

**保证分布式数据原子性操作**

ZooKeeper中为数据节点引入了版本的概念，每个数据节点都具有三种类型的版本信息，对数据节点的任何更新操作都会引起版本号的变化 

| 版本类型 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| version  | 当前数据节点数据内容的版本号；每次对节点进行更新操作，version的值都会增加1 |
| cversion | 当前数据节点子节点的版本号                                   |
| aversion | 当前数据节点ACL变更版本号                                    |



## ZooKeeper 原理

### **ZooKeeper 一致性**

简单来说：顺序一致性是针对单个操作，单个数据对象。属于CAP 中C这个范畴。一个数据被更新后，能够立马被后续的读操作读到。

client 会记录自己已经读取到的最大的zxid，如果client 重连到server 发现 client 的zxid 更大。连接会失败；client 只要连接过一次ZooKeeper ，就不会有历史的倒退

### **ZooKeeper 分布式锁**

**利用节点的唯一性来实现分布式锁**

多个进程往ZooKeeper 的指定节点下创建一个相同名称的节点，只有一个能成功，其余创建失败；创建失败的节点全部通过ZooKeeper 的watcher 机制来监听ZooKeeper 这个子节点的变化，一旦监听到子节点的删除事件，则再次触发所有进程去写锁；

这种实现方式很简单，但是会产生“惊群效应”，简单来说就是如果存在许多的客户端在等待获取锁，当成功获取到锁的进程释放该节点后，所有处于等待状态的客户端都会被唤醒，这个时候ZooKeeper 在短时间内发送大量子节点变更事件给所有待获取锁的客户端，然后实际情况是只会有一个客户端获得锁。如果在集群规模比较大的情况下，会对ZooKeeper 服务器的性能产生比较的影响。


**利用临时有序节点来实现分布式锁**

- 多个进程往ZooKeeper 的指定节点下创建一个临时顺序节点
- 如果自己不是第一个节点，就对自己上一个节点加监听器
- 只要上一个节点释放锁，自己就可以获得锁，相当于是一个排队机制。

## Leader 选举

**myid：**每个ZooKeeper服务器，都需要在数据文件夹下创建一个名为myid的文件，该文件包含整个ZooKeeper集群唯一的ID（整数），编号越大在选举算法中的权重越大

**zxid：**事务ID，值越大说明数据越新，在选举算法中的权重也越大

**逻辑时钟（epoch –logicalclock）：**或者叫投票的次数，同一轮投票过程中的逻辑时钟值是相同的。每投完一次票这个数据就会增加，然后与接收到的其它服务器返回的投票信息中的数值相比，根据不同的值做出不同的判断。

**服务器状态：**

* LOOKING，竞选状态。
* FOLLOWING，随从状态，同步leader 状态，参与投票。
* OBSERVING，观察状态,同步leader 状态，不参与投票。
* LEADING，领导者状态。



### 启动期间的Leader 选举

每个节点启动的时候状态都是LOOKING，处于观望状态，接下来就开始进行选主流程

若进行Leader 选举，则至少需要两台机器，这里选取3 台机器组成的服务器集群为例。在集群初始化阶段，当有一台服务器Server1 启动时，其单独无法进行和完成Leader 选举，当第二台服务器Server2 启动时，此时两台机器可以相互通信，每台机器都试图找到Leader，于是进入Leader选举过程。选举过程如下

1. **每个Server 发出一个投票：**由于是初始情况，Server1 和Server2 都会将自己作为Leader 服务器来进行投票，每次投票会包含所推举的服务器的myid 和ZXID、epoch，使用(myid, ZXID,epoch)来表示，此时Server1 的投票为(1, 0)，Server2 的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。

2. **接受来自各个服务器的投票：**集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票（epoch）、是否来自LOOKING 状态的服务器。

3. **处理投票：**针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK 规则如下

   i. 优先比较epoch

   ii. 其次检查ZXID。ZXID 比较大的服务器优先作为Leader

   iii. 如果ZXID 相同，那么就比较myid。myid 较大的服务器作为Leader 服务器。

   对于Server1 而言，它的投票是(1, 0)，接收Server2 的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此Server2 的myid 最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2 而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。
   
4. **统计投票：**每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于Server1、Server2 而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。

5. **改变服务器状态：**一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。

### 运行期间的Leader 选举

当集群中的leader 服务器出现宕机或者不可用的情况时，那么整个集群将无法对外提供服务，而是进入新一轮的Leader 选举，服务器运行期间的Leader 选举和启动时期的Leader 选举基本过程是一致的。

1. **变更状态**：Leader 挂后，余下的非Observer 服务器都会将自己的服务器状态变更为LOOKING，然后开始进入Leader 选举过程。
2. **每个Server 会发出一个投票**：在运行期间，每个服务器上的ZXID 可能不同，此时假定Server1 的ZXID 为123，Server3 的ZXID 为122；在第一轮投票中，Server1 和Server3 都会投自己，产生投票(1, 123)，(3, 122)，然后各自将投票发送给集群中所有机器。接收来自各个服务器的投票。与启动时过程相同。
3. **处理投票**：与启动时过程相同，此时，Server1 将会成为Leader。
4. **统计投票**：与启动时过程相同。
5. **改变服务器的状态**：与启动时过程相同



# 源码分析

## 服务端

### ZooKeeperServer

![ZooKeeperServer](img/ZooKeeperServer.png)

**ZooKeeperServer**: 所有服务器的父类，请求处理链为PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor。

**QuorumZooKeeperServer**: 其是所有参与选举的服务器的父类，是抽象类，其继承了ZooKeeperServer类。

**LeaderZooKeeperServer**: Leader服务器，继承了QuorumZooKeeperServer类，其请求处理链为PrepRequestProcessor -> ProposalRequestProcessor -> CommitProcessor -> Leader.ToBeAppliedRequestProcessor -> FinalRequestProcessor。

**LearnerZooKeeper**: Follower和Observer的父类，为抽象类，也继承了QuorumZooKeeperServer类。

**FollowerZooKeeperServer**: Follower服务器，继承了LearnerZooKeeper，其请求处理链为FollowerRequestProcessor -> CommitProcessor -> FinalRequestProcessor。

**ObserverZooKeeperServer**: Observer服务器，继承了LearnerZooKeeper。

**ReadOnlyZooKeeperServer**: 只读服务器，不提供写服务；其处理链的第一个处理器为ReadOnlyRequestProcessor。

~~~java
public class ZooKeeperServer implements SessionExpirer, ServerStats.Provider {
    // JMX服务
    protected ZooKeeperServerBean jmxServerBean;
    protected DataTreeBean jmxDataTreeBean; 
    // 默认心跳频率
    public static final int DEFAULT_TICK_TIME = 3000;
    protected int tickTime = DEFAULT_TICK_TIME; 
    // 最小会话过期时间
    protected int minSessionTimeout = -1; 
    // 最大会话过期时间
    protected int maxSessionTimeout = -1;
    // 会话跟踪器
    protected SessionTracker sessionTracker;
    // 事务日志快照
    private FileTxnSnapLog txnLogFactory = null;
    // Zookeeper内存数据库
    private ZKDatabase zkDb;
    // zxid
    private final AtomicLong hzxid = new AtomicLong(0);
    public final static Exception ok = new Exception("No prob");
    // 请求处理器
    protected RequestProcessor firstProcessor;
    // 运行状态
    protected volatile State state = State.INITIAL;

    protected enum State {
        INITIAL, RUNNING, SHUTDOWN, ERROR
    } 
    static final private long superSecret = 0XB3415C00L;
    // 处理中的请求数
    private final AtomicInteger requestsInProcess = new AtomicInteger(0);
    final Deque<ChangeRecord> outstandingChanges = new ArrayDeque<>();
    // this data structure must be accessed under the outstandingChanges lock
    final HashMap<String, ChangeRecord> outstandingChangesForPath =
        new HashMap<String, ChangeRecord>();
 	// 连接工厂
    protected ServerCnxnFactory serverCnxnFactory;
    protected ServerCnxnFactory secureServerCnxnFactory;
    // server 状态信息
    private final ServerStats serverStats;
    // 监听器
    private final ZooKeeperServerListener listener;
    private ZooKeeperServerShutdownHandler zkShutdownHandler;
    private volatile int createSessionTrackerServerId = 1;
    
    // 构造方法
    public ZooKeeperServer(FileTxnSnapLog txnLogFactory, int tickTime,
            int minSessionTimeout, int maxSessionTimeout, ZKDatabase zkDb) {
        // 服务器状态信息
        serverStats = new ServerStats(this);
        this.txnLogFactory = txnLogFactory;
        this.txnLogFactory.setServerStats(this.serverStats);
        // 数据存储
        this.zkDb = zkDb;
        this.tickTime = tickTime;
        setMinSessionTimeout(minSessionTimeout);
        setMaxSessionTimeout(maxSessionTimeout);
        // 创建监听器
        listener = new ZooKeeperServerListenerImpl(this); 
    }
    
}
~~~

#### startup

~~~java
public synchronized void startup() {
    if (sessionTracker == null) {
        // 创建 会话跟踪器
        createSessionTracker();
    }
    // 启动线程 SessionTracker
    startSessionTracker();
    // 创建请求处理链, 启动 firstProcessor（是一个线程）
    setupRequestProcessors();

    registerJMX();
    // 设置状态
    setState(State.RUNNING);
    // 释放锁
    notifyAll();
}
protected void setupRequestProcessors() {
    // FinalRequestProcessor 请求处理链的最后一个处理器
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);
    // FinalRequestProcessor 同步请求的处理器
    RequestProcessor syncProcessor = new SyncRequestProcessor(this, finalProcessor);
    ((SyncRequestProcessor)syncProcessor).start();
    // PrepRequestProcessor 请求处理链的第一个处理器
    firstProcessor = new PrepRequestProcessor(this, syncProcessor);
    ((PrepRequestProcessor)firstProcessor).start();
}
~~~

#### LeaderZooKeeperServer

~~~java
public class LeaderZooKeeperServer extends QuorumZooKeeperServer {
    
    // 重写方法
    @Override
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor toBeAppliedProcessor = 
            new Leader.ToBeAppliedRequestProcessor(finalProcessor, getLeader());
        commitProcessor = new CommitProcessor(toBeAppliedProcessor,
                Long.toString(getServerId()), false,
                getZooKeeperServerListener());
        commitProcessor.start();
        ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
                commitProcessor);
        proposalProcessor.initialize();
        prepRequestProcessor = new PrepRequestProcessor(this, proposalProcessor);
        prepRequestProcessor.start();
        firstProcessor = new LeaderRequestProcessor(this, prepRequestProcessor);

        setupContainerManager();
    }
}
~~~





#### FollowerZooKeeperServer

~~~java
public class FollowerZooKeeperServer extends LearnerZooKeeperServer {
    
}
~~~



#### ObserverZooKeeperServer

~~~java
public class ObserverZooKeeperServer extends LearnerZooKeeperServer {
    
}
~~~



### Server启动

启动类QuorumPeerMain

~~~java
public class QuorumPeerMain {
    public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            // 初始化并运行
            main.initializeAndRun(args);
        } catch  // ..  
    }
    protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException{
        // 解析配置
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        } 
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        if (args.length == 1 && config.isDistributed()) {
            // 集群启动
            runFromConfig(config);
        } else { 
            // 单机启动
            ZooKeeperServerMain.main(args);
        }
    }
    // 集群启动
    public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException  {
        try {
            ManagedUtil.registerLog4jMBeans();
        } catch (JMException e) { 
        } 
        try {
            ServerCnxnFactory cnxnFactory = null;
            ServerCnxnFactory secureCnxnFactory = null;
			// 创建通信工厂 ServerCnxnFactory
            if (config.getClientPortAddress() != null) {
                cnxnFactory = ServerCnxnFactory.createFactory();
                cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), false);
            } 
            if (config.getSecureClientPortAddress() != null) {
                secureCnxnFactory = ServerCnxnFactory.createFactory();
                secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(),
                                            true);
            }
            // 创建QuorumPeer ZooKeeperThread的子类, 是一个线程
            quorumPeer = getQuorumPeer(); 
            // ... 省略 quorumPeer set 参数

            quorumPeer.initialize();
            // quorumPeer 是一个线程, 启动
            quorumPeer.start();
            quorumPeer.join();
        } catch (InterruptedException e) {
            // warn, but generally this is ok
            LOG.warn("Quorum Peer interrupted", e);
        }
    }
}
~~~

#### QuorumPeer 

QuorumPeer是ZooKeeper执行同步、选主过程的线程

~~~java
public class QuorumPeer extends ZooKeeperThread implements QuorumStats.Provider {
    // 当前服务器角色
    private ServerState state = ServerState.LOOKING;
    public enum ServerState {
        LOOKING, FOLLOWING, LEADING, OBSERVING;
    }
    @Override
    public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
        }
        // 恢复快照数据
        loadDataBase();
        // 启动 网络通信 NIO 或 Netty
        startServerCnxnFactory();
        try {
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        // 开始选举
        startLeaderElection();
        // 线程启动 执行run()
        super.start();
    }
}
~~~

##### 线程运行

~~~java
@Override
public void run() {
    updateThreadName(); 
    // ... 省略 JMX 初始化代码  
    try { 
        while (running) {
            // 判断当前服务器参加选举后的角色
            switch (getPeerState()) {

                case LOOKING: 
                    // 当前服务是否 只读服务
                    if (Boolean.getBoolean("readonlymode.enabled")) { 
                        final ReadOnlyZooKeeperServer roZk =
                            new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb); 
                        Thread roZkMgr = new Thread() {
                            public void run() {
                                try {
                                    // lower-bound grace period to 2 secs
                                    sleep(Math.max(2000, tickTime));
                                    if (ServerState.LOOKING.equals(getPeerState())) {
                                        roZk.startup();
                                    }
                                } // catch ... 
                            }
                        };
                        try {
                            roZkMgr.start();
                            reconfigFlagClear();
                            if (shuttingDownLE) {
                                shuttingDownLE = false;
                                startLeaderElection();
                            }
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) { 
                            setPeerState(ServerState.LOOKING);
                        } finally { 
                            roZkMgr.interrupt();
                            roZk.shutdown();
                        }
                    } else {
                        // ZooKeeper 启动时所有节点初始状态为Looking
                        // Follower 失去 Leader 后也会进入 Looking
                       
                        try {
                            reconfigFlagClear();
                            if (shuttingDownLE) {
                                shuttingDownLE = false;
                                 // 开始 Leader 选举
                                startLeaderElection();
                            }
                            setCurrentVote(makeLEStrategy().lookForLeader());
                        } catch (Exception e) { 
                            setPeerState(ServerState.LOOKING);
                        }                        
                    }
                    break;
                case OBSERVING:
                    // 创建 Observer 服务
                    try { 
                        setObserver(makeObserver(logFactory));
                        observer.observeLeader();
                    } catch (Exception e) { 
                    } finally {
                        observer.shutdown();
                        setObserver(null);  
                        updateServerState();
                    }
                    break;
                case FOLLOWING:
                    // 创建 Follower 服务
                    try { 
                        setFollower(makeFollower(logFactory));
                        follower.followLeader();
                    } catch (Exception e) { 
                    } finally {
                        follower.shutdown();
                        setFollower(null);
                        updateServerState();
                    }
                    break;
                case LEADING:
                    // 创建 Leader 服务 
                    try {
                        setLeader(makeLeader(logFactory));
                        leader.lead();
                        setLeader(null);
                    } catch (Exception e) { 
                    } finally {
                        if (leader != null) { 
                            setLeader(null);
                        }
                        updateServerState();
                    }
                    break;
            }
            start_fle = Time.currentElapsedTime();
        }
    } finally { 
        //  ... 省略 JMX 代码   
    }
}
~~~

#### Leader

##### makeLeader

~~~java
protected Leader makeLeader(FileTxnSnapLog logFactory) throws IOException, X509Exception {
    // 创建 LeaderZooKeeperServer
    return new Leader(this, new LeaderZooKeeperServer(logFactory, this, this.zkDb));
}
~~~

##### leader.lead()



#### Follower

##### makeFollower

~~~java
protected Follower makeFollower(FileTxnSnapLog logFactory) throws IOException {
    // 创建 FollowerZooKeeperServer
    return new Follower(this, new FollowerZooKeeperServer(logFactory, this, this.zkDb));
} 
~~~

##### follower.followLeader()

~~~java
void followLeader() throws InterruptedException {
    self.end_fle = Time.currentElapsedTime();
    long electionTimeTaken = self.end_fle - self.start_fle;
    self.setElectionTimeTaken(electionTimeTaken); 
    self.start_fle = 0;
    self.end_fle = 0;
    fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
    try {
        // 根据sid找到对应leader，拿到lead连接信息
        QuorumServer leaderServer = findLeader();            
        try {
            // 连接到Leader
            connectToLeader(leaderServer.addr, leaderServer.hostname);
            long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
            if (self.isReconfigStateChange())
                throw new Exception("learned about role change"); 
            // 如果leader的epoch比当前follow节点的epoch还小,抛异常
            long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
            if (newEpoch < self.getAcceptedEpoch()) { 
                throw new IOException("Error: Epoch of leader is lower");
            }
            // 和leader进行数据同步, 同步过程中会启动 FollowerZooKeeperServer
            syncWithLeader(newEpochZxid);                
            QuorumPacket qp = new QuorumPacket();
            while (this.isRunning()) {
                // 从leader读取数据包
                readPacket(qp);
                // 处理packet
                processPacket(qp);
            }
        } catch (Exception e) { 
            try {
                sock.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            } 
            // clear pending revalidations
            pendingRevalidations.clear();
        }
    } finally {
        zk.unregisterJMX((Learner)this);
    }
}
~~~



#### Observer

##### makeObserver

~~~java
protected Observer makeObserver(FileTxnSnapLog logFactory) throws IOException {
    // 创建 ObserverZooKeeperServer
    return new Observer(this, new ObserverZooKeeperServer(logFactory, this, this.zkDb));
}
~~~

##### observer.observeLeader()

~~~java
void observeLeader() throws Exception {
    zk.registerJMX(new ObserverBean(this, zk), self.jmxLocalPeerBean); 
    try {
        // 根据sid找到对应leader，拿到lead连接信息
        QuorumServer leaderServer = findLeader(); 
        try {
            // 连接到Leader
            connectToLeader(leaderServer.addr, leaderServer.hostname);
            long newLeaderZxid = registerWithLeader(Leader.OBSERVERINFO);
            if (self.isReconfigStateChange())
                throw new Exception("learned about role change");
            // 和leader进行数据同步, 同步过程中会启动 ObserverZooKeeperServer
            syncWithLeader(newLeaderZxid);
            QuorumPacket qp = new QuorumPacket();
            while (this.isRunning()) {
                // 从leader读取数据包
                readPacket(qp);
                // 处理packet
                processPacket(qp);
            }
        } catch (Exception e) { 
            try {
                sock.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            } 
            // clear pending revalidations
            pendingRevalidations.clear();
        }
    } finally {
        zk.unregisterJMX(this);
    }
}
~~~





### Leader选举

~~~java
synchronized public void startLeaderElection() {
    try {
        if (getPeerState() == ServerState.LOOKING) {
            // 初始化投票, 用于记录投票
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
        }
    } catch(IOException e) {
        RuntimeException re = new RuntimeException(e.getMessage());
        re.setStackTrace(e.getStackTrace());
        throw re;
    } 
    if (electionType == 0) {
        try {
            udpSocket = new DatagramSocket(getQuorumAddress().getPort());
            responder = new ResponderThread();
            responder.start();
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }
    }
    // 选举算法
    this.electionAlg = createElectionAlgorithm(electionType);
}
protected Election createElectionAlgorithm(int electionAlgorithm){
    Election le=null;

    // 根据配置选择算法 默认值位3
    switch (electionAlgorithm) {
        case 0:
            le = new LeaderElection(this);
            break;
        case 1:
            le = new AuthFastLeaderElection(this);
            break;
        case 2:
            le = new AuthFastLeaderElection(this, true);
            break;
        case 3:
            // 默认 
            QuorumCnxManager qcm = createCnxnManager();
            QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
            if (oldQcm != null) { 
                oldQcm.halt();
            }
            QuorumCnxManager.Listener listener = qcm.listener;
            if(listener != null){
                listener.start();
                // 当前默认选择FastLeaderElection,初始化
                FastLeaderElection fle = new FastLeaderElection(this, qcm);
                fle.start();
                le = fle;
            } else { 
            }
            break;
        default:
            assert false;
    }
    return le;
}
~~~

#### FastLeaderElection

~~~java
public class FastLeaderElection implements Election {
    // 构造方法
    public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
        this.stop = false;
        this.manager = manager;
        starter(self, manager);
    }
    private void starter(QuorumPeer self, QuorumCnxManager manager) {
        this.self = self;
        proposedLeader = -1;
        proposedZxid = -1;
        // 发送队列
        sendqueue = new LinkedBlockingQueue<ToSend>();
        // 接收队列
        recvqueue = new LinkedBlockingQueue<Notification>();
        this.messenger = new Messenger(manager);
    }
}
~~~

##### Messenger

~~~java
protected class Messenger {
    Messenger(QuorumCnxManager manager) {
        // WorkerSender ZooKeeperThread的子类 用来发送消息
        this.ws = new WorkerSender(manager); 
        this.wsThread = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
        this.wsThread.setDaemon(true);
        // WorkerReceiver ZooKeeperThread的子类 用来接收消息
        this.wr = new WorkerReceiver(manager);

        this.wrThread = new Thread(this.wr,  "WorkerReceiver[myid=" + self.getId() + "]");
        this.wrThread.setDaemon(true);
    }
}
~~~

##### start

~~~java
public class FastLeaderElection implements Election {	
	public void start() {
        this.messenger.start();
    }
}
protected class Messenger {
    void start(){
        // 启动两个线程,用于发送和接收消息
        this.wsThread.start();
        this.wrThread.start();
    }
}
~~~

#### 选举流程

选择好选举算法后，QuorumPeer线程启动，当前状态为Looking，执行下面的方法

~~~Java
// 设置投票
setCurrentVote(makeLEStrategy().lookForLeader());
~~~

##### lookForLeader

~~~java
public class FastLeaderElection implements Election {	
	public Vote lookForLeader() throws InterruptedException {
        try {
            self.jmxLeaderElectionBean = new LeaderElectionBean();
            MBeanRegistry.getInstance().register(
                    self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
        } catch (Exception e) {
            self.jmxLeaderElectionBean = null;
        }
        if (self.start_fle == 0) {
           self.start_fle = Time.currentElapsedTime();
        }
        try {
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

            int notTimeout = finalizeWait;

            synchronized(this){
                // 更新逻辑时钟,每进行一轮选举,都需要更新逻辑时钟
                logicalclock.incrementAndGet();
                // 更新选票  把当前节点的myid,zxid,epoch更新到本地的成员属性
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }
            // 向其他服务器发送自己的选票 把信息放入阻塞队列sendqueue, 由wsThread 发送信息
            sendNotifications();
            // 状态为LOOKING并且还未选出leader
            while ((self.getPeerState() == ServerState.LOOKING) &&
                    (!stop)){
                // 从recvqueue接收队列中取出投票
                Notification n = recvqueue.poll(notTimeout,TimeUnit.MILLISECONDS);
                // 如果没有获取到外部的投票 有可能是集群之间的节点没有真正连接上
                if(n == null){
                    // 判断发送队列是否有数据
                    if(manager.haveDelivered()){
                        // 如果发送队列为空,重新发一次自己的选票
                        sendNotifications();
                    } else { // 发送队列不为空, 可能没有连接
                        // 重新连接
                        manager.connectAll();
                    }
                    int tmpTimeOut = notTimeout*2;
                    notTimeout = (tmpTimeOut < maxNotificationInterval?tmpTimeOut : maxNotificationInterval);
                }
                // 校验投票信息 判断服务器id是否在投票服务器集合中
                else if (validVoter(n.sid) && validVoter(n.leader)) {
                    switch (n.state) {
                        // LOOKING 服务器也是在找leader
                    case LOOKING: 
                        // 如果收到的投票的逻辑时钟大于当前的节点的逻辑时钟
                        if (n.electionEpoch > logicalclock.get()) {
                            // 重新赋值逻辑时钟
                            logicalclock.set(n.electionEpoch);
                            recvset.clear();
                            // 选出较优的服务器
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                            } else {
                                // 否则,说明当前节点的票据优先级更高,再次更新自己的票据
                                updateProposal(getInitId(),
                                        getInitLastLoggedZxid(),
                                        getPeerEpoch());
                            }
                            // 再次发送消息
                            sendNotifications();
                        } else if (n.electionEpoch < logicalclock.get()) {
                            // 如果小于，说明收到的票据已经过期了，直接把这张票丢掉 
                            break;
                        // epoch相同,并且能选出较优的服务器
                        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                proposedLeader, proposedZxid, proposedEpoch)) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                            // 把收到的票据再发出去,告诉大家我要选n.leader为leader
                            sendNotifications();
                        }
 
                        // 将收到的投票信息放入投票的集合recvset 中, 用来作为  最终的"过半原则" 判断
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        // 判断选举是否结束
                        if (termPredicate(recvset,
                                new Vote(proposedLeader, proposedZxid,
                                        logicalclock.get(), proposedEpoch))) { 
                            // 遍历已经接收的投票集合
                            while((n = recvqueue.poll(finalizeWait,
                                    TimeUnit.MILLISECONDS)) != null){
                                // 能够选出较优的服务器
                                if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                        proposedLeader, proposedZxid, proposedEpoch)){
                                    recvqueue.put(n);
                                    break;
                                }
                            }

                            // 如果notifaction为空，说明Leader节点是可以确定好了
                            if (n == null) {
                                // 设置当前节点的状态
                                // 判断leader节点是不是自己,如果是,直接更新当前节点的state为LEADING 
                                // 否则,根据当前节点的特性进行判断 FOLLOWING还是OBSERVING
                                self.setPeerState((proposedLeader == self.getId()) ?
                                        ServerState.LEADING: learningState());
                                // 组装生成这次Leader 选举最终的投票的结果
                                Vote endVote = new Vote(proposedLeader,
                                        proposedZxid, logicalclock.get(), 
                                        proposedEpoch);
                                // 清空 recvqueue
                                leaveInstance(endVote);
                                // 返回最终的票据
                                return endVote;
                            }
                        }
                        break;
                    case OBSERVING:
                        // OBSERVING不参与leader选举
                        LOG.debug("Notification from observer: " + n.sid);
                        break;
                    case FOLLOWING:
                    case LEADING: 
                        if(n.electionEpoch == logicalclock.get()){
                            // 将该服务器和选票信息放入recvset中
                            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                            // 判断是否完成了leader选举
                            if(termPredicate(recvset, new Vote(n.version, n.leader,
                                            n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                                            && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());
                                Vote endVote = new Vote(n.leader, 
                                        n.zxid, n.electionEpoch, n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }
 
                        outofelection.put(n.sid, new Vote(n.version, n.leader, 
                                n.zxid, n.electionEpoch, n.peerEpoch, n.state));
                        if (termPredicate(outofelection, new Vote(n.version, n.leader,
                                n.zxid, n.electionEpoch, n.peerEpoch, n.state))
                                && checkLeader(outofelection, n.leader, n.electionEpoch)) {
                            synchronized(this){
                                logicalclock.set(n.electionEpoch);
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());
                            }
                            Vote endVote = new Vote(n.leader, n.zxid, 
                                    n.electionEpoch, n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                        break;
                    default: 
                        break;
                    }
                } else { 
                }
            }
            return null;
        } finally {
            try {
                if(self.jmxLeaderElectionBean != null){
                    MBeanRegistry.getInstance().unregister(  self.jmxLeaderElectionBean);
                }
            } catch (Exception e) { 
            }
            self.jmxLeaderElectionBean = null; 
        }
    }
}
~~~

###### sendNotifications

遍历所有的参与者投票集合，然后将自己的选票信息发送给所有的投票者集合，其并非同步发送，而是将ToSend消息放置于sendqueue中，之后由WorkerSender线程进行发送

~~~java
private void sendNotifications() {
    // 遍历投票参与者集合
    for (long sid : self.getCurrentAndNextConfigVoters()) {
        QuorumVerifier qv = self.getQuorumVerifier();
        // 构造发送消息
        ToSend notmsg = new ToSend(ToSend.mType.notification, proposedLeader,proposedZxid,
            logicalclock.get(),QuorumPeer.ServerState.LOOKING,sid,proposedEpoch, qv.toString().getBytes());
       
        // 将发送消息加入队列
        sendqueue.offer(notmsg);
    }
}
~~~

totalOrderPredicate

接收的投票与自身投票进行PK，查看是否消息中包含的服务器id是否更优，其按照epoch、zxid、id的优先级进行PK

~~~java
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, 
    long curZxid, long curEpoch) {
    // 使用计票器判断当前服务器的权重是否为0
    if(self.getQuorumVerifier().getWeight(newId) == 0){
        return false;
    }
    // 1. 判断消息里的epoch是不是比当前的大，如果大则消息中id对应的服务器就是leader
    // 2. 如果epoch相等则判断zxid，如果消息里的zxid大，则消息中id对应的服务器就是leader
    // 3. 如果前面两个都相等那就比较服务器id，如果大，则其就是leader
    return ((newEpoch > curEpoch) ||
            ((newEpoch == curEpoch) &&
             ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
}
~~~

###### termPredicate

用于判断Leader选举是否结束

~~~java
protected boolean termPredicate(Map<Long, Vote> votes, Vote vote) {
    SyncedLearnerTracker voteSet = new SyncedLearnerTracker();
    voteSet.addQuorumVerifier(self.getQuorumVerifier());
    if (self.getLastSeenQuorumVerifier() != null
        && self.getLastSeenQuorumVerifier().getVersion() > self
        .getQuorumVerifier().getVersion()) {
        voteSet.addQuorumVerifier(self.getLastSeenQuorumVerifier());
    } 
    // 遍历已经接收的投票集合
    for (Map.Entry<Long, Vote> entry : votes.entrySet()) {
        // 将等于当前投票的项放入set
        if (vote.equals(entry.getValue())) {
            voteSet.addAck(entry.getKey());
        }
    }
    // 统计set,查看投某个id的票数是否超过一半
    return voteSet.hasAllQuorums();
}
~~~

图解

![选举流程](img/选举流程.png)

### 网络通信





### 请求处理链



## 客户端

### ZooKeeper

~~~java
public class ZooKeeper implements AutoCloseable {
	public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
    		long sessionId, byte[] sessionPasswd, boolean canBeReadOnly,
    		HostProvider aHostProvider, ZKClientConfig clientConfig) throws IOException { 
        // 客户端配置信息
        if (clientConfig == null) {
            clientConfig = new ZKClientConfig();
        }
        this.clientConfig = clientConfig;
        // 使用 watchManager 管理监听; 默认ZKWatchManager
        watchManager = defaultWatchManager();
        watchManager.defaultWatcher = watcher;
        // 解析 连接的地址
        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);
        hostProvider = aHostProvider;
        // 创建 ClientCnxn, 调用cnxn.start()
        cnxn = createConnection(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
        // 启动线程
        cnxn.start();
    }
    
}
~~~

#### ClientCnxn

~~~java
public class ClientCnxn {	
	public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
            ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
            long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
        this.zooKeeper = zooKeeper;
        this.watcher = watcher;
        this.sessionId = sessionId;
        this.sessionPasswd = sessionPasswd;
        this.sessionTimeout = sessionTimeout;
        this.hostProvider = hostProvider;
        this.chrootPath = chrootPath; 
        connectTimeout = sessionTimeout / hostProvider.size();
        readTimeout = sessionTimeout * 2 / 3;
        readOnly = canBeReadOnly;
        // send 线程  父类 ZooKeeperThread -> Thread
        sendThread = new SendThread(clientCnxnSocket);
        // event 线程 父类 ZooKeeperThread -> Thread
        eventThread = new EventThread();
        this.clientConfig=zooKeeper.getClientConfig();
        initRequestTimeout();
    }
}
~~~



#### SendThread

~~~java
class SendThread extends ZooKeeperThread {
    @Override
    public void run() {
        clientCnxnSocket.introduce(this, sessionId, outgoingQueue);
        clientCnxnSocket.updateNow();
        clientCnxnSocket.updateLastSendAndHeard();
        int to;
        long lastPingRwServer = Time.currentElapsedTime();
        final int MAX_SEND_PING_INTERVAL = 10000; //10 seconds
        InetSocketAddress serverAddress = null;
        while (state.isAlive()) {
            try {
                if (!clientCnxnSocket.isConnected()) {
                    // don't re-establish connection if we are closing
                    if (closing) {
                        break;
                    }
                    if (rwServerAddress != null) {
                        serverAddress = rwServerAddress;
                        rwServerAddress = null;
                    } else {
                        serverAddress = hostProvider.next(1000);
                    }
                    startConnect(serverAddress);
                    clientCnxnSocket.updateLastSendAndHeard();
                }

                if (state.isConnected()) {
                    // determine whether we need to send an AuthFailed event.
                    if (zooKeeperSaslClient != null) {
                        boolean sendAuthEvent = false;
                        if (zooKeeperSaslClient.getSaslState() == ZooKeeperSaslClient.SaslState.INITIAL) {
                            try {
                                zooKeeperSaslClient.initialize(ClientCnxn.this);
                            } catch (SaslException e) {
                                LOG.error("SASL authentication with Zookeeper Quorum member failed: " + e);
                                state = States.AUTH_FAILED;
                                sendAuthEvent = true;
                            }
                        }
                        KeeperState authState = zooKeeperSaslClient.getKeeperState();
                        if (authState != null) {
                            if (authState == KeeperState.AuthFailed) {
                                // An authentication error occurred during authentication with the Zookeeper Server.
                                state = States.AUTH_FAILED;
                                sendAuthEvent = true;
                            } else {
                                if (authState == KeeperState.SaslAuthenticated) {
                                    sendAuthEvent = true;
                                }
                            }
                        }

                        if (sendAuthEvent) {
                            eventThread.queueEvent(new WatchedEvent(
                                Watcher.Event.EventType.None,
                                authState,null));
                            if (state == States.AUTH_FAILED) {
                                eventThread.queueEventOfDeath();
                            }
                        }
                    }
                    to = readTimeout - clientCnxnSocket.getIdleRecv();
                } else {
                    to = connectTimeout - clientCnxnSocket.getIdleRecv();
                }

                if (to <= 0) {
                    String warnInfo;
                    warnInfo = "Client session timed out, have not heard from server in "
                        + clientCnxnSocket.getIdleRecv()
                        + "ms"
                        + " for sessionid 0x"
                        + Long.toHexString(sessionId);
                    LOG.warn(warnInfo);
                    throw new SessionTimeoutException(warnInfo);
                }
                if (state.isConnected()) {
                    //1000(1 second) is to prevent race condition missing to send the second ping
                    //also make sure not to send too many pings when readTimeout is small 
                    int timeToNextPing = readTimeout / 2 - clientCnxnSocket.getIdleSend() - 
                        ((clientCnxnSocket.getIdleSend() > 1000) ? 1000 : 0);
                    //send a ping request either time is due or no packet sent out within MAX_SEND_PING_INTERVAL
                    if (timeToNextPing <= 0 || clientCnxnSocket.getIdleSend() > MAX_SEND_PING_INTERVAL) {
                        sendPing();
                        clientCnxnSocket.updateLastSend();
                    } else {
                        if (timeToNextPing < to) {
                            to = timeToNextPing;
                        }
                    }
                }

                // If we are in read-only mode, seek for read/write server
                if (state == States.CONNECTEDREADONLY) {
                    long now = Time.currentElapsedTime();
                    int idlePingRwServer = (int) (now - lastPingRwServer);
                    if (idlePingRwServer >= pingRwTimeout) {
                        lastPingRwServer = now;
                        idlePingRwServer = 0;
                        pingRwTimeout =
                            Math.min(2*pingRwTimeout, maxPingRwTimeout);
                        pingRwServer();
                    }
                    to = Math.min(to, pingRwTimeout - idlePingRwServer);
                }
                // 传输
                clientCnxnSocket.doTransport(to, pendingQueue, ClientCnxn.this);
            } catch (Throwable e) {
                if (closing) { 
                    break;
                } else { 
                    cleanAndNotifyState();
                }
            }
        }
        synchronized (state) { 
            cleanup();
        }
        clientCnxnSocket.close();
        if (state.isAlive()) {
            eventThread.queueEvent(new WatchedEvent(Event.EventType.None,
                                                    Event.KeeperState.Disconnected, null));
        }
        // 向队列中添加 Watcher
        eventThread.queueEvent(new WatchedEvent(Event.EventType.None,Event.KeeperState.Closed, null)); 
    }


}
~~~

#### EventThread

~~~java
class EventThread extends ZooKeeperThread {
    // 阻塞队列
    private final LinkedBlockingQueue<Object> waitingEvents = new LinkedBlockingQueue<Object>();
    // Event 线程 运行方法
    @Override 
    public void run() {
        try {
            isRunning = true;
            while (true) {
                // 从阻塞队列里取数据
                Object event = waitingEvents.take();
                if (event == eventOfDeath) {
                    wasKilled = true;
                } else {
                    // 处理 event
                    processEvent(event);
                }
                if (wasKilled)
                    synchronized (waitingEvents) {
                    if (waitingEvents.isEmpty()) {
                        isRunning = false;
                        break;
                    }
                }
            }
        } catch (InterruptedException e) { 
        } 
    }
    // 处理 event
    private void processEvent(Object event) {
        try {
            if (event instanceof WatcherSetEventPair) {
                // each watcher will process the event
                WatcherSetEventPair pair = (WatcherSetEventPair) event;
                for (Watcher watcher : pair.watchers) {
                    try {
                        // 调用 process 方法
                        watcher.process(pair.event);
                    } catch (Throwable t) { 
                    }
                }
            } else if (event instanceof LocalCallback) {
                // 省略 ... 
            } else {
                // 省略 ...  
            }
        } catch (Throwable t) { 
        }
    }
}
~~~





## Watcher 机制

### WatchedEvent

~~~java
public class WatchedEvent {	
    // Zookeeper的状态
	final private KeeperState keeperState;
    // 事件类型
    final private EventType eventType;
    // 节点路径
    private String path;
    // 构造方法
    public WatchedEvent(EventType eventType, KeeperState keeperState, String path) {
        this.keeperState = keeperState;
        this.eventType = eventType;
        this.path = path;
    } 
    // 将 WatcherEvent 修改为 WatchedEvent ,两者是不一样的
    public WatchedEvent(WatcherEvent eventMessage) {
        keeperState = KeeperState.fromInt(eventMessage.getState());
        eventType = EventType.fromInt(eventMessage.getType());
        path = eventMessage.getPath();
    }
}
~~~

#### KeeperState

KeeperState是一个枚举类，其定义了在事件发生时ZooKeeper所处的各种状态

~~~java
public enum KeeperState {
    // 未知状态,不再使用 
    @Deprecated
    Unknown (-1),
	// 断开
    Disconnected (0), 
    // 未同步连接,不再使用
    @Deprecated
    NoSyncConnected (1),
    // 同步连接
    SyncConnected (3),
	// 认证失败
    AuthFailed (4),
    // 只读连接
    ConnectedReadOnly (5), 
    // SASL认证通过
    SaslAuthenticated(6), 
    // 过期
    Expired (-112), 
    // 关闭
    Closed (7);
} 
~~~

#### EventType

EventType是一个枚举类，其定义了事件的类型

~~~java
public enum EventType {
    // 无
    None (-1),
    // 创建节点
    NodeCreated (1),
    // 删除节点
    NodeDeleted (2),
    // 节点数据变化
    NodeDataChanged (3),
    // 当前节点的子节点变化
    NodeChildrenChanged (4),
    // 节点Watch 移除
    DataWatchRemoved (5),
    // 子节点Watch 移除
    ChildWatchRemoved (6);
}
~~~



### ClientWatchManager

客户端的watcher管理器，根据event找到该event的所有观察者，返回需要被通知的Watcher集合

~~~java
public interface ClientWatchManager { 
    // 返回需要被通知的Watcher集合
    public Set<Watcher> materialize(Watcher.Event.KeeperState state,
        Watcher.Event.EventType type, String path);
}

static class ZKWatchManager implements ClientWatchManager {
    // 数据变化的Watchers  
    private final Map<String, Set<Watcher>> dataWatches = new HashMap<String, Set<Watcher>>();
    // 节点存在与否的Watchers  
    private final Map<String, Set<Watcher>> existWatches = new HashMap<String, Set<Watcher>>();
    // 子节点变化的Watchers 
    private final Map<String, Set<Watcher>> childWatches = new HashMap<String, Set<Watcher>>();
    private boolean disableAutoWatchReset;
    
    @Override
    public Set<Watcher> materialize(Watcher.Event.KeeperState state,
        Watcher.Event.EventType type, String clientPath) {
        // 需要被通知的Watcher集合
        Set<Watcher> result = new HashSet<Watcher>();

        switch (type) {
            // 无类型 全部添加
            case None:
                result.add(defaultWatcher);
                boolean clear = disableAutoWatchReset && state != Watcher.Event.KeeperState.SyncConnected;
                synchronized(dataWatches) {
                    for(Set<Watcher> ws: dataWatches.values()) {
                        result.addAll(ws);
                    }
                    if (clear) {
                        dataWatches.clear();
                    }
                } 
                synchronized(existWatches) {
                    for(Set<Watcher> ws: existWatches.values()) {
                        result.addAll(ws);
                    }
                    if (clear) {
                        existWatches.clear();
                    }
                } 
                synchronized(childWatches) {
                    for(Set<Watcher> ws: childWatches.values()) {
                        result.addAll(ws);
                    }
                    if (clear) {
                        childWatches.clear();
                    }
                }

                return result;
            // 节点数据变化 和 创建节点
            case NodeDataChanged:
            case NodeCreated:
                // 移除clientPath对应的Watcher, 加入集合
                synchronized (dataWatches) {
                    addTo(dataWatches.remove(clientPath), result);
                }
                synchronized (existWatches) {
                    addTo(existWatches.remove(clientPath), result);
                }
                break;
            // 子节点变化
            case NodeChildrenChanged:
                synchronized (childWatches) {
                    addTo(childWatches.remove(clientPath), result);
                }
                break;
            // 删除节点
            case NodeDeleted:
                synchronized (dataWatches) {
                    addTo(dataWatches.remove(clientPath), result);
                } 
                synchronized (existWatches) {
                    Set<Watcher> list = existWatches.remove(clientPath);
                    if (list != null) {
                        addTo(list, result);
                        LOG.warn("We are triggering an exists watch for delete! Shouldn't happen!");
                    }
                }
                synchronized (childWatches) {
                    addTo(childWatches.remove(clientPath), result);
                }
                break;
            default:
                String msg = "Unhandled watch event type " + type
                    + " with state " + state + " on path " + clientPath;
                LOG.error(msg);
                throw new RuntimeException(msg);
        } 
        // 返回watcher集合
        return result;
    }
    
}
~~~

### WatchManager

服务端的watcher管理器

~~~java
class WatchManager { 
    // watcher表     路径 -> watcher集合
    private final HashMap<String, HashSet<Watcher>> watchTable = new HashMap<String, HashSet<Watcher>>();
	// watcher到节点路径的映射   watcher -> 路径集合
    private final HashMap<Watcher, HashSet<String>> watch2Paths = new HashMap<Watcher, HashSet<String>>(); 
}
~~~

#### addWatch 

向watch2Paths和watchTable添加watcher

~~~java
synchronized void addWatch(String path, Watcher watcher) {
    // 根据路径获取对应的 Watcher集合
    HashSet<Watcher> list = watchTable.get(path);
    if (list == null) { 
        // 新建 Watcher 集合
        list = new HashSet<Watcher>(4);
        watchTable.put(path, list);
    }
    // 将watcher直接添加至 Watcher 集合
    list.add(watcher);
    // 通过watcher获取对应的 路径集合
    HashSet<String> paths = watch2Paths.get(watcher);
    if (paths == null) { 
        // 新建 paths 集合
        paths = new HashSet<String>();
        watch2Paths.put(watcher, paths);
    }
    // 将路径添加至paths集合
    paths.add(path);
}
~~~

#### removeWatcher

从watch2Paths和watchTable中移除该watcher

~~~java
synchronized void removeWatcher(Watcher watcher) {
    // 从wach2Paths中移除watcher , 返回watcher对应的path集合
    HashSet<String> paths = watch2Paths.remove(watcher);
    if (paths == null) {
        return;
    }
    // 遍历 paths
    for (String p : paths) {
        HashSet<Watcher> list = watchTable.get(p);
        if (list != null) {
            // 从list中移除该watcher
            list.remove(watcher);
            if (list.size() == 0) {
                watchTable.remove(p);
            }
        }
    }
}
~~~

#### triggerWatch

触发watch事件，并对事件进行处理，触发过程中会移除watcher，所以每个watcher只会触发一次

~~~java
Set<Watcher> triggerWatch(String path, EventType type) {
    return triggerWatch(path, type, null);
} 
Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    // 根据事件类型、连接状态、节点路径创建WatchedEvent
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        // 从watchTable中移除path,返回对应的watcher集合
        watchers = watchTable.remove(path);
        if (watchers == null || watchers.isEmpty()) { 
            return null;
        }
        // 遍历watcher集合
        for (Watcher w : watchers) {
            // 根据watcher和path 从watch2Paths移除对应的路径
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                paths.remove(path);
            }
        }
    }
    // 遍历watcher集合
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
        // 调用process,对watcher进行处理
        w.process(e);
    }
    return watchers;
}
~~~

 



客户端注册Watcher



服务器处理Watcher 



客户端回调Watcher