# 温故Java基础（一）多线程编程—多线程之间的通讯 #
**本文是多线程系列第三弹，前两篇文章见：**
> **多线程编程——多线程入门：https://blog.csdn.net/qq_26628329/article/details/89209019**

> **多线程编程——线程安全：https://blog.csdn.net/qq_26628329/article/details/89249209**

----------

## 三、多线程之间的通讯 ##
**与上两篇文章一样先列大纲，本篇将从下面几个地方温故多线程之间的通讯问题：**

1.**为什么要进行多线程间的通讯**

2.**什么是多线程之间的通讯**

3.**多线程间如何实现通讯**

4.**关于并发包里的Lock锁**

5.**如何停止线程**

6.**关于ThreadLoca**

### 1.为什么要进行多线程间的通讯 ###

当我们开启**多个线程**来**共同完成一件任务**的时候，CPU是随机切换线程来执行的，如果我们希望它们**有规律**的执行，那么就需要多个线程之间进行**协调通信**，通过这个方式来实现**多个线程共同操作一份数据。**

虽然我们再不使用多线程通讯的时候也可以共同操作一份数据，但是很容易就造成多线程对同一共享变量的争夺，会引发很多问题，造成损失，因此使用多线程通信就很有必要性。

### 2.什么是多线程之间的通讯 ###

**多个线程共同完成一件任务时，对共享资源做不同操作，通过互相协调通信，从而高效准确的完成工作任务。**

下面分析下的多线程间通讯案例：我们有个需求，偶数打印：小明，数学课代表 奇数打印：小花，语文课代表。代码实现，我们可能会这么写:
第一步：定义共同资源UserRes,第二步定义一个写线程和一个读线程

共同资源类

    class UserRes{
	public String type;
	public String userName;
	}

写线程

    class WriteThrad extends Thread {
	private UserRes userRes;

	public WriteThrad(UserRes userRes) {
		this.userRes = userRes;
	}

	@Override
	public void run() {
		int count = 0;
		while (true) {
				if (count == 0) {
					userRes.userName = "小明";
					userRes.type = "数学课代表";
				} else {
					userRes.userName = "小花";
					userRes.type = "语文课代表";
				}
				count = (count + 1) % 2;
			}
	}
	}

读的线程

    class ReadThread extends Thread {
	private UserRes userRes;

	public ReadThread(UserRes userRes) {
		this.userRes = userRes;
	}

	@Override
	public void run() {
		while (true) {
				System.out.println(userRes.userName + "," + userRes.type);
		}
	}
	}

运行一下,思考下打印结果

    UserRes res = new UserRes();
	WriteThrad mWriteThrad = new WriteThrad(res);
	ReadThread mReadThread = new ReadThread(res);
	mWriteThrad.start();
	mReadThread.start();

比对下以下结果和我们期待的一样么

    小明，数学课代表
	小花，语文课代表
	小明，数学课代表
	小花，语文课代表
	小花，语文课代表
	小花，语文课代表
	小明，数学课代表
	小明，数学课代表
	小明，数学课代表
	小明，语文课代表

以上数据是打印出来的部分结果,是不是发现了有错乱数据，我们明明写入的是小花，语文课代表，结果却出现了小花是数学课代表的情况，发生了上篇文章讲到的线程安全问题，如何解决呢？对的，加对象锁；注意这里不能使用this锁，因为我们需要用同一把锁，而this代表的一个是WriteThrad，一个是ReadThread，这里我们需要保障加锁的时候是同一把锁，直接使用UserRes这个对象吧。

加锁后的代码：

    class WriteThrad extends Thread {
	private UserRes userRes;

	public WriteThrad(UserRes userRes) {
		this.userRes = userRes;
	}

	@Override
	public void run() {
		int count = 0;
		
		while (true) {
		synchronized (userRes) {
				if (count == 0) {
					userRes.userName = "小明";
					userRes.type = "数学课代表";
				} else {
					userRes.userName = "小花";
					userRes.type = "语文课代表";
				}
				count = (count + 1) % 2;
			}
		}
		}
	}
	
	class ReadThread extends Thread {
	private UserRes userRes;

	public ReadThread(UserRes userRes) {
		this.userRes = userRes;
	}

	@Override
	public void run() {
		while (true) {
		synchronized (userRes) {
				System.out.println(userRes.userName + "," + userRes.type);
				}
		}
	}
	}
打印结果：

    小明，数学课代表
    小明，数学课代表
	小明，数学课代表
	小明，数学课代表
	小明，数学课代表
	小明，数学课代表
	小明，数学课代表
	小花，语文课代表
	小花，语文课代表
	小花，语文课代表
	小花，语文课代表
	小花，语文课代表
	小花，语文课代表
	小明，数学课代表
	小明，数学课代表
	小明，数学课代表
	小明，数学课代表
	小花，语文课代表
	小花，语文课代表
	小花，语文课代表
那这次数据数据没有错乱问题，但是观察发现有重复读取的现象，如果我们不想要重复读取，需求是这样的：

1.不重复读取

2.写一个就读一个

3.如果没有写入就不读取

4.没有读取完就不写入

这次怎么实现呢？



### 3.如何实现多线程间的通讯 ###
其实，上面的需求，就是经典的生产者消费者案例，就需要使用到，多线程间通讯来协调两个线程进行工作了。下面需要介绍一下多线程通讯用到的几个API：

wait()、notify()、notifyAll()是三个定义在Object类里的方法，可以用来控制线程的状态。
这三个方法最终调用的都是jvm级的native方法。随着jvm运行平台的不同可能有些许差异。

如果对象调用了wait方法就会使持有该对象的线程把该对象的控制权交出去，然后处于等待状态。

如果对象调用了notify方法就会通知某个正在等待这个对象的控制权的线程可以继续运行。

如果对象调用了notifyAll方法就会通知所有等待这个对象控制权的线程继续运行。

**注意:一定要在线程同步中使用,并且是同一个锁的资源**

实现上述需求：

共同资源：

    class UserRes{
	public String type;
	public String userName;
	}
读写线程：

    class WriteThrad extends Thread {
	private UserRes userRes;

	public WriteThrad(UserRes userRes) {
		this.userRes = userRes;
	}

	@Override
	public void run() {
		int count = 0;
		
		while (true) {
		synchronized (userRes) {
				if (userRes.flag) {
					try {
					   // 当前线程变为等待，但是可以释放锁
						res.wait();
					} catch (Exception e) {

					}
				}
				if (count == 0) {
					userRes.userName = "小明";
					userRes.type = "数学课代表";
				} else {
					userRes.userName = "小花";
					userRes.type = "语文课代表";
				}
				count = (count + 1) % 2;
				userRes.flag = true;
				// 唤醒当前线程
				userRes.notify();
			}
		}
		}
	}
	
	class ReadThread extends Thread {
	private UserRes userRes;

	public ReadThread(UserRes userRes) {
		this.userRes = userRes;
	}

	@Override
	public void run() {
		while (true) {
		synchronized (userRes) {
				if (!userRes.flag) {
					try {
						userRes.wait();
					} catch (Exception e) {
						// TODO: handle exception
					}
				}
				System.out.println(userRes.userName + "," + userRes.type);
				userRes.flag = false;
				userRes.notify();
				}
		}
	}
	}
以上就是传统的synchronized+wait+notify方式实现多线程间的协调通讯。

再复习下wait和sleep的区别：

对于sleep()方法，我们首先要知道该方法是属于Thread类中的。而wait()方法，则是属于Object类中的。
sleep()方法导致了程序暂停执行指定的时间，让出cpu该其他线程，但是他的监控状态依然保持者，当指定的时间到了又会自动恢复运行状态。
在调用sleep()方法的过程中，线程不会释放对象锁。
而当调用wait()方法的时候，线程会放弃对象锁，进入等待此对象的等待锁定池，只有针对此对象调用notify()方法后本线程才进入对象锁定池准备
获取对象锁进入运行状态。

### 4.关于并发包里的Lock锁 ###

JDK1.5之后引入Lock 接口提供了与 synchronized 关键字类似的同步功能，但需要在使用时手动获取锁和释放锁。

使用Lock锁：

    Lock lock  = new ReentrantLock();
	lock.lock();
	try{
		//可能会出现线程安全的操作
	}finally{
		//一定在finally中释放锁
		//也不能把获取锁在try中进行，因为有可能在获取锁的时候抛出异常
  		lock.ublock();
	}
Condition使用：

    Condition condition = lock.newCondition();
	res. condition.await();  类似wait
	res. Condition. Signal() 类似notify
使用Lock锁改写后的代码：

        class UserRes{
		public String type;
		public String userName;
		Lock lock = new ReentrantLock();
	}

	class WriteThrad extends Thread {
		rivate UserRes userRes;
		Condition newCondition;//添加Condition

		public WriteThrad(UserRes userRes，Condition newCondition ) {
			this.userRes = userRes;
			this.newCondition=newCondition;
		}

		@Override
		public void run() {
			int count = 0;
		
			while (true) {
				try {
					if (userRes.flag) {
						newCondition.await();
					}
					if (count == 0) {
						userRes.userName = "小明";
						userRes.type = "数学课代表";
					} else {
						userRes.userName = "小花";
						userRes.type = "语文课代表";
					}
					count = (count + 1) % 2;
					userRes.flag = true;
					newCondition.signal();
					} catch (Exception e) {
						// TODO: handle exception
				
					}finally{			
						res.lock.unlock();
					}

				}
		
			}
		}
	
	class ReadThread extends Thread {
		private UserRes userRes;
		Condition newCondition;//添加Condition

		public ReadThread(UserRes userRes，Condition newCondition) {
			this.userRes = userRes;
			this.newCondition=newCondition;
		}

		@Override
		public void run() {
			while (true) {
				res.lock.lock();
				try {
					if (!userRes.flag) {
					newCondition.await();
					}
					System.out.println(userRes.userName + "," + userRes.type);
					userRes.flag = false;
					newCondition.signal();
				} catch (Exception e) {
						// TODO: handle exception	
				}finally{
					res.lock.unlock();
				}
			}
		}


### 5.如何停止线程 ###
停止线程的方式：

a.Thread.stop方法虽然可以实现停止线程，但是不安全，已经废弃，不建议使用

b.interrupt方法，实质上是在当前线程打了一个退出标志，不是真正停止线程

c.借住异常法停止线程

先来了解下Thread的两个方法interrupted和isInterrupted：

方法interrupted用来判断当前线程是否是停止状态，执行后清除当前线程的中断状态，也就是说连续两次调用，第二次可定是false;

**使用interrupted方法：**

    public class Main {

    public static void main(String[] args) {
        Thread thread = new MyThread();
        thread.start();
        try {
            thread.interrupt();
            System.out.println(" 1  is thread stoped?"+thread.interrupted());
            System.out.println(" 2  is thread stoped?"+thread.interrupted());
        } catch (Exception e) {
            System.out.println(" e = "+e.toString());
            e.printStackTrace();
        }
    }
	}

	class MyThread  extends Thread{
    	@Override
    	public void run() {
       		for (int i=0;i<50000;i++){
            	System.out.println("i="+(i+1));
        	}
    	}
	}

打印结果：

     1  is thread stoped?false
	 2  is thread stoped?false
	 i=1
	 i=2
	 i=3
	 i=4
	 i=5
	 i=6
	 i=7
	 ...
	 i=50000
从打印结果看：

1.我们在代码中调用了thread.interrupt()，但是程序还是执行完了，证明了interrupt方法不是真正的中断线程，他只是设置了一个中断标识;

2.我们调用interrupt方法后打印线程状态返回都是false,虽然我们用的是thread对象，但是当前线程是Main,Main线程并未终止，证明interrupted返回的是当前线程是否是停止状态。

**使用isInterrupted方法：**
不清除线程停止状态。

有了上面的基础，我们研究下如何停止线程：

**interrupt方法停止线程**

    public class Main {

    public static void main(String[] args) {
        Thread thread = new MyThread();
        thread.start();
        try {
            thread.sleep(200);
            thread.interrupt();
        } catch (Exception e) {
            System.out.println(" e = "+e.toString());
            e.printStackTrace();
        }
    }
	}

	class MyThread  extends Thread{
    @Override
    public void run() {
        for (int i=0;i<50000;i++){
            if (this.interrupted()){
                System.out.println("线程中止，不在执行");
                break;
            }
            System.out.println("i="+(i+1));
        }
        System.out.println("这是for循环外面的代码");
    	}
	}
打印结果：

	...
    i=18572
	i=18573
	i=18574
	i=18575
	i=18576
	i=18577
	i=18578
	i=18579
	i=18580
	i=18581
	线程中止，不在执行
	这是for循环外面的代码
确实上述方法停止了线程，但是有个问题，for循环的外面还会继续执行，我们的break只是跳出for循环。这里**把break改成return**就可以了。

**异常法停止线程：**

    public class Main {

    public static void main(String[] args) {
        Thread thread = new MyThread();
        thread.start();
        try {
            thread.sleep(200);
            thread.interrupt();
        } catch (Exception e) {
            System.out.println(" e = " + e.toString());
            e.printStackTrace();
        }
    }
	}

	class MyThread extends Thread {
    	@Override
    	public void run() {
        	try {


           		for (int i = 0; i < 50000; i++) {
                	if (this.interrupted()) {
                    	System.out.println("线程中止，不在执行");
                    	throw new InterruptedException();
                	}
                	System.out.println("i=" + (i + 1));
            	}
            	System.out.println("这是for循环外面的代码");
        	} catch (InterruptedException e) {
            	e.printStackTrace();
            	System.out.println("捕获异常");
        	}
    	}
	}
看下打印的日志：

	...
    i=21406
	i=21407
	i=21408
	i=21409
	i=21410
	线程中止，不在执行
	java.lang.InterruptedException
		at MyThread.run(Main.java:25)
	捕获异常
的确按照我们的需求停止了线程。

接下来看一个有意思的事情，如果我们停止了一个正在sleep的线程会怎样？

    public class Main {

    public static void main(String[] args) {
        Thread thread = new MyThread();
        thread.start();
        try {
            thread.interrupt();
        } catch (Exception e) {
            System.out.println(" e = " + e.toString());
            e.printStackTrace();
        }
    }
	}

	class MyThread extends Thread {
    @Override
    public void run() {
        try {
            System.out.println("线程开始。。。");
            Thread.sleep(2000);
            System.out.println("线程结束。。。");
        } catch (InterruptedException e) {
            e.printStackTrace();
            System.out.println("捕获异常,正在sleep中的线程被停止，isInterrupted = "+this.isInterrupted());
        }
    }
	}

打印结果：

    线程开始。。。
	捕获异常,正在sleep中的线程被停止，isInterrupted = false
	java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at MyThread.run(Main.java:20)

	Process finished with exit code 0
会报异常，并且清除停止状态值，使之变为false。

### 6.关于ThreadLoca实现原理###
**ThreadLocal是什么？**
ThreadLocal为每一个线程提供一个局部变量，每一个线程独立改变自己的副本，不会影响其他线程的副本。
**如何使用ThreadLocal？**
第一步：声明一个共享资源对象MyRes：

    public class MyRes {
    public Integer count =0;
    public  ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>(){
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };
    public String getCount(){
        count = threadLocal.get()+1;
        threadLocal.set(count);
        return count+"";
    }
	}
第二步声明Thread:

    class MyThread extends Thread {
    private MyRes myRes;

    public MyThread(MyRes myRes) {
        this.myRes = myRes;
    }

    @Override
    public void run() {
        for (int i = 0;i<5;i++){
            System.out.println(getName()+",count = "+myRes.getCount());
        }
    }
第三步，Main中调用：

    public class Main {

    public static void main(String[] args) {
        MyRes myRes = new MyRes();
        Thread thread1 = new MyThread(myRes);
        Thread thread2 = new MyThread(myRes);
        Thread thread3 = new MyThread(myRes);
        thread1.start();
        thread2.start();
        thread3.start();

    }
	}
看执行结果：

    Thread-2,count = 1
	Thread-2,count = 2
	Thread-2,count = 3
	Thread-2,count = 4
	Thread-2,count = 5
	Thread-1,count = 1
	Thread-1,count = 2
	Thread-1,count = 3
	Thread-1,count = 4
	Thread-1,count = 5
	Thread-0,count = 1
	Thread-0,count = 2
	Thread-0,count = 3
	Thread-0,count = 4
	Thread-0,count = 5
	Process finished with exit code 0
**ThreadLocal的实现原理：**
ThreadLocal的实现原理很简单，通过map集合实现，put("当前线程"，值)；
看下面的图就比较清楚啦：


欢迎关注我的个人公众号，精彩不容错过！


