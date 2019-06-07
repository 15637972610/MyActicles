# 温故Java基础（一）多线程编程 #

**本文是多线程编程系列第二弹，多线程编程之线程安全，第一篇多线程入门：**
## 二、线程安全 ##
**本文将从以下几个方面温故线程安全问题**

1. **什么是线程安全问题**
2. **如何解决线程安全问题**
3. **多线程死锁问题**
4. **多线程的三大特性**
5. **Java内存模型**
6. **volatile关键字**

### 1.什么是线程安全问题 ###

话不多说，先看一段代码：

    public class MyRunnable implements Runnable {
	private int count = 100;

	@Override
	public void run() {
		while (count > 0) {
			try {
				Thread.sleep(50);
			} catch (Exception e) {
				Log.d(TAG,"异常信息打印 e = "+e.toString());
			}
			System.out.println(Thread.currentThread().getName() + ",出售第" + (100 - count + 1) + "票");
				}
			}

		}

	public class ThreadDemo {
	public static void main(String[] args) {
		MyRunnable runnable = new MyRunnable();
		Thread t1 = new Thread(runnable, "线程①");
		Thread t2 = new Thread(runnable, "线程②");
		t1.start();
		t2.start();
		}
	}

以上是一个熟知的案例两个窗口卖100张票，思考下，如果这种写法会有什么问题呢？
看下打印的结果：

    线程①，出售第1张票
	线程②，出售第1张票
	线程②，出售第3张票
	线程①，出售第4张票
	线程①，出售第5张票
	线程②，出售第6张票
	线程①，出售第7张票
	线程②，出售第7张票

出现了线程①和线程②同时出售同一张票的情况，这是我们在现实需求中不想见到的。

想一下出现这个问题的原因，CPU在执行多线程时，在执行过程中可能随时切换到其他线程上执行，有可能出现多个线程同事操作同一个变量的情况。

总结一下，多个线程同时共享一个成员变量，对这个变量做更改，也就是写操作的时候可能会出现冲突问题，也就是线程安全问题。

### 2.如何解决线程安全问题 ###
知道了出现上面线程安全问题的原因，想一下，那如果我们能保证一个线程在执行对变量写操作的时候，不让其他线程执行这部分的操作是不是就行了。Java中为我们提供了（synchronized修饰符）同步代码块技术解决这个问题。

- 同步代码块
- 同步函数
- 静态同步函数

**使用同步代码块改造后的售票案例，如下**：


    public class MyRunnable implements Runnable {
	private int count = 100;
	private Object oj = new Object();

	@Override
	public void run() {
    while (count > 0) {
        try {
            Thread.sleep(50);
        } catch (Exception e) {
            Log.d(TAG,"异常信息打印 e = "+e.toString());
        }
		synchronized（oj）{
        System.out.println(Thread.currentThread().getName() + ",出售第" + (100 - count + 1) + "票");
					}
        		}
       		}
		}
    }
其中锁可以为任意对象，也可以是this。持有锁的线程可以执行同步代码块中的代码，没持有锁的线程，即使获得了CPU的执行权也只能等待锁被释放后抢到锁才能执行。
同步代码块中的锁，在代码执行完自动释放。

同步的前提：

1. 必须是两个及以上线程
2. 必须是多个线程使用同一个锁

**好处：解决多线程的安全问题**；
**弊端：多个线程需要判断锁，抢锁较为消耗资源**



同步代码块写法：

        synchronized(对象锁){
		需要被同步的代码
		}

同步函数写法：

    public synchronized void methodA(){
		需要同步的代码
	}
静态同步函数写法：

     public static synchronized void methodA(){
		需要同步的代码
	}
跟同步函数相比方法上多了一个static修饰符，下面说一下静态同步函数和非静态同步函数的区别：

a.同步使用的锁是this对象锁

证明方式：一个线程使用同步代码块（this锁）一个线程使用同步函数，如果两个线程售票出现数据错误，证明不是this锁。看代码：

    class MyRunnable2 implements Runnable {
	private int count = 100;
	public boolean flag = true;
	@Override
	public void run() {
		if (flag) {

			while (count > 0) {

				synchronized (this) {
					if (count > 0) {
						try {
							Thread.sleep(50);
						} catch (Exception e) {
							// TODO: handle exception
						}
						System.out.println(Thread.currentThread().getName() + ",出售第" + (100 - count + 1) + "票");
						count--;
					}
				}

			}

		} else {
			while (count > 0) {
				sale();
			}
		}

	}

	public synchronized void sale() {
		if (count > 0) {
			try {
				Thread.sleep(50);
			} catch (Exception e) {
				// TODO: handle exception
			}
			System.out.println(Thread.currentThread().getName() + ",出售第" + (100 - count + 1) + "票");
			count--;
		}
	}
	}

	public class ThreadDemo2 {
	public static void main(String[] args) throws InterruptedException {
		MyRunnable2 myRunnable2 = new MyRunnable2();
		Thread t1 = new Thread(myRunnable2, "①号窗口");
		Thread t2 = new Thread(myRunnable2, "②号窗口");
		t1.start();
		Thread.sleep(40);
		threadTrain1.flag = false;
		t2.start();
		}
	}


b.静态同步函数使用的锁是类的字节码文件 ×××.class不是同一个对象

证明方式与1类似不再重复。

### 3. 多线程死锁问题###
两个问题：什么是死锁？为什么会发生死锁现象？

a.**什么是多线程死锁？**

同步中嵌套同步，导致锁无法释放

看下如下代码：

    class MyRunnable3 implements Runnable {

	private int trainCount = 100;
	public boolean flag = true;
	private Object ob = new Object();

	@Override
	public void run() {
		if (flag) {
			while (true) {
				synchronized (ob) {
					sale();
				}
			}
		} else {
			while (true) {
				sale();
			}
		}
	}


	public synchronized void sale() {
		synchronized (ob) {
			if (trainCount > 0) {
				try {
					Thread.sleep(40);
				} catch (Exception e) {

				}
				System.out.println(Thread.currentThread().getName() + ",出售 第" + (100 - trainCount + 1) + "张票.");
				trainCount--;
			}
		}
	}
	}

	public class DeadlockThread {

	public static void main(String[] args) throws InterruptedException {

		MyRunnable3 myRunnable3 = new MyRunnable3();
		Thread thread1 = new Thread(myRunnable3, "①窗口");
		Thread thread2 = new Thread(myRunnable3, "②窗口");
		thread1.start();
		Thread.sleep(40);
		threadTrain.flag = false;
		thread2.start();
	}

	}


b.**为什么会发生死锁现象？**
上述代码中，

线程1先拿到同步代码块中的ob锁，然后再拿到同步函数中的this锁；

线程2先拿到同步函数中的this锁，然后再拿到同步代码块中的ob锁。

同步中嵌套同步，互相不释放锁。

c.**如何避免死锁？**

不要在同步中嵌套同步

### 4.多线程的三大特性 ###

a.**原子性**

一个或多个操作，要么全部执行，并且不会被打断，要么都不执行。例如银行转账，一个账户做减法，一个账户做加法，这两个操作必须具备原子性才不会出现差错。原子性保证了数据的一致性。

b.**可见性**

多个线程共享一个变量，做写操作时，一个线程修改这个变量的值，其他线程能够立即看到修改的值


c.**有序性**

程序执行的顺序按照代码的先后顺序执行。
一般来说处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的。如下：

int a = 10;    //语句1

int r = 2;    //语句2

a = a + 3;    //语句3

r = a*a;     //语句4

则因为重排序，他还可能执行顺序为 2-1-3-4，1-3-2-4
但绝不可能 2-1-4-3，因为这打破了依赖关系。
显然重排序对单线程运行是不会有任何问题，而多线程就不一定了，所以我们在多线程编程时就得考虑这个问题了。

### 5.Java内存模型 ###

Java内存模型即共享内存模型，简称JMM，它决定一个线程对共享变量写入时，能对另一个线程可见。线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了改线程读写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化。

来看一张图简单梳理下JMM：


### 6.volatile关键字 ###

a.**volatile有什么用**

让多线程的共享变量在诸线程间可见

    class ThreadVolatileDemo extends Thread {
	public    boolean flag = true;
	@Override
	public void run() {
		System.out.println("开始执行子线程....");
		while (flag) {
		}
		System.out.println("线程停止");
	}
	public void setRuning(boolean flag) {
		this.flag = flag;
	}

	}

	public class ThreadVolatile {
	public static void main(String[] args) throws InterruptedException {
		ThreadVolatileDemo threadVolatileDemo = new ThreadVolatileDemo();
		threadVolatileDemo.start();
		Thread.sleep(3000);
		threadVolatileDemo.setRuning(false);
		System.out.println("flag 已经设置成false");
		Thread.sleep(1000);
		System.out.println(threadVolatileDemo.flag);

	}
	}
运行：

    开始执行子线程....
代码中我们已经设置setRuning（false）了，为什么感觉不起作用呢？

根据上面的Java内存模型知道，线程间不可见，本地内存读取的是主内存的副本，主内存改变值了，本地内存并没有取主内存拿改变后的值。

解决：使用volatile 修饰flag

    public  volatile   boolean flag = true;
保证共享变量可见性。

b.**volatile的非原子性**

可以使用原子类：

    public class VolatileNoAtomic extends Thread {
	static int count = 0;
	private static AtomicInteger atomicInteger = new AtomicInteger(0);

	@Override
	public void run() {
		for (int i = 0; i < 1000; i++) {
			//等同于i++
			atomicInteger.incrementAndGet();
		}
		System.out.println(atomicInteger);
	}

	public static void main(String[] args) {
		// 初始化10个线程
		VolatileNoAtomic[] volatileNoAtomic = new VolatileNoAtomic[10];
		for (int i = 0; i < 10; i++) {
			// 创建
			volatileNoAtomic[i] = new VolatileNoAtomic();
		}
		for (int i = 0; i < volatileNoAtomic.length; i++) {
			volatileNoAtomic[i].start();
		}
	}

 	}

c.**volatile和synchronized区别**

仅靠volatile不能保证线程的安全性。（原子性）

①volatile轻量级，只能修饰变量。synchronized重量级，还可修饰方法

②volatile只能保证数据的可见性，不能用来同步，因为多个线程并发访问volatile修饰的变量不会阻塞。

synchronized不仅保证可见性，而且还保证原子性，因为，只有获得了锁的线程才能进入临界区，从而保证临界区中的所有语句都全部执行。多个线程争抢synchronized锁对象时，会出现阻塞。

线程安全性

线程安全性包括两个方面，①可见性。②原子性。

从上面自增的例子中可以看出：仅仅使用volatile并不能保证线程安全性。而synchronized则可实现线程的安全性。


参考阅读：
> https://www.cnblogs.com/cb0327/p/4986286.html#_label6
