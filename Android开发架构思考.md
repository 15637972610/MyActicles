# Android开发架构思考 #


**三层架构模型：**

三层架构是一个分层式的软件体系架构设计，它可适用于任何一个项目。通常意义上的三层架构就是将整个业务应用划分为：界面层（User Interface layer）、业务逻辑层（Business Logic Layer）、数据访问层（Data access layer）。区分层次的目的即为了“高内聚低耦合”的思想。在软件体系架构设计中，分层式结构是最常见，也是最重要的一种结构。微软推荐的分层式结构一般分为三层，从下至上分别为：数据访问层、业务逻辑层（又或称为领域层）、表示层


**三层架构目的：**

通过设计使程序模块化，做到模块内部的高聚合和模块之间的低耦合。这样做的好处是使得程序在开发的过程中，开发人员只需要专注于一点，提高程序开发的效率，并且更容易进行后续的测试以及定位问题。但设计不能违背目的，对于不同量级的工程，具体架构的实现方式必然是不同的，切忌犯为了设计而设计，为了架构而架构的毛病。

举个简单的例子：

一个Android App如果只有3个Java文件，那只需要做点模块和层次的划分就可以，引入框架或者架构反而提高了工作量，降低了生产力；

但如果当前开发的App最终代码量在10W行以上，本地需要进行复杂操作，同时也需要考虑到与其余的Android开发者以及后台开发人员之间的同步配合，那就需要在架构上进行一些思考！


**MVC:**


- M ：数据模型 
- V ：视图层 
- C ：Controller层


优点：使项目有了很好的扩展性，便于维护。

使用场景：页面较多，项目较大

缺点：
在Android开发过程中采用这种模式的不足之处是Controller层由于要处理View展示，不过更多情况下在实际应用开发中Activity不能够完全充当Controller，而是Controller和View的合体。于是Activity既要负责视图的显示，又要负责对业务逻辑的处理。这样在Activity中代码达到上千行，甚至几千行都不足为其，同时这样的Activity也显得臃肿不堪;后期需求变更需要大量改动，导致BUG较多，版本不可控。


**MVP:**

- M:对于Model层也是数据层

它区别于MVC架构中的Model，在这里不仅仅只是数据模型。在MVP架构中Model它负责对数据的存取操作，例如对数据库的读写，网络的数据的请求等


- P：对于Presenter层他是连接View层与Model层的桥梁并对业务逻辑进行处理。

在MVP架构中Model与View无法直接进行交互。所以在Presenter层它会从Model层获得所需要的数据，进行一些适当的处理后交由View层进行显示。这样通过Presenter将View与Model进行隔离，使得View和Model之间不存在耦合，同时也将业务逻辑从View中抽离。

- V:视图层

在View层中只负责对数据的展示，提供友好的界面与用户进行交互。在Android开发中通常将Activity或者Fragment作为View层。

实现见demo:

[https://github.com/15637972610/mvp_better](https://github.com/15637972610/mvp_better "mvp_better")

问题：

1.View即我们的Activity,在P中持有Activity的引用可能存在内存泄漏问题。加载数据完成，通过V的方法更新界面的时候。可通过软引用解决。抽取BasePresenter类 封装attach 和dettach方法

2.如果项目比较庞大会导致类比较繁多

MVP与MVC的区别：他们之间最大区别就是MVC中View层和Model层直接交互，而在MVP中他们是通过Persenter进行交互，不直接交互。

**MVVM:**

MVP中我们说过随着业务逻辑的增加，UI的改变多的情况下，会有非常多的跟UI相关的case，这样就会造成View的接口会很庞大。而MVVM就解决了这个问题，通过双向绑定的机制，实现数据和UI内容，只要想改其中一方，另一方都能够及时更新的一种设计理念，这样就省去了很多在View层中写很多case的情况，只需要改变数据就行。


参考：

[https://blog.csdn.net/ljd2038/article/details/51477475](https://blog.csdn.net/ljd2038/article/details/51477475 "https://blog.csdn.net/ljd2038/article/details/51477475")
[https://www.tianmaying.com/tutorial/AndroidMVC](https://www.tianmaying.com/tutorial/AndroidMVC "https://www.tianmaying.com/tutorial/AndroidMVC")

[https://juejin.im/post/5b3a3a44f265da630e27a7e6](https://juejin.im/post/5b3a3a44f265da630e27a7e6 "https://juejin.im/post/5b3a3a44f265da630e27a7e6")