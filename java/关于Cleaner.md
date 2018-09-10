# 主要内容



## 为啥看这个东西

JVM中创建直接内存，netty在4.1中==取消==默认构建使用Cleaner

java.nio.DirectByteBuffer 构造器

在构造器中会默认创建Cleaner

```java
    DirectByteBuffer(int cap) {                   // package-private

        super(-1, 0, cap, cap);
        //是否是页对齐
        boolean pa = VM.isDirectMemoryPageAligned();
        //内存页大小
        int ps = Bits.pageSize();
        //需要申请的内存大小
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        //预备内存申请
        //记录内存申请次数及容量
        //计算内存是否可用，否则会抛OOM
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            //申请内存,返回内存地址
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        //设置内存中数据 
        //base 地址
        //sise 长度
        // 0   设置值
        unsafe.setMemory(base, size, (byte) 0);
        //设置数据起始位置
        //如果需要内存对齐并且内存地址非内存页大小整倍数
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        //创建自动回收器
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
```



## Cleaner的实现逻辑

```java
//Cleaner 继承 PhantomReference 幻引用
//PhantomReference 继承 Reference 抽象引用基类
public class Cleaner extends PhantomReference {
    private static final ReferenceQueue dummyQueue = new ReferenceQueue();
    private static Cleaner first = null;
    private Cleaner next = null;
    private Cleaner prev = null;
    private final Runnable thunk;
    
    //封装构造函数
    //创建一个Cleaner并添加到Cleaner队列中
    public static Cleaner create(Object var0, Runnable var1) {
        return var1 == null ? null : add(new Cleaner(var0, var1));
    }
    
    //私有构造函数
    //var1 需要清理的对象
    //var2 清理逻辑
    private Cleaner(Object var1, Runnable var2) {
        super(var1, dummyQueue);
        this.thunk = var2;
    }
    
    //添加当前Cleaner到队列中
    private static synchronized Cleaner add(Cleaner var0) {
        if (first != null) {
            var0.next = first;
            first.prev = var0;
        }

        first = var0;
        return var0;
    }
    
    //Reference回调函数
    public void clean() {
        if (remove(this)) {
            try {
                //回调指定函数
                this.thunk.run();
            } catch (final Throwable var2) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        if (System.err != null) {
                            (new Error("Cleaner terminated abnormally", var2)).printStackTrace();
                        }

                        System.exit(1);
                        return null;
                    }
                });
            }

        }
    }
}
```



```java
//PhantomReference 
public class PhantomReference<T> extends Reference<T> {

    //重写get()方法，始终返回null
    public T get() {
        return null;
    }

    //创建一个指定对象的幻引用，并注册到指定的队列中
    //如果指定队列为null，将一直不会enqueued,这个引用将是无用的
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}
```



## 关于Reference的介绍



```
A Reference instance is in one of four possible internal states:

引用实例可能处于四种内部状态之一:

     Active: Subject to special treatment by the garbage collector.  Some
     time after the collector detects that the reachability of the
     referent has changed to the appropriate state, it changes the
     instance's state to either Pending or Inactive, depending upon
     whether or not the instance was registered with a queue when it was
     created.  In the former case it also adds the instance to the
     pending-Reference list.  Newly-created instances are Active.
	
	 激活:由垃圾收集器进行特殊处理。在收集器检测到referent的可达性已更改为适当的状态后，它将实例的状态更改为挂起状态或非活动状态，这取决于创建实例时是否向队列注册了实例。在前一种情况下，它还将实例添加到挂起引用列表中。新创建的实例是活动的。

     Pending: An element of the pending-Reference list, waiting to be
     enqueued by the Reference-handler thread.  Unregistered instances
     are never in this state.
     
     挂起:挂起引用列表中的一个元素，等待被引用处理程序线程加入队列。未注册的实例不会处于这种状态。

     Enqueued: An element of the queue with which the instance was
     registered when it was created.  When an instance is removed from
     its ReferenceQueue, it is made Inactive.  Unregistered instances are
     never in this state.
     
     
	 排队:创建实例时使用的队列的一个元素。当一个实例从它的ReferenceQueue中删除时，它就会变成非活动的。未注册的实例不会处于这种状态。
     
     Inactive: Nothing more to do.  Once an instance becomes Inactive its
     state will never change again.
     
     非激活:无事可做。一旦实例变为非活动状态，它的状态将永远不会改变。

 The state is encoded in the queue and next fields as follows:

 引用中queue和next字段的状态码如下:

     Active: queue = ReferenceQueue with which instance is registered, or
     ReferenceQueue.NULL if it was not registered with a queue; next =
     null.
     
     激活:queue = 注册实例的一个ReferenceQueue,如没有注册到一个队列则为ReferenceQueue.NULL, next =null。

     Pending: queue = ReferenceQueue with which instance is registered;
     next = Following instance in queue, or this if at end of list.
     
     挂起: queue = 注册实例的一个ReferenceQueue, next = 在队列中跟随的实例， 如它是队列的最后一个则是它自身。

     Enqueued: queue = ReferenceQueue.ENQUEUED; next = Following instance
     in queue, or this if at end of list.
     
     已入队: queue = ReferenceQueue.ENQUEUED; next = 在队列中跟随的实例， 如它是队列的最后一个则是它自身。

     Inactive: queue = ReferenceQueue.NULL; next = this.
     
     非激活: queue = ReferenceQueue.NULL; next = this.

 With this scheme the collector need only examine the next field in order
 to determine whether a Reference instance requires special treatment: If
 the next field is null then the instance is active; if it is non-null,
 then the collector should treat the instance normally.
 
使用这种方案，收集器只需检查下一个字段，以确定引用实例是否需要特殊处理:如果下一个字段为空，则实例为活动的;如果它是非空的，那么收集器应该正常地处理实例。

 To ensure that concurrent collector can discover active Reference
 objects without interfering with application threads that may apply
 the enqueue() method to those objects, collectors should link
 discovered objects through the discovered field.
 
 为了确保并发收集器能够发现活动引用对象，而不会干扰应用程序线程(应用程序线程可能会将enqueue()方法应用到这些对象)，收集器应该通过discovered字段链接已发现的对象。
```



```java
public abstract class Reference<T> {
    //由GC处理的对象
    private T referent;
	//注册队列
    ReferenceQueue<? super T> queue;
	//引用的next,构建链表使用
    Reference next;
    //链接已发现的对象
    transient private Reference<T> discovered;
    
    //用于垃圾回收的对象，收集器在每个回收周期必须先获取此锁，
    //因此，任何具有此锁的代码都必须尽快完成，不分配新对象，并避免调用用户代码。
    static private class Lock { };
    private static Lock lock = new Lock();
    
    //等待入队的引用链表
    //由VM设置 ??
    private static Reference pending = null;
    
    //引用回收处理器
    private static class ReferenceHandler extends Thread {

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            for (;;) {

                Reference r;
                synchronized (lock) {
                    //检查pending
                    if (pending != null) {
                        r = pending;
                        Reference rn = r.next;
                        //如果this = next 表示this是最后一个引用
                        pending = (rn == r) ? null : rn;
                        r.next = r;
                    } else {
                        try {
                            lock.wait();
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }

                // Cleaner 特别处理
                if (r instanceof Cleaner) {
                    ((Cleaner)r).clean();
                    continue;
                }
				
                //回收对象引用入队
                ReferenceQueue q = r.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }

    //启动处理器线程
    //最高优先级
    //守护进程
    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
}
```



## 简单总结

1. 创建Cleaner，并把指定对象和回收处理逻辑传进去
2. 当指定对象不再被引用时，VM标记幻引用为pending状态，并添加到Reference.pending链表中
3. 然后唤醒Reference的引用处理线程
4. 逐个读取Reference.pending
5. 如果当前引用属于Cleaner对象或其子类，直接调动Cleaner.clean();
6. 在Cleaner.clean()调用回收处理逻辑

