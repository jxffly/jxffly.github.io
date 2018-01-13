---
layout: post
title:  "netty的时间轮算法解读"
date: 2018-01-12 00:00:00
tags: netty java
---

### 1.为什么使用netty

​	大量的调度任务如果每一个都使用自己的调度器来管理任务的生命周期的话，浪费cpu的资源并且很低效。

​	时间轮是一种高效来利用线程资源来进行批量化调度的一种调度模型。把大批量的调度任务全部都绑定到同一个的调度器上面，使用这一个调度器来进行所有任务的管理（manager），触发（trigger）以及运行（runnable）。能够高效的管理各种延时任务，周期任务，通知任务等等。
![netty](/image/netty-hasedwheeltimer1.png)


​	缺点，时间轮调度器的时间精度可能不是很高，对于精度要求特别高的调度任务可能不太适合。因为时间轮算法的精度取决于，时间段“指针”单元的最小粒度大小，比如时间轮的格子是一秒跳一次，那么调度精度小于一秒的任务就无法被时间轮所调度。而且时间轮算法没有做宕机备份，因此无法再宕机之后恢复任务重新调度。

### 2.实例

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
//生成了一个时间轮，时间步长为300ms
HashedWheelTimer hashedWheelTimer = new HashedWheelTimer(300, TimeUnit.MILLISECONDS);
System.out.println("start:" + LocalDateTime.now().format(formatter));
//提交一个任务在三秒之后允许
hashedWheelTimer.newTimeout(timeout -> {
  System.out.println("task :" + LocalDateTime.now().format(formatter));
  }, 3, TimeUnit.SECONDS);
  Thread.sleep(100000);
```

输出为：

>start:2018-01-09 22:10:55  
>task :2018-01-09 22:11:05

以上生成了一个轮转间隔为300ms的时间轮调度器，表示时间轮每300ms向前跳一个并且激活对应格子里面的任务并且激活，但是这个有个缺陷，就是任务是串行的，也就是所有的任务是依次执行，如果调度的任务耗时比较长的话，容易出现调度超时的情况，因此很可能产生任务堆集的情况出现。

```java
 DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
 HashedWheelTimer hashedWheelTimer = new HashedWheelTimer(100, TimeUnit.MILLISECONDS);
 System.out.println("start:" + LocalDateTime.now().format(formatter));
 hashedWheelTimer.newTimeout(timeout -> {
 Thread.sleep(3000);
     System.out.println("task1:" + LocalDateTime.now().format(formatter));
        }, 3, TimeUnit.SECONDS);
 hashedWheelTimer.newTimeout(timeout -> System.out.println("task2:" + LocalDateTime.now().format(
                formatter)), 4, TimeUnit.SECONDS);
 Thread.sleep(10000);
```

> start:2018-01-09 22:16:44     
> task1:2018-01-09 22:16:50     
> task2:2018-01-09 22:16:50

可以看到task2的任务已经被task1阻塞了，task2本来的执行时间应该是44+4在48秒的时候执行，但是task1任务是阻塞的，因此这就导致了串行化的任务调度延时，因此，应该避免耗时比较长的任务。

### 3.源码系列

#### 3.1构造方法

```java
  public HashedWheelTimer(
          ThreadFactory threadFactory, // 用来创建worker线程
          long tickDuration, // tick的时长，也就是指针多久转一格
          TimeUnit unit, // tickDuration的时间单位
          int ticksPerWheel, // 一圈有几格
          boolean leakDetection // 是否开启内存泄露检测
          ) {
      // 一些参数校验
      if (threadFactory == null) {
          throw new NullPointerException("threadFactory");
      }
      if (unit == null) {
          throw new NullPointerException("unit");
      }
      if (tickDuration <= 0) {
          throw new IllegalArgumentException("tickDuration must be greater than 0: " + tickDuration);
      }
      if (ticksPerWheel <= 0) {
          throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
      }
      // 创建时间轮基本的数据结构，一个数组。长度为不小于ticksPerWheel的最小2的n次方
      wheel = createWheel(ticksPerWheel);
      // 这是一个标示符，用来快速计算任务应该呆的格子。
      // 我们知道，给定一个deadline的定时任务，其应该呆的格子=deadline%wheel.length.但是%操作是个相对耗时的操作，所以使用一种变通的位运算代替：
      // 因为一圈的长度为2的n次方，mask = 2^n-1后低位将全部是1，然后deadline&mast == deadline%wheel.length
      // java中的HashMap在进行hash之后，进行index的hash寻址寻址的算法也是和这个一样的
      mask = wheel.length - 1;
      // 转换成纳秒处理
      this.tickDuration = unit.toNanos(tickDuration);
      // 校验是否存在溢出。即指针转动的时间间隔不能太长而导致tickDuration*wheel.length>Long.MAX_VALUE
      if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
          throw new IllegalArgumentException(String.format(
                  "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                  tickDuration, Long.MAX_VALUE / wheel.length));
      }
      // 创建worker线程
      workerThread = threadFactory.newThread(worker);
// 这里默认是启动内存泄露检测：当HashedWheelTimer实例超过当前cpu可用核数*4的时候，将发出警告
      leak = leakDetection || !workerThread.isDaemon() ? leakDetector.open(this) : null;
    //任务的超时等待时间，如果设置了超时了之后会抛出异常  
    this.maxPendingTimeouts = maxPendingTimeouts;
    //当HashedWheelTimer实例超过当前cpu可用核数最大的实例数为64个，也就是一个jvm最多只能有64个时间轮对象实例，因为时间轮是一个非常耗费资源的结构所以实例数目不能太高
	  if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
            WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
            reportTooManyInstances();
        //超过会报警实例过多
        //  String resourceType = simpleClassName(HashedWheelTimer.class);
        //logger.error("You are creating too many " + resourceType + " instances. " +
        //        resourceType + " is a shared resource that must be reused across the 	//JVM," +
  //              "so that only a few instances are created.");
        }
  }
```

生成时间轮的格子的数目，会发现格子的数目始终是2^n次方，但是感觉这种方式没有hashMap的tableSizeFor优雅

```
    private static int normalizeTicksPerWheel(int ticksPerWheel) {
        int normalizedTicksPerWheel = 1;
        while (normalizedTicksPerWheel < ticksPerWheel) {
            normalizedTicksPerWheel <<= 1;
        }
        return normalizedTicksPerWheel;
    }

```

```java
//来自于hashMap里面的算法，把一个数，变成最接近的2^n的数字，
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

#### 3.2HashedWheelTimer源码之启动、停止与添加任务

`start()`启动时间轮的方法：

```java
// 启动时间轮。这个方法其实不需要显示的主动调用，因为在添加定时任务（newTimeout()方法）的时候会自动调用此方法。
// 这个是合理的设计，因为如果时间轮里根本没有定时任务，启动时间轮也是空耗资源，因此使用的类似于hashMap的懒加载的形式，只有当有任务提交之后才会启动这个调度器
public void start() {
    // 判断当前时间轮的状态，如果是初始化，则启动worker线程，启动整个时间轮；如果已经启动则略过；如果是已经停止，则报错
    // 这里是一个Lock Free的设计。因为可能有多个线程调用启动方法，这里使用AtomicIntegerFieldUpdater原子的更新时间轮的状态，因此使用这个方法来获取实例的允许状态，防止重入
    switch (WORKER_STATE_UPDATER.get(this)) {
        case WORKER_STATE_INIT:
        //使用cas来获取启动调度的权力，只有竞争到的线程允许来进行实例启动
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                workerThread.start();
            }
            break;
        case WORKER_STATE_STARTED:
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }

    // 等待worker线程初始化时间轮的启动时间
    while (startTime == 0) {
        try {
          //这里使用countDownLauch来确保调度的线程已经被启动
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}
```

AtomicIntegerFieldUpdater是JUC里面的类，原理是利用安全的反射进行原子操作，来获取实例的本身的属性。有比AtomicInteger更好的性能和更低得内存占用。跟踪这个类的github 提交记录，可以看到更详细的[原因](https://github.com/netty/netty/commit/1f68479e3cd94deb3172edd3c01aa74f35032b9b)

`stop()`停止时间轮的方法：

```java
public Set<Timeout> stop() {
    // worker线程不能停止时间轮，也就是加入的定时任务，不能调用这个方法。
    // 不然会有恶意的定时任务调用这个方法而造成大量定时任务失效
    if (Thread.currentThread() == workerThread) {
        throw new IllegalStateException(
                HashedWheelTimer.class.getSimpleName() +
                        ".stop() cannot be called from " +
                        TimerTask.class.getSimpleName());
    }
    // 尝试CAS替换当前状态为“停止：2”。如果失败，则当前时间轮的状态只能是“初始化：0”或者“停止：2”。直接将当前状态设置为“停止：2“
    if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
        // workerState can be 0 or 2 at this moment - let it always be 2.
        WORKER_STATE_UPDATER.set(this, WORKER_STATE_SHUTDOWN);

        if (leak != null) {
            leak.close();
        }

        return Collections.emptySet();
    }

    // 终端worker线程，尝试把正在进行任务的线程中断掉,如果某些任务正在执行则会，抛出interrupt异常，并且任务会尝试立即中断
    boolean interrupted = false;
    while (workerThread.isAlive()) {
        workerThread.interrupt();
        try {
          //当前前程会等待stop的结果
            workerThread.join(100);
        } catch (InterruptedException ignored) {
            interrupted = true;
        }
    }

    // 从中断中恢复
    if (interrupted) {
          // 如果中断掉了所有工作的线程，那么当前关闭时间轮调度的线程会在随后关闭
        Thread.currentThread().interrupt();
    }

    if (leak != null) {
        leak.close();
    }
    // 返回未处理的任务
    return worker.unprocessedTimeouts();
}
```

`newTimeout()`添加定时任务：

```java
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    // 参数校验
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (unit == null) {
        throw new NullPointerException("unit");
    }
    // 如果时间轮没有启动，则启动
    start();

    // Add the timeout to the timeout queue which will be processed on the next tick.
    // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
    // 计算任务的deadline
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
    // 这里定时任务不是直接加到对应的格子中，而是先加入到一个队列里，然后等到下一个tick的时候，会从队列里取出最多100000个任务加入到指定的格子中
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    timeouts.add(timeout);
    return timeout;
}
```

这里使用的Queue不是普通java自带的Queue的实现，而是使用[JCTool](https://github.com/JCTools/JCTools)–一个高性能的的并发Queue实现包。

#### 3.3 HashedWheelTimer源码之HashedWheelTimeout

`HashedWheelTimeout`是一个定时任务的内部包装类，双向链表结构。会保存定时任务到期执行的任务、deadline、round等信息。

```java
private static final class HashedWheelTimeout implements Timeout {

    // 定义定时任务的3个状态：初始化、取消、过期
    private static final int ST_INIT = 0;
    private static final int ST_CANCELLED = 1;
    private static final int ST_EXPIRED = 2;
    // 用来CAS方式更新定时任务状态
    private static final AtomicIntegerFieldUpdater<HashedWheelTimeout> STATE_UPDATER;

    static {
        AtomicIntegerFieldUpdater<HashedWheelTimeout> updater =
                PlatformDependent.newAtomicIntegerFieldUpdater(HashedWheelTimeout.class, "state");
        if (updater == null) {
            updater = AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimeout.class, "state");
        }
        STATE_UPDATER = updater;
    }

    // 时间轮引用
    private final HashedWheelTimer timer;
    // 具体到期需要执行的任务
    private final TimerTask task;
    private final long deadline;

    @SuppressWarnings({"unused", "FieldMayBeFinal", "RedundantFieldInitialization" })
    private volatile int state = ST_INIT;

    // 离任务执行的轮数，当将次任务加入到格子中是计算该值，每过一轮，该值减一。
    long remainingRounds;

    // 双向链表结构，由于只有worker线程会访问，这里不需要synchronization / volatile
    HashedWheelTimeout next;
    HashedWheelTimeout prev;

    // 定时任务所在的格子
    HashedWheelBucket bucket;

    HashedWheelTimeout(HashedWheelTimer timer, TimerTask task, long deadline) {
        this.timer = timer;
        this.task = task;
        this.deadline = deadline;
    }

    @Override
    public Timer timer() {
        return timer;
    }

    @Override
    public TimerTask task() {
        return task;
    }

    @Override
    public boolean cancel() {
        // 这里只是修改状态为ST_CANCELLED，会在下次tick时，在格子中移除      
        if (!compareAndSetState(ST_INIT, ST_CANCELLED)) {
            return false;
        }       
        // 加入到时间轮的待取消队列，并在每次tick的时候，从相应格子中移除。
        timer.cancelledTimeouts.add(this);
        return true;
    }

    // 从格子中移除自身
    void remove() {
        HashedWheelBucket bucket = this.bucket;
        if (bucket != null) {
            bucket.remove(this);
        }
    }

    public boolean compareAndSetState(int expected, int state) {
        return STATE_UPDATER.compareAndSet(this, expected, state);
    }

    public int state() {
        return state;
    }

    @Override
    public boolean isCancelled() {
        return state() == ST_CANCELLED;
    }

    @Override
    public boolean isExpired() {
        return state() == ST_EXPIRED;
    }

    // 过期并执行任务
    public void expire() {
        if (!compareAndSetState(ST_INIT, ST_EXPIRED)) {
            return;
        }

        try {
            task.run(this);
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("An exception was thrown by " + TimerTask.class.getSimpleName() + '.', t);
            }
        }
    }

    // 略过toString()
}
```

#### 3.4 HashedWheelTimer源码之HashedWheelBucket

`HashedWheelBucket`用来存放HashedWheelTimeout，结构类似于LinkedList。提供了`expireTimeouts(long deadline)`方法来过期并执行格子中的定时任务

```java
private static final class HashedWheelBucket {
    // 指向格子中任务的首尾
    private HashedWheelTimeout head;
    private HashedWheelTimeout tail;

    // 基础的链表添加操作
    public void addTimeout(HashedWheelTimeout timeout) {
        assert timeout.bucket == null;
        timeout.bucket = this;
        if (head == null) {
            head = tail = timeout;
        } else {
            tail.next = timeout;
            timeout.prev = tail;
            tail = timeout;
        }
    }

    // 过期并执行格子中的到期任务，tick到该格子的时候，worker线程会调用这个方法，根据deadline和remainingRounds判断任务是否过期
    public void expireTimeouts(long deadline) {
        HashedWheelTimeout timeout = head;

        // 遍历格子中的所有定时任务
        while (timeout != null) {
            boolean remove = false;
            if (timeout.remainingRounds <= 0) { // 定时任务到期
                if (timeout.deadline <= deadline) {
                    timeout.expire();
                } else {
                    // 如果round数已经为0，deadline却>当前格子的deadline，说放错格子了，这种情况应该不会出现
                    throw new IllegalStateException(String.format(
                            "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                }
                remove = true;
            } else if (timeout.isCancelled()) {
                remove = true;
            } else { //没有到期，轮数-1
                timeout.remainingRounds --;
            }
            // 先保存next，因为移除后next将被设置为null
            HashedWheelTimeout next = timeout.next;
            if (remove) {
                remove(timeout);
            }
            timeout = next;
        }
    }

    // 基础的链表移除node操作
    public void remove(HashedWheelTimeout timeout) {
        HashedWheelTimeout next = timeout.next;
        // remove timeout that was either processed or cancelled by updating the linked-list
        if (timeout.prev != null) {
            timeout.prev.next = next;
        }
        if (timeout.next != null) {
            timeout.next.prev = timeout.prev;
        }

        if (timeout == head) {
            // if timeout is also the tail we need to adjust the entry too
            if (timeout == tail) {
                tail = null;
                head = null;
            } else {
                head = next;
            }
        } else if (timeout == tail) {
            // if the timeout is the tail modify the tail to be the prev node.
            tail = timeout.prev;
        }
        // null out prev, next and bucket to allow for GC.
        timeout.prev = null;
        timeout.next = null;
        timeout.bucket = null;
    }

    /**
     * Clear this bucket and return all not expired / cancelled {@link Timeout}s.
     */
    public void clearTimeouts(Set<Timeout> set) {
        for (;;) {
            HashedWheelTimeout timeout = pollTimeout();
            if (timeout == null) {
                return;
            }
            if (timeout.isExpired() || timeout.isCancelled()) {
                continue;
            }
            set.add(timeout);
        }
    }

    // 链表的poll操作
    private HashedWheelTimeout pollTimeout() {
        HashedWheelTimeout head = this.head;
        if (head == null) {
            return null;
        }
        HashedWheelTimeout next = head.next;
        if (next == null) {
            tail = this.head =  null;
        } else {
            this.head = next;
            next.prev = null;
        }

        // null out prev and next to allow for GC.
        head.next = null;
        head.prev = null;
        head.bucket = null;
        return head;
    }
}
```

#### 3.5HashedWheelTimer源码之Worker

`Worker`是时间轮的核心线程类。tick的转动，过期任务的处理都是在这个线程中处理的。

```java
private final class Worker implements Runnable {
    private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

    private long tick;

    @Override
    public void run() {
        // 初始化startTime.只有所有任务的的deadline都是想对于这个时间点
        startTime = System.nanoTime();
        // 由于System.nanoTime()可能返回0，甚至负数。并且0是一个标示符，用来判断startTime是否被初始化，所以当startTime=0的时候，重新赋值为1
        if (startTime == 0) {
            startTime = 1;
        }

        // 唤醒阻塞在start()的线程
        startTimeInitialized.countDown();

        // 只要时间轮的状态为WORKER_STATE_STARTED，就循环的“转动”tick，循环判断响应格子中的到期任务
        do {
            // waitForNextTick方法主要是计算下次tick的时间, 然后sleep到下次tick
            // 返回值就是System.nanoTime() - startTime, 也就是Timer启动后到这次tick, 所过去的时间
            final long deadline = waitForNextTick();
            if (deadline > 0) { // 可能溢出或者被中断的时候会返回负数, 所以小于等于0不管
                // 获取tick对应的格子索引
                int idx = (int) (tick & mask);
                // 移除被取消的任务
                processCancelledTasks();
                HashedWheelBucket bucket =
                        wheel[idx];
                // 从任务队列中取出任务加入到对应的格子中
                transferTimeoutsToBuckets();
                // 过期执行格子中的任务
                bucket.expireTimeouts(deadline);
                tick++;
            }
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

        // 这里应该是时间轮停止了，清除所有格子中的任务，并加入到未处理任务列表，以供stop()方法返回
        for (HashedWheelBucket bucket: wheel) {
            bucket.clearTimeouts(unprocessedTimeouts);
        }
        // 将还没有加入到格子中的待处理定时任务队列中的任务取出，如果是未取消的任务，则加入到未处理任务队列中，以供stop()方法返回
        for (;;) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                break;
            }
            if (!timeout.isCancelled()) {
                unprocessedTimeouts.add(timeout);
            }
        }
        // 处理取消的任务
        processCancelledTasks();
    }

    // 将newTimeout()方法中加入到待处理定时任务队列中的任务加入到指定的格子中
    private void transferTimeoutsToBuckets() {
        // 每次tick只处理10w个任务，以免阻塞worker线程
        for (int i = 0; i < 100000; i++) {
            HashedWheelTimeout timeout = timeouts.poll();
            // 如果没有任务了，直接跳出循环
            if (timeout == null) {
                break;
            }
            // 还没有放入到格子中就取消了，直接略过
            if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                continue;
            }

            // 计算任务需要经过多少个tick
            long calculated = timeout.deadline / tickDuration;
            // 计算任务的轮数
            timeout.remainingRounds = (calculated - tick) / wheel.length;

            //如果任务在timeouts队列里面放久了, 以至于已经过了执行时间, 这个时候就使用当前tick, 也就是放到当前bucket, 此方法调用完后就会被执行.
            final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
            int stopIndex = (int) (ticks & mask);

            // 将任务加入到响应的格子中
            HashedWheelBucket bucket = wheel[stopIndex];
            bucket.addTimeout(timeout);
        }
    }

    // 将取消的任务取出，并从格子中移除
    private void processCancelledTasks() {
        for (;;) {
            HashedWheelTimeout timeout = cancelledTimeouts.poll();
            if (timeout == null) {
                // all processed
                break;
            }
            try {
                timeout.remove();
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown while process a cancellation task", t);
                }
            }
        }
    }

    /**
     * calculate goal nanoTime from startTime and current tick number,
     * then wait until that goal has been reached.
     * @return Long.MIN_VALUE if received a shutdown request,
     * current time otherwise (with Long.MIN_VALUE changed by +1)
     */
    //sleep, 直到下次tick到来, 然后返回该次tick和启动时间之间的时长
    private long waitForNextTick() {
        //下次tick的时间点, 用于计算需要sleep的时间
        long deadline = tickDuration * (tick + 1);

        for (;;) {
            // 计算需要sleep的时间, 之所以加999999后再除10000000,前面是1所以这里需要减去1，才能计算准确，还有通过这里可以看到，其实线程是以睡眠一定的时候再来执行下一个ticket的任务的，这样如果ticket的间隔设置的太小的话，系统会频繁的睡眠然后启动，其实感觉影响部分的性能，所以为了更好的利用系统步长可以稍微设置大点
            final long currentTime = System.nanoTime() - startTime;
            long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;
//这里无需等待，立即返回下一个指针时间
            if (sleepTimeMs <= 0) {
	// 以下为个人理解：（如有错误，欢迎大家指正）
                // 这里的意思应该是从时间轮启动到现在经过太长的时间(跨度大于292年...)，以至于让long装不下，都溢出了...对于netty的严谨，我服！
                if (currentTime == Long.MIN_VALUE) {
                    return -Long.MAX_VALUE;
                } else {
                    return currentTime;
                }
            }

            // Check if we run on windows, as if thats the case we will need
            // to round the sleepTime as workaround for a bug that only affect
            // the JVM if it runs on windows.
            //
            // See https://github.com/netty/netty/issues/356
            if (PlatformDependent.isWindows()) { // 这里是因为windows平台的定时调度最小单位为10ms，如果不是10ms的倍数，可能会引起sleep时间不准确
                sleepTimeMs = sleepTimeMs / 10 * 10;
            }

            try {
                Thread.sleep(sleepTimeMs);
            } catch (InterruptedException ignored) {
	// 调用HashedWheelTimer.stop()时优雅退出
                if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                    return Long.MIN_VALUE;
                }
            }
        }
    }

    public Set<Timeout> unprocessedTimeouts() {
        return Collections.unmodifiableSet(unprocessedTimeouts);
    }
}
```

### 四、总结

​	读完netty的时间轮的源代码，心情感觉变化了很多，才开始感觉很有一些地方写的很不好，感觉实现的太麻烦了，后来想想，发现其实是很多地方自己没有想到，就比如，才开始的任务并没有放入时间轮的格子里面，其实任务是维护在一个双向链表当中的，这种数据结构非常便于插入移除，而且使用了一种队列的操作的，避免多线程插入任务的冲突，使用了lock free的思想，使用了队列来面对并发的插入，然后批量由自己的woker自动对任务进行归集和分类，插入到对应的槽位里面。

​	netty的时间轮算法总的来说从上一个指针到下一个指针是使用sleep的方式来完成等待的，比较朴素，要想想有没有更好的方法。

当然还有其他方面的一些注意事项：

1. 操作数字型要考虑溢出问题
2. System.nanoTime(）返回值
3. Atomic*FieldUpdater类的运用
4. 一些代码设计方式
5. 不断优化性能，Lock Less代替Lock；Lock Free代替Lock Less
6. JCTool高性能队列的使用

引用的文章：[netty源码解读之时间轮算法实现-HashedWheelTimer](https://zacard.net/2016/12/02/netty-hashedwheeltimer/)