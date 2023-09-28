---
title: "Future"
date: 2019-05-07T20:21:27+08:00
draft: false
---

## Future 类
Java Future相关的代码基本都在java.util.concurrent的包里面，Future.java是一个接口，定义了最基本的一些任务操作和状态判断。
```java
// Future.java

boolean cancel(boolean mayInterruptIfRunning);
boolean isCancelled();
boolean isDone();
V get() throws InterruptedException, ExecutionException;
V get(long timeout, TimeUnit unit);
```
我们从FutureTask.java去了解Future的内部机制，FutureTask对Future有比较接地气的实现，其他的实现或多或少都加入了新的一些特性，对了解原理没太大帮助。
FutureTask.java是对Futre和Runnable最简单的实现，所以FutureTask是一个可执行的异步对象。

## FutureTask的状态
FutureTask定义了7种状态，这7种状态里面包含了简单的状态机，使用了一个用volatile修饰的int来记录状态。如下：

```java
/** Possible state transitions:
* NEW -> COMPLETING -> NORMAL
* NEW -> COMPLETING -> EXCEPTIONAL
* NEW -> CANCELLED
* NEW -> INTERRUPTING -> INTERRUPTED
*/
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

状态的变换基本在FutureTask.java的所有函数中都有体现，我们来看看几个典型的状态变换：

1. 构造函数
构造函数完成时，将状态置为NEW。

2. 取消操作
取消操作对任务状态进行判断。
    1. 如果任务正在执行但没有完成时，发出中断，并将任务状态置为中断中状态，并在执行线程完成后，置为中断完成状态NEW -> INTERRUPTING -> INTERRUPTED
    2. 当任务还没有执行，则直接置为取消状态 NEW -> CANCELLED

3. 执行操作
执行分执行成功和执行异常两种，状态转换路径是这两个。
NEW -> COMPLETING -> NORMAL和NEW -> COMPLETING -> EXCEPTIONAL

## FutureTask的执行

`FutureTask`因为封装了`Runnable`的接口，实现了`run`函数，所以可以直接执行，直接执行使用主线程；`FutureTask`的另外一种执行方式是提交到线程池去执行，由线程池去分配执行线程；只有提交到线程池去执行才能体现异步的特性。不过我们不关注执行的方式，我们关注执行的逻辑。

`FutureTask`的构造函数传入了`Callable`或`Runnable`对象，也即是需要执行的业务逻辑，他是业务逻辑的基本表现形式，保存在类属性`callable`，在`run`函数里面，调用`callalbe.call()`来执行业务逻辑。`run`函数主要完成以下几个操作。

1. 判断当前状态是否为`NEW`，如果不是，说明任务被执行过，或者已被取消，直接返回。
2. 如果状态为`NEW`，**接着会通过unsafe类把任务执行线程保存在runner字段中，如果保存失败，则直接返回**。
3. 执行任务
4. 任务执行成功，`set`保存结果，不成功`setException`保存异常信息，任务的状态在这里改变。

以下是`run`函数的逻辑。

```
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread())) // 1, 2
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call(); // 3
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex); // 4
            }
            if (ran)
                set(result); // 4
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

由于在任务执行的过程中，可能被取消，所以在`finally`块里，会任务的根据状态来做一些善后的工作。

## FutureTask的取消

任务的取消比较简单，我们知道，在执行的时候，执行任务的线程会保存在`runner`属性中，所以对于正在执行的任务，**取消的本质就是将执行的线程取出来，向该线程发出interupt信号**。但对于一个较为完备的取消动作，`cancel`做了一下几个动作。

1. 判断任务当前执行状态，如果任务状态不为`NEW`，则说明任务或者已经执行完成，或者执行异常，不能被取消，直接返回`false`表示执行失败。

2. 判断需要中断任务执行线程，则

   1. 把任务状态从`NEW`转化到`INTERRUPTING`。这是个中间状态。
   2. 中断任务执行线程。
   3. 修改任务状态为`INTERRUPTED`。

3. 如果不需要中断任务执行线程，直接把任务状态从`NEW`转化为`CANCELLED`。如果转化失败则返回`false`表示取消失败。

4. 调用`finishCompletion`

   这个函数将阻塞在等待这个任务完成的线程唤醒，具体操作是`LockSupport.unpark(t)`，这些线程都是在`awaitDone`函数内的`LockSupport.park(this)`中阻塞的，关于`awaitDone`函数的作用后文还会继续介绍。`finishCompletion`除了在这里有调用以外，在`set`和`setException`中也有调用。

   以下是`finishCompletion`的实现

   ```
   /**
    * Removes and signals all waiting threads, invokes done(), and
    * nulls out callable.
    */
   private void finishCompletion() {
       // assert state > COMPLETING;
       for (WaitNode q; (q = waiters) != null;) {
           if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
               for (;;) {
                   Thread t = q.thread;
                   if (t != null) {
                       q.thread = null;
                       LockSupport.unpark(t);
                   }
                   WaitNode next = q.next;
                   if (next == null)
                       break;
                   q.next = null; // unlink to help gc
                   q = next;
               }
               break;
           }
       }
       done();
       callable = null;        // to reduce footprint
   }
   ```

以下是`cancel`函数的实现。

```
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED))) // 1, 2
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED); // 3
            }
        }
    } finally {
        finishCompletion(); // 4
    }
    return true;
}
```

## FutureTask的结果

异步任务提交到线程池去执行后，由于无法预知什么时候结束，所以必须得提供接口来获取任务执行的结果，在`FutureTask`中，`get`函数用来获取任务执行的结果。该函数有了设置超时和不超时的两种实现。

除去超时的差异，`get`操作对任务的状态进行判断，当状态还没有完成的时候，调用`awaitDone`函数来等待完成，我们在上面提到过这个函数，对于未完成的任务`awaitDone`阻塞，而`finishCompletion`唤醒阻塞。

```
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

我们来详细看看awaitDone函数做了什么操作，它是如何阻塞的，它是怎么设置超时和完成返回的。

`awaitDone`主题是一个死循环，轮询判断任务的状态。

1. 当执行的线程被中断时，调用`removeWaiter`移除等待节点`WaitNode`，抛出中断异常

2. 当状态为已经完成，直接返回

3. 当状态为完成中，通过`Thread.yield()`让出CPU时间

4. 如果当前线程还没有创建`WaitNode`等待节点保存到等待队列里面去，则新建一个等待节点，插入到等待链表，表明当前线程也准备进入等待该任务完成的队列中去。

5. 最后是进入阻塞的动作，通过`LockSupport.park`，如果设置了超时的时间，则将时间作为参数传递到`park`中去。

   综上，`awaitDone`函数除了对状态的判断以外，核心就是`LockSupport`的阻塞等待完成的操作了。我们后面还会探讨一下`LockSupport`。

   ```
   private int awaitDone(boolean timed, long nanos)
       throws InterruptedException {
       final long deadline = timed ? System.nanoTime() + nanos : 0L;
       WaitNode q = null;
       boolean queued = false;
       for (;;) {
           if (Thread.interrupted()) {
               removeWaiter(q);
               throw new InterruptedException();
           }
   
           int s = state;
           if (s > COMPLETING) {
               if (q != null)
                   q.thread = null;
               return s;
           }
           else if (s == COMPLETING) // cannot time out yet
               Thread.yield();
           else if (q == null)
               q = new WaitNode();
           else if (!queued)
               queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                    q.next = waiters, q);
           else if (timed) {
               nanos = deadline - System.nanoTime();
               if (nanos <= 0L) {
                   removeWaiter(q);
                   return state;
               }
               LockSupport.parkNanos(this, nanos);
           }
           else
               LockSupport.park(this);
       }
   }
   ```

当`awaitDone`的阻塞完成以后，就会将结果返回，将结果返回是通过`report`函数来实现的，返回的是执行完成的结果或者是执行中获得的异常信息。

## LockSupport

异步执行任务，在获取任务状态时，阻塞是必然的。在这里是通过`LockSupport`来实现线程的阻塞(park)和唤醒(unpark)。

```
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}

public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);
}
```

可以看到`LockSupport`则是通过`UNSAFE`的同名函数来实现的。**java不能直接访问操作系统底层，而是通过本地方法来访问。Unsafe类提供了硬件级别的原子操作。**

> Unsafe类是在sun.misc包下，不属于java标准。但是很多Java的基础类库，包括一些被广泛使用的高性能开发库都是基于Unsafe类开发的，比如Netty、Cassandra、Hadoop、Kafka等。Unsafe类在提升Java运行效率，增强Java语言底层操作能力方面起了很大的作用。**Unsafe类使Java拥有了像C语言的指针一样操作内存空间的能力，同时也带来了指针的问题**。过度的使用Unsafe类会使得出错的几率变大，因此Java官方并不建议使用的，官方文档也几乎没有。

所以在这里Java也是使用了C级别的线程同步机制来完成这些操作的，在这就不再展开了。

