# AbstractQueuedSynchronizer

## AQS 结构

先来看看 AQS 有哪些属性，搞清楚这些基本就知道 AQS 是什么套路了。

```java
// 头结点，你直接把它当做 当前持有锁的线程 可能是最好理解的
private transient volatile Node head;

// 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
private transient volatile Node tail;

// 这个是最重要的，代表当前锁的状态，0代表没有被占用，大于 0 代表有线程持有当前锁
// 这个值可以大于 1，是因为锁可以重入，每次重入都加上 1
private volatile int state;

// 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
// if (currentThread == getExclusiveOwnerThread()) {state++}
private transient Thread exclusiveOwnerThread; //继承自AbstractOwnableSynchronizer
```

AbstractQueuedSynchronizer 的等待队列示意如下所示，注意了，之后分析过程中所说的 queue，也就是阻塞队列**不包含 head，不包含 head，不包含 head**。

![aqs-0](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer/aqs-0.png)

等待队列中每个线程被包装成一个 Node 实例，数据结构是链表，一起看看源码吧：

```java
static final class Node {
    // 标识节点当前在共享模式下
    static final Node SHARED = new Node();
    // 标识节点当前在独占模式下
    static final Node EXCLUSIVE = null;

    // ======== 下面的几个int常量是给waitStatus用的 ===========
    /** waitStatus value to indicate thread has cancelled */
    // 代码此线程取消了争抢这个锁
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    // 本文不分析condition，所以略过吧，下一篇文章会介绍这个
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    // 同样的不分析，略过吧
    static final int PROPAGATE = -3;
    // =====================================================


    // 取值为上面的1、-1、-2、-3，或者0(以后会讲到)
    // 这么理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待，
    //    ps: 半天抢不到锁，不抢了，ReentrantLock是可以指定timeouot的。。。
    volatile int waitStatus;
    // 前驱节点的引用
    volatile Node prev;
    // 后继节点的引用
    volatile Node next;
    // 这个就是线程本尊
    volatile Thread thread;

}
```

Node 的数据结构其实也挺简单的，就是 thread + waitStatus + pre + next 四个属性而已，大家先要有这个概念在心里。



## 线程抢锁

以ReentrantLockg公平锁为例，公平锁和非公平锁区别就是每个线程来抢锁的时候看看队列里是不是又线程在等，有的话公平直接去等，非公平先试着抢一下再去等。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
      // 争锁
    final void lock() {
        acquire(1);
    }
      // 来自父类AQS，我直接贴过来这边，下面分析的时候同样会这样做，不会给读者带来阅读压力
    // 我们看到，这个方法，如果tryAcquire(arg) 返回true, 也就结束了。
    // 否则，acquireQueued方法会将线程压到队列中
    public final void acquire(int arg) { // 此时 arg == 1
        // 首先调用tryAcquire(1)一下，名字上就知道，这个只是试一试
        // 因为有可能直接就成功了呢，也就不需要进队列排队了，
        // 对于公平锁的语义就是：本来就没人持有锁，根本没必要进队列等待(又是挂起，又是等待被唤醒的)
        if (!tryAcquire(arg) &&
            // tryAcquire(arg)没有成功，这个时候需要把当前线程挂起，放到阻塞队列中。
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {
              selfInterrupt();
        }
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    // 尝试直接获取锁，返回值是boolean，代表是否获取到锁
    // 返回true：1.没有线程在等待锁；2.重入锁，线程本来就持有锁，也就可以理所当然可以直接获取
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // state == 0 此时此刻没有线程持有锁
        if (c == 0) {
            // 虽然此时此刻锁是可以用的，但是这是公平锁，既然是公平，就得讲究先来后到，
            // 看看有没有别人在队列中等了半天了
            if (!hasQueuedPredecessors() &&
                // 如果没有线程在等待，那就用CAS尝试一下，成功了就获取到锁了，
                // 不成功的话，只能说明一个问题，就在刚刚几乎同一时刻有个线程抢先了 =_=
                // 因为刚刚还没人的，我判断过了
                compareAndSetState(0, acquires)) {

                // 到这里就是获取到锁了，标记一下，告诉大家，现在是我占用了锁
                setExclusiveOwnerThread(current);
                return true;
            }
        }
          // 会进入这个else if分支，说明是重入了，需要操作：state=state+1
        // 这里不存在并发问题
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 如果到这里，说明前面的if和else if都没有返回true，说明没有获取到锁
        // 回到上面一个外层调用方法继续看:
        // if (!tryAcquire(arg) 
        //        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
        //     selfInterrupt();
        return false;
    }

    // 假设tryAcquire(arg) 返回false，那么代码将执行：
      //        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)，
    // 这个方法，首先需要执行：addWaiter(Node.EXCLUSIVE)

    /**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    // 此方法的作用是把线程包装成node，同时进入到队列中
    // 参数mode此时是Node.EXCLUSIVE，代表独占模式
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // 以下几行代码想把当前node加到链表的最后面去，也就是进到阻塞队列的最后
        Node pred = tail;

        // tail!=null => 队列不为空(tail==head的时候，其实队列是空的，不过不管这个吧)
        if (pred != null) { 
            // 将当前的队尾节点，设置为自己的前驱 
            node.prev = pred; 
            // 用CAS把自己设置为队尾, 如果成功后，tail == node 了，这个节点成为阻塞队列新的尾巴
            if (compareAndSetTail(pred, node)) { 
                // 进到这里说明设置成功，当前node==tail, 将自己与之前的队尾相连，
                // 上面已经有 node.prev = pred，加上下面这句，也就实现了和之前的尾节点双向连接了
                pred.next = node;
                // 线程入队了，可以返回了
                return node;
            }
        }
        // 仔细看看上面的代码，如果会到这里，
        // 说明 pred==null(队列是空的) 或者 CAS失败(有线程在竞争入队)
        // 读者一定要跟上思路，如果没有跟上，建议先不要往下读了，往回仔细看，否则会浪费时间的
        enq(node);
        return node;
    }

    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    // 采用自旋的方式入队
    // 之前说过，到这个方法只有两种可能：等待队列为空，或者有线程竞争入队，
    // 自旋在这边的语义是：CAS设置tail过程中，竞争一次竞争不到，我就多次竞争，总会排到的
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 之前说过，队列为空也会进来这里
            if (t == null) { // Must initialize
                // 初始化head节点
                // 细心的读者会知道原来 head 和 tail 初始化的时候都是 null 的
                // 还是一步CAS，你懂的，现在可能是很多线程同时进来呢
                if (compareAndSetHead(new Node()))
                    // 给后面用：这个时候head节点的waitStatus==0, 看new Node()构造方法就知道了

                    // 这个时候有了head，但是tail还是null，设置一下，
                    // 把tail指向head，放心，马上就有线程要来了，到时候tail就要被抢了
                    // 注意：这里只是设置了tail=head，这里可没return哦，没有return，没有return
                    // 所以，设置完了以后，继续for循环，下次就到下面的else分支了
                    tail = head;
            } else {
                // 下面几行，和上一个方法 addWaiter 是一样的，
                // 只是这个套在无限循环里，反正就是将当前线程排到队尾，有线程竞争的话排不上重复排
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }


    // 现在，又回到这段代码了
    // if (!tryAcquire(arg) 
    //        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
    //     selfInterrupt();

    // 下面这个方法，参数node，经过addWaiter(Node.EXCLUSIVE)，此时已经进入阻塞队列
    // 注意一下：如果acquireQueued(addWaiter(Node.EXCLUSIVE), arg))返回true的话，
    // 意味着上面这段代码将进入selfInterrupt()，所以正常情况下，下面应该返回false
    // 这个方法非常重要，应该说真正的线程挂起，然后被唤醒后去获取锁，都在这个方法里了
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                // p == head 说明当前节点虽然进到了阻塞队列，但是是阻塞队列的第一个，因为它的前驱是head
                // 注意，阻塞队列不包含head节点，head一般指的是占有锁的线程，head后面的才称为阻塞队列
                // 所以当前节点可以去试抢一下锁
                // 这里我们说一下，为什么可以去试试：
                // 首先，它是队头，这个是第一个条件，其次，当前的head有可能是刚刚初始化的node，
                // enq(node) 方法里面有提到，head是延时初始化的，而且new Node()的时候没有设置任何线程
                // 也就是说，当前的head不属于任何一个线程，所以作为队头，可以去试一试，
                // tryAcquire已经分析过了, 忘记了请往前看一下，就是简单用CAS试操作一下state
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 到这里，说明上面的if分支没有成功，要么当前node本来就不是队头，
                // 要么就是tryAcquire(arg)没有抢赢别人，继续往下看
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 什么时候 failed 会为 true???
            // tryAcquire() 方法抛异常的情况
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    // 刚刚说过，会到这里就是没有抢到锁呗，这个方法说的是："当前线程没有抢到锁，是否需要挂起当前线程？"
    // 第一个参数是前驱节点，第二个参数才是代表当前线程的节点
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // 前驱节点的 waitStatus == -1 ，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;

        // 前驱节点 waitStatus大于0 ，之前说过，大于0 说明前驱节点取消了排队。
        // 这里需要知道这点：进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的。
        // 所以下面这块代码说的是将当前节点的prev指向waitStatus<=0的节点，
        // 简单说，就是为了找个好爹，因为你还得依赖它来唤醒呢，如果前驱节点取消了排队，
        // 找前驱节点的前驱节点做爹，往前遍历总能找到一个好爹的
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            // 仔细想想，如果进入到这个分支意味着什么
            // 前驱节点的waitStatus不等于-1和1，那也就是只可能是0，-2，-3
            // 在我们前面的源码中，都没有看到有设置waitStatus的，所以每个新的node入队时，waitStatu都是0
            // 正常情况下，前驱节点是之前的 tail，那么它的 waitStatus 应该是 0
            // 用CAS将前驱节点的waitStatus设置为Node.SIGNAL(也就是-1)
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        // 这个方法返回 false，那么会再走一次 for 循序，
        //     然后再次进来此方法，此时会从第一个分支返回 true
        return false;
    }

    // private static boolean shouldParkAfterFailedAcquire(Node pred, Node node)
    // 这个方法结束根据返回值我们简单分析下：
    // 如果返回true, 说明前驱节点的waitStatus==-1，是正常情况，那么当前线程需要被挂起，等待以后被唤醒
    //        我们也说过，以后是被前驱节点唤醒，就等着前驱节点拿到锁，然后释放锁的时候叫你好了
    // 如果返回false, 说明当前不需要被挂起，为什么呢？往后看

    // 跳回到前面是这个方法
    // if (shouldParkAfterFailedAcquire(p, node) &&
    //                parkAndCheckInterrupt())
    //                interrupted = true;

    // 1. 如果shouldParkAfterFailedAcquire(p, node)返回true，
    // 那么需要执行parkAndCheckInterrupt():

    // 这个方法很简单，因为前面返回true，所以需要挂起线程，这个方法就是负责挂起线程的
    // 这里用了LockSupport.park(this)来挂起线程，然后就停在这里了，等待被唤醒=======
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

    // 2. 接下来说说如果shouldParkAfterFailedAcquire(p, node)返回false的情况

   // 仔细看shouldParkAfterFailedAcquire(p, node)，我们可以发现，其实第一次进来的时候，一般都不会返回true的，原因很简单，前驱节点的waitStatus=-1是依赖于后继节点设置的。也就是说，我都还没给前驱设置-1呢，怎么可能是true呢，但是要看到，这个方法是套在循环里的，所以第二次进来的时候状态就是-1了。

    // 解释下为什么shouldParkAfterFailedAcquire(p, node)返回false的时候不直接挂起线程：
    // => 是为了应对在经过这个方法后，node已经是head的直接后继节点了。剩下的读者自己想想吧。
}
```

说到这里，也就明白了，多看几遍 `final boolean acquireQueued(final Node node, int arg)` 这个方法吧。自己推演下各个分支怎么走，哪种情况下会发生什么，走到哪里。

总结一些，还是以公平锁为例：

1- 首先tryAcquire(),尝试抢锁，成功则直接setExclusiveOwnerThread，设置为当前线程。失败进入2；

2- addWaiter(),把当前线程封装成节点，同时进入到队列中。首先尝试快速加入队列，cas把自己设置为队尾，成功后当前节点是队尾，把之前队尾与自己链起来后返回，失败则进入enq()方法

​	2.1 enq()采用自旋的方式入队,如果进到这里，说明 pred==null(队列是空的) 或者 CAS失败(有线程在竞争入队)。cas入队的方法和2中类似，只是多了队列为空时的进行队列的初始化，初始化完成后还是cas竞争入队。

​	2.2 这里前后连起来不用担心线程安全问题，因为先把当前的pre设置为尾节点，如果尾节点发生变化不会cas成功；就算cas成功之后cpu被其他线程夺走，也不会应该next连起来。

3- acquireQueued(),这个方法，还是一个循环cas过程。就是入队之后为当前线程找个合适的位置挂起，合适就是让前面的线程知道要唤醒挂起的自己。

​	循环内容：

​	3.1 如果是队列首节点，则尝试tryAcquire(),失败进3.2。因为正在运行的线程是不在队列中的。

​	3.2  shouldParkAfterFailedAcquire：进到这里说明没抢到锁，需要根据前面节点的状态判断当前线程是否需要挂起。

​		这里回顾一下这里会涉及到的节点状态：

​			signal：-1；后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，那么就会通知后继节点，让后继节点的线程能够运行；

​			cancelled：1；值为1，由于超时或中断，该节点被取消。 节点进入该状态将不再变化。特别是具有取消节点的线程永远不会再次阻塞；

​			initial：0；初始状态；

* 如果前驱节点，状态为SIGNAL，当前挂起；return true，进入parkAndCheckInterrupt线程挂起

* if (ws > 0)
      		//大于0 说明前驱节点取消了排队。
      		//这里需要知道这点：进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的。
      		//所以需要将当前节点的prev指向waitStatus<=0的节点，简单说，就是为了找个好爹，因为你还得依赖它来唤醒呢，如果前驱节点取消了排队，找前驱节点的前驱节点做爹，往前遍历总能找到一个好爹的

* 前驱节点的waitStatus不等于-1和1，那也就是只可能是0，-2，-3，-2和-3这里用不到，所以就是0。0代表初始话，因为每个节点的状态都要靠后面的节点来修改，所以是0，通过cas将前驱节点状态设为SIGNAL。

* 线程挂起后，如果前驱节点执行完毕唤醒自己，还是在acquireQueued方法的循环中抢锁以及判断是否需要挂起。

## 解锁操作

最后，就是还需要介绍下唤醒的动作了。我们知道，正常情况下，如果线程没获取到锁，线程会被 `LockSupport.park(this);` 挂起停止，等待被唤醒。

```java
// 唤醒的代码还是比较简单的，你如果上面加锁的都看懂了，下面都不需要看就知道怎么回事了
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    // 往后看吧
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// 回到ReentrantLock看tryRelease方法
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 是否完全释放锁
    boolean free = false;
    // 其实就是重入的问题，如果c==0，也就是说没有嵌套锁了，可以释放了，否则还不能释放掉
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
// 唤醒后继节点
// 从上面调用处知道，参数node是head头结点
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    // 如果head节点当前waitStatus<0, 将其修改为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    // 下面的代码就是唤醒后继节点，但是有可能后继节点取消了等待（waitStatus==1）
    // 从队尾往前找，找到waitStatus<=0的所有节点中排在最前面的
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从后往前找，仔细看代码，不必担心中间有节点取消(waitStatus==1)的情况
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒线程
        LockSupport.unpark(s.thread);
}
```

这里从后往前找，源码的解释是 “通常只是下一个节点。 但如果后继节点被取消或明显为空，则从尾部向后遍历以找到实际的未取消后继者。”。

首先后继节点为空的情况：回到enq方法里，在执行“if(compareAndSetTail(t, node)){//...}”时，cas是原子操作，但是当cas成功（tail是当前node，只保证了当前node的前驱是对的，而当前node的前驱节点的后继节点不一定是当前node），执行if代码块里的内容时，此时不是同步操作。
这个时刻，node与前一个节点t之间，node的prev指针在cas操作之前已经建立，而t的next指针还未建立。此时若其他线程调用了unpark操作，从头开始找就无法遍历完整的队列，所以从后往前找就可以。

后继节点被取消的情况类似：

唤醒线程以后，被唤醒的线程将从以下代码中继续往前走：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 刚刚线程被挂起在这里了
    return Thread.interrupted();
}
// 又回到这个方法了：acquireQueued(final Node node, int arg)，这个时候，node的前驱是head了
```

