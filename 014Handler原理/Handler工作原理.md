##Android的消息机制

###1.Handler工作原理

Android的消息机制，主要是值Handler的运行机制，而Handler的运行依赖MessageQueue和Loop的支撑。
MessageQueue是消息队列，用来存储一组消息，以队列的形式对外提供插入和删除工作，内部结构通过单链表实现。

Looper消息循环，以无限循环的形式查询是否是否有新消息，如果有就处理新消息，否则就一会等待。

Message消息载体

Handler创建时会使用当前线程的Looper构建内部的消息循环，主线程自动创建了Looper,通过Handler的send方法发送一个Message消息，send方法会调用MessageQueue的enqueueMessage把这个消息添加到消息丢了中，然后Looper发现新消息到来就会处理这个消息，最终Runnable或者handleMessage方法会被调用，handler中的业务就被切换到创建Handler的线程中执行了。

###2.主线程的消息循环
Android 的主线程是ActivityThread 主线程入口方法为main,在main方法中系统会通过Looper.prepareMainLooper创建主线程的Looper以及MessageQueue,并通过Looper.loop开启主线程的消息循环，在ActivityThread的内部有一个Handler H,它内部定义了一组消息类型，主要包含四大组件的启动和停止过程。

ActivityThread通过ApplicationThread 和AMS进行进程间通讯，AMS以进程间通讯的方式完成AcitivityThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息，调用ActivityThread处理消息，切换到主线程。这个过程就是主线程的消息循环模型。