# app启动流程 #
点开桌面图标，冷启动调用ActivityThread 的main函数：
的main函数里：

1.Looper.prepareMainLooper()为主线程创建Looper。
2.thread.attach(false) false不是系统应用

attach里获取ActivityManagerNative.getDefault()



# 面向切面编程 #

面向对象各模块独立，一个功能使用了多个模块式，当这个功能需求变更，需要改动多个模块才能实现我们的需求。那么有没有一个很好的，不用改动那么多的解决方案呢？

答案是，有的。这就是我们要了解的AOP，面向切面编程。

什么是AOP呢？

把我们这个模块的功能提出来，与一批对象隔离，这样与一批对象之间降耦合运行，就可以对某个功能进行编程。