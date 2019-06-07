
上篇我们知道请求构建完成并不是立马会去加载资源，而是对请求做了管理，从而知道当前需不需要加载资源，最大化的优化性能及用户体验，这一点是十分值得我们学习借鉴的。

问题：

1.数据资源加载过程是怎样的？

2.首次加载资源是怎么缓存的？

带着这两个问题，接下来我们接着上篇继续探讨：

	//Engine.java
	public synchronized <R> LoadStatus load(
	      GlideContext glideContext,
	      Object model,
	      Key signature,
	      int width,
	      int height,
	      Class<?> resourceClass,
	      Class<R> transcodeClass,
	      Priority priority,
	      DiskCacheStrategy diskCacheStrategy,
	      Map<Class<?>, Transformation<?>> transformations,
	      boolean isTransformationRequired,
	      boolean isScaleOnlyOrNoTransform,
	      Options options,
	      boolean isMemoryCacheable,
	      boolean useUnlimitedSourceExecutorPool,
	      boolean useAnimationPool,
	      boolean onlyRetrieveFromCache,
	      ResourceCallback cb,
	      Executor callbackExecutor) {
	    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
		//1.根据我们的配置参数构建key,每次加载资源的唯一标示
	    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
	        resourceClass, transcodeClass, options);
		//2.根据key从当前存活的缓存加载资源
	    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
	    if (active != null) {
	      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
	      if (VERBOSE_IS_LOGGABLE) {
	        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
	      }
	      return null;
	    }
		//3.根据key从当前内存缓存加载资源
	    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
	    if (cached != null) {
	      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
	      if (VERBOSE_IS_LOGGABLE) {
	        logWithTimeAndKey("Loaded resource from cache", startTime, key);
	      }
	      return null;
	    }
		//4.根据key从已有的EngineJobs加载
	    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
	    if (current != null) {
	      current.addCallback(cb, callbackExecutor);
	      if (VERBOSE_IS_LOGGABLE) {
	        logWithTimeAndKey("Added to existing load", startTime, key);
	      }
	      return new LoadStatus(cb, current);
	    }
		//4.根据key构建一个EngineJob和DecodeJob
	    EngineJob<R> engineJob =
	        engineJobFactory.build(
	            key,
	            isMemoryCacheable,
	            useUnlimitedSourceExecutorPool,
	            useAnimationPool,
	            onlyRetrieveFromCache);
	
	    DecodeJob<R> decodeJob =
	        decodeJobFactory.build(
	            glideContext,
	            model,
	            key,
	            signature,
	            width,
	            height,
	            resourceClass,
	            transcodeClass,
	            priority,
	            diskCacheStrategy,
	            transformations,
	            isTransformationRequired,
	            isScaleOnlyOrNoTransform,
	            onlyRetrieveFromCache,
	            options,
	            engineJob);
		//添加到jobs缓存起来
	    jobs.put(key, engineJob);
		//添加ResourceCallback回调
	    engineJob.addCallback(cb, callbackExecutor);
		//5.添加任务到线程池里
	    engineJob.start(decodeJob);
	
	    if (VERBOSE_IS_LOGGABLE) {
	      logWithTimeAndKey("Started new load", startTime, key);
	    }
	    return new LoadStatus(cb, engineJob);
	 }

大致流程分为以上5步，接下来我们跟踪每一步看看他们分别做了什么，是如何实现的。**由于构建key的过程比较简单，就不赘述了。2,3步都是从内存中加载资源放在一起；4,5构建任务，添加到线程池一起分析。**

#### 1.根据key从内存缓存加载资源 

	EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
	    if (active != null) {
	      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
	      if (VERBOSE_IS_LOGGABLE) {
	        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
	      }
	      return null;
	}

获取一个软引用缓存，它是用了一个Map<Key, WeakReference<EngineResource<?>>>装载起来的,计数引用数，在图片加载完成后进行判断，如果引用计数为空则回收掉。
这个缓存主要是谁来放进去呢？ 可以参考内存缓存loadFromCache方法。
	//Engine.java
	EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
	    if (cached != null) {
	      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
	      if (VERBOSE_IS_LOGGABLE) {
	        logWithTimeAndKey("Loaded resource from cache", startTime, key);
	      }
	      return null;
	}
	//Engine.java
	private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
	    if (!isMemoryCacheable) {
	      return null;
	    }
	
	    EngineResource<?> cached = getEngineResourceFromCache(key);
	    if (cached != null) {
	      cached.acquire();
	      activeResources.activate(key, cached);
	    }
	    return cached;
	}
	//Engine.java
	private EngineResource<?> getEngineResourceFromCache(Key key) {
	    Resource<?> cached = cache.remove(key);
	
	    final EngineResource<?> result;
	    if (cached == null) {
	      result = null;
	    } else if (cached instanceof EngineResource) {
	      // Save an object allocation if we've cached an EngineResource (the typical case).
	      result = (EngineResource<?>) cached;
	    } else {
	      result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
	    }
	    return result;
	}
loadFromCache根据key从cache.remove(key)中得到

	//LruCache.java
	public class LruCache<T, Y> {
	private final LinkedHashMap<T, Y> cache = new LinkedHashMap<T, Y>(100, 0.75f, true);
我们在LruCache.java中看到cache是一个LinkedHashMap
#### 2.根据key从已有的EngineJobs加载 
假如从内存缓存中没有拿到资源，就会根据key从已有的EngineJobs缓存中获取一个EngineJob
	//EngineJob.java	
	public void addCallback(ResourceCallback cb) {
	        Util.assertMainThread();
	        if (hasResource) {
	            cb.onResourceReady(engineResource);
	        } else if (hasException) {
	            cb.onException(exception);
	        } else {
	            cbs.add(cb);
	        }
	}
然后通过EngineJob的addCallback把资源回传给UI线程。
那这个engineResource是从哪里来的呢？

#### 3.根据key构建一个EngineJob和DecodeJob 
如果以上都没有成功拿到资源，就构建EngineJob和DecodeJob对象，

**构建EngineJob：**

	//EngineJobFactory.java
    @Synthetic final Pools.Pool<EngineJob<?>> pool =
        FactoryPools.threadSafe(
            JOB_POOL_SIZE,
            new FactoryPools.Factory<EngineJob<?>>() {
              @Override
              public EngineJob<?> create() {
                return new EngineJob<>(
                    diskCacheExecutor,
                    sourceExecutor,
                    sourceUnlimitedExecutor,
                    animationExecutor,
                    listener,
                    pool);
              }
            });

	<R> EngineJob<R> build(
	        Key key,
	        boolean isMemoryCacheable,
	        boolean useUnlimitedSourceGeneratorPool,
	        boolean useAnimationPool,
	        boolean onlyRetrieveFromCache) {
	      EngineJob<R> result = Preconditions.checkNotNull((EngineJob<R>) pool.acquire());
	      return result.init(
	          key,
	          isMemoryCacheable,
	          useUnlimitedSourceGeneratorPool,
	          useAnimationPool,
	          onlyRetrieveFromCache);
	}
我们看到通过EngineJobFactory 构建EngineJob时传入一些列的初始化参数，磁盘缓存线程池，资源加载线程池，动画使用的线程池，监听器等。由此可知EngineJob是负责将资源缓存到磁盘的。

**再看DecodeJob的构建**：

	@Synthetic final Pools.Pool<DecodeJob<?>> pool =
	        FactoryPools.threadSafe(JOB_POOL_SIZE,
	            new FactoryPools.Factory<DecodeJob<?>>() {
	          @Override
	          public DecodeJob<?> create() {
	            return new DecodeJob<>(diskCacheProvider, pool);
	          }
	        });
	    private int creationOrder;
	
	    DecodeJobFactory(DecodeJob.DiskCacheProvider diskCacheProvider) {
	      this.diskCacheProvider = diskCacheProvider;
	    }
	
	    @SuppressWarnings("unchecked")
	    <R> DecodeJob<R> build(GlideContext glideContext,
	        Object model,
	        EngineKey loadKey,
	        Key signature,
	        int width,
	        int height,
	        Class<?> resourceClass,
	        Class<R> transcodeClass,
	        Priority priority,
	        DiskCacheStrategy diskCacheStrategy,
	        Map<Class<?>, Transformation<?>> transformations,
	        boolean isTransformationRequired,
	        boolean isScaleOnlyOrNoTransform,
	        boolean onlyRetrieveFromCache,
	        Options options,
	        DecodeJob.Callback<R> callback) {
	      DecodeJob<R> result = Preconditions.checkNotNull((DecodeJob<R>) pool.acquire());
	      return result.init(
	          glideContext,
	          model,
	          loadKey,
	          signature,
	          width,
	          height,
	          resourceClass,
	          transcodeClass,
	          priority,
	          diskCacheStrategy,
	          transformations,
	          isTransformationRequired,
	          isScaleOnlyOrNoTransform,
	          onlyRetrieveFromCache,
	          options,
	          callback,
	          creationOrder++);
	    }
	}
参数主要有，初始化的缓存策略构建构建diskCacheStrategy，资源源地址model，资源类型resourceClass，主线程回调callback等，由此可知DecodeJob是负责加载资源的。在这里对家在资源需要的参数进行一系列初始化。初始化完成以后开启线城池：

	engineJob.addCallback(cb, callbackExecutor);
	engineJob.start(decodeJob);

	public synchronized void start(DecodeJob<R> decodeJob) {
	    this.decodeJob = decodeJob;
	    GlideExecutor executor = decodeJob.willDecodeFromCache()
	        ? diskCacheExecutor
	        : getActiveSourceExecutor();
	    executor.execute(decodeJob);
	}

我们看到DecodeJob是Runnable
	class DecodeJob<R> implements DataFetcherGenerator.FetcherReadyCallback,
    Runnable,
    Comparable<DecodeJob<?>>,
    Poolable {

线程开启，他肯定有run方法：

	@SuppressWarnings("PMD.AvoidRethrowingException")
  	@Override
	public void run() {
	    
	    DataFetcher<?> localFetcher = currentFetcher;
	    try {
	      if (isCancelled) {
	        notifyFailed();
	        return;
	      }
	      runWrapped();
	    } catch (CallbackException e) {
	     
	      throw e;
	    } catch (Throwable t) {
	      
	      if (Log.isLoggable(TAG, Log.DEBUG)) {
	        Log.d(TAG, "DecodeJob threw unexpectedly"
	            + ", isCancelled: " + isCancelled
	            + ", stage: " + stage, t);
	      }
	      
	      if (stage != Stage.ENCODE) {
	        throwables.add(t);
	        notifyFailed();
	      }
	      if (!isCancelled) {
	        throw t;
	      }
	      throw t;
	    } finally {
	      
	      if (localFetcher != null) {
	        localFetcher.cleanup();
	      }
	      GlideTrace.endSection();
	    }
	  }
	
	  private void runWrapped() {
	    switch (runReason) {
	      case INITIALIZE:
	        stage = getNextStage(Stage.INITIALIZE);
	        currentGenerator = getNextGenerator();
	        runGenerators();
	        break;
	      case SWITCH_TO_SOURCE_SERVICE:
	        runGenerators();
	        break;
	      case DECODE_DATA:
	        decodeFromRetrievedData();
	        break;
	      default:
	        throw new IllegalStateException("Unrecognized run reason: " + runReason);
	    }
	  }
果真，run方法又调用了runWrapped

	//DecodeJob.java
	private DataFetcherGenerator getNextGenerator() {
		    switch (stage) {
		      case RESOURCE_CACHE:
		        return new ResourceCacheGenerator(decodeHelper, this);
		      case DATA_CACHE:
		        return new DataCacheGenerator(decodeHelper, this);
		      case SOURCE:
		        return new SourceGenerator(decodeHelper, this);
		      case FINISHED:
		        return null;
		      default:
		        throw new IllegalStateException("Unrecognized stage: " + stage);
		    }
		}
	
		private void runGenerators() {
		    currentThread = Thread.currentThread();
		    startFetchTime = LogTime.getLogTime();
		    boolean isStarted = false;
		    while (!isCancelled && currentGenerator != null
		        && !(isStarted = currentGenerator.startNext())) {
		      stage = getNextStage(stage);
		      currentGenerator = getNextGenerator();
		
		      if (stage == Stage.SOURCE) {
		        reschedule();
		        return;
		      }
		    }
		    // We've run out of stages and generators, give up.
		    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
		      notifyFailed();
		    }
		
		    // Otherwise a generator started a new load and we expect to be called back in
		    // onDataFetcherReady.
		  }
		private Stage getNextStage(Stage current) {
		    switch (current) {
		      case INITIALIZE:
		        return diskCacheStrategy.decodeCachedResource()
		            ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
		      case RESOURCE_CACHE:
		        return diskCacheStrategy.decodeCachedData()
		            ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
		      case DATA_CACHE:
		        // Skip loading from source if the user opted to only retrieve the resource from cache.
		        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
		      case SOURCE:
		      case FINISHED:
		        return Stage.FINISHED;
		      default:
		        throw new IllegalArgumentException("Unrecognized stage: " + current);
		    }
	}
再分析runWrapped过程的时候先介绍几个方法的作用，

1. getNextStage方法：获取下一步执行的策略

	一共5种策略：INITIALIZE初始，RESOURCE_CACHE资源缓存，DATA_CACHE本地缓存，SOURCE数据源，FINISHED

2. getNextGenerator方法：根据Stage获取到相应的Generator

		//对应关系
		case RESOURCE_CACHE:
		    return new ResourceCacheGenerator(decodeHelper, this);
		case DATA_CACHE:
		    return new DataCacheGenerator(decodeHelper, this);
		case SOURCE:
		    return new SourceGenerator(decodeHelper, this);


3. runGenerators方法：加载资源

通过调用currentGenerator.startNext()方法触发对应的next方法加载资源

	//SourceGenerator.java
	@Override
	public boolean startNext() {
	    if (dataToCache != null) {
	      Object data = dataToCache;
	      dataToCache = null;
	      cacheData(data);
	    }
	
	    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
	      return true;
	    }
	    sourceCacheGenerator = null;
	
	    loadData = null;
	    boolean started = false;
	    while (!started && hasNextModelLoader()) {
	      loadData = helper.getLoadData().get(loadDataListIndex++);
	      if (loadData != null
	          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
	          || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
	        started = true;
	        loadData.fetcher.loadData(helper.getPriority(), this);
	      }
	    }
	    return started;
	}
loadData.fetcher获取对应资源加载器，通过loadData方法加载资源。再次执行的时候内存缓存已经存在，因此直接缓存数据cacheData(data)。

4.4 decodeFromRetrievedData方法：

	private void decodeFromRetrievedData() {
	    if (Log.isLoggable(TAG, Log.VERBOSE)) {
	      logWithTimeAndKey("Retrieved data", startFetchTime,
	          "data: " + currentData
	              + ", cache key: " + currentSourceKey
	              + ", fetcher: " + currentFetcher);
	    }
	    Resource<R> resource = null;
	    try {
	      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
	    } catch (GlideException e) {
	      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
	      throwables.add(e);
	    }
	    if (resource != null) {
	      notifyEncodeAndRelease(resource, currentDataSource);
	    } else {
	      runGenerators();
	    }
	}

获取数据成功后，进行处理，也就是数据的装载过程，给我们的View装载数据并显示。

我们来总结一下整体的流程，当请求构建完成，发起加载资源的请求以后，Glide会通过Engine的load方法获取资源，先看存活的内存缓存里能不能找到，如果找到就回主线程，找不到，去内存缓存找，找不到再去本地存储找，本地找不到就构建了EngineJob负责处理资源加载完成后的缓存逻辑，构建DecodeJob并对加载数据需要的参数进行初始化，DecodeJob实现了Runnable，这里通过创建线程池开启加载资源和处理缓存的任务，DecodeJob的run方法调用了runWrapped方法，这里分为四步去加载数据处理数据：

1.通过 getNextStage方法：获取下一步执行的策略

2.通过 getNextGenerator方法：根据Stage获取到相应的Generator

3.通过runGenerators方法：触发startNext方法，loadData.fetcher.loadData加载资源，缓存资源

4.decodeFromRetrievedData方法数据获取成功后给View装载数据。

至此，数据的加载和缓存就分析完了。

