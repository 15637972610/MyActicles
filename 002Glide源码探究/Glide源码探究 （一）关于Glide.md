
参考网上几篇Glide源码的介绍，也想看看Glide内部怎么实现的，顺便膜拜下Google大佬们的设计思想，话不多说，来吧~

本篇不介绍Glide的具体使用，后面的操作符也只代表glide的常用功能，详细使用可以参考官方文档。本文主要介绍了以下几个方面：

- glide是什么，它能干什么，为什么是它？
- 和之前的图片加载框架的比较
- Glide的操作符

## 1.glide是什么，它能干什么，为什么是它？ ##

官方对glide的介绍:[Glide官方文档地址](Glide官方文档地址 "https://muyangmin.github.io/glide-docs-cn/")

> Glide是一个快速高效的Android图片加载库，注重于平滑的滚动。Glide提供了易用的API，高性能、可扩展的图片解码管道（decode pipeline），以及自动的资源池技术。
> 
> Glide 支持拉取，解码和展示视频快照，图片，和GIF动画。Glide的Api是如此的灵活，开发者甚至可以插入和替换成自己喜爱的任何网络栈。默认情况下，Glide使用的是一个定制化的基于HttpUrlConnection的栈，但同时也提供了与Google Volley和Square OkHttp快速集成的工具库。
> 
> 虽然Glide 的主要目标是让任何形式的图片列表的滚动尽可能地变得更快、更平滑，但实际上，Glide几乎能满足你对远程图片的拉取/缩放/显示的一切需求。

嗯，通俗易懂，总的来说它是一个开源的图片加载库，能加载视频快照，图片，和GIF动画，使用的时候API简洁，加载图片功能强大，性能牛批！！！

> Glide 充分考虑了Android图片加载性能的两个关键方面：
> 
> - 图片解码速度
> - 解码图片带来的资源压力
> 为了让用户拥有良好的App使用体验，图片不仅要快速加载，而且还不能因为过多的主线程I/O或频繁的垃圾回收导致页面的闪烁和抖动现象。
> 
> Glide使用了多个步骤来确保在Android上加载图片尽可能的快速和平滑：
> 
> - 自动、智能地下采样(downsampling)和缓存(caching)，以最小化存储开销和解码次数；
> - 积极的资源重用，例如字节数组和Bitmap，以最小化昂贵的垃圾回收和堆碎片影响；
> - 深度的生命周期集成，以确保仅优先处理活跃的Fragment和Activity的请求，并有利于应用在必要时释放资源以避免在后台时被杀掉。

## 2.和之前的图片加载框架的比较 ##

- ImagerLoader

是早期用于加载图片的使用比较广泛的框架，其中 Cache 分为 MemoryCache 和 DiskCache 两部分

默认实现多种内存缓存算法 这几个图片缓存都可以配置缓存算法，不过 ImageLoader 默认实现了较多缓存算法，如 Size 最大先删除、使用最少先删除、最近最少使用、先进先删除、时间最长先删除等

- Glide与Picasso

总的来说二者极为相似，有着近乎相同的 API 风格，但 Glide 在缓存策略和加载 gif 方面略胜一筹

1. 两者使用方式类似，但Glide的with()接受的不仅仅是Context，还可以是Activity或是Fragment，Context会自动的从他们获取。同时将Activity/Fragment作为with()参数的好处是：图片加载会和Activity/Fragment的生命周期保持一致，比如Paused状态在暂停加载，在Resumed的时候又自动重新加载。所以我建议传参的时候传递Activity 和 Fragment给Glide，而不是Context。
2. Glide加载的图片质量要略差于Picasso，这又是为什么呢？这是因为Glide默认的Bitmap格式是RGB_565，比ARGB_8888格式的内存开销要小一半。Glide当然也可以通过GlideModule设置格式。
3. 两者在磁盘缓存策略上有很大的不同。Picasso缓存的是全尺寸的，而Glide缓存的是跟ImageView尺寸相同的。Glide的这种方式优点是加载显示非常快。而Picasso的方式则因为需要在显示之前重新调整大小而导致一些延迟
4. Glide可以加载GIF动态图，而Picasso不能
5. Picasso (v2.5.1)大小约为118KB，然而Glide (v3.5.2)的大小约为430KB。Picasso的方法数大约480，然而Glide的方法数约2678

Glide 的速度比 Picasso 更快，Glide 的长处是处理大型的图片流，如 gif、video，如果要制作视频类应用，Glide 当为首选




## 3.Glide的操作符 ##
1. 占位符(Placeholder)

    通常设置默认加载图片

2. 错误符(Error)

	设置请求永久性失败时展示的图片
3. 后备回调符(Fallback）

	主要目的是允许用户指示 null 是否为可接受的正常情况

问：占位符是异步加载的吗？

> No。占位符是在主线程从Android Resources加载的。我们通常希望占位符比较小且容易被系统资源缓存机制缓存起来

问：变换是否会被应用到占位符上？

> No。Transformation仅被应用于被请求的资源，而不会对任何占位符使用。可以考虑自定义一个View来剪裁(clip)你的占位符，而达到你想要的变换效果

问：在多个不同的View上使用相同的Drawable可行么？

> 通常可以，但不是绝对的。任何无状态(non-stateful)的 Drawable（例如 BitmapDrawable ）通常都是ok的。但是有状态的 Drawable 不一样，在同一时间多个 View 上展示它们通常不是很安全，因为多个View会立刻修改(mutate) Drawable 。对于有状态的 Drawable ，建议传入一个资源ID，或者使用 newDrawable() 来给每个请求传入一个新的拷贝。

4.request options 可以实现（包括但不限于）：具体用法见官方文档

- 占位符(Placeholders)
- 转换(Transformations)
- 缓存策略(Caching Strategies)
- 组件特有的设置项，例如编码质量，或Bitmap的解码配置等。

5.使用 RequestBuilder 可以指定：

- 你想加载的资源类型(Bitmap, Drawable, 或其他)
- 你要加载的资源地址(url/model)
- 你想最终加载到的View
- 任何你想应用的（一个或多个）RequestOption 对象
- 任何你想应用的（一个或多个）TransitionOption 对象
- 任何你想加载的缩略图 thumbnail()

6.TransitionOption 可以应用以下变换：

- View淡入
- 与占位符交叉淡入
- 或者什么都不发生

7.缩略图 (Thumbnail) 请求

允许你指定一个 RequestBuilder 以与你的主请求并行启动

8.在失败时开始新的请求error

9.Option 类是给Glide的组件添加参数的通用办法

Option 通过 RequestOptions 类应用到请求上：

10.transform关于变换，通常变换操作是用来完成剪裁或对位图应用过滤器

默认情况下，每个 transform() 调用，或任何特定转换方法(fitCenter(), centerCrop(), bitmapTransform() etc)的调用都会替换掉之前的变换

11.多重变换

    GlideApp.with(fragment)
  	.load(url)
  	.transforms(new FitCenter(), new    YourCustomTransformation())
  	.into(imageView);
注意，你向 MultiTransformation 的构造器传入变换参数的顺序，决定了这些变换的应用顺序

12.定制变换，可以实现你自己的 Transformation

13.关于Target是介于请求和请求者之间的中介者的角色

14.指定目标

into(Target) 方法不仅仅用于启动每个请求，它同时也指定了接收请求结果的 Target

15.取消和重用

 使用into(Target) 和 into(ImageView) 都返回了一个 Target 实例。如果你重用这个 Target 来在将来开始一个新的加载，则之前开始的任何请求都会被取消，它们使用的资源将被释放：

16.清理

Glide.with(fragment).clear(imageView);

17.尺寸 (Sizes and dimensions)

Glide 通过 getSize 方法提供的尺寸来作为请求的目标尺寸,有利于选取合适的 URL，下采样，裁剪和变换合适的图片以减少内存占用，并确保加载尽可能快地完成。

18.ViewTarget

ViewTarget 通过检查 View 的属性，使用一个 OnPreDrawListener 在 View 绘制之前直接测量尺寸。因此， Glide 可以自动调整大部分图片以匹配目标 View。加载更小的图片可使 Glide 更快地完成加载 (在缓存到磁盘以后)，并使用更少的内存，在图片尺寸一致时还可以增加 Glide 的 BitmapPool 的命中率

19.动画资源和定制目标

Glide的大部分 ViewTarget 实现已经为您处理了 Drawable 动画。如果你确实必须使用定制的 ViewTarget，请确保继承自 ViewTarget 或在新请求开始之前和展示资源结束之后严格地清理从 into(Target) 返回的 Target。

如果你并非往 View 中加载图片，而直接使用 ViewTarget 或使用了定制的 Target 比如 SimpleTarget 且你正在加载一个动画资源例如 GifDrawable，你需要确保在 onResourceReady 中调用 start() 来启动这个动画

如果你加载的是 Bitmap 或 GifDrawable，你可以判断这个可绘制对象是否实现了 Animatable

20.Transitions(直译为”过渡”) 允许你定义 Glide 如何从占位符到新加载的图片，或从缩略图到全尺寸图像过渡。

....

总结：本篇重在介绍glide是什么，能干什么，以及与别的加载库相比较的优点，以及推荐快速阅读理解Glide内部原理的参考文章，不是使用教程。

参考文章：
 
> 几种图片加载框架对比
> 
> https://www.cnblogs.com/linghu-java/p/5741358.html
> https://www.jianshu.com/p/ca5ce4444c37
> https://inthecheesefactory.com/blog/get-to-know-glide-recommended-by-google/en
> 
> Glide源码解析：http://frodoking.github.io/2015/10/10/android-glide/
> https://juejin.im/entry/586766331b69e60063d889ea
> 
> Glide操作符：
> https://muyangmin.github.io/glide-docs-cn/
