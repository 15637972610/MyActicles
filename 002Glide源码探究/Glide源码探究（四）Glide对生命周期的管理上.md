
用过Glide的小伙伴都知道，图片加载会和Activity/Fragment的生命周期保持一致，并且有对应的 trimMemory 接口实现可供调用。那么Glide是怎么实现和Activity/Fragment的生命周期保持一致的么？带着这个疑问我们来探究下源码。

仍然以Glide.with(context).load(url).into(mImageView)说起

我们知道with方法有很多重载，支持FragmentActivity，android.support.v4.app.Fragment、android.app.Fragment、Activity、Context、View

简单说一下区别：

	android.app.Fragment使用 getFragmentManager().findFragmentById(R.id.userList)  
	android.support.v4.app.Fragment使用 (ListFragment)getSupportFragmentManager().findFragmentById(R.id.userList) 

我们这里以android.support.v4.app.Fragment为例跟踪源码看下过程：

	@NonNull
	public static RequestManager with(@NonNull Fragment fragment) {
	    return getRetriever(fragment.getActivity()).get(fragment);
	}
	//getRetriever返回一个RequestManagerRetriever.java下面看一下这个类
	
	//RequestManagerRetriever.java
	@NonNull
	public RequestManager get(@NonNull FragmentActivity activity) {
	    if (Util.isOnBackgroundThread()) {
	      return get(activity.getApplicationContext());
	    } else {
	      assertNotDestroyed(activity);
	      FragmentManager fm = activity.getSupportFragmentManager();
	      return supportFragmentGet(
	          activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
	    }
	}

    RequestManager supportFragmentGet(Context context, FragmentManager fm) {
    //第一步：创建SupportRequestManagerFragment
	SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);
	//第二步：创建RequestManager
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
			//注意这里传递的参数current.getLifecycle是ActivityFragmentLifecycle
			//构造方法里给
            requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());

            current.setRequestManager(requestManager);
        }
        return requestManager;
    }

	//第一步源码分析：
	//RequestManagerRetriever.java
	@NonNull
	private SupportRequestManagerFragment getSupportRequestManagerFragment(
	      @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
	    SupportRequestManagerFragment current =
	        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
	    if (current == null) {
	      current = pendingSupportRequestManagerFragments.get(fm);
	      if (current == null) {

			//这行代码返回SupportRequestManagerFragment对象，构造函数里new ActivityFragmentLifecycle()并让SupportRequestManagerFragment持有他的引用

	        current = new SupportRequestManagerFragment();


	        current.setParentFragmentHint(parentHint);
	        if (isParentVisible) {
	          current.getGlideLifecycle().onStart();
	        }
	        pendingSupportRequestManagerFragments.put(fm, current);
			//创建了SupportRequestManagerFragment对象,绑定给当前上下文，TAG为FRAGMENT_TAG并加入到当前上下文的FragmentTransaction事务中，从而与当前上下文Fragment的生命周期保持一致。
	        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
	        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
	      }
	    }
	    return current;
	}

	//第二步源码分析
	//RequestManager.java的构造
	RequestManager(
	      Glide glide,
	      Lifecycle lifecycle,
	      RequestManagerTreeNode treeNode,
	      RequestTracker requestTracker,
	      ConnectivityMonitorFactory factory,
	      Context context) {
	    this.glide = glide;
	    this.lifecycle = lifecycle;
	    this.treeNode = treeNode;
	    this.requestTracker = requestTracker;
	    this.context = context;
	
	    connectivityMonitor =
	        factory.build(
	            context.getApplicationContext(),
	            new RequestManagerConnectivityListener(requestTracker));
	
	   
	    if (Util.isOnBackgroundThread()) {
	      mainHandler.post(addSelfToLifecycle);
	    } else {
		  //lifecycle是ActivityFragmentLifecycle对象，参考第一步的
		  //current.getLifecycle就可以发现

	      lifecycle.addListener(this);
	    }
	    lifecycle.addListener(connectivityMonitor);
	
	    defaultRequestListeners =
	        new CopyOnWriteArrayList<>(glide.getGlideContext().getDefaultRequestListeners());
	    setRequestOptions(glide.getGlideContext().getDefaultRequestOptions());
	
	    glide.registerRequestManager(this);
	}
	//ActivityFragmentLifecycle.java 
	class ActivityFragmentLifecycle implements Lifecycle {

  	private final Set<LifecycleListener> lifecycleListeners =
      Collections.newSetFromMap(new WeakHashMap<LifecycleListener, Boolean>());//创建set集合

	//把listener添加到集合，
	//可以看一下RequestManager的构造里lifecycle.addListener(this)把自己对象的引用交给ActivityFragmentLifecycle管理
	@Override
	public void addListener(@NonNull LifecycleListener listener) {
	    lifecycleListeners.add(listener);
	
	    if (isDestroyed) {
	      listener.onDestroy();
	    } else if (isStarted) {
	      listener.onStart();
	    } else {
	      listener.onStop();
	    }
	}

总结一下：

如果glide.with（fragment）传入的参数是Fragment:
会调用supportFragmentGet方法，创建一个SupportRequestManagerFragment对象,绑定给当前上下文，TAG为FRAGMENT_TAG，并加入到当前上下文的FragmentTransaction事务中，在attach的时候调用registerFragmentWithRoot(getActivity())把该fragment作为rootRequestManagerFragment，与当前上下文Fragment的生命周期保持一致。

创建SupportRequestManagerFragment对象时，在它的无参构造里创建了ActivityFragmentLifecycle对象，让当前SupportRequestManagerFragment持有它的引用，目的是根据当前上下文的声明周期对RequestManager进行管理。在接下来创建requestManager的时候把ActivityFragmentLifecycle对象传递给requestManager，requestManager把自己的引用添加到ActivityFragmentLifecycle类的一个Set集合中。当前Fragment生命周期有变化时通过ActivityFragmentLifecycle的回调对RequestManager的生命周期进行管理。
