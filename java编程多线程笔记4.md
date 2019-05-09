# 温故 java基础（一）多线程编程——并发编程 #
本文是多线程系列第四弹，前三篇地址：
> 1.多线程入门：https://blog.csdn.net/qq_26628329/article/details/89209019
> 
> 2.线程安全：https://blog.csdn.net/qq_26628329/article/details/89249209
> 
> 3.线程间通讯：https://blog.csdn.net/qq_26628329/article/details/89334840
	
昨天睡太晚，今天早上起床心情不美丽，以后还是要早睡早起呀，话不多说，看正文。
## 四、并发编程 ##
先说下并发编程的**目的**是什么，**是为了让我们的程序运行的更快，高效的完成一件任务**。

回想一下之前使用多线程的时候出现的安全问题，我们使用**synchronized**关键字给代码块或者函数加同步锁方式解决多线程安全问题。

举个例子，拿面试中经常问到的Vector、LinkedList与ArrayList、Hashtable和HashMap来看下：

**Vector、ArrayList、LinkedList**

Vector、ArrayList两个都是集合类，本质上都是Object[]数组,而且数组大小可变，不同之处Vector是线程安全的、方法上会加synchronized修饰，效率低，ArrayList是非线程安全的，效率比Vector高。

LinkedList是一个功能强大的集合类，与前两者不同，它底层是链表结构，明显区别就是插入和删除块，查询速度较慢。

另外，LinkedList集合实现了Deque(Queue接口的子接口)，所以它是一个双向队列，还有一些pop(出栈)和push(入栈)两个方法。可以当做栈来使用。

**Hashtable和HashMap**

这两个键值对集合都实现了Map接口，HashMap是非synchronized，可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行。Hashtable是synchronized，Hashtable是线程安全的效率低。

Collections.synchronized*(m) 将线程不安全额集合变为线程安全集合

看到了他们的优缺点，有没有更好的方式既能保证线程安全，又比较好用？答案是肯定的jdk1.5之后提供了大量的并发API，下面我们就来探讨下。


本文将从以下方面探讨：

1. **1.ConcurrentHashMap原理简述**
2. **2.CountDownLatch**
3. **3.CyclicBarrier**
4. **4.Semaphore信号量**
5. **5.并发队列ConcurrentLinkedDeque、BlockingQueue阻塞队列用法**

### 1.ConcurrentHashMap使用分析 ###
线程安全，内部使用段（Segment）来表示不同的部分，每个段就是一个小的Hashtable,每个段有自己各自的锁，在多个修改操作发生在不同的段上的情况下，他们可以实现并发进行。并且代码中大多共享变量使用volatile关键字声明，性能比较好。
### 2.CountDownLatch ###

CountDownLatch类位于java.util.concurrent包下，利用它可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了

### 3.CyclicBarrier ###

CyclicBarrier初始化时规定一个数目，然后计算调用了CyclicBarrier.await()进入等待的线程数。当线程数达到了这个数目时，所有进入等待状态的线程被唤醒并继续。 
 CyclicBarrier就象它名字的意思一样，可看成是个障碍， 所有的线程必须到齐后才能一起通过这个障碍。 
CyclicBarrier初始时还可带一个Runnable的参数， 此Runnable任务在CyclicBarrier的数目达到后，所有其它线程被唤醒前被执行。

### 4.Semaphore信号量 ###
Semaphore是一种基于计数的信号量。它可以设定一个阈值，基于此，多个线程竞争获取许可信号，做自己的申请后归还，超过阈值后，线程申请许可信号将会被阻塞。Semaphore可以用来构建一些对象池，资源池之类的，比如数据库连接池，我们也可以创建计数为1的Semaphore，将其作为一种类似互斥锁的机制，这也叫二元信号量，表示两种互斥状态。它的用法如下：
availablePermits函数用来获取当前可用的资源数量
wc.acquire(); //申请资源
wc.release();// 释放资源

    	// 创建一个计数阈值为5的信号量对象  
    	// 只能5个线程同时访问  
    	Semaphore semp = new Semaphore(5);  
    	  
    	try {  
    	    // 申请许可  
    	    semp.acquire();  
    	    try {  
    	        // 业务逻辑  
    	    } catch (Exception e) {  
    	  
    	    } finally {  
    	        // 释放许可  
    	        semp.release();  
    	    }  
    	} catch (InterruptedException e) {  
    	  
    	}  

需求: 一个厕所只有3个坑位，但是有10个人来上厕所，那怎么办？假设10的人的编号分别为1-10，并且1号先到厕所，10号最后到厕所。那么1-3号来的时候必然有可用坑位，顺利如厕，4号来的时候需要看看前面3人是否有人出来了，如果有人出来，进去，否则等待。同样的道理，4-10号也需要等待正在上厕所的人出来后才能进去，并且谁先进去这得看等待的人是否有素质，是否能遵守先来先上的规则。

    class Parent implements Runnable {
	private String name;
	private Semaphore wc;
	public Parent(String name,Semaphore wc){
		this.name=name;
		this.wc=wc;
	}
	@Override
	public void run() {
		try {
			// 剩下的资源(剩下的茅坑)
			int availablePermits = wc.availablePermits();
			if (availablePermits > 0) {
				System.out.println(name+"天助我也,终于有茅坑了...");
			} else {
				System.out.println(name+"怎么没有茅坑了...");
			}
			//申请茅坑 如果资源达到3次，就等待
			wc.acquire();
			System.out.println(name+"终于轮我上厕所了..爽啊");
			   Thread.sleep(new Random().nextInt(1000)); // 模拟上厕所时间。
			System.out.println(name+"厕所上完了...");
			wc.release();
			
		} catch (Exception e) {

		}
	}
	}
	public class TestSemaphore02 {
	public static void main(String[] args) {
		// 一个厕所只有3个坑位，但是有10个人来上厕所，那怎么办？假设10的人的编号分别为1-10，并且1号先到厕所，10号最后到厕所。那么1-3号来的时候必然有可用坑位，顺利如厕，4号来的时候需要看看前面3人是否有人出来了，如果有人出来，进去，否则等待。同样的道理，4-10号也需要等待正在上厕所的人出来后才能进去，并且谁先进去这得看等待的人是否有素质，是否能遵守先来先上的规则。
         Semaphore semaphore = new Semaphore(3);
		for (int i = 1; i <=10; i++) {
			 Parent parent = new Parent("第"+i+"个人,",semaphore);
			 new Thread(parent).start();
		}
	}
	}


### 5.并发队列ConcurrentLinkedDeque、BlockingQueue阻塞队列 ###
