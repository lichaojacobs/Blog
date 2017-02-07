---
title: ReentrantLock与condition应用后的思考
date: 2017-01-20 16:02:15
tags:
    - 多线程
    - ReentrantLock 
---
一直在断断续续看《java并发编程实战》这本书，每次看都有不一样的体会，前些日子在知乎上回答了一个关于ReentrantLock的问题[java里是怎么通过condition接口是获取监视器方法的][1] ,那次回答之后也引发了我对其实现的进一步探究。

### 一个例子
举一个简单的例子，是基本上各个公司招聘的时候都会出现的关于多线程间通信的问题：利用多线程循环打印n次"ABC"。当然，这个题目有很多实现方法，有经典的wait和notify的原生方法,也有时髦一点的ReentrantLock写法，如下:
```
/**
 * Created by lichao on 2016/1/20.
 */
public class PrintABC {

  static ReentrantLock lock = new ReentrantLock();
  static Condition conditionA = lock.newCondition();
  static Condition conditionB = lock.newCondition();
  static Condition conditionC = lock.newCondition();
  static int signal = 1;//1=>A, 2=>B 3=>C
  static int loopValue = 10;

  class taskA implements Runnable {
    @Override
    public void run() {
      lock.lock();
      try {
        for (int i = 0; i < loopValue; i++) {
          if (signal != 1) {
            conditionA.await();
            conditionB.signalAll();
            conditionC.signalAll();
          }
          signal = 2;
          System.out.print("A");
          conditionB.signal();
          conditionA.await();
        }
      } catch (Exception ex) {

      } finally {
        lock.unlock();
      }
    }
  }

  class taskB implements Runnable {

    @Override
    public void run() {
      lock.lock();
      try {
        for (int i = 0; i < loopValue; i++) {
          if (signal != 2) {
            conditionB.await();
            conditionA.signalAll();
            conditionC.signalAll();
          }
          signal = 3;
          System.out.print("B");
          conditionC.signal();
          conditionB.await();

        }
      } catch (Exception ex) {

      } finally {
        lock.unlock();
      }
    }
  }

  class taskC implements Runnable {

    @Override
    public void run() {
      lock.lock();
      try {
        for (int i = 0; i < loopValue; i++) {
          if (signal != 3) {
            conditionC.await();
            conditionB.signalAll();
            conditionA.signalAll();
          }
          signal = 1;
          System.out.print("C");
          conditionA.signal();
          conditionC.await();
        }
      } catch (Exception ex) {

      } finally {
        lock.unlock();
      }
    }
  }

  public static void main(String[] args) {
    ExecutorService executorService = Executors.newFixedThreadPool(3);
    executorService.submit(new PrintABC().new taskC());
    executorService.submit(new PrintABC().new taskB());
    executorService.submit(new PrintABC().new taskA());
  }
}
```
首先定义了一个可重入锁，然后起了三个监视器用于控制任务a,b,c的状态，signal作为信号量用于表明当前应该哪个线程执行打印操作，实现无论任务往线程池中提交的顺序如何都能正确打印ABC的顺序

### 讲讲原理
- ReentrantLock（重入锁）是jdk的concurrent包提供的一种独占锁的实现。它继承自Dong Lea的 AbstractQueuedSynchronizer（同步器）。回到上面的代码，我们提交任务是按照CBA的次序来提交的，也就是打印C的任务会先开始执行，而当前的信号量signal为1,也就是A而不是3，所以通过conditonC.await()来释放锁，同时线程休眠等待唤醒，这时A拿到了，并且打印后将signal置为2即B，同时通过conditionA.await()方法使自己休眠，并唤醒B进行打印。以此类推，总的来说，ReentrantLock�与condition配合，优雅的完成了wait和notify做的事情。
- 我们来看看其中是如何实现这种线程的调度过程的：reentrantLock.newCondition() 返回的是Condition的一个实现，该类在AbstractQueuedSynchronizer中被实现，可以访问AbstractQueuedSynchronizer中的方法和其余内部类,await被调用时的代码如下：
```
public final void await() throws InterruptedException {
if (Thread.interrupted())
 throw new InterruptedException();
 //将当前线程包装下后，添加到Condition自己维护的一个链表中。
 Node node = addConditionWaiter(); 
 //释放当前线程占有的锁
int savedState = fullyRelease(node);
int interruptMode = 0;
 while (!isOnSyncQueue(node)) {
 //释放完毕后，不断AQS的队列，看当前节点是否在队列中，不在 说明它还没有竞争锁的资格，所以继续将自己沉睡。直到它被重新加入到队列中.
 LockSupport.park(this);
 if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
 break;
 }
//被唤醒后，重新开始正式竞争锁，同样，如果竞争不到还是会将自己沉睡，等待唤醒重新开始竞争。
if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
 interruptMode = REINTERRUPT;
 if (node.nextWaiter != null)
 unlinkCancelledWaiters();
 if (interruptMode != 0)
 reportInterruptAfterWait(interruptMode);
 }
```
signal方法的调用代码
```
public final void signal() {
 if (!isHeldExclusively())
 throw new IllegalMonitorStateException();
 Node first = firstWaiter; //firstWaiter为condition自己维护的一个链表的头结点，
                          //取出第一个节点后开始唤醒操作
 if (first != null)
 doSignal(first);
 }
 
 
 private void doSignal(Node first) {
 do {
 if ( (firstWaiter = first.nextWaiter) == null) //修改头结点，完成旧头结点的移出工作
 lastWaiter = null;
 first.nextWaiter = null;
 } while (!transferForSignal(first) &&//将老的头结点，加入到AQS的等待队列中
 (first = firstWaiter) != null);
 }

final boolean transferForSignal(Node node) {
 /*
 * If cannot change waitStatus, the node has been cancelled.
 */
 if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
 return false;

/*
 * Splice onto queue and try to set waitStatus of predecessor to
 * indicate that thread is (probably) waiting. If cancelled or
 * attempt to set waitStatus fails, wake up to resync (in which
 * case the waitStatus can be transiently and harmlessly wrong).
 */
 Node p = enq(node);
 int ws = p.waitStatus;
//如果该结点的状态为cancel 或者修改waitStatus失败，则直接唤醒。
 if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
 LockSupport.unpark(node.thread);
 return true;
 }
```
其实condition内部维护了一个等待队列，用于存放等待signal的任务
```
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
        
**
         * Adds a new waiter to wait queue.
         * @return its new wait node
         */
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
}
```
而AbstractQueuedSynchronizer中也维护了一个队列，就是获取当前资源的等待队列，当资源释放掉之后，会依次从队列中恢复线程，直至为空。每个线程会在这俩个队列中来回切换，但同一时刻仅存在于一个队列中。
![ReentrantLock流程](http://jacobs.wanhb.cn/images/reentrantLock.jpg)
```seq
线程1->AQS: ReentrantLock.lock（加入AQS队列，获取资源）
线程1->AQS: condition.await()（移除队列，释放资源）
AQS->condition队列: 线程1加入condition等待队列
线程2->AQS: 线程2加入AQS队列并获取资源
线程2->AQS: condition2.await()（移除AQS队列，释放资源）
线程2->线程1: condition.signal()（线程2调用signal唤醒线程1）
线程1->AQS: ReentrantLock.lock(重新加入AQS队列，获取资源)
```



###总结
还是得深入源码去看问题，不能只关注业务，否则会成为彻头彻尾的搬砖工，不仅要会用轮子，还要会造轮子。最后分享一片陈浩写的技术人员的职业生涯文章，共勉。[技术人员的发展之路][2]


  [1]: https://www.zhihu.com/question/52273413
  [2]: http://coolshell.cn/articles/17583.html