# 温故 java基础（一）之多线程编程 #

----------
**俗话说好记性不如lai笔头,子曰：温故而知新可以为师矣~ 言归正传，本文将从以下方面复习Java多线程编程相关的知识**。



-  **多线程入门**
-  **线程安全**
-  **多线程之间的通讯**
-  **并发编程**
-  **线程池**
## 一、多线程入门 ##
1. 线程与进程的区别
2. 为什么使用多线程
3. 如何创建线程
4. 线程的几种状态
5. 守护线程与非守护线程
6. 线程常用API
 
### 1.线程与进程的区别 ###
进程：个人理解进程是指正在运行的一个应用程序；定义为：具有一定独立功能的程序关于某个数据集合上的一次运行活动，进程是系统进行资源分配和调度的独立单位。

线程：是进程的一个实体，是CPU调度和分派的基本单位，比进程更小的能独立运行的基本单位；或者说线程是进程的一条执行路径。

区别：

a. 根本区别：进程是资源分配的基本单位，线程是程序执行的基本单位。

b. 主要差别：它们是操作系统不同的资源管理方式。系统会为每个进程分配不同的内存空间，进程间切换会有比较大的开销；同一进程内的线程共享进程的内存资源，除了CPU外，系统不会给线程分配内存，每个线程有自己独立的运行栈和程序计数器，线程间切换开销小。
### 2.为什么使用多线程 ###
使多	CPU系统更加高效，提高工作效率。例如多线程上传或下载资源。

### 3.如何创建线程 ###
a.继承Thread类，重写run方法。

创建CreateThread类：

    public class CreateThread extends Thread {
	
	public void run() {
		for (inti = 0; i< 10; i++) {
			System.out.println("i:" + i);
			}
		}
	}
开启线程方式：

    public static void main(String[] args) {

		CreateThread createThread = new CreateThread();
		createThread.start();
		
	}

b.实现Runnable接口，重写run方法（推荐）

创建CreateRunnable类：
    
    public class CreateRunnable implements Runnable {

	@Override
	public void run() {
		for (inti = 0; i< 10; i++) {
			System.out.println("i:" + i);
			}
		}

	}
	
开启线程：

    public static void main(String[] args) {
		CreateRunnable createThread = new CreateRunnable();
		Thread createThread = new Thread(createThread);
		createThread.start();
		
	}
    
    
    

c.使用匿名内部类

    Thread thread = new Thread(new Runnable() {
			public void run() {
				for (int i = 0; i< 10; i++) {
					System.out.println("i:" + i);
				}
			}
		});
	thread.start();


为什么推荐第二种，因为Java的单一继承，多实现原则。第三种不方便管理，容易造成资源浪费。
**注意：别忘调用Thread类的start()方法开启线程**
### 4.线程的几种状态 ###


### 5.守护线程与非守护线程 ###

### 6.线程常用API ###


