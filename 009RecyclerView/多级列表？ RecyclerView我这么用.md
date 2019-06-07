# 多级列表？ RecyclerView我这么用 #

源码是最好的老师，RecyclerView有多好用我就不多说，来上一个开源库自己去体会：

[推荐一个封装RecyclerView的开源库地址](推荐一个封装RecyclerView的开源库地址 "https://github.com/jdsjlzx/LRecyclerView")

本篇从以下角度介绍一下RecyclerView:

1. 回答个面试题RecyclerView和ListView的区别？
2. 简述RecyclerView使用步骤
3. 如何用RecyclerView 实现多级列表？

## 1.RecyclerView和ListView的区别  ##
① 首先我们知道RecyclerView 被用来替换ListView的，它高度解耦；

② 缓存上的异同

[RecyclerView和ListView的性能对比](RecyclerView和ListView的性能对比 "https://zhuanlan.zhihu.com/p/23339185")

③ 与ListView相比，RecyclerView可以通过设置不同的LayoutManager实现横向滑动列表，网格列表，瀑布流等效果。

④ RecyclerView高度自定义，扩展方便，使用灵活；

⑤ RecyclerView需要自己去实现Item的点击事件，长按时间，以及ItemDecoration 画Item的装饰；


## 2.简述RecyclerView使用步骤 ##
由于网上关于RecyclerView怎么使用已经有很多文章了，不再赘述，简述回忆一下使用步骤：


- gradle添加依赖，布局文件引用RecyclerView
- 创建Adapter集成RecyclerView.Adapter
- UI页面拿到RecyclerView的实例，拿到Adapter实例
- 给RecyclerView设置LayoutManager,设置Adapter


## 3. 如何用RecyclerView 实现多级列表？ ##

这是今天的重点，如何实现多级列表？相信很多小伙伴都遇见过这样的需求，篇就就是在做上个项目中用到的一种方案，来一起了解一下吧。

分析：多级列表，我们肯定是通过RecyclerView这样的可以实现列表效果的View来展示的，那么我们不妨先先想一下他的数据源是什么样的呢？以一个三级列表为例，这里我们采用的是一节列表嵌套二级列表，二级列表嵌套三级列表 这种形式。
并把每级列表抽象成一个对象，用type来区分他们。我们来看代码

	AbstractExpandableItem类抽取了各级了列表共有属性，各级列表是它的子类：
	
	    	package com.dkp.viewdemo.expandablelist;
	
	
	import java.util.ArrayList;
	import java.util.List;
	
	/**
	 * <p>A helper to implement expandable item.</p>
	 * <p>if you don't want to extent a class, you can also implement the interface IExpandable</p>
	 */
	public abstract class AbstractExpandableItem<T> implements IExpandable<T> {
	    protected boolean mExpandable = false;//是否展开状态，默认是折叠状态
	    protected List<T> mSubItems; //下级列表的数据源
	
	    @Override
	    public boolean isExpanded() {
	        return mExpandable;
	    }
	
	    @Override
	    public void setExpanded(boolean expanded) {
	        mExpandable = expanded;
	    }
	
	    @Override
	    public List<T> getSubItems() {
	        return mSubItems;
	    }
	
	    public boolean hasSubItem() {
	        return mSubItems != null && mSubItems.size() > 0;
	    }
	
	    public void setSubItems(List<T> list) {
	        mSubItems = list;
	    }
	
	    public T getSubItem(int position) {
	        if (hasSubItem() && position < mSubItems.size()) {
	            return mSubItems.get(position);
	        } else {
	            return null;
	        }
	    }
	
	    public int getSubItemPosition(T subItem) {
	        return mSubItems != null ? mSubItems.indexOf(subItem) : -1;
	    }
	
	    public void addSubItem(T subItem) {
	        if (mSubItems == null) {
	            mSubItems = new ArrayList<>();
	        }
	        mSubItems.add(subItem);
	    }
	
	    public void addSubItem(int position, T subItem) {
	        if (mSubItems != null && position >= 0 && position < mSubItems.size()) {
	            mSubItems.add(position, subItem);
	        } else {
	            addSubItem(subItem);
	        }
	    }
	
	    public boolean contains(T subItem) {
	        return mSubItems != null && mSubItems.contains(subItem);
	    }
	
	    public boolean removeSubItem(T subItem) {
	        return mSubItems != null && mSubItems.remove(subItem);
	    }
	
	    public boolean removeSubItem(int position) {
	        if (mSubItems != null && position >= 0 && position < mSubItems.size()) {
	            mSubItems.remove(position);
	            return true;
	        }
	        return false;
	    }
	}

主要属性介绍：

① mExpandable属性标记是否是展开着的状态

② mSubItems 属性是下级列表的数据源

以下定义每级列表对应的对象，只贴部分代码，详细见后面的demo，
一级列表

	  package com.dkp.viewdemo.expandablelist;

	    public class Level0Item extends AbstractExpandableItem<Level1Item> implements MultiItemEntity {
	    public String title;
	    public String subTitle;
	
	    public Level0Item(String title, String subTitle) {
	        this.subTitle = subTitle;
	        this.title = title;
	    }
	
	    @Override
	    public int getItemType() {
	        return ExpandableItemAdapter.TYPE_LEVEL_ZERO;
	    }
	
	    @Override
	    public int getLevel() {
	        return 0;
	    }
	}
二级列表，三级列表同理

数据源有了，那么怎么在RecyclerView展示出来，并实现展开折叠效果呢？有同学可能会想用RecyclerView嵌套Recycler,这样可以么？答案是可以，但是没有必要这么做。

本例中主要是展开折叠是如何实现的，以下贴出核心代码：


	//展开功能的核心代码，mDataList是Adapter的数据源
    public int expand(@IntRange(from = 0) int position, boolean animate, boolean shouldNotify) {
        position -= getHeaderLayoutCount();//position - headersize 去掉头部占用的位置

        IExpandable expandable = getExpandableItem(position);//通过position获取一个bean
        if (expandable == null) {
            return 0;//如果数据为空 返回0
        }
        if (!hasSubItems(expandable)) {//如果这个Item没有子节点；依据是expandable.getSubItems返回的list 为空或 size是0
            expandable.setExpanded(false);//设置不可展开并返回
            return 0;
        }
        int subItemCount = 0;
        if (!expandable.isExpanded()) {//当前节点没有展开
            List list = expandable.getSubItems();//获取子节点的集合
            mDataList.addAll(position + 1, list);//把数据插入到mDataList中
            subItemCount += recursiveExpand(position + 1, list);//展开的子节点的数量

            expandable.setExpanded(true);//设置为展开状态
            subItemCount += list.size();//统计当前节点所有展开子节点数
        }
        int parentPos = position + getHeaderLayoutCount();//在列表中的实际位置，相对mDataList
        if (shouldNotify) {//是否刷新，是
            if (animate) {//有动画
                notifyItemChanged(parentPos);//刷新当前Item
                notifyItemRangeInserted(parentPos + 1, subItemCount);//刷新展开的所有item
            } else {//没动画
                notifyDataSetChanged();
            }
        }
        return subItemCount;
    }


    public int expandAll(int position, boolean animate, boolean notify) {
        position -= getHeaderLayoutCount();//除去header后的list的

        T endItem = null;
        if (position + 1 < this.mDataList.size()) {
            endItem = getItem(position + 1);//或取当前Item对应的bean
        }

        IExpandable expandable = getExpandableItem(position);//校验 这个Item bean 是IExpandable类型
        if (!hasSubItems(expandable)) {//如果当前节点没有子节点，返回
            return 0;
        }

        //展开当前节点，统计展开的节点的数量
        int count = expand(position + getHeaderLayoutCount(), false, false);
        for (int i = position + 1; i < this.mDataList.size(); i++) {
            T item = getItem(i);

            if (item == endItem) {
                break;
            }
            if (isExpandable(item)) {
                count += expand(i + getHeaderLayoutCount(), false, false);
            }
        }
        //刷新view
        if (notify) {
            if (animate) {
                notifyItemRangeInserted(position + getHeaderLayoutCount() + 1, count);
            } else {
                notifyDataSetChanged();
            }
        }
        return count;
    }




    private int recursiveCollapse(@IntRange(from = 0) int position) {
        T item = getItem(position);
        if (!isExpandable(item)) {
            return 0;
        }
        IExpandable expandable = (IExpandable) item;//IExpandable类型校验
        int subItemCount = 0;
        if (expandable.isExpanded()) {//当前节点是展开状态
            List<T> subItems = expandable.getSubItems();//获取子节点的集合
            for (int i = subItems.size() - 1; i >= 0; i--) {//从后往前
                T subItem = subItems.get(i);
                int pos = getItemPosition(subItem);
                if (pos < 0) {
                    continue;
                }
                if (subItem instanceof IExpandable) {
                    subItemCount += recursiveCollapse(pos);
                }
                mDataList.remove(pos);//从当前list里移除
                subItemCount++;
            }
        }
        return subItemCount;
    }

    private int recursiveExpand(int position, @NonNull List list) {
        int count = 0;
        int pos = position + list.size() - 1;
        for (int i = list.size() - 1; i >= 0; i--, pos--) {
            if (list.get(i) instanceof IExpandable) {
                IExpandable item = (IExpandable) list.get(i);//倒序，获取每一个具体的子节点
                if (item.isExpanded() && hasSubItems(item)) {//如果当前是展开状态，并且该子节点还有子节点
                    List subList = item.getSubItems();//获取子节点集合
                    mDataList.addAll(pos + 1, subList);//插入到mDataList
                    int subItemCount = recursiveExpand(pos + 1, subList);
                    count += subItemCount;
                }
            }
        }
        return count;

    }

    /**
     * 判断是否有子节点
     * @param item
     * @return
     */
    private boolean hasSubItems(IExpandable item) {
        List list = item.getSubItems();
        return list != null && list.size() > 0;
    }

    /**
     * 对当前item进行类型检查
     * @param item
     * @return
     */
    private boolean isExpandable(T item) {
        return item != null && item instanceof IExpandable;
    }

    /**
     * 返回一个IExpandable类型的item
     * @param position
     * @return
     */
    private IExpandable getExpandableItem(int position) {
        T item = getItem(position);
        if (isExpandable(item)) {
            return (IExpandable) item;
        } else {
            return null;
        }
    }


    public int collapse(@IntRange(from = 0) int position, boolean animate, boolean notify) {
        position -= getHeaderLayoutCount();

        IExpandable expandable = getExpandableItem(position);
        if (expandable == null) {
            return 0;
        }
        int subItemCount = recursiveCollapse(position);//移除该节点下展开的节点，返回移除的个数
        expandable.setExpanded(false);
        int parentPos = position + getHeaderLayoutCount();
        if (notify) {
            if (animate) {
                notifyItemChanged(parentPos);
                notifyItemRangeRemoved(parentPos + 1, subItemCount);
            } else {
                notifyDataSetChanged();
            }
        }
        return subItemCount;
    }


	}



**展开列表思想**：

当我们点击一级列表的时候

① 我们判断下当前节点（我们把每个Item看成一个节点）是不是折叠状态，

② 如果是获取当前节点的子节点的集合，也就是下级列表的数据源，

③ 把数据从当前节点的位置插入的当前的数据源中mDataList(mDataList刚开始只是充当一级列表的数据源)，

④ 然后通过递归方法获取插入的数据里所有展开的节点，并根据当前位置插入到mDataList中，这样就实现了展开列表的功能。

**折叠列表思想**：

① 先判断是否是展开状态

② 获取当前点击节点的子节点的集合

③ 递归判断子节点的下级节点，从后往前一个个的把数据从数据源里移除

④刷新View,至此折叠完成。

本文只是介绍多级列表的实现思想，用法见[本文多级列表的Demo](本文多级列表的Demo "https://github.com/15637972610/ViewDemo")

源码是最好的老师，感谢大家阅读，水平有限，如有错误请不吝赐教，感谢~
