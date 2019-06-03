# 自定义View(三)RecyclerView源码 #
本文基于android_27，我的Recycler源码位置：
> H:\AndroidSDK\sources\android-27\android\support\v7\widget\RecyclerView.java


我们从源码里看到RecyclerView是继承自ViewGroup的，分析它的源码，那我们就从它的measure、layout、draw入手吧。

onMeasure：重写ViewGroup的onMeasure方法
