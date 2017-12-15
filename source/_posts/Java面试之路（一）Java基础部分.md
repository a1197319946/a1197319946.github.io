引言
==

最近辞职，开始了一轮又一轮腥风血雨的面试，开个专栏，记录下面试中的各种疑难杂症问题，面试的公司有58企服、便利蜂，美团等，给大家分享下。专栏将分为几部分：java基础、数据库部分、分布式架构中间件部分、网络及算法部分。本篇来说下基础部分。博主刚毕业一年，加上大四一年外包经验，工作两年，本以为面试应该比较轻松，面了发现还是挺难的……

> 可谓雄关漫道真如铁，而今迈步从头越，从头越，苍山如海，残阳如血。


----------


Java基础
======

 

一、关于线程池
-------


 **1. java自带的线程池**
 
	

  a. newCachedThreadPool(创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。) 	
  b.  newFixedThreadPool(创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。) 	 
  c. newScheduledThreadPool(创建一个定长线程池，支持定时及周期性任务执行。) 
  d.newSingleThreadExecutor(创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。)

 **2. 自己如何实现线程池**
 
```
    ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
        .setNameFormat("demo-pool-%d").build();

    //Common Thread Pool
    ExecutorService pool = new ThreadPoolExecutor(5, 200,
         0L, TimeUnit.MILLISECONDS,
         new LinkedBlockingQueue<Runnable>(1024), namedThreadFactory, new ThreadPoolExecutor.AbortPolicy());

    pool.execute(()-> System.out.println(Thread.currentThread().getName()));
    pool.shutdown();//gracefully shutdown
      
```
* corePoolSize：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；
* maximumPoolSize：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；
* keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；
* unit：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性：

			TimeUnit.DAYS;               //天 TimeUnit.HOURS;             //小时
			TimeUnit.MINUTES;           //分钟 TimeUnit.SECONDS;           //秒
			TimeUnit.MILLISECONDS;      //毫秒 TimeUnit.MICROSECONDS;      //微妙
			TimeUnit.NANOSECONDS;       //纳秒

* workQueue：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：


		ArrayBlockingQueue; LinkedBlockingQueue; SynchronousQueue;

ArrayBlockingQueue和PriorityBlockingQueue使用较少，一般使用LinkedBlockingQueue和Synchronous。线程池的排队策略与BlockingQueue有关。
* threadFactory：线程工厂，主要用来创建线程；
* handler：表示当拒绝处理任务时的策略，有以下四种取值：

		 ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
		 ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
		 ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
		 ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务

二、关于Lock和Synchronized
---------------------

 **1. 两种锁有什么区别**

 1、ReentrantLock 拥有Synchronized相同的并发性和内存语义，但是增加了锁投票，定时锁等候和中断锁等候，。比如ReentrantLock获取锁定与三种方式：
 
	a) lock(),如果获取了锁立即返回，如果别的线程持有锁，当前线程则一直处于休眠状态，直到获取锁
	b) tryLock(),如果获取了锁立即返回true，如果别的线程正持有锁，立即返回false； 
	c)tryLock(long timeout,TimeUnit unit)， 如果获取了锁定立即返回true，如果别的线程正持有锁，会等待参数给定的时
	间，在等待的过程中，如果获取了锁定，就返回true，如果等待超时，返回false；
	d) lockInterruptibly:如果获取了锁定立即返回，如果没有获取锁定，当前线程处于休眠状态，直到或者锁定，或者当前线程
	被别的线程中断


**2 、ReentrantLock增加了等待条件，可以看ArrayBlockingQuene的实现**

```
public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();//设置不空条件
        notFull =  lock.newCondition();//设置不满条件
    }
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();//等待队列不满条件
            insert(e);
        } finally {
            lock.unlock();
        }
    }

```

3、synchronized是在JVM层面上实现的，不但可以通过一些监控工具监控synchronized的锁定，而且在代码执行时出现异常，JVM会自动释放锁定，但是使用Lock则不行，lock是通过代码实现的，要保证锁定一定会被释放，就必须将unLock()放到finally{}中
 
 4、在资源竞争不是很激烈的情况下，Synchronized的性能要优于ReetrantLock，但是在资源竞争很激烈的情况下，Synchronized的性能会下降几十倍，但是ReetrantLock的性能能维持常态；



 **2. Synchronized修饰普通方法和静态方法有什么区别**
	

	 修饰普通方法为对象锁，修饰静态方法为类锁

三、HashMap底层原理
-------------

		
 **1. 底层结构**
 


	 数组+链表；jdk8优化后链表长度超过8会转化为红黑树




 **2. 散列算法**

```
//jdk8已经进行简化，改为1次16位右移异或混合，而不是4次
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }

```

```
//hashcode用之前还要先做对数组的长度取模运算，得到的余数才能用来访问数组下标。获取下标
 static int indexFor(int h, int length) {
        return h & (length-1);
    }
```
**3. 扩容原理**
 

```
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
//两倍扩容
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}
```

 **4. jdk8对HashMap的优化**
 

```
// 链表长度大于8时，改为红黑树结构
// 实现如下：
 private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
//红黑树的具体实现需要了解下
```


 
 **5.  HashMap put方法原理**
 

```
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
//允许key为null
    if (key == null)
        return putForNullKey(value);
//key不是null时计算哈希值得到数组下标
    int hash = hash(key);
    int i = indexFor(hash, table.length);
//遍历该位置链表，如果已有此value则直接进行替换，并返回原来的值
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
//如果不存在此value,则插入节点
    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```

 **6. HashMap如何调整性能**
 


	适当修改分组组数及加载因子（默认16*0.75），保证分组组数* 加载因子>元素总量，避免由于出发扩容重新散列印象效率。另，
	分组组数越大，数据将被散列到更多的小组，查询效率高；加载因子越小，扩容发生的越早，也就是在尽可能的保证效率。

      
 **7. HashMap是否线程安全**


 
	 HashMap不支持多个线程并发操作，会出现并发错误，在并发的场景下可以用HashTable，可以使用Collections工具类中的
	 synchronizedMap方法或者有更好的并发性能的ConcurrentHashMap

    

四、TreeMap如何保证有序
---------------



```
//在初始化时，构造器会传入一个比较器参数
 public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }
//put时根据比较原则进行插入元素
public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) {
            compare(key, key); // type (and possibly null) check

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
            if (key == null)
                throw new NullPointerException();
            Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }
```

五、NIO与IO的区别及何种场景使用
------------------
**1 使用场景**


	**NIO**
	
	优势在于一个线程管理多个通道；但是数据的处理将会变得复杂；
	如果需要管理同时打开的成千上万个连接，这些连接每次只是发送少量的数据，采用这种； 
	
	**传统的IO** 
	
	适用于一个线程管理一个通道的情况；因为其中的流数据的读取是阻塞的；
	如果需要管理同时打开不太多的连接，这些连接会发送大量的数据；




**2 NIO vs IO区别**


1 IO是面向流的，NIO是面向缓冲区的

	a Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方；
	b NIO则能前后移动流中的数据，因为是面向缓冲区的

2 IO流是阻塞的，NIO流是不阻塞的

     a Java IO的各种流是阻塞的。这意味着，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或
     数据完全写入。该线程在此期间不能再干任何事情了
     b Java NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用
     时，就什么都不会获取。NIO可让您只使用一个（或几个）单线程管理多个通道（网络连接或文件），但付出的代价是解析数据可
     能会比从一个阻塞流中读取数据更复杂。 
     c 非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。
3 选择器

	     Java NIO的选择器允许一个单独的线程来监视多个输入通道，你可以注册多个通道使用一个选择器，然后使用一个单独的线
	     程来“选择”通道：这些通道里已经有可以处理的输入，或者选择已准备写入的通道。这种选择机制，使得一个单独的线程很容
	     易来管理多个通道。 



六、CAS和CAP
---------

 **1. CAS**
 

 CAS:Compare and Swap, 翻译成比较并交换。 
java.util.concurrent包中借助CAS实现了区别于synchronouse同步锁的一种乐观锁。CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

```
//可以看下AtomicInteger的源码 i++操作
private volatile int value;
public final int incrementAndGet() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return next;
    }
}
public final boolean compareAndSet(int expect, int update) {   
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
//compareAndSwapInt方法即拿预期值expect和this当前内存值比较，相同则返回true,并赋新值update.
```

 **2. CAP**
 

>  引用一下  ~
> [CAP原理](http://www.blogjava.net/hello-yun/archive/2012/04/27/376744.html)

七、JVM内存模型，jdk8有何种改进
-------------------
  jvm内存由堆、栈、本地方法栈、方法区等组成

 1. 堆。存放通过new创建的对象，分新生代、老年代
 2. 栈。每个线程执行每个方法的时候都会在栈中申请一个栈帧，每个栈帧包括局部变量区和操作数栈，用于存放此次方法调用过程中的临时变量、参数和中间结果。  -xss:设置每个线程的堆栈大小. JDK1.5+ 每个线程堆栈大小为 1M，一般来说如果栈不是很深的话， 1M 是绝对够用了的。
 3. 本地方法栈。用于支持native方法的执行，存储了每个native方法调用的状态
 4. 方法区 。即持久代Permanet Space.

下图为堆内存模型（图有点问题，持久带为方法区，不应该画在堆内存中）。
![这里写图片描述](http://img.blog.csdn.net/20171117144601868?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZnJpZ2h0aW5nZm9yYW1iaXRpb24=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

普及一下gc算法：

		年轻代：
		复制算法。 此算法把内存空间划为两个相等的区域，每次只使用其中一个区域。垃圾回收时，遍历当前使用区域，把正在使用中的对象复
		制到另外一个区域中。算法每次只处理正在使用中的对象，因此复制成本比较小，同时复制过去以后还能进行相应的内存整理，不会出现“碎片”问
		题。当然，此算法的缺点也是很明显的，就是需要两倍内存空间。
		
		老年代：
		并行（标记清除）。关于标记清除算法：此算法执行分两阶段。第一阶段从引用根节点开始标记所有被引用的对象，第二阶段遍历整个堆，
		把未标记的对象清除。此算法需要暂停整个应用，同时，会产生内存碎片。
		
		并行回收（标记整理）.关于标记整理算法： 此算法结合了“标记-清除”和“复制”两个算法的优点。也是分两阶段，第一阶段从根节点开始标记所
		有被引用对象，第二阶段遍历整个堆，把清除未标记对象并且把存活对象“压缩”到堆的其中一块，按顺序排放。此算法避免了“标记-清除”的碎片问
		题，同时也避免了“复制”算法的空间问题。

顺便提一下对象何时进行销毁：

		这也是面试官喜欢问的一个问题，一般我们会回答对象在gc的时候进行销毁，那面试官又会问，gc的时候jvm怎么知道该销毁哪些对象呢？这个问
		题有点恶心，多次碰到简单说下：一个对象创建后被放置在JVM的堆内存中，当永远不再引用这个对象时，它将被JVM在堆内存中回收。即当对象在
		JVM运行空间中无法通过根集合到达(找到)时,这个对象被称为垃圾对象。根集合是由类中的静态引用域与本地引用域组成的。JVM通过根集合索引
		对象。 
		这时面试官又会问，那什么情况下对象不会被回收呢（问的是内存泄露什么时候会发生），请把滚动条拉到最下面基础中基础，第一题有总结。



主要来说下jdk8的改进：
```

以元空间取代永久代。元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间
并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制，但可以通过以下参数来指定元空间的大小：

　　-XX:MetaspaceSize，初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值。
　　-XX:MaxMetaspaceSize，最大空间，默认是没有限制的。

　　除了上面两个指定大小的选项以外，还有两个与 GC 相关的属性：
　　-XX:MinMetaspaceFreeRatio，在GC之后，最小的Metaspace剩余空间容量的百分比，减少为分配空间所导致的垃圾收集
　　-XX:MaxMetaspaceFreeRatio，在GC之后，最大的Metaspace剩余空间容量的百分比，减少为释放空间所导致的垃圾收集。

jvm参数不再需要指定 PermSize 和 MaxPermSize。而是指定 MetaSpaceSize 和 MaxMetaSpaceSize的大小。
```


八、JVM调优
-------

 **1. jvm参数修改包括堆内存设置及垃圾回收器设置**

```
JVM内存的系统级的调优主要的目的是减少GC的频率和Full GC的次数，过多的GC和Full GC是会占用很多的系统资源（主要是CPU），
影响系统的吞吐量。特别要关注Full GC，因为它会对整个堆进行整理。
出现Full GC的几种原因：

 1. 老年代空间不足
 2. 新生代设置过小或过大
   过小会导致大对象直接进入老年代，诱发full gc;
   过大会导致老年代过小（堆总量一定），诱发full gc，另外会导致新生代gc耗时严重,
   新生代最好占堆内存 1/3 或 3/8
 3. Survivor或Eden区设置过小或过大
   过小导致对象从Eden区直接进去到老年代
   过大导致Eden过小，增加了GC频率
   可以使用-XX:SurvivorRatio=n 手动设置，默认n为8，即Survivor1：Survivor0：Eden=1:1:8
 


另外，可选择适合该应用的垃圾回收器：

 XX:+UseSerialGC:设置串行收集器
-XX:+UseParallelGC:设置并行收集器
-XX:+UseParalledlOldGC:设置并行年老代收集器
-XX:+UseConcMarkSweepGC:设置并发收集器


垃圾回收统计信息：
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
```

 
 **2. 代码调优，防止大对象，大数组频繁创建**

九、CPU和内存飙升的一般情况和解决思路
--------------------

 **1. 内存飙升可能与线程阻塞、死循环有关，解决方法如下：**
 
 

```
1 使用jps查找出java进程的pid，如3707
2 使用top -p 3707观察进程情况，然后Shift+h,显示该进程的所有线程。
3 找出CPU消耗较多的线程id，如3720，将3720转换为16进制0x7d0，注意是小写哦
4 使用jstack 3707 | grep -A 10 0x7d0 来查询出具体的线程状态。
```

 
 **2. 内存占用高一般是创建对象太多，或有大数组之类，排查方式：**
 

```
 通过jstat -gc -h10 pid 1000 查看gc情况
 通过jmap -heap pid 查看堆内存情况
 通过jmap -histo pid 查看堆中对象创建情况    
 关于内存溢出除了对象创建频繁另外的一种可能性：由于方法区主要存储类的相关信息，所以对于动态生成类的情况比较容易出现永久
代的内存溢出。最典型的场景就是，在JSP页面比较多的情况，容易出现永久代内存溢出。
```

十、Quene的原理
----------


```
主要说一下java阻塞队列BlockingQuene。
它的入队出队主要有几种方法：

 1. add remove 队列满的时候会抛异常
 2. offer poll 队列满了会返回false
 3. put take 队列满了会阻塞（等待）
当然我们用阻塞的话只能用最后一种方式，
常用的实现有两种ArrayBlockingQueue、LinkedBlockingQueue
ArrayBlockingQueue实现如下：

  public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return extract();
        } finally {
            lock.unlock();
        }
    }
    可以看到使用一个ReentrantLock实现的，而LinkedBlockingQueue的实现略有不同，用了两个ReentrantLock。来看下：
     public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
//入队和出队是不同的锁
```

十一、介绍了解的设计模式
------------

 **1. 单例**

```
    //醉汉
    public class Singleton{
        private Singleton(){};
        private static Singleton instance = new Singleton();
        public static Singleton getInstance (){
            return instance;
        } 
    }
    //懒汉
    public class Singleton{
        private static Singleton instance = null;
        public static synchronized Singleton getInstance(){
                if(instance == null {
                        instance = new Singleton();
                }
               return instance;
        }
    }
    //对懒汉模式加锁的优化（volatile+双重锁）
     public class Singleton{
        private volatile static Singleton instance = null;
        public static Singleton getInstance(){
                if(instance == null {
                    synchronized（Singleton.class）{
			            if(instance == null){
                            instance = new Singleton();
                        }
                    }
                }
               return instance;
        }
    }
    
```

 **2. 工厂**
 **3. 代理**
 **4. 包装**
 ...........

十二、volatile为什么能支持同步，却不支持原子性？
----------------------------
这两句话看起来好像是矛盾的，我们来一一分析下。

　　首先，我们知道volatile是一个线程安全的修饰符，当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。
　　在访问volatile变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此volatile变量是一种比sychronized关键字更轻量级的同步机制。
　　当对非 volatile 变量进行读写的时候，每个线程先从内存拷贝变量到CPU缓存中。如果计算机有多个CPU，每个线程可能在不同的CPU上被处理，这意味着每个线程可以拷贝到不同的 CPU cache 中。

　　
当一个变量定义为 volatile 之后，将具备两种特性：

　　1.保证此变量对所有的线程的可见性，当一个线程修改了这个变量的值，volatile 保证了新值能立即同步到主内存，以及每次get时立即从主内存刷新。
　　2.禁止指令重排序优化。有volatile修饰的变量，赋值后多执行了一个“load addl $0x0, (%esp)”操作，这个操作相当于一个内存屏障（指令重排序时不能把后面的指令重排序到内存屏障之前的位置），所以就禁止了cpu的重排序操作。
　　
volatile 性能：

　　volatile 的读性能消耗与普通变量几乎相同，但是写操作稍慢，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。


**当变量用volatile修饰是，从上面可以知道set/get 都是线程安全的，但这是否说明volatile是原子的？**
答案是否定的，比如i++ , 这个操作实际是 i = i + 1, 它不是一个原子的行为，在操作的过程中，是不安全的，如要实现安全，可采用CAS方案进行同步，像AtomicInteger、AtomicLong等就是基于volatile的可见性 + CAS控制 i++操作的原子性的。
　　

----------
以上为我接触的感觉基础中比较深入的问题，下面的问题我感觉比较简单但是必须熟练掌握，答案不一一列举。有问题的自行百度，如若无果请留言。


----------


Java基础中的基础
------

一、内存溢出和内存泄露发生原因
1 内存溢出（比较简单暂不列举）
2 内存泄露是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成的内存空间的浪费称为内存泄露。
内存泄露的原因：

		1、静态集合类引起内存泄露： 
		像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放，因为他们也将一直被Vector等引用着。 
		2 各种连接、流未关闭

二、集合框架对比

 1. ArrayList与linkedList、Vector的区别
 2. HashMap与HashTable的区别
 3. TreepMap,SortedMap比较器的用法
 4. ArrayList扩容机制
	 jdk1.7 前 a*1.5 + 1
	jdk1.7 后 a + a>>1


三、接口抽象类的区别

 1. 接口中定义的变量默认即常量；抽象类中变量为普通属性，每个对象都有一份。
 2. 接口中的方法都是抽象方法不能写实现，抽象类中中可以有抽象方法也可以有非抽象方法，也就是说可以有实现
 3. jdk8开始接口中的方法可以写具体实现，static修饰的静态方法和default修饰的默认方法

四、线程的生命周期
新建、就绪、运行、阻塞、死亡

> 先写这么多，后面待续...........

 