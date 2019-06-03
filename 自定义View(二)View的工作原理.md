# 自定义View(二) #

本章主要讲下View的工作原理，以及实现一个自定义View。


要想弄明白View的工作原理需要从三大流程入手，即测量流程（Mesure）、布局流程（layout）、绘制流程(draw),除此之外View的常见回调也需要掌握，如构造，onAttach、onVisibilityChanged、onDetach,还有一些需要处理的滑动等。

本篇将从以下几点来复习：

1. View的测量流程
2. View的布局流程
3. View的绘制流程
4. 自定义View实现

## 1. View的测量流程 ##

回顾测量流程之前，先了解一下MeasureSpec,翻译出来大概是测量规格的意思，在测量过程中，系统会将View的LayoutParams根据父容器所施加的规则

> （父容器是精确大小<ps:match_parent和固定大小的值都是精确大小  对应规则是exactly;父容器是wrap_content对应规则是 at_most；还有一种unspecified 父容器不对view有任何限制，要多大给多大，这个规则一般用于系统内部，表示一种测量状态）

转换成对应的MeasureSpace。

MeasureSpec代表一个32位int值，高两位代表mode,低30位代表size，通过调用makeMeasureSpace方法将 mode和size 合成MeasureSpace。

### 1.1系统是怎么通过父类的规格将View的LayoutParams转换成对应的MeasureSpace的 ###

看下源码：

1.measureChildWithMargins方法：

        protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {

        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
2.getChildMeasureSpec方法：

        public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {

        case MeasureSpec.EXACTLY://父容器有固定值或者设置了match_parent
            if (childDimension >= 0) {//子View固定值的情况，因为Match_parent和wrap_content的值都小于0
                resultSize = childDimension;//子View设置多大就多大，值可以确定
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {

                resultSize = size;//子View等于父的大小，值可以确定
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {

                resultSize = size;//这时size是子View的最大上线，值不确定
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        case MeasureSpec.AT_MOST://父设置了wrap_content
            if (childDimension >= 0) {//子View固定值的情况
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;//子View等于父的大小，值不确定
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;子View等于父的大小，值不确定
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        case MeasureSpec.UNSPECIFIED://一般多次测量的时候调用
            if (childDimension >= 0) {//子View固定值的情况，值可以确定
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //将size和mode 通过makeMeasureSpec合成MeasureSpace
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }

### 1.2.View的Measure过程  ###
由上面我们知道了怎么确定View的测量规格，现在就看一下测量的过程吧。View的测量过程由ViewGroup传递而来。measure方法是final类型不可重写，会调用OnMeasure(),在OnMeasure里通过调用setMeasureDimension设置View的宽高，下面说一下View宽高的来历

      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

调用了getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec)；widthMeasureSpec是测量的宽的规格，调用了getDefaultSize：

        public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);//测量值

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST://如果测量完的VIEW是这两种模式的话
        case MeasureSpec.EXACTLY:
            result = specSize;
			//大小就是测量大小，如果父View和子View都设置wrap_content,那这里子View
			//的大小就是父View的大小，相当于match_parent,所以在onMeasure里处理宽高
			//modle为AT_MOST的情况，处理方法见后面附录。
            break;
        }
        return result;
    }



看下getSuggestedMinimumWidth 返回的Size是什么

    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
这里判断了View如果没设置背景返回布局文件设置的mMinWidth，不设置就是0，有背景的话返回mBackground.getMinimumWidth()又是什么呢？返回的是drawable的原始宽度，前提是这个drawable有宽度，否则返回0；

总结下getSuggestedMinimumWidth方法会先判断View是否有背景,如果没有，返回minWidth指定的值，可以是0；如果有则返回minwidth和背景的最小宽度中的较大者。getSuggestedMinimumWidth是 View测量模式为UNSPECIFIED的测量宽高。


总的来说View 测量过程是父类先通过getChildMeasureSpace方法获得子View测量规格，然后通过子View的measure(widMeasureSpace,heightMeasureSpace)传递给子View,接着调用子View的onMeasure方法，通过setMeasuredDimension方法设置View测量后的大小。

附录：自定义View继承自View需要处理AT_MOST的情况，如下：

    /**
     * 两个参数是从父类传过来的，由父类测量完以后生成
     * @param widthMeasureSpec 当前View宽的测量规格
     * @param heightMeasureSpec 当前View高的测量规格
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int mWidMode = MeasureSpec.getMode(widthMeasureSpec);
        int mWidSize = MeasureSpec.getSize(widthMeasureSpec);
        int mHeightMode = MeasureSpec.getMode(heightMeasureSpec);
        int mHeightSize = MeasureSpec.getSize(heightMeasureSpec);
        if (mHeightMode==MeasureSpec.AT_MOST && mWidMode == MeasureSpec.AT_MOST){
            setMeasuredDimension(mWidSize,mHeightSize);
        }else if (mHeightMode==MeasureSpec.AT_MOST){
            setMeasuredDimension(widthMeasureSpec,mHeightSize);
        }else if (mWidMode==MeasureSpec.AT_MOST){
            setMeasuredDimension(mWidSize,heightMeasureSpec);
        }

    }
    
### 1.3.ViewGroup的Measure过程 ###
ViewGroup除了测量自身外，还要遍历子View测量他们，它是一个抽象类，所以它没有重写OnMeasure,提供一个measureChildren方法。

    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

直接把测量交给子View,ViewGroup自己没实现OnMeasure,他是一个抽象类，具体过程交给子类去实现。
## 2. View的layout流程 ##
layout 用来确定View本身的位置，onLayout 方法会遍历调用所有字View的layout。
layout方法首先会调用setFrame方法确定四个顶点的位置，接着调用onLayout,这个方法是父类用来确定子View的位置，onLayout的实现和布局有关，所以在View和ViewGroup均没有具体实现。



## 3. View的draw流程 ##
1.draw背景            canvas

2.draw自己            onDraw

3.draw children      dispatchDraw

4.draw装饰            onDrawScrollBar

