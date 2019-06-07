上篇我们看到了RequestManager的创建，以及上层ActivityFragmentLifecycle对RequestManager生命周期的管理，今天我们来继续探究Glide源码里对生命周期的管理。

	//RequestManager.java
    public RequestBuilder<Drawable> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
    	return asDrawable().load(resourceId);
  	}
    public RequestBuilder<Drawable> asDrawable() {
    	return as(Drawable.class);
    }
	public <ResourceType> RequestBuilder<ResourceType> as(
	      @NonNull Class<ResourceType> resourceClass) {
	   return new RequestBuilder<>(glide, this, resourceClass, context);
	}

看下上面的过程，
第一步：调用了asDrawable()，asDrawable里调用了as（Drawable.class）默认resourceClass类型是Drawable.class 
as方法创建一个RequestBuilder对象，并返回
第二步：接着调用RequestBuilder对象的load方法：

	public RequestBuilder<TranscodeType> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
		return loadGeneric(resourceId).apply(signatureOf(ApplicationVersionSignature.obtain(context)));
	}
	//将我们传进来的Integer 类型的参数 赋值给model ，返回RequestBuilder
	private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
	    this.model = model;
	    isModelSet = true;
	    return this;
	}
	//RequestBuilder的apply
	public RequestBuilder<TranscodeType> apply(@NonNull BaseRequestOptions<?> requestOptions) {
	    Preconditions.checkNotNull(requestOptions);
	    //调用父类的apply
	    return super.apply(requestOptions);
	}

	//BaseRequestOptions.java 的 app方法，
	public T apply(@NonNull BaseRequestOptions<?> o) {
	    if (isAutoCloneEnabled) {
	      return clone().apply(o);
	    }
	    BaseRequestOptions<?> other = o;
	
	    if (isSet(other.fields, SIZE_MULTIPLIER)) {
	      sizeMultiplier = other.sizeMultiplier;
	    }
	    if (isSet(other.fields, USE_UNLIMITED_SOURCE_GENERATORS_POOL)) {
	      useUnlimitedSourceGeneratorsPool = other.useUnlimitedSourceGeneratorsPool;//资源缓存池重新赋值
	    }
	    if (isSet(other.fields, USE_ANIMATION_POOL)) {
	      useAnimationPool = other.useAnimationPool;//动画缓存池重新赋值
	    }
	    if (isSet(other.fields, DISK_CACHE_STRATEGY)) {
	      diskCacheStrategy = other.diskCacheStrategy;//缓存策略重新赋值
	    }
	    if (isSet(other.fields, PRIORITY)) {
	      priority = other.priority;//优先级
	    }
	    if (isSet(other.fields, ERROR_PLACEHOLDER)) {
	      errorPlaceholder = other.errorPlaceholder;//error操作符
	      errorId = 0;
	      fields &= ~ERROR_ID;
	    }
	    if (isSet(other.fields, ERROR_ID)) {
	      errorId = other.errorId;
	      errorPlaceholder = null;
	      fields &= ~ERROR_PLACEHOLDER;
	    }
	    if (isSet(other.fields, PLACEHOLDER)) {
	      placeholderDrawable = other.placeholderDrawable;//placeholder操作符
	      placeholderId = 0;
	      fields &= ~PLACEHOLDER_ID;
	    }
	    if (isSet(other.fields, PLACEHOLDER_ID)) {
	      placeholderId = other.placeholderId;
	      placeholderDrawable = null;
	      fields &= ~PLACEHOLDER;
	    }
	    if (isSet(other.fields, IS_CACHEABLE)) {
	      isCacheable = other.isCacheable;
	    }
	    if (isSet(other.fields, OVERRIDE)) {
	      overrideWidth = other.overrideWidth;
	      overrideHeight = other.overrideHeight;
	    }
	    if (isSet(other.fields, SIGNATURE)) {
	      signature = other.signature;
	    }
	    if (isSet(other.fields, RESOURCE_CLASS)) {
	      resourceClass = other.resourceClass;
	    }
	    if (isSet(other.fields, FALLBACK)) {
	      fallbackDrawable = other.fallbackDrawable;
	      fallbackId = 0;
	      fields &= ~FALLBACK_ID;
	    }
	    if (isSet(other.fields, FALLBACK_ID)) {
	      fallbackId = other.fallbackId;
	      fallbackDrawable = null;
	      fields &= ~FALLBACK;
	    }
	    if (isSet(other.fields, THEME)) {
	      theme = other.theme;
	    }
	    if (isSet(other.fields, TRANSFORMATION_ALLOWED)) {
	      isTransformationAllowed = other.isTransformationAllowed;
	    }
	    if (isSet(other.fields, TRANSFORMATION_REQUIRED)) {
	      isTransformationRequired = other.isTransformationRequired;
	    }
	    if (isSet(other.fields, TRANSFORMATION)) {
	      transformations.putAll(other.transformations);
	      isScaleOnlyOrNoTransform = other.isScaleOnlyOrNoTransform;
	    }
	    if (isSet(other.fields, ONLY_RETRIEVE_FROM_CACHE)) {
	      onlyRetrieveFromCache = other.onlyRetrieveFromCache;
	    }
		//以上是一系列的操作符
	
	    // Applying options with dontTransform() is expected to clear our transformations.
	    if (!isTransformationAllowed) {
	      transformations.clear();
	      fields &= ~TRANSFORMATION;
	      isTransformationRequired = false;
	      fields &= ~TRANSFORMATION_REQUIRED;
	      isScaleOnlyOrNoTransform = true;
	    }
	
	    fields |= other.fields;
	    options.putAll(other.options);
	
	    return selfOrThrowIfLocked();
	}

我们看到load方法内部调用了loadGeneric方法把我们的int值给了model,返回一个RequestBuilder对象，调用apply方法，然后执行super的apply方法，可以看到它的父类是BaseRequestOptions，父类的apply主要对一些缓存，缓存策略，操作符重新赋值（因此我们可以通过RequstBuilder在代码中更改配置），返回一个RequestBuilder对象。

通过上面的代码我们知道其实load过程就是对我们的一些配置重新赋值的一个过程。


接着我们看into方法,它有很多重载，这里我们传进去的是ImageView那么来看下这个方法

	//RequestBuilder.java

	public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
	    Util.assertMainThread();
	    Preconditions.checkNotNull(view);
	
	    BaseRequestOptions<?> requestOptions = this;
	    if (!requestOptions.isTransformationSet()
	        && requestOptions.isTransformationAllowed()
	        && view.getScaleType() != null) {
	      
	      switch (view.getScaleType()) {
	        case CENTER_CROP:
	          requestOptions = requestOptions.clone().optionalCenterCrop();
	          break;
	        case CENTER_INSIDE:
	          requestOptions = requestOptions.clone().optionalCenterInside();
	          break;
	        case FIT_CENTER:
	        case FIT_START:
	        case FIT_END:
	          requestOptions = requestOptions.clone().optionalFitCenter();
	          break;
	        case FIT_XY:
	          requestOptions = requestOptions.clone().optionalCenterInside();
	          break;
	        case CENTER:
	        case MATRIX:
	        default:
	          // Do nothing.
	      }
	    }
	
	    return into(
	        glideContext.buildImageViewTarget(view, transcodeClass),
	        /*targetListener=*/ null,
	        requestOptions,
	        Executors.mainThreadExecutor());
	}
我们看到into方法会根据我们ImageView的ScaleType返回requestOptions然后会调用into方法的另一个重载方法

> 参数1.glideContext.buildImageViewTarget(view,
> transcodeClass)方法会根据View和transcodeClass生成对应的ViewTarget,例如：DrawableImageViewTarget、CustomViewTarget、DrawableThumbnailImageViewTarget等。
> 
> 参数2.看源码我们知道Executors.mainThreadExecutor()方法里创建一个UI线程的Handler,通过execute把任务发送到主线程

## 生命周期下 重点部分：


	//RequestBuilder.java
	private <Y extends Target<TranscodeType>> Y into(
	      @NonNull Y target,
	      @Nullable RequestListener<TranscodeType> targetListener,
	      BaseRequestOptions<?> options,
	      Executor callbackExecutor) {
	    Preconditions.checkNotNull(target);
	    if (!isModelSet) {
	      throw new IllegalArgumentException("You must call #load() before calling #into()");
	    }
		//看到这里构建请求，target这里就是我们的imageView，listener是null
	    Request request = buildRequest(target, targetListener, options, callbackExecutor);
	
	    Request previous = target.getRequest();
	    if (request.isEquivalentTo(previous)
	        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
	      request.recycle();

		  //1.如果Request的状态是completed,重复begin将确保结果正常返回completed
			2.它触发RequestListeners和Targets，如果request是failed那么会重启request，给它完成任务的机会
			3.如果request正在运行，那么不会打断它的状态，让它继续运行
	      if (!Preconditions.checkNotNull(previous).isRunning()) {

	        previous.begin();
	      }
	      return target;
	    }
	
	    requestManager.clear(target);
	    //分析1：
	    target.setRequest(request);
	    //分析2：
	    requestManager.track(target, request);
	
	    return target;
	}

buildRequest(target)创建一个Request,在Request的创建中会针对是否有缩略图来创建不同尺寸的请求;

buildRequest-->buildRequestRecursive-->...SingleRequest.obtain( 不管何种类型的 最终会调用到这里：

	target.setRequest(request);
	requestManager.track(target, request);
分析1：
首先，target.setRequest(request)

如果target是ViewTarget，最终调用了ViewTarget的setTag方法，那么request会被设置到View的tag上。这样其实是有一个好处，每一个View有一个自己的Request，如果有重复请求，那么都会先去拿到上一个已经绑定的Request，并且从RequestManager中清理回收掉。这应该是去重的功能。

	//ViewTarget.java ---->target.setRequest
	private void setTag(@Nullable Object tag) {
	    if (tagId == null) {
	      isTagUsedAtLeastOnce = true;
	      view.setTag(tag);
	    } else {
	      view.setTag(tagId, tag);
	    }
	}
分析2：

	//RequestManager.java
	private final TargetTracker targetTracker = new TargetTracker();
	synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
	    targetTracker.track(target);
	    requestTracker.runRequest(request);
	}
分析2.1先看targetTracker.track(target, request);targetTracker是RequestManager类创建的时候就创建了的，下面是TargetTracker 类：

	//实现LifecycleListener 接口
	public final class TargetTracker implements LifecycleListener {
	//创建set集合
	  private final Set<Target<?>> targets =
	      Collections.newSetFromMap(new WeakHashMap<Target<?>, Boolean>());
	  //添加target进集合，这里我们的对应是ViewTarget
	  public void track(@NonNull Target<?> target) {
	    targets.add(target);
	  }
	//移除Target
	  public void untrack(@NonNull Target<?> target) {
	    targets.remove(target);
	  }
	
	  @Override
	  public void onStart() {
	    for (Target<?> target : Util.getSnapshot(targets)) {
	      target.onStart();
	    }
	  }
	
	  @Override
	  public void onStop() {
	    for (Target<?> target : Util.getSnapshot(targets)) {
	      target.onStop();
	    }
	  }
	
	  @Override
	  public void onDestroy() {
	    for (Target<?> target : Util.getSnapshot(targets)) {
	      target.onDestroy();
	    }
	  }
	
	  @NonNull
	  public List<Target<?>> getAll() {
	    return Util.getSnapshot(targets);
	  }
	
	  public void clear() {
	    targets.clear();
	  }
	  }

TargetTracker实现生命周期的回调，维护一个View的集合，在RequestManager的生命周期回调里分别调用targetTracker的生命周期方法，从而和当前的Activity或者Fragment生命周期进行绑定。

分析2.2 调用了requestTracker.runRequest(request);
requestTracker看注释，它是一个根据当前状态，控制Request请求开始，取消的一个管理类。内部维护着存储请求的Set集合。

	//RequestTracker.java
	public void runRequest(@NonNull Request request) {
	    requests.add(request);//添加到内存缓存
	    if (!isPaused) {
	      request.begin();//启动SingleRequest
	    } else {
	      request.clear();
	      if (Log.isLoggable(TAG, Log.VERBOSE)) {
	        Log.v(TAG, "Paused, delaying request");
	      }
	      pendingRequests.add(request);
	    }
	}
request.begin();--->因为buildRequest出来的的request是SingleRequest类型，所以看它的begin方法实现

	//SingleRequest.java
	@Override
	public synchronized void begin() {
	    assertNotCallingCallbacks();
	    stateVerifier.throwIfRecycled();
	    startTime = LogTime.getLogTime();
	    if (model == null) {//model本案例中也就是我们的资源id，还有可能是url,bitmap等，
	      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
	        width = overrideWidth;
	        height = overrideHeight;
	      }

	      int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
	      onLoadFailed(new GlideException("Received null model"), logLevel);
	      return;//抛异常返回
	    }
	
	    if (status == Status.RUNNING) {
	      throw new IllegalArgumentException("Cannot restart a running request");
	    }
	
	    
	    if (status == Status.COMPLETE) {//onResourceReady去缓存取
	      onResourceReady(resource, DataSource.MEMORY_CACHE);
	      return;
	    }
	
	    status = Status.WAITING_FOR_SIZE;
		// 如果当前的View尺寸已经加载获取到了，那么就会进入真正的加载流程。
	    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
	      onSizeReady(overrideWidth, overrideHeight);
	    } else {
	      target.getSize(this);
	    }
		// 如果等待和正在执行状态，那么当前会加载占位符Drawable
	    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
	        && canNotifyStatusChanged()) {
	      target.onLoadStarted(getPlaceholderDrawable());
	    }
	    if (IS_VERBOSE_LOGGABLE) {
	      logV("finished run method in " + LogTime.getElapsedMillis(startTime));
	    }
	}
通过跟踪源码我们知道SingleRequest中根据具体状态status去执行不同策略。
//SingleRequest.java

	@Override
	public synchronized void onSizeReady(int width, int height) {
	    stateVerifier.throwIfRecycled();
	    if (IS_VERBOSE_LOGGABLE) {
	      logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
	    }
	    if (status != Status.WAITING_FOR_SIZE) {
	      return;
	    }
	    status = Status.RUNNING;
	
	    float sizeMultiplier = requestOptions.getSizeMultiplier();
	    this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
	    this.height = maybeApplySizeMultiplier(height, sizeMultiplier);
	
	    if (IS_VERBOSE_LOGGABLE) {
	      logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
	    }
	    loadStatus =
	        engine.load(
	            glideContext,
	            model,
	            requestOptions.getSignature(),
	            this.width,
	            this.height,
	            requestOptions.getResourceClass(),
	            transcodeClass,
	            priority,
	            requestOptions.getDiskCacheStrategy(),
	            requestOptions.getTransformations(),
	            requestOptions.isTransformationRequired(),
	            requestOptions.isScaleOnlyOrNoTransform(),
	            requestOptions.getOptions(),
	            requestOptions.isMemoryCacheable(),
	            requestOptions.getUseUnlimitedSourceGeneratorsPool(),
	            requestOptions.getUseAnimationPool(),
	            requestOptions.getOnlyRetrieveFromCache(),
	            this,
	            callbackExecutor);
	
	    // This is a hack that's only useful for testing right now where loads complete synchronously
	    // even though under any executor running on any thread but the main thread, the load would
	    // have completed asynchronously.
	    if (status != Status.RUNNING) {
	      loadStatus = null;
	    }
	    if (IS_VERBOSE_LOGGABLE) {
	      logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
	    }
	}
最终是通过engine的load方法加载资源。

总结一下吧，本篇是生命周期管理的下篇，上篇我们知到了当前Activity的生命周期是如何和Glide关联的，以及上层的ActivityFragmentLifecycle是如何管理RequestManager的，本片主要介绍了三个类，
一、TargetTracker类 ，它实现了生命周期的接口，内部维护者一个set集合，添加Target,
它在RequestManager创建的时候创建，在RequestManager的生命周期方法中通过TargetTracker 引用调用相应回调方法实现对Target和当前Activity或Fragment生命周期的绑定。
二、RequestTracker创建时机是在RequestManger的构造函数通过new 关键字实例化，主要用来根据当前维护的状态控制Request的，开始、取消、请除
三、Request的具体实现类SingleRequest，在RequestBuilder的into方法内 通过buildRequest方法构建实例，通过这个类发起数据加载。
	
