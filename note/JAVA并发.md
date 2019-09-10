## 为什么使用并发编程？  

* 并发编程可以充分利用多核心CPU资源，在单核性能达到瓶颈的情况下能有效提高程序的执行效率，好的并发模型会比单线程应用花费更少的执行时间 
* 我们可以对业务进行拆分，让每个线程运行一个任务，会比顺序执行效率更高 

例如：我们苦逼的程序员，当一个程序员的价值达到瓶颈时，老板首先会想到招兵买马，一个人干不完，多个人一起干，怎么让多个人协作完成一个任务就是并发编程需要解决的问题，肯定会充分利用每个人的能力，不能闲着；我们可以把任务进行拆分，每个员工一个任务模块，多任务并行提高开发效率

缺点：  

1. 高并发会带来频繁的线程切换，这个过程资源消耗很大，当问题没有达到一定规模时，并发编程的效率甚至不如单线程;

   多个程序员开发同一个任务，有时候沟通成本会很大，当意见不统一时会导致项目停滞，另外：一些简单的任务，写个静态HTML页面，一个人做反而更快  

2. 并发编程由于共享变量会带来线程安全问题

   在我们程序员开发过程中，会有一些公共的代码模块，如果程序员A修改了公共模块，如果没有及时沟通，会导致其他成员的代码报错甚至崩溃，这种就是线程安全问题

**线程安全的定义：**多个线程访问同一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他操作，调用这个对象的行为都可以获得正确的结果，那么这个对象就是线程安全的。

**线程安全的根本原因：**CPU高速缓存中缓存的数据和主内存中不一致，指令重排序; 线程安全的问题一般是因为主内存和工作内存数据不一致性和重排序导致的

## 多线程中基本概念
1）同步VS异步  
同步：同步调用,调用者一旦调用方法，必须等到方法结束才能进行下一步操作;  
异步调用：调用者调用完改方法，不需要等待久可以执行下一步；
例如泡茶，必须等到水烧开才能开始泡茶，这个是同步；等待烧水的过程中可以读书或者干别的，这个属于异步

2）并发VS并行  
并发指多个任务交替执行，并行指任务同时进行

3）阻塞VS非阻塞  
一个线程占用了资源，导致别的线程不能访问，被挂起，被称为阻塞
4）临界区  
临界区表示公共资源或者共享数据，可以被多个线程共同使用，但是一旦临界区被一个线程使用，其他线程就必须等待

## JAVA创建线程 
创建线程的方式只有一种Thread 类，Runnable和线程中最终都是创建一个Thread对象去执行Runnable接口中的run方法

1）实现Runnable接口2）继承Thread 类 3）使用ExecutorService线程池 4）Callable、Future 实现带返回结果的多线程

**线程状态**
线程一共有6 种状态（NEW、RUNNABLE、BLOCKED、WAITING、TIME_WAITING、TERMINATED）
**NEW：**初始状态，线程被构建，但是还没有调用start 方法
**RUNNABLED：**运行状态，JAVA 线程把操作系统中的就绪和运行两种状态统一称为“运行中”
**WAITING：**无限期等待，处于这种状态的线程不会被分配到CPU,需要等到被其他线程唤醒
Object.wait(),Thread.join(),LockSupport.park() 会是线程进入这种状态
**BLOCKED：**阻塞状态，表示线程进入等待状态,也就是线程因为某种原因放弃了CPU 使用权，阻塞也分为几种情况
➢ 等待阻塞：运行的线程执行wait 方法，jvm 会把当前线程放入到等待队列
➢ 同步阻塞：运行的线程在获取对象的同步锁时，若该同步锁被其他线程锁占用了，那么jvm 会把当前的线程放入到锁池中
➢ 其他阻塞：运行的线程执行Thread.sleep 或者t.join 方法，或者发出了I/O 请求时，JVM 会把当前线程设置为阻塞状态，当sleep 结束、join 线程终止、io 处理完毕则线程恢复

**TIME_WAITING：**超时等待状态，超时以后自动返回
**TERMINATED：**终止状态，表示当前线程执行完毕

## 线程基本操作
interrupt：线程中断，
InterruptedException线程复位，怎么理解？
当我们对某个线程发起中断请求时（interrupt状态置为true），线程会自己判断是否可以中断，然后如果线程判断当前不能中断，就会把把interrupt状态恢复成false,并抛出InterruptedException异常；

join：
sleep：睡眠，会阻塞
yield：
wait：
nodify：

## JMM(java内存模型)

由于各种CPU指令集（cpu高速缓存），操作系统对内存的控制存在差异，JVM通过创建一种规范来屏蔽这种差异，这个就是JMM，JMM分为两个区域，工作内存和主内存，JMM规定所有变量（包含实例字段，静态字段，构成数组对象的元素，不包括局部变量和方法参数，因为这两个是线程私有的）都存储在主内存中（对应硬件中的主内存），然后每个线程会有自己的工作内存（对应CPU的高速缓存），线程对变量的所有操作都必须在工作内存中执行；
注：如果变量是引用类型，那么线程中只会复制变量的引用，而不会复制变量中的所有对象，先拿到变量的引用，然后在根据引用去工作内存中取数据

## 重排序

1、编译器优化重排序，在不改变单线程程序语义的前提下，重新排序（多线程下可能有问题）
2、指令级并行，如果指令直接没有依赖关系，处理器会将多条指令重叠执行
3、内存重排序，由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行

**Happens-Before**

1. 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。

   保证在一个线程中，前面的代码先执行，后面后执行 

2. 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。

   保证被锁的代码执行完释放锁之后，其他线程才能加锁  

3. volatile变量规则：对一个volatile变量的写，happens-before于任意后续对这个volatile变量的读。  

   保证volatile的可见性  

4. 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

   保证程序的逻辑顺序 

5. start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。  

6. join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。



**单例模式双重检查锁**

~~~java
public class Singleton {
   private static Singleton singleton;
   private Singleton(){}
   public static Singleton getInstance(){
       if(singleton == null){                              // 1
           synchronized (Singleton.class){                 // 2
               if(singleton == null){                      // 3
                   singleton = new Singleton();            // 4
               }
           }
       }
       return singleton;
   }
}
~~~

上面的单例创建是有问题的；原因如下：

创建对象过程，实例化一个对象要分为三个步骤：

1、分配内存空间；2、初始化对象；3、将内存空间的地址赋值给对应的引用

但是由于重排序的缘故，步骤2、3可能会发生重排序，其过程如下：

1、分配内存空间；2、将内存空间的地址赋值给对应的引用；3、初始化对象

所以变量**singleton**必须加volatile关键字，禁止重排序

## CAS

CAS: compareAndSet/compareAndSwap，是一种乐观锁，乐观锁认为并发是不常存在的（偶尔有并发），只有不断尝试，直到成功为止就可以了

CAS操作过程： CAS(V,O,N)，包含三个值分别为：V 内存地址存放的实际值；O 预期的值（旧值）；N 更新的新值；当V和O相同时，即内存中值和预期值相同，表示V没被修改过，这是把V的值修改为N是没问题的，CAS成功；反之，内存中的值和预期值不同时，说明V已经被别的线程修改过了，CAS失败；需要重新获取内存中的值然后重新CAS,直到成功

CAS存在的问题：

**1、ABA问题**

ABA问题，如果变量从A->B->A 变量版本发生了变化，但是CAS操作无法识别，解决思路添加版本号，每次更新版本号加1

**2、自旋问题**

循环环开销时间大，如果CAS一直不成功，就会一直占用线程，造成资源浪费

**3、只能保证一个共享变量的原子操作**

如果要同时修改多个变量，保证多个变量的一致性，CAS操作无法解决，需要通过锁来实现



## synchronized  

synchronized锁的是什么，怎么锁的  
java中实现锁需要哪些东西  
1、锁对象，记录哪个线程获取到了锁，并让其他线程等待，  
2、锁状态：记录当前锁的状态，如果锁可以重入，还需要记录重入次数，  
3、未获取到锁的进程怎么处理，  
4、锁释放后如何唤醒阻塞的线程

Synchronized只能锁Object及子类，（8大基本类型锁不了）  
对于普通同步方法，锁是当前实例对象。  
对于静态同步方法，锁是当前类的Class对象。  
对于同步方法块，锁是Synchonized括号里配置的对象。   

Object头文件会在内存中开辟空间，Object中有一个MarkWord存储对象运行时的数据

<table>
    <thead>
        <tr>
            <th rowspan="2">锁状态</th>
            <th colspan="2">25bit</th>
            <th rowspan="2">4bit</th>
            <th >1bit</th>
            <th >2bit</th>
        </tr>
          <tr> 
            <th>23bit</th>
            <th>2bit</th> 
            <th>是否偏向锁</th> 
            <th>锁标志</th> 
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>无状态</td>
            <td colspan="2">对象的hashCode</td>
            <td>分代年龄</td>
            <td>0</td>
            <td>01</td>
        </tr>
        <tr>
            <td>偏向锁</td>
            <td >线程ID</td>
            <td>Epoch</td>
            <td>分代年龄</td>
            <td>1</td>
            <td>01</td>
        </tr>
        <tr>
            <td>轻量级锁</td>
            <td colspan="4">指向栈中锁记录的指针</td> 
            <td>00</td>
        </tr>
        <tr>
            <td>重量级锁</td>
            <td colspan="4">指向重量级锁的指针</td> 
            <td>10</td>
        </tr>
        <tr>
            <td>GC标记</td>
            <td colspan="4">空</td> 
            <td>11</td>
        </tr>
    </tbody> 
</table>

为什么任何对象都可以实现锁
1. 首先，Java 中的每个对象都派生自Object 类，而每个Java Object 在JVM 内部都有一个native 的C++对象oop/oopDesc 进行对应。
2. 线程在获取锁的时候，实际上就是获得一个监视器对象(monitor) ,monitor 可以认为是一个同步对象，所有的Java 对象是天生携带monitor。

无锁 --> 偏向锁 --> 轻量级锁 --> 重量级锁

### 偏向锁

通过统计发现，大部分情况下锁不仅不存在竞争，而且锁总是由同一个线程多次获得，为了让锁获取的代价更低，减少不必要的CAS操作，引入了偏向锁；JDK1.6之后默认开启偏向锁；我们可以通过参数启用或禁用偏向锁

- 开启偏向锁：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
- 关闭偏向锁：-XX:-UseBiasedLocking=false

**偏向锁获取锁过程**

1. 检查MarkWord是否偏向锁，锁标志为01，是否偏向锁为1；
2. 如果是偏向锁，判断当前线程ID与MarkWord中的线程ID是否一致；如果一致执行5，否则执行3
3. 如果线程ID不一致，通过CAS竞争锁，竞争成功后修改MarkWord中的线程ID，否则执行4；
4. 通过CAS竞争锁失败，证明当前存在多线程竞争情况，到达全局安全点（获得偏向锁的线程被挂起，不执行）；然后偏向锁升级为轻量级锁
5. 执行同步代码

**释放锁过程**

偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。偏向锁的撤销需要等待全局安全点（这个时间点是上没有正在执行的代码）。其步骤如下：

1. 暂停拥有偏向锁的线程，判断锁对象是否还处于被锁定状态；
2. 撤销偏向锁，恢复到无锁状态（01）或者轻量级锁的状态；

![1559525875439](img/偏向锁.png)

### 轻量级锁

在实际场景中，会存在这种场景，多个线程交替执行同步代码（存在锁竞争，但是不激烈），或者同步代码很快就可以执行完，这种场景下，重量级锁的开销就会非常大；所以JDK引入了轻量级锁

**轻量级锁加锁**

1. 判断当前对象是否处于无锁状态(锁标志01，偏向锁0)，如果是无锁状态，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word；否则执行步骤3
2. 然后线程尝试通过CAS将对象头中的Mark Word替换为指向锁记录的指针，CAS成功表示获取到锁，把锁状态修改为00（轻量级锁），然后执行同步代码块，CAS失败则执行步骤3
3. 通过自旋CAS修改对象的Mark Word(通过自旋获取锁)，有次数限制，超过一定次数如果没有获取到锁，会导致锁膨胀成重量级锁

**轻量级锁解锁**

轻量级锁解锁是，会使用CAS将Displaced Mark Word替换为原来的对象头，如果成功表示没有锁竞争，失败会导致锁膨胀；



![1559525875439](img/轻量级锁.png)

### 重量级锁

重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高



**锁的优缺点对比**

| 锁       | 优点                                                         | 缺点                                         | 适用场景                           |
| -------- | ------------------------------------------------------------ | -------------------------------------------- | ---------------------------------- |
| 偏向锁   | 加锁和解锁不需要额外的消耗，和执行非同步方法（不加锁）相比仅存在纳秒级的差距 | 如果线程存在锁竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问同步块的场景 |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度                     | 如果线程始终竞争不到锁，会自旋消耗CPU        | 追求响应时间，同步块执行非常快     |
| 重量级锁 | 线程竞争不会自旋，不会消耗CPU                                | 线程阻塞，响应非常慢                         | 追求吞吐量，同步块执行很慢         |



## volatile

Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。

通俗的就，volatile修饰的变量，java可以保证这个变量所有线程拿到值的时候是一致的，如果某个线程更新了这个变量，其他线程可以立马看到这个更新，这就是所谓的线程可见性。

原理：汇编指令中：在修改带有volatile 修饰的成员变量时，会多一个lock 指令

1. Lock前缀的指令会引起处理器缓存写回内存；
2. 一个处理器的缓存回写到内存会导致其他处理器的缓存失效；（缓存一致性协议MESI）
3. 当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。

volatile变量自身具有下列特性。

1. 可见性。对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
2. 原子性：对任意单个volatile变量的读/写具有原子性（对volatile变量直接赋值 a = 1），但类似于volatile++这种复合操作不具有原子性。

volatile变量规则：对一个volatile变量的写，happens-before于任意后续对这个volatile变量的读。

> volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。

上面那段话，有两层语义

1. 保证可见性、不保证原子性
2. 禁止指令重排序

## final

final可以修饰方法、变量和类，被final修饰的内容一旦被赋值就不会改变



## JAVA中的锁

### lock

lock本质是一个接口，定义锁的标准规范，最重要的两个方法获得锁lock和释放锁unlock

~~~java
Lock lock = new ReentrantLock();
lock.lock();// 获得锁
try {
} finally {
    // 释放锁
	lock.unlock();
}
~~~

### AQS(同步队列模版)

AQS全称AbstractQueuedSynchronizer，定义了一个实现锁机制的抽象模版

包含：state锁状态，同步队列（head,tail 通过双向链表实现队列），独占锁和共享锁

void acquire(int arg)：独占式获取同步状态，如果获取失败则插入同步队列进行等待； 

void acquireShared(int arg)：共享式获取同步状态，失败进入同步队列；与独占式的区别在于同一时刻有多个线程获取同步状态；

![1559525875439](img/AQS.png)

内部类，同步队列中的结点

```java
static final class Node {
    /** 共享锁中线程可以同时访问，共享锁在同步队列中共享同一个结点 */
    static final Node SHARED = new Node();
    /** 独占线程结点 */
    static final Node EXCLUSIVE = null; 
    /** 关闭状态，在队列中等待超时或被中断，会进入这种状态 */
    static final int CANCELLED =  1;
    /** 后继结点的线程处于等待状态，如果当前结点的线程被释放锁或者取消，需要唤醒下一个结点的线程 */
    static final int SIGNAL    = -1;
    /** 结点在等待队列中，结点线程在Condition队列中 */
    static final int CONDITION = -2; 
    static final int PROPAGATE = -3;
 	//等待状态
    volatile int waitStatus;
 	// AQS同步队列 前驱结点(双向链表)
    volatile Node prev; 
    // AQS同步队列 后继结点(双向链表)
    volatile Node next;
 	// 当前结点的线程
    volatile Thread thread;
  	// Condition等待队列的后继结点(单链表)
  	Node nextWaiter;
}
```

AQS独占锁  -  获得锁和释放锁方法

```java
// 独占式获取同步状态，如果获取失败则插入同步队列进行等待； 
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // tryAcquire定义好模版，有子类实现，成功则把当前线程设为独占，失败
        // 如果当前线程获取锁失败
        // addWaiter 将当前线程封装成Node结点 插入队尾
        // acquireQueued 排队获取到锁
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
// addWaiter 将当前线程封装成Node结点 插入队尾
private Node addWaiter(Node mode) {
    // 将当前线程封装成Node结点
    Node node = new Node(Thread.currentThread(), mode); 
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        // 先通过CAS操作，把 node 设为 tail
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // tail=null 或者 CAS失败，循环入队
    enq(node);
    return node;
}
// node结点入队
private Node enq(final Node node) {
    // 自旋操作
    for (;;) {
        Node t = tail;
        // 如果tail为空,通CAS设置head结点
        if (t == null) {  
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {// 否则，把node结点放入队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

// acquireQueued 排队获取到锁
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 自旋
        for (;;) {
            // 当前结点的前置结点
            final Node p = node.predecessor();
            // 如果前置结点是head节点，说明有资格去争抢锁 
            // tryAcquire 争抢独占锁，成功
            if (p == head && tryAcquire(arg)) {
                setHead(node); // 报当前结点是指为头结点
                p.next = null; // 把原来的头结点断开 
                failed = false;
                return interrupted;
            }
            // tryAcquire 失败
            // shouldParkAfterFailedAcquire 判断当前线程是否需要park
            // 如果shouldParkAfterFailedAcquire返回true；挂起当前线程parkAndCheckInterrupt()
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 判断当前线程是否需要park
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 如果前驱结点是SIGNAL 状态，前驱结点排完队会通知，直接返回true
    if (ws == Node.SIGNAL) 
        return true;
    if (ws > 0) { // 前驱结点是 关闭 CANCELLED 状态
        do {
            // 往前找  前驱结点的前驱结点
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 如果没有关闭，就把前驱结点的状态修改为 SIGNAL 然后等待通知
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
} 

// 释放锁
public final boolean release(int arg) {
    if (tryRelease(arg)) { // tryRelease 释放独占的线程，定义好模版，子类重写
        Node h = head; // 找到头结点
        if (h != null && h.waitStatus != 0)
            // 唤醒下一个节点
            unparkSuccessor(h); 
        return true;
    }
    return false;
} 
```

AQS共享锁  -  获得锁和释放锁方法

```java
// 共享式获取同步状态，如果获取失败（一般由于存在线程写操作获取了独占锁）则插入同步队列进行等待； 
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

// 共享锁线程排队
private void doAcquireShared(int arg) {
    // 共享锁线程统一为一个结点 Node.SHARED，然后插入队尾
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

总结：AQS可以理解为现实中的叫号，我们去银行办事，会给我们一个排队的号码，如果等了半天，还没轮到我们，我们就会休息，休息之前会去找我们前面的人（如果走了就找前面的前面）让他办完事通知我们下





### ReentrantLock（重入锁、独占）

重入锁ReentrantLock，顾名思义，就是支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁

ReentrantLock获取锁的入口

```java
public void lock() {
    sync.lock();
} 
private final Sync sync; //Sync是内部抽象类，继承了AbstractQueuedSynchronizer（AQS）
// 有两个子类FairSync（公平锁） NonfairSync（非公平锁） 默认NonfairSync

```



```java
// 非公平锁
final void lock() {
    // 线程进来直接CAS（不管有没有排队直接CAS）
    if (compareAndSetState(0, 1))
        // 插队成功后，把当前线程设为独占线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 插队失败,排队获取锁
        acquire(1);
}
// 非公平锁获取锁方法 父类定义好的模版子类实现
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
} 
// 非公平获取锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();// 当前线程
    int c = getState(); // 获取当前锁状态
    if (c == 0) { // 为0说明没有被锁
        // 通过CAS操作把当前线程设置为锁的独占线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果 当前线程和锁的独占线程相同-> 重入，并记录重入次数
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



```java
// 公平锁
final void lock() {
    // 直接排队获取锁
    acquire(1);
}

// 公平锁获取锁方法 父类定义好的模版子类实现
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread(); // 当前线程
    int c = getState();// 获取当前锁状态
    if (c == 0) { // 为0说明没有被锁
        // 公平锁首先需要判断 有没有线程在排队，
        // 没有正在排队 通过CAS操作把当前线程设置为锁的独占线程
        // 有正在排队的线程 往下走 ——>返回false
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果 当前线程和锁的独占线程相同-> 重入，并记录重入次数
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

```

### 公平锁VS非公平锁

公平锁遵循新进先出原则，先来的线程会优先获得锁，保证了线程请求的时间顺序，而非公平锁允许刚来的线程插队，可能导致部分线程永远在排队，造成饥饿现象

公平锁为了保证时间上的绝对公平，需要进行大量的上下文切换，会增加性能开销，而非公平锁能一点程度减少上下文切换，提高吞吐量，ReentrantLock默认使用非公平锁，保证系统有更大的吞吐量



### ReentrantReadWriteLock（读写锁）

ReentrantReadWriteLock在初始化时会创建读锁和写锁; 
同时会根据参数创建FairSync或者NonfairSync，两者都继承了内部类Sync,本质是一个同步阻塞队列 

需要明确：读锁是共享锁，读线程在同一时刻可以允许多个线程访问; 
写锁是独占锁,当线程获得写锁后,任何的读写操作都会被阻塞 

写锁（独占锁）

```java
// 写锁获取锁方法 父类定义好的模版子类实现
protected final boolean tryAcquire(int acquires) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);// 写锁state 二进制后16位
    if (c != 0) {// 已经有线程获得了锁 
        // 不存在写锁 且 c > 0,说明写锁（共享锁）状态不为0,有线程读取数据,
        // 不能进行写操作，或出现脏数据
        // 或者 当前线程 <> 获得锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 修改线程状态 独占锁重入次数+1
        setState(c + acquires);
        return true;
    }
    // c == 0 线程没有被锁
    // writerShouldBlock():判断是否需要阻塞,非公平锁不需要直接CAS
    // 公平锁需要判断阻塞队列中是否存在排队的线程,存在排队直接阻塞,不会CAS
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 把当前线程设为独占线程
    setExclusiveOwnerThread(current);
    return true;
}
// 释放独占锁
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases; // 当前state - 释放次数
    boolean free = exclusiveCount(nextc) == 0;
    if (free) // 如果锁状态为0 ，释放独占线程
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

读锁

```java
// 获取共享锁  父类定义好的模版子类实现
protected final int tryAcquireShared(int unused) { 
    Thread current = Thread.currentThread();
    int c = getState();
    // 写锁state 二进制后16位 写锁状态不为0 说明存在写线程在做写操作
    // 当前线程和正在写的线程不是一个，阻塞
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 写锁state 二进制前16位 
    int r = sharedCount(c);
    // readerShouldBlock():判断是否需要阻塞,
    // 非公平锁需要判断阻塞队列的第一个结点是不是独占锁(写锁),不是独占锁,CAS
    // 公平锁需要判断阻塞队列中是否存在排队的线程,不存在排队,CAS
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        // 在原来基础加2^16 因为前16位标识共享锁状态
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) { // 读锁状态为0,没有线程读操作
            firstReader = current; // 设置当前线程为第一个读线程
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // 当前线程 == 第一个读线程 读count自增
            firstReaderHoldCount++;
        } else {
            // 从ThreadLocal共享变量中获取当前线程的HoldCounter,记录次数
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    // 需要阻塞 或者CAS失败
    return fullTryAcquireShared(current);
}

// 需要阻塞 或者CAS失败 需要再次尝试获取锁（自旋）
final int fullTryAcquireShared(Thread current) {
    
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        
        // 自旋过程中有写锁抢占了线程
        // 写锁状态不为0 说明存在写线程在做写操作
        if (exclusiveCount(c) != 0) {
            // 当前线程和正在写的线程不是一个，阻塞
            if (getExclusiveOwnerThread() != current)
                return -1; // 获取写锁失败
        } else if (readerShouldBlock()) {
             
            if (firstReader == current) {
                 
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}

// 创建readHolds ThreadLocal变量,记录每个线程的读取次数
private transient ThreadLocalHoldCounter readHolds;

static final class HoldCounter {
    int count = 0; 
    final long tid = getThreadId(Thread.currentThread());
}
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

### Condition接口

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。**Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式**

Condition 是一个多线程协调通信的工具类，可以让某些线程一起等待某个条件（condition），只有满足条件时，线程才会被唤醒

首先，执行Condition方法之前必须先获得锁

Condition是一个接口，实现类是AQS（AbstractQueuedSynchronizer）的内部类ConditionObject；主要方法

> **针对Object的wait方法**

1. void await()：当前线程进入等待状态，如果其他线程调用condition的signal或者signalAll方法并且当前线程获取Lock从await方法返回，如果在等待状态中被中断会抛出被中断异常；
2. long awaitNanos(long nanosTimeout)：当前线程进入等待状态直到被通知，中断或者超时；
3. boolean await(long time, TimeUnit unit)：同第二种，支持自定义时间单位
4. boolean awaitUntil(Date deadline) ：当前线程进入等待状态直到被通知，中断或者到了某个时间

> **针对Object的notify/notifyAll方法**

1. void signal()：唤醒一个等待在condition上的线程，将该线程从等待队列中转移到同步队列中，如果在同步队列中能够竞争到Lock则可以从等待方法中返回。
2. void signalAll()：与1的区别在于能够唤醒所有等待在condition上的线程

```java
// 是当前线程进入等待队列
public final void await() throws InterruptedException {
    if (Thread.interrupted()) // 判断当前线程是否被中断
        throw new InterruptedException();
    // 把当前线程包装成Node,结点状态 CONDITION，然后放到队列尾部
    Node node = addConditionWaiter();
    // 释放当前锁，并唤醒AQS队列中下一个结点（进入await方法的前提是获得了锁，
    // 要进入wait状态就必须释放锁，不然别的线程没办法获得锁了）
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // isOnSyncQueue判断当前线程是不是在AQS队列中 
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // park挂起当前线程
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 当这个线程醒来,会尝试拿锁, 当acquireQueued 返回false 就是拿到锁了.
    // interruptMode != THROW_IE -> 表示这个线程没有成功将node 入队 

    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
// 把当前线程包装成Node, 然后放到队列尾部
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果lastWaiter不为空且状态不为CONDITION 
    if (t != null && t.waitStatus != Node.CONDITION) {
        // 从first结点遍历把状态不为CONDITION的结点移除
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 当前线程封装成Node，状态为CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)// 如果链表尾结点为空，把当前结点设为头结点
        firstWaiter = node;
    else // 如果链表尾结点为空，把当前结点设为头lastWaiter的next结点
        t.nextWaiter = node;
    lastWaiter = node;//当前结点加入到last
    return node;
}
```

```java
// 唤醒当前线程
public final void signal() {
    // 判断当前线程是否获得了锁，没有锁抛异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
// 从first结点开始，把每个结点移入AQS队列，等待执行
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) { 
    // 把当前结点的状态设置为0，初始状态
    // 如果更新失败，只有一种可能就是节点被CANCELLED了 
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 结点node入队
    Node p = enq(node);
    int ws = p.waitStatus;
    // 
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);//唤醒结点上的线程
    // 如果node的prev节点已经是signal状态，那么被阻塞的ThreadA的唤醒工作由AQS队列来完成
    return true;
}
```

![1559728986268](img/condition.png)

### LockSupport工具

LockSupport定义了一组的公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也成为构建同步组件的基础工具。
LockSupport定义了一组以park开头的方法用来阻塞当前线程，以及unpark(Thread thread)方法来唤醒一个被阻塞的线程



> **阻塞线程**

1. void park()：阻塞当前线程，如果调用unpark方法或者当前线程被中断，从能从park()方法中返回
2. void park(Object blocker)：功能同方法1，入参增加一个Object对象，用来记录导致线程阻塞的阻塞对象，方便进行问题排查；
3. void parkNanos(long nanos)：阻塞当前线程，最长不超过nanos纳秒，增加了超时返回的特性；
4. void parkNanos(Object blocker, long nanos)：功能同方法3，入参增加一个Object对象，用来记录导致线程阻塞的阻塞对象，方便进行问题排查；
5. void parkUntil(long deadline)：阻塞当前线程，直到deadline（当前时间，用毫秒）；
6. void parkUntil(Object blocker, long deadline)：功能同方法5，入参增加一个Object对象，用来记录导致线程阻塞的阻塞对象，方便进行问题排查；

> **唤醒线程**

void unpark(Thread thread):唤醒处于阻塞状态的指定线程



## 线程池（Executor体系）

### 线程池的好处
1、降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。  
2、提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。  
3、提高线程的可管理性。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。合理的设置线程池大小可以避免因为线程数超过硬件资源瓶颈带来的问题 

**线程池执行原理**

![Image text](img/thread-pool.png)

### java中的线程池API

 newFixedThreadPool：该方法返回一个固定数量的线程池，线程数不变，当有一个任务提交时，若线程池中空闲，则立即执行，若没有，则会被暂缓在一个任务队列中，等待有空闲的线程去执行。
newSingleThreadExecutor: 创建一个线程的线程池，若空闲则执行，若没有空闲线程则暂缓在任务队列中。
newCachedThreadPool：返回一个可根据实际情况调整线程个数的线程池，不限制最大线程数量，若用空闲的线程则执行任务，若无任务则不创建线程。并且每一个空闲线程会在60 秒后自动回收
newScheduledThreadPool: 创建一个可以指定线程的数量的线程池，但是这个线程池还带有延迟和周期性执行任务的功能，类似定时器。

```java
// 线程池核心构造函数
public ThreadPoolExecutor(int corePoolSize,// 核心线程数
    int maximumPoolSize,// 最大线程数
    long keepAliveTime, // 线程池的工作线程空闲后，保持存活的时间
    TimeUnit unit, // 时间单位
    BlockingQueue<Runnable> workQueue,//用于保存等待执行的任务的阻塞队列
    ThreadFactory threadFactory,// 用于设置创建线程的工厂
    RejectedExecutionHandler handler) { // 拒绝策略 当队列和线程池都满了，
    // 说明线程池处于饱和状态，那么必须采取一种策略 处理提交的新任务
    
}
//  workQueue 的类型为BlockingQueue，通常可以取下面三种类型：
// 1. ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；
// 2. LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为
// Integer.MAX_VALUE；
// 3. SynchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个
// 线程来执行新来的任务。一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用
// 移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工
// 厂方法Executors.newCachedThreadPool使用了这个队列。

// RejectedExecutionHandler handler 拒绝策略
// 1、AbortPolicy：直接抛出异常，默认策略；
// 2、CallerRunsPolicy：用调用者所在的线程来执行任务；
// 3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
// 4、DiscardPolicy：直接丢弃任务；
```

### 线程池执行原理

```java
// ctl 用来保存线程数量和线程池的状态
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 阻塞队列
private final BlockingQueue<Runnable> workQueue;
// HashSet Worker的Set集合，保存
private final HashSet<Worker> workers = new HashSet<Worker>();
// 线程池执行方法execute
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException(); 
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        // 当前线程数 < 核心线程数，新建一个线程执行任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 核心池已满，但任务队列未满，添加到队列中 
    if (isRunning(c) && workQueue.offer(command)) {
        // 任务成功添加到队列以后，再次检查是否需要添加新的线程，因为已存在的线程可能被销毁了
        int recheck = ctl.get(); 
        // 如果线程池处于非运行状态，并且把当前的任务从任务队列中移除成功，则拒绝该任务
        if (! isRunning(recheck) && remove(command))
            reject(command);//拒绝该任务
        // 如果之前的线程已被销毁完，新建一个线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3.核心池已满，队列已满，试着创建一个新线程
    else if (!addWorker(command, false))
        // 如果创建新线程失败了，说明线程池被关闭或者线程池完全满了，拒绝任务
        reject(command);
}
// 新建一个线程执行任务
private boolean addWorker(Runnable firstTask, boolean core) {
    // 第一步 通过CAS操作记录线程数
    retry: // goto语句，双重循环中跳出循环
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c); 
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry; 
        }
    }
	// 第二步 新建一个Worker
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 把当前Runnable任务创建线程-》封装为Work
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 重入锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try { 
                int rs = runStateOf(ctl.get()); 
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) 
                        throw new IllegalThreadStateException();
                    workers.add(w);// 加入到workers集合
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 如果work创建成功
            if (workerAdded) {
                t.start(); // 启动线程 ——》 线程是启动了，然后执行run方法
                workerStarted = true;
            }
        }
    } finally {
        //如果添加失败，需要从workers集合中移除w，因为前面已经add了 ，并减少线程数
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
} 
// 启动线程后线程run
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // 运行中断
    boolean completedAbruptly = true;
    try {
        // 通过getTask()从队列中获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock(); 
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 空方法，如果继承线程池可以重新这个方法，在线程运行前执行
                beforeExecute(wt, task); 
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // // 空方法，如果继承线程池可以重新这个方法，在线程运行后执行
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
// 获取队列中的任务
private Runnable getTask() {
    boolean timedOut = false;  
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c); 
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 判断核心线程是否允许超时，或者当前线程数 > 核心线程数
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 如果线程数 > 最大线程数
        // 如果有超时机制timed 且 上次获取队列超时
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;//返回null，则当前worker线程会退出
            continue;
        }

        try {
            // 如果有超时机制 或者超过了核心线程数
            Runnable r = timed ?
                // 有超时机制，poll只会阻塞keepAliveTime的时间
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take(); // 没有超时机制，在队列为空时take会一直阻塞
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
// 按过去执行已提交任务的顺序发起一个有序的关闭，但是不接受新任务。
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN); // 通过CAS把状态设为SHUTDOWN,有新线程进来会拒绝
        interruptIdleWorkers(); // 中断空闲的线程
        onShutdown(); // 交给子类实现
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
// 尝试停止所有的活动执行任务、暂停等待任务的处理，并返回等待执行的任务列表。
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP); // 通过CAS把状态设为STOP，现象运行是会尝试中断
        interruptWorkers();
        tasks = drainQueue(); // 返回等待执行的任务列表
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

## ThreadLocal

ThreadLocal可以称为线程本地变量，或者线程本地存储；作用是提供线程内的局部变量（变量只在当前线程有效，并且在线程的生命周期内起作用，线程消失后变量也会消失）

~~~java
public class ThreadLocal<T> {
    public T get() {
        Thread t = Thread.currentThread(); // 获取当前线程
        // 获取当前线程的成员变量 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    //获取当前线程所对应的ThreadLocalMap，如果不为空，则调用ThreadLocalMap的set()方法，key就是当前ThreadLocal，
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else // 如果不存在，则调用createMap()方法新建一个
            createMap(t, value);
    }
    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    
    public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
    } 
}
~~~

- ThreadLocal 不是用于解决共享变量的问题的，也不是为了协调线程同步而存在，而是为了方便每个线程处理自己的状态而引入的一个机制。这点至关重要。
- 每个Thread内部都有一个ThreadLocal.ThreadLocalMap类型的成员变量，该成员变量用来存储实际的ThreadLocal变量副本。
- ThreadLocal并不是为线程保存对象的副本，它仅仅只起到一个索引的作用。它的主要木得视为每一个线程隔离一个类的实例，这个实例的作用范围仅限于线程内部。

**ThreadLocal注意事项**

**脏数据**

线程复用会产生脏数据。由于线程池会重用Thread对象，那么与Thread绑定的类的静态属性ThreadLocal变量也会被重用。如果在实现的线程run()方法体中不显式地调用remove() 清理与线程相关的ThreadLocal信息，那么倘若下一个结程不调用set() 设置初始值，就可能get() 到重用的线程信息，包括 ThreadLocal所关联的线程对象的value值。

**内存泄漏**

通常我们会使用使用static关键字来修饰ThreadLocal（这也是在源码注释中所推荐的）。在此场景下，其生命周期就不会随着线程结束而结束，寄希望于ThreadLocal对象失去引用后，触发弱引用机制来回收Entry的Value就不现实了。如果不进行remove() 操作，那么这个线程执行完成后，通过ThreadLocal对象持有的对象是不会被释放的。

以上两个问题的解决办法很简单，就是在每次用完ThreadLocal时， 必须要及时调用 remove()方法清理。

**父子线程共享线程变量**

很多场景下通过ThreadLocal来透传全局上下文，会发现子线程的value和主线程不一致。比如用ThreadLocal来存储监控系统的某个标记位，暂且命名为traceId。某次请求下所有的traceld都是一致的，以获得可以统一解析的日志文件。但在实际开发过程中，发现子线程里的traceld为null，跟主线程的并不一致。这就需要使用InheritableThreadLocal来解决父子线程之间共享线程变量的问题，使整个连接过程中的traceId一致。

## 并发容器（线程安全集合）

### ConcurrentHashMap



### CopyOnWriteArrayList和CopyOnWriteArraySet

CopyOnWriteArrayList

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    /** 通过重入锁保证线程安全 */
    final transient ReentrantLock lock = new ReentrantLock();

    /** 通过数组记录元素，并通过volatile保证可见性 */
    private transient volatile Object[] array;
    // 新增元素，通过ReentrantLock保证线程安全 
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock(); // 获得锁
        try {
            Object[] elements = getArray();
            int len = elements.length;
            // 每次把原来的数组复制
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();// 释放锁
        }
    }
    // get操作，通过volatile保证可见性 
    public E get(int index) {
        return get(getArray(), index);
    }
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
}
```

CopyOnWriteArraySet 通过对CopyOnWriteArrayList新增时判断元素是否存在，保证Set元素不重复

```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    // 内部封装了一个CopyOnWriteArrayList    
    private final CopyOnWriteArrayList<E> al;   
    
    // 新增方法 CopyOnWriteArrayList 先判断元素e,是否在数组中存在，不存在新增
    public boolean add(E e) {
        return al.addIfAbsent(e); 
    }
}
```

### ConcurrentSkipListMap和ConcurrentSkipListSet



ConcurrentSkipListSet内部封装了ConcurrentSkipListMap实现Set功能

### ArrayBlockingQueue 和LinkedBlockingQueue

ArrayBlockingQueue 





LinkedBlockingQueue





### LinkedTransferQueue



### PriorityBlockingQueue

