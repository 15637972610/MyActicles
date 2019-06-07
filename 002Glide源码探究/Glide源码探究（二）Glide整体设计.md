
本文参考前辈们的文章，结合源码，从以下角度继续探究Glide的源码设计。个人觉得如果先有一个整体的认识，然后再深入探讨是比较舒服的。本文从以下角度继续探究Glide：

- Glide整体框架
- Glide模块间的调用流程
- Glide库 目录结构
- Glide的类之间的关系

## 1.Glide整体框架 ##
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190605095246944.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NjI4MzI5,size_16,color_FFFFFF,t_70)
结合上面的图，我们分两步探究这个库的工作原理，
第一步：当我们使用这个Glide库加载图片时，它内部先初始化Glide以及它需要用到的相关组件，关联生命周期，构建请求RequestBuilder，初始化Engine；
第二步：Engine获取数据，具体步骤是先从缓存拿数据（Memory和Active都是内存缓存），拿到了就返回到MainThread；拿不到就调度一个DecodeJob任务从数据源获取数据，拿到数据后通过DecodeJob处理数据，缓存起来然后回调到MainThread。
第三步：回到主线程给View加载对应的资源
## 2.Glide模块间的调用流程 ##
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190605095558688.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NjI4MzI5,size_16,color_FFFFFF,t_70)

## 3.Glide库 目录结构 ##
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190605100636721.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NjI4MzI5,size_16,color_FFFFFF,t_70)
## 4.Glide的类之间的关系 ##
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190605095633873.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NjI4MzI5,size_16,color_FFFFFF,t_70)
这张图反映了类之间的实现，继承，组合关系，其中实心箭头代表组合，空心箭头代表实现implements ,空心圆形代表接口。组合不了解的可以搜一下组合模式。

结合3和4可以看到整个库分为 RequestManager(请求管理器)，Engine(数据获取引擎)、 Fetcher(数据获取器)、MemoryCache(内存缓存)、DiskLRUCache（磁盘缓存）、Transformation(图片处理)、Registry(图片类型及解析器配置)、Target(目标) 等模块

通过这篇呢对Glide的整体设计有了一个初步的了解，接下来结合源码分析每一个模块是怎么工作的
