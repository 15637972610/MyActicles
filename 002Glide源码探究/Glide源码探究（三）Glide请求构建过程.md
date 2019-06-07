本篇结合源码讲解Glilde的构建过程，我们知道Glide的使用是非常简单的，例如：

    Glide.with(this)
	.load(android.R.mipmap.sym_def_app_icon)
	.into(mImageView);
通过上面的一行代码就可以给我们的mImageView加载一张图片；Glide的内部为我们做了大量的工作，都有些什么呢？今天我们来探究的它的第一步（RequestManager）请求构建过程。

通过源码我们知道Glide的with方法有很多实现，允许我们传入类型有Context、Activity、FragmentActivity、Fragment、android.app.Fragment、View。

    @NonNull
    public static RequestManager with(@NonNull Activity activity) {
	    return getRetriever(activity).get(activity);
    }
	......
    private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    return Glide.get(context).getRequestManagerRetriever();
    }
上述方法中：
第一步：with方法内的 getRetriever(activity).get(activity)
getRetriever（activity）返回RequestManagerRetriever 对象然后通过get(activity)返回RequestManger对象。
第二步：getRetriever方法内的Glide.get(context).getRequestManagerRetriever();
Glide.get(context)返回的是Glide对象，然后调用getRequestManagerRetriever()返回RequestManagerRetriever对象。

我们来分析Glide.get(context)这一具体过程：
	
	  public static Glide get(@NonNull Context context) {
	    if (glide == null) {
	      synchronized (Glide.class) {
	        if (glide == null) {
	         //首次加载
	          checkAndInitializeGlide(context);
	        }
	      }
	    }
	
	    return glide;
	  }

	  private static void checkAndInitializeGlide(@NonNull Context context) {
	    if (isInitializing) {
	      throw new IllegalStateException("You cannot call Glide.get() in registerComponents(),"
	          + " use the provided Glide instance instead");
	    }
	    isInitializing = true;
	    //调用initializeGlide
	    initializeGlide(context);
	    isInitializing = false;
	  }

	private static void initializeGlide(@NonNull Context context) {
		//初始化GlideBuilder对象
	    initializeGlide(context, new GlideBuilder());
	}

    private static void initializeGlide(Context context,GlideBuilder builder) {
	...
	//主要看build过程，通过glideBuilder构建Glide
    Glide glide = builder.build(applicationContext);

    ...
    Glide.glide = glide;
    }
	//-----------GlideBuilder 的build方法--------------
	@NonNull
    Glide build(@NonNull Context context) {
    if (sourceExecutor == null) {
      sourceExecutor = GlideExecutor.newSourceExecutor();//初始化加载资源的线程池
    }

    if (diskCacheExecutor == null) {
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();//初始化磁盘缓存线程池
    }

    if (animationExecutor == null) {
      animationExecutor = GlideExecutor.newAnimationExecutor();//初始化加载动画线程池
    }

    if (memorySizeCalculator == null) {
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();//内存大小计算
    }

    if (connectivityMonitorFactory == null) {
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();//默认创建网络接听的工厂
    }

    if (bitmapPool == null) {//位图缓存
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }

    if (arrayPool == null) {//数组缓存池
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }

    if (memoryCache == null) {//资源缓存
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {//磁盘缓存工厂类
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }

    if (engine == null) {//初始化engine
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              GlideExecutor.newAnimationExecutor(),
              isActiveResourceRetentionAllowed);
    }

    if (defaultRequestListeners == null) {
      defaultRequestListeners = Collections.emptyList();
    } else {
      defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }
	//初始化RequestManagerRetriever
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptions.lock(),
        defaultTransitionOptions,
        defaultRequestListeners,
        isLoggingRequestOriginsEnabled);
    }
	//-----------Glide 的有参构造方法--------------
	Glide(
      @NonNull Context context,
      @NonNull Engine engine,
      @NonNull MemoryCache memoryCache,
      @NonNull BitmapPool bitmapPool,
      @NonNull ArrayPool arrayPool,
      @NonNull RequestManagerRetriever requestManagerRetriever,
      @NonNull ConnectivityMonitorFactory connectivityMonitorFactory,
      int logLevel,
      @NonNull RequestOptions defaultRequestOptions,
      @NonNull Map<Class<?>, TransitionOptions<?, ?>> defaultTransitionOptions,
      @NonNull List<RequestListener<Object>> defaultRequestListeners,
      boolean isLoggingRequestOriginsEnabled) {
    this.engine = engine;
    this.bitmapPool = bitmapPool;
    this.arrayPool = arrayPool;
    this.memoryCache = memoryCache;
    this.requestManagerRetriever = requestManagerRetriever;
    this.connectivityMonitorFactory = connectivityMonitorFactory;

    DecodeFormat decodeFormat = defaultRequestOptions.getOptions().get(Downsampler.DECODE_FORMAT);
    bitmapPreFiller = new BitmapPreFiller(memoryCache, bitmapPool, decodeFormat);

    final Resources resources = context.getResources();

    registry = new Registry();
    registry.register(new DefaultImageHeaderParser());
    
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O_MR1) {
      registry.register(new ExifInterfaceImageHeaderParser());
    }

    List<ImageHeaderParser> imageHeaderParsers = registry.getImageHeaderParsers();
    Downsampler downsampler =
        new Downsampler(
            imageHeaderParsers,
            resources.getDisplayMetrics(),
            bitmapPool,
            arrayPool);
    ByteBufferGifDecoder byteBufferGifDecoder =
        new ByteBufferGifDecoder(context, imageHeaderParsers, bitmapPool, arrayPool);
    ResourceDecoder<ParcelFileDescriptor, Bitmap> parcelFileDescriptorVideoDecoder =
        VideoDecoder.parcel(bitmapPool);
    ByteBufferBitmapDecoder byteBufferBitmapDecoder = new ByteBufferBitmapDecoder(downsampler);
    StreamBitmapDecoder streamBitmapDecoder = new StreamBitmapDecoder(downsampler, arrayPool);
    ResourceDrawableDecoder resourceDrawableDecoder =
        new ResourceDrawableDecoder(context);
    ResourceLoader.StreamFactory resourceLoaderStreamFactory =
        new ResourceLoader.StreamFactory(resources);
    ResourceLoader.UriFactory resourceLoaderUriFactory =
        new ResourceLoader.UriFactory(resources);
    ResourceLoader.FileDescriptorFactory resourceLoaderFileDescriptorFactory =
        new ResourceLoader.FileDescriptorFactory(resources);
    ResourceLoader.AssetFileDescriptorFactory resourceLoaderAssetFileDescriptorFactory =
        new ResourceLoader.AssetFileDescriptorFactory(resources);
    BitmapEncoder bitmapEncoder = new BitmapEncoder(arrayPool);

    BitmapBytesTranscoder bitmapBytesTranscoder = new BitmapBytesTranscoder();
    GifDrawableBytesTranscoder gifDrawableBytesTranscoder = new GifDrawableBytesTranscoder();

    ContentResolver contentResolver = context.getContentResolver();

    registry
        .append(ByteBuffer.class, new ByteBufferEncoder())
        .append(InputStream.class, new StreamEncoder(arrayPool))
        /* Bitmaps */
        .append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)
        .append(Registry.BUCKET_BITMAP, InputStream.class, Bitmap.class, streamBitmapDecoder)
        .append(
            Registry.BUCKET_BITMAP,
            ParcelFileDescriptor.class,
            Bitmap.class,
            parcelFileDescriptorVideoDecoder)
        .append(
            Registry.BUCKET_BITMAP,
            AssetFileDescriptor.class,
            Bitmap.class,
            VideoDecoder.asset(bitmapPool))
        .append(Bitmap.class, Bitmap.class, UnitModelLoader.Factory.<Bitmap>getInstance())
        .append(
            Registry.BUCKET_BITMAP, Bitmap.class, Bitmap.class, new UnitBitmapDecoder())
        .append(Bitmap.class, bitmapEncoder)
        /* BitmapDrawables */
        .append(
            Registry.BUCKET_BITMAP_DRAWABLE,
            ByteBuffer.class,
            BitmapDrawable.class,
            new BitmapDrawableDecoder<>(resources, byteBufferBitmapDecoder))
        .append(
            Registry.BUCKET_BITMAP_DRAWABLE,
            InputStream.class,
            BitmapDrawable.class,
            new BitmapDrawableDecoder<>(resources, streamBitmapDecoder))
        .append(
            Registry.BUCKET_BITMAP_DRAWABLE,
            ParcelFileDescriptor.class,
            BitmapDrawable.class,
            new BitmapDrawableDecoder<>(resources, parcelFileDescriptorVideoDecoder))
        .append(BitmapDrawable.class, new BitmapDrawableEncoder(bitmapPool, bitmapEncoder))
        /* GIFs */
        .append(
            Registry.BUCKET_GIF,
            InputStream.class,
            GifDrawable.class,
            new StreamGifDecoder(imageHeaderParsers, byteBufferGifDecoder, arrayPool))
        .append(Registry.BUCKET_GIF, ByteBuffer.class, GifDrawable.class, byteBufferGifDecoder)
        .append(GifDrawable.class, new GifDrawableEncoder())
        /* GIF Frames */
        // Compilation with Gradle requires the type to be specified for UnitModelLoader here.
        .append(
            GifDecoder.class, GifDecoder.class, UnitModelLoader.Factory.<GifDecoder>getInstance())
        .append(
            Registry.BUCKET_BITMAP,
            GifDecoder.class,
            Bitmap.class,
            new GifFrameResourceDecoder(bitmapPool))
        /* Drawables */
        .append(Uri.class, Drawable.class, resourceDrawableDecoder)
        .append(
            Uri.class, Bitmap.class, new ResourceBitmapDecoder(resourceDrawableDecoder, bitmapPool))
        /* Files */
        .register(new ByteBufferRewinder.Factory())
        .append(File.class, ByteBuffer.class, new ByteBufferFileLoader.Factory())
        .append(File.class, InputStream.class, new FileLoader.StreamFactory())
        .append(File.class, File.class, new FileDecoder())
        .append(File.class, ParcelFileDescriptor.class, new FileLoader.FileDescriptorFactory())
        // Compilation with Gradle requires the type to be specified for UnitModelLoader here.
        .append(File.class, File.class, UnitModelLoader.Factory.<File>getInstance())
        /* Models */
        .register(new InputStreamRewinder.Factory(arrayPool))
        .append(int.class, InputStream.class, resourceLoaderStreamFactory)
        .append(
            int.class,
            ParcelFileDescriptor.class,
            resourceLoaderFileDescriptorFactory)
        .append(Integer.class, InputStream.class, resourceLoaderStreamFactory)
        .append(
            Integer.class,
            ParcelFileDescriptor.class,
            resourceLoaderFileDescriptorFactory)
        .append(Integer.class, Uri.class, resourceLoaderUriFactory)
        .append(
            int.class,
            AssetFileDescriptor.class,
            resourceLoaderAssetFileDescriptorFactory)
        .append(
            Integer.class,
            AssetFileDescriptor.class,
            resourceLoaderAssetFileDescriptorFactory)
        .append(int.class, Uri.class, resourceLoaderUriFactory)
        .append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
        .append(Uri.class, InputStream.class, new DataUrlLoader.StreamFactory<Uri>())
        .append(String.class, InputStream.class, new StringLoader.StreamFactory())
        .append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())
        .append(
            String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())
        .append(Uri.class, InputStream.class, new HttpUriLoader.Factory())
        .append(Uri.class, InputStream.class, new AssetUriLoader.StreamFactory(context.getAssets()))
        .append(
            Uri.class,
            ParcelFileDescriptor.class,
            new AssetUriLoader.FileDescriptorFactory(context.getAssets()))
        .append(Uri.class, InputStream.class, new MediaStoreImageThumbLoader.Factory(context))
        .append(Uri.class, InputStream.class, new MediaStoreVideoThumbLoader.Factory(context))
        .append(
            Uri.class,
            InputStream.class,
            new UriLoader.StreamFactory(contentResolver))
        .append(
            Uri.class,
            ParcelFileDescriptor.class,
             new UriLoader.FileDescriptorFactory(contentResolver))
        .append(
            Uri.class,
            AssetFileDescriptor.class,
            new UriLoader.AssetFileDescriptorFactory(contentResolver))
        .append(Uri.class, InputStream.class, new UrlUriLoader.StreamFactory())
        .append(URL.class, InputStream.class, new UrlLoader.StreamFactory())
        .append(Uri.class, File.class, new MediaStoreFileLoader.Factory(context))
        .append(GlideUrl.class, InputStream.class, new HttpGlideUrlLoader.Factory())
        .append(byte[].class, ByteBuffer.class, new ByteArrayLoader.ByteBufferFactory())
        .append(byte[].class, InputStream.class, new ByteArrayLoader.StreamFactory())
        .append(Uri.class, Uri.class, UnitModelLoader.Factory.<Uri>getInstance())
        .append(Drawable.class, Drawable.class, UnitModelLoader.Factory.<Drawable>getInstance())
        .append(Drawable.class, Drawable.class, new UnitDrawableDecoder())
        /* Transcoders */
        .register(
            Bitmap.class,
            BitmapDrawable.class,
            new BitmapDrawableTranscoder(resources))
        .register(Bitmap.class, byte[].class, bitmapBytesTranscoder)
        .register(
            Drawable.class,
            byte[].class,
            new DrawableBytesTranscoder(
                bitmapPool, bitmapBytesTranscoder, gifDrawableBytesTranscoder))
        .register(GifDrawable.class, byte[].class, gifDrawableBytesTranscoder);

    ImageViewTargetFactory imageViewTargetFactory = new ImageViewTargetFactory();
    glideContext =
        new GlideContext(
            context,
            arrayPool,
            registry,
            imageViewTargetFactory,
            defaultRequestOptions,
            defaultTransitionOptions,
            defaultRequestListeners,
            engine,
            isLoggingRequestOriginsEnabled,
            logLevel);
  }
  
 将GlideBuider的 build 方法分两部分理解下：
 

 1. GlideBuilder 的build方法
我们看到在这个方法里主要进行的操作有资源加载线程池，动画加载线程池，本地缓存线程池、网络监听工厂、bitmapPool、arrayPool、memoryCache、engine、defaultTransitionOptions等初始化

 2. Glide的构造方法
DecodeFormat配置、Registry初始化、各种Decoder初始化，注册、GlideContext初始化等。

至此Glide.get执行完毕，我们接着看第二步的getRequestManagerRetriever（）的调用
返回了一个RequestManagerRetriever对象，紧接着执行with方法的第二步RequestManagerRetriever的get(activity);

	   //RequestManagerRetriever.java
	  @SuppressWarnings("deprecation")
	  @NonNull
	  public RequestManager get(@NonNull Activity activity) {
	    if (Util.isOnBackgroundThread()) {//后台线程
	      return get(activity.getApplicationContext());
	    } else {
	      assertNotDestroyed(activity);
	      //获取当前activity的FragmentManager对象
	      android.app.FragmentManager fm = activity.getFragmentManager();
	      return fragmentGet(
	          activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
	    }
	  }
	  //你要找的fragmentGet方法
	  @SuppressWarnings({"deprecation", "DeprecatedIsStillUsed"})
	  @Deprecated
	  @NonNull
	  private RequestManager fragmentGet(@NonNull Context context,
	      @NonNull android.app.FragmentManager fm,
	      @Nullable android.app.Fragment parentHint,
	      boolean isParentVisible) {
	      //获取当前Activity对应的RequestManagerFragment 
	    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
	    //得到请求管理类
	    RequestManager requestManager = current.getRequestManager();
	    if (requestManager == null) {
	      Glide glide = Glide.get(context);
	      //如果请求是null,通过RequestManagerFactory生产，看源码知道其实就是相当于new RequestManager
	      requestManager =
	          factory.build(
	              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
	       //将requestManager添加到RequestManagerFragment 中
	      current.setRequestManager(requestManager);
	    }
	    //返回一个请求管理类
	    return requestManager;
	  }


至此我们的请求构建完毕！附上简单的时序图，内部省略与流程无关步骤。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190605192717333.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2NjI4MzI5,size_16,color_FFFFFF,t_70)
总结一下吧：我们使用Glide的时候，通过调用Glide.with方法，它内部帮助我们创建一个GlideBuilder对象，然后通过它的build方法，初始化一下线程池，缓存等，然后通过new Glide 对Registry进行初始化，并注册一些默认的配置，池，工厂，缓存等等然后通过getRequestManagerRetriever返回一个RequestManagerRetriever对象，通过这个对象的get(activity)方法给当前的activity初始化一个RequestManagerFragment对象，和RequestManager对象。至此请求构建流程结束。
