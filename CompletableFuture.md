# 背景

​		某天在写业务代码的时候突然想到了很久以前写过一个逻辑类似的代码，想着能不能复用一下，于是找到了尘封已久的老代码。

```java
private Map<String, Object> getAccountInfo(List<String> accountIds) {
    Map<String, Object> bucAccountInfoMap = new new HashMap<>(16);
    for (String accountId : accountIds) {
            bucAccountInfoMap.put(accountId,
                rpcService.getAccountInfoByAccountId(accountId).getResult());
        }
    return bucAccountInfoMap;
}
```

​		代码逻辑很简单，大概就是拿到一些账号，通过账号去调RPC接口查询具体的账户信息。但是这种写法肯定是有问题的。在此之前这块儿代码只在一个后台定时调度任务里面用到了，对实时性要求并不高，这样写好像也没什么问题(没问题？菜就是菜，还给自己找借口)。但是用在需要实时响应的接口里肯定就是不行的了，经过一些调研以后决定使用CompletableFuture进行异步调用提升接口性能。

```java
private Map<String, Object> getAccountInfo(List<String> accountIds) {
    Map<String, Object> bucAccountInfoMap = new new ConcurrentHashMap<>(16);
      CompletableFuture.allOf(accountIds.stream()
          .map(accountId -> CompletableFuture.supplyAsync(
                  () -> rpcService.getAccountInfoByAccountId(accountId).getResult(), EXECUTOR_SERVICE)
              .exceptionally(e -> {
                  log.error("getAccountInfoByAccountId error: " + e.getMessage());
                  return null;
              })
              .thenAccept(result -> {
                  if (Objects.nonNull(result)) {
                      bucAccountInfoMap.put(accountId, result);
                  }
              }))
          .toArray(CompletableFuture[]::new));
  return bucAccountInfoMap;
}
```

​		改造后，接口响应时间大幅缩短，正好趁着这个机会深入研究一下CompletableFuture。

# Future

## 1.Future类的用法和局限性

### 1.1用法

经典八股：创建线程的几种方式？

- 继承Thread类

- 实现Runnable接口
- 通过Callable接口和FutureTask类
- 通过线程池创建

其中第三种就是通过实现Callable接口的**call()**方法，该call()方法作为线程的执行体，并有返回值。然后就可以使用FutureTask类获取该返回值了。

```java
// 下面的代码通过lambda表达式可以直接优化成一行
// Callable<String> callable = () -> "test";
Callable<String> callable = new Callable<String>() {
    @Override
    public String call() throws Exception {
        return "test";
    }
};

//包装返回值
FutureTask<String> future = new FutureTask<>(callable);
//启动子线程
new Thread(future).start();
//打印子线程回调结果
System.out.println(future.get());
```

### 1.2局限性

​		使用Future获得异步执行结果时，要么调用阻塞方法get()，要么轮询看isDone()是否为true，这两种方法都不是很好，因为主线程也会被迫等待。并且多个任务，轮询get()，使用get(long,TimeUnit)方法限制超时时间，实际只是限制每个任务的时间，不能做到限制所有任务都完成的耗时时间。如以下案例：

```java
    ExecutorService executor = Executors.newFixedThreadPool(4);
    Callable<String> task1 = () -> {
        Thread.sleep(200);
        return "first";
    };

    Callable<String> task2 = () -> {
        Thread.sleep(400);
        return "second";
    };

    Future<String> future1 = executor.submit(task1);
    Future<String> future2 = executor.submit(task2);
		// 这时task1和task2已经交由线程池执行了
    long now = System.currentTimeMillis();
		// 此时future1.get会阻塞到task1执行成功，下面的代码不会执行。
    System.out.println(future1.get(200, TimeUnit.MILLISECONDS));
		// 此时再执行future2.get，但是task2已经执行了200ms，虽然整体sleep时间是400ms，但是已执行的200ms加上参数限制的200ms正好等于400ms，于是可以在规定时间内获取到返回值不会报错。
    System.out.println(future2.get(200, TimeUnit.MILLISECONDS));

    System.err.println("整体耗时，cost=" + (System.currentTimeMillis() - now));
```

定义了两个任务，任务1耗时200ms，任务2耗时400ms，获取回调结果限制时间都为200ms，实际运行结果：

![img](https://oss-ata.alibaba.com/article/2023/11/ceff5462-4c28-4385-8519-f8e7d1f6ef7f.png)

​		可以看到任务2没有抛出`TimeoutException`，总耗时远远超过200ms，如果上层调用方限制服务时间不可超过200ms，则整个调用都会因为超时被抛弃，实际上任务1的结果是可以正常返回的。

​		任务1和任务2是并发执行的，但是获取关系是先后进行，任务1首先执行get()，执行的过程中任务2同时进行，任务2的get()起点是在任务1get()完成点，因此任务2的实际执行时间为任务1执行时间+任务2get时间段。

## 2.Future源码解读

FutureTask是一个包装类，同时实现了Runnable和Future接口，获取结果实际上就是用到了Future接口。

```java
public class FutureTask<V> implements RunnableFuture<V> 
  
public interface RunnableFuture<V> extends Runnable, Future<V>
  
public interface Future<V> {
  	// 可以取消这个执行逻辑，如果这个逻辑已经正在执行，提供可选的参数来控制是否取消已经正在执行的逻辑；
    boolean cancel(boolean mayInterruptIfRunning);
  	// 判断执行逻辑是否已经被取消；
    boolean isCancelled();
    // 判断执行逻辑是否已经执行完成。
    boolean isDone();
  	// 获取执行逻辑的执行结果； 注意get是一个阻塞操作
  	V get();
  	// 可以允许在一定时间内去等待获取执行结果，如果超过这个时间，抛TimeoutException
  	V get(long timeout, TimeUnit unit);
}
```

FutureTask又是怎么实现获取线程返回值的呢？来看看FutureTask的源码

### 2.1核心参数

```java
public class FutureTask<V> implements RunnableFuture<V> {

    /**
     * 线程执行程序时可能的状态。
     * 可能的状态转换过程:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
   	/** 记录的状态值 */
    private volatile int state;
  	/** 初始状态值 */
    private static final int NEW          = 0;
    /** 完成中状态，这里需要注意，run方法执行的时候state还是一直是NEW，只有在跑完run方法进入到set(result)或者setException(ex)方法中设置为完成或者异常状态之前会进行一个状态的转换，把状态转换成COMPLETING，方法里是放在if里完成的，后面我们会讲到 */
    private static final int COMPLETING   = 1;
    /** 正常完成 */
    private static final int NORMAL       = 2;
  	/** 执行异常 */
    private static final int EXCEPTIONAL  = 3;
  	/** 取消执行 */
    private static final int CANCELLED    = 4;
  	/** 中断执行中 */
    private static final int INTERRUPTING = 5;
  	/** 中断执行 */ 
    private static final int INTERRUPTED  = 6;

    /** 初始化时传入的callable，注意在任务执行结束后会被置空，所有完成方法最后都会调用finishCompletion()方法，这里会把callable置为null */
    private Callable<V> callable;
    /** 存放结果值，也可能是报错，在调用get()方法时返回 */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** 用来执行任务的线程，在之前前会用CAS的方法判断现在有没有线程正在执行任务 */
    private volatile Thread runner;
    /** 等待队列，当一个线程调用get()方法并且任务还没有完成时，该线程会被添加到等待队列中。
    当任务完成时，会调用finishCompletion方法。在这个方法里，会遍历waiters链表，并唤醒所有在等待的线程。*/
    private volatile WaitNode waiters;
}
```

### 2.2 run方法

```java
public void run() {
  	// 判断state状态是否为NEW，并通过CAS (Compare-And-Swap) 操作将 runner 设置为当前线程，如果都满足则执行，否则直接返回。
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
              	// 执行call方法，这里是个阻塞操作。直到执行结束才进行下一步，所有run方法过程中state始终为NEW。
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
				// 执行完run方法后保障runner为null
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}

protected void set(V v) {
  	// 通过CAS将状态设置为正在完成中
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
      // 赋值result
      outcome = v;
      // 设置state为最终状态
      STATE.setRelease(this, NORMAL); // final state
      // 这个方法的作用是唤醒所有在waiter里面的等待线程，并把callable设置为null
      finishCompletion();
    }
}
```

### 2.3 get方法

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
// awaitDone方法会创建一个WaitNode对象并将其添加到等待队列的头部。这个方法会使当前线程阻塞，直到任务完成。
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    long startTime = 0L; 
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
      int s = state;
      if (s > COMPLETING) {
        if (q != null)
          q.thread = null;
        return s;
      }
      else if (s == COMPLETING)
        // We may have already promised (via isDone) that we are done
        // so never return empty-handed or throw InterruptedException
        Thread.yield();
      else if (Thread.interrupted()) {
        removeWaiter(q);
        throw new InterruptedException();
      }
      else if (q == null) {
        if (timed && nanos <= 0L)
          return s;
        q = new WaitNode();
      }
      else if (!queued)
        queued = WAITERS.weakCompareAndSet(this, q.next = waiters, q);
      else if (timed) {
        final long parkNanos;
        if (startTime == 0L) { // first time
          startTime = System.nanoTime();
          if (startTime == 0L)
            startTime = 1L;
          parkNanos = nanos;
        } else {
          long elapsed = System.nanoTime() - startTime;
          if (elapsed >= nanos) {
            removeWaiter(q);
            return state;
          }
          parkNanos = nanos - elapsed;
        }
        // nanoTime may be slow; recheck before parking
        if (state < COMPLETING)
          LockSupport.parkNanos(this, parkNanos);
      }
      else
        LockSupport.park(this);
    }
}
```

# CompletableFuture

## 1. 介绍

​		CompletableFuture是JAVA 8引入的Future的实现类，针对Future做了改进，当异步任务完成或者发生异常时，自动调用回调对象的回调方法，并且提供了函数式编程的能力。CompletableFuture还实现了另外一个接口—CompletionStage，CompletableFuture的众多方法以及函数式能力都是这个接口赋予的。

## 2.用法



## 3.核心源码解读

​		在读源码之前先贴一个小用例，跟着用例去阅读源码。

```java
public class Test1 {

    private static final Logger log = LoggerFactory.getLogger(Test1.class);
    private static int[] timeArray = new int[]{0, 1, 2, 3};

    private static final ThreadPoolExecutor executor = new ThreadPoolExecutor(3, 3, 0, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(100), new ThreadPoolExecutor.AbortPolicy());

    public static void main(String[] args) {

        int size = timeArray.length - 1;
        CompletableFuture[] cfArray = new CompletableFuture[size];
        List<CompletableFuture<Integer>> cfList = new ArrayList<>(size);
        log.info("start CompletableFuture task by async");
        for (int i = 0; i <= size; i++) {
            final int index = i;
            CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> sleepTimeAndReturn(index), executor);
            cfList.add(future);
        }
        log.info("start join");
        CompletableFuture.allOf(cfList.toArray(cfArray)).join();
        log.info("end join");
        int sum = 0;
        for (CompletableFuture<Integer> future : cfList) {
            try {
                Integer i = future.get();
                if (i != null) {
                    sum += i;
                }
            } catch (Exception e) {
                log.error("main error={}", e);
            }
        }
        log.info("sum={}", sum);
        executor.shutdown();
    }

    // 模拟查询接口耗时和返回值
    private static Integer sleepTimeAndReturn(int index) {
        Integer time = null;
        try {
            log.info("index:" + index + ",start sleep");
            if (index == 2) {
                throw new RuntimeException("RPC exception"); // 模拟异常
            }
            time = timeArray[index];
            Thread.sleep(time * 1000L); // 模拟接口耗时时间
            log.info("index:" + index + ",end sleep");
            return time;
        } catch (Exception e) {
            log.error("index:" + index + ",system error", e);
        }
        return time;
    }
}
```

​		代码模拟了CompletableFuture调用RPC接口的情况，通过timeArray模拟RPC接口不同的响应时间，并模拟了其中一个RPC接口报错的情况。

执行情况如下

![image-20240822175359729](/Users/leiyu/Library/Application Support/typora-user-images/image-20240822175359729.png)

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
  	// 存储运行结果
  	volatile Object result;      
  	// 栈，流程执行CompletableFuture时会压入栈中，栈顶一直是自己
    volatile Completion stack;
}
```
