
先上一张Android平台架构图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190531211832310.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NjI4MzI5,size_16,color_FFFFFF,t_70)
[Android Developers](https://developer.android.com/guide/platform?hl=zh-cn)
## 1.Android系统的启动 ##
名词介绍：

①APP service：也就是我们APP内的普通应用服务

②Core Service:系统核心服务，，开机的时候会启动几十个（40）核心服务，开机完成全部核心服务也就启动完成。

这些核心服务就是我们常见的各种×××ManagerService.java

![系统开机过程](https://img-blog.csdnimg.cn/20190531211342184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NjI4MzI5,size_16,color_FFFFFF,t_70)
**开机过程分析：**

用户点击开机键，Android系统先检测LinuxKernel+Drivers+Hw 内核和驱动程序是否准备好，如果准备好了，进入系统的第一个进程Init进程，init进程读取配置文件configration,然后创建Runtime 进程，zygote进程，启动一个VM ，Runtime 进程启动ServiceManager进程；zygote进程启动一个SystemServer进程，SystemServer里拥有大部分核心服务执行（Java 和 Native Service两种）。具体有多少呢，都在init进程的init.rc配置文件中;init进程依照配置文件启动SurfaceFlinger,MediaServer服务（AudioFlinger、MediaPlayerService、CameraService）至此系统启动起来。


用户启动一个APP 是通过AMS启动的，它会通过Socket请求Zygote来fork(复制，繁殖)一个进程给这个即将要启动的App，App进程和系统服务进程是不一样的他们之间通信需要跨进程。所以系统服务会提供IBinder接口供给App使用。

我们的app在使用的时候，通过IBinder接口和Java写的核心服务通信，而这些Java核心服务通过JNI的方式调用驱动程序，从而使用我们的硬件服务。
NativeService是实现于Runtime层里的Server,在系统服务的开发上，既可以写成JavaService再通过JNI与驱动HAL通信，也可以让App直接通过JNI与AndroidService通信。


关于核心服务的启动，开机时加载LinuxKernel部分，进行内核空间的初始化，加载硬件驱动，启动了Linux系统，岁后切换至用户空间，创建Init进程，都去init.rc文档，根据配置文件启动NativeService,再启动AndroidService 完成核心服务启动。

系统核心服务诞生后，会请求ServiceManger将该对象参考值加入到BinderDriver里。

###1.1.常见的AndroidService ###

ActivityManager是ActivityManagerService的包装类（活动管理器）：应用程序的生命周期，以及Activity之间的互动

WindowManager(窗口管理器)：屏幕窗口管理服务

ResourceManager(资源管理器)：管理如字符串，图片，layout等。

LocationManager:(位置管理器)：提供位置服务

TelephonyManager(电话管理器)：手机通话有关的服务

###1.2常见的NativeService ###

用C、C++写的服务

ServiceManager(服务管理器)：协助登录与绑定系统服务

MediaPlayerService():

Zygote:Android Java层的孵化器，fork子进程给APP使用

CameraService(摄像服务):调用摄像头，提供摄像相关服务

## 2.APP的启动过程 ##

**第一步：**
当用户点击一个桌面图标的时候，LuncherActivity接收到点击事件，然后会调用startActivity(intent)方法，通过IPC机制最终调用ActivityManagerService方法，这个调用过程先不说，等下会结合AIDL跨进程通讯说一下，这个ActivityService里做了如下操作：

①通过PackageManager的resolveIntent收集对象的指向信息，包含我们在Manifest.xml里配置的Interfilter 信息

②通过grantUriPermissionLocked方法验证用户是否有足够权限调用该Intent对象指向的Activity

③检查并在新的Task中启动目标activity

④检查这个进程的pocessRecord是否存在，如果是null通过Socket通道传递参数给Zygote进程

**第二步：**
Zygote进程调用main方法实例化一个ActivityThread对象，创建一个VM，通过ActivityThread对象的bindApplication绑定新进程和Application,该方法会发送一个BIND_APPLICATION的消息到消息队列，最终通过handleBindApplication方法处理消息，然后调用makeApplication方法加载App的classes文件到内存中。

以上参考自：[应用启动分析](应用启动分析 "https://www.jianshu.com/p/a5532ecc8377")

**从源码角度分析一下：**

① 从源码看Activity的启动过程呢，startActivity(Intent)最终会调用Activity.startActivityForResult方法；

② 而startActivityForResult方法会调用Instrumentation的execStartActivity,在这个方法里会调用ActivityManagerNative.getDefult.startActivity方法；

先看一下ActivityManagerNative.getDefult返回的是一个IActivityManager类型，我们看源码知道ActivityManagerService即AMS继承自ActivityManagerNative,而ActivityManagerNative继承自Binder并实现了IActivityManager,因此ActivityManagerNative.getDefult返回的是一个IActivityManager类型的Binder,他的具体实现是AMS，get方法会返回一个单例的AMS.

③ 因此启动Activity的任务交给了AMS.startActivity;

④AMS的startActivity方法里把任务交给了ActivityStackSupervisor的startActivityMayWait

⑤ActivityStackSupervisor通过一系列的方法调用，把启动Activity事件交给了ActivityStack,ActivityStack最终又会把任务交给ActivityStackSupervisor.realStartActivityLocked

⑥ActivityStackSupervisor.realStartActivityLocked方法会调用app.Thread的ScheduleLaunchActivity方法把事件交给了ApplicationThread，最终把事件交给PerformLaunchActivity方法
**PerformLaunchActivity方法主要工作：**

1. 从ActivityRecord中获取待启动的Activity的组件信息
2. Activity实例的创建是通过Instrumentation的newActivity方法，使用类加载器ClassLoader创建Activity对象
3. 通过LoadedApk的makeApplication方法尝试创建Application对象，如果已存在不会重复创建，其实Application方法也是通过Instrumentaion使用类加载器创建的，创建完成会通过callApplicationOnCreate方法调用Application的onCreate方法
4. 创建ContexImpl对象，并通过activity的attach方法初始化一些重要参数，比如关联上下文ContexImpl，创建Window并建立自己和Window的关联等
5. 调用Activity的onCreate方法。

