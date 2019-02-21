# LeakCanary原理浅析
## 1.LeakCanary简介
LeakCanary是一个Android和Java的内存泄漏检测库，可以大幅可以大幅度减少了开发中遇到的OOM问题。</br>

## 2.如何使用LeakCanary
使用LeakCanary非常简单，只需要在Application的onCreate()方法里面调用LeakCanary.install(this)方法，就像下面一样：
   
           public class ExampleApplication extends Application {
            @Override 
            public void onCreate() {
            super.onCreate();
            // 如果是在HeapAnalyzer进程里，则返回，因为该进程是专门用来堆内存分析的。
            if (LeakCanary.isInAnalyzerProcess(this)) {
            // This process is dedicated to LeakCanary for heap analysis.
            // You should not init your app in this process.
            return;
            }
            //调用LeakCanary.install()的方法来进行必要的初始化工作，来监听内存泄漏。
            LeakCanary.install(this);
            // Normal app init code...
            }
            }

如果只想检测Activity的内存泄漏，而且只想使用默认的报告方式，则只需按照上面的实例，在Application里面添加一行代码：
LeakCanary.install(this);</br>

如果想监控Fragment等其他有生命周期函数的方法的类是否有内存泄漏时，可以先返回一个RefWatcher对象，然后通过RefWatcher监控那些本该被回收的对象。使用示例如下：</br>

    public class ExampleApplication extends Application {
    public static RefWatcher getRefWatcher(Context context) {
    ExampleApplication application = (ExampleApplication) context.getApplicationContext();
    return application.refWatcher;
    }
    private RefWatcher refWatcher;
    @Override public void onCreate() {
    super.onCreate();
    refWatcher = LeakCanary.install(this);
    }
    }

使用RefWatcher监控Fragment：

    public abstract class BaseFragment extends Fragment {
    @Override 
    public void onDestroy() {
    super.onDestroy();
    RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    refWatcher.watch(this);
    }
    }

可以看到，不管哪种方式，都需要调用LeakCanary.install()方法，因此接下来的源码分析将从这个方法开始。
## 3.LeakCanary源码分析
第一部分:初始化工作

接下来将从LeakCanary的install()方法开始分析LeakCanary的工作流程。
3.1 LeakCanary.install()

    /*
    * 创建一个RefWatcher，用来监控Activity的引用
    */
    public static RefWatcher install(Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
    .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
    .buildAndInstall();//见3.2,3.3,3.4
    }

    refWatcher(application)方法返回的是一个AndroidRefWatcherBuilder，通过这个Builder来配置RefWatcher的一些设置参数。
    public static AndroidRefWatcherBuilder refWatcher(Context context) {
    return new AndroidRefWatcherBuilder(context);
    }

3.2  AndroidRefWatcherBuilder.listenerServiceClass()

    /*
    * 设置一个自定义的AbstractAnalysisResultService服务来监听内存分析结果
    */
    public AndroidRefWatcherBuilder listenerServiceClass(
    Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
    }

    //通过ServiceHeapDumpListener类来保存listenerServiceClass服务，然后再通过RefWatcherBuilder来保存HeapDump的listener。
    public ServiceHeapDumpListener(Context context,
    Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabled(context, listenerServiceClass, true);
    setEnabled(context, HeapAnalyzerService.class, true);
    this.listenerServiceClass = checkNotNull(listenerServiceClass, "listenerServiceClass");//保存listenerServiceClass
    this.context = checkNotNull(context, "context").getApplicationContext();
    }

    public final T heapDumpListener(HeapDump.Listener heapDumpListener) {
    this.heapDumpListener = heapDumpListener;//保存HeapDump的listener
    return self();
    }

可以看到，listenerServiceClass()方法主要保存监听内存分析结果的listener，当内存分析结果完成后，则回调该listener，在这里这个listener是DisplayLeakService服务。</br>
3.3 RefWatcherBuilder.excludedRefs()

    /*
    * 该方法主要是排除一些开发可以忽略的泄漏路径(一般是系统级别BUG)，这些枚举在AndroidExcludedRefs这个类当中定义 
    */
    public final T excludedRefs(ExcludedRefs excludedRefs) {
    this.excludedRefs = excludedRefs;
    return self();
    }

由于AndroidRefWatcherBuilder继承自RefWatcherBuilder，excludedRefs()方法只在RefWatcherBuilder类中定义，因此调用的是RefWatcherBuilder.excludedRefs()方法，该方法的主要作用是告诉需要忽略哪些已知的系统泄漏Bug，避免这些结果产生干扰。
</br>
3.4 AndroidRefWatcherBuilder.buildAndInstall()

    /*
    * 创建一个RefWatcher实例，并且开始监控Activity实例引用
    */
    public RefWatcher buildAndInstall() {
    RefWatcher refWatcher = build();//构建一个RefWatcher，见3.5
    // 如果RefWatcher可用，则让泄漏通知显示服务生效，并且注册监听。
    if (refWatcher != DISABLED) {
    LeakCanary.enableDisplayLeakActivity(context);
    ActivityRefWatcher.install((Application) context, refWatcher);//注册监听，见3.6
    }
    return refWatcher;
    }

该方法的主要作用是构建一个RefWatcher类，并判断RefWatcher是否可用，如果可用的话，则让泄漏通知显示服务生效，并且注册监听。</br>
3.5 RefWatcherBuilder.build()

    /*
    * 构建一个RefWatcher对象
    */
    public final RefWatcher build() {
    //如果禁用了，则返回
    if (isDisabled()) {
    return RefWatcher.DISABLED;
    }
    // 保留被排出系统的泄漏Bug
    ExcludedRefs excludedRefs = this.excludedRefs;
    if (excludedRefs == null) {
    excludedRefs = defaultExcludedRefs();
    }
    // 保存HeapDump的listener，监听内存分析结果
    HeapDump.Listener heapDumpListener = this.heapDumpListener;
    if (heapDumpListener == null) {
    heapDumpListener = defaultHeapDumpListener();
    }
    // 保存调试控制，如果是处理调试阶段，则忽略内存泄漏监控
    DebuggerControl debuggerControl = this.debuggerControl;
    if (debuggerControl == null) {
    debuggerControl = defaultDebuggerControl();
    }
    // 保存HeapDumper，如果没有定义，则使用默认的HeapDumper，为AndroidHeapDumper。
    HeapDumper heapDumper = this.heapDumper;
    if (heapDumper == null) {
    heapDumper = defaultHeapDumper();
    }
    // 保存WatchExecutor，如果没有定义，则使用默认的WatchExecutor，为AndroidWatchExecutor。
    WatchExecutor watchExecutor = this.watchExecutor;
    if (watchExecutor == null) {
    watchExecutor = defaultWatchExecutor();
    }

    // 保存GcTrigger，如果没有定义，则使用默认的GcTrigger，为GcTrigger.DEFAULT
    GcTrigger gcTrigger = this.gcTrigger;
    if (gcTrigger == null) {
    gcTrigger = defaultGcTrigger();
    }

    return new RefWatcher(watchExecutor, debuggerControl, gcTrigger, heapDumper, heapDumpListener,
    excludedRefs);//调用RefWatcher的构造方法，保留设置的参数
    }

    RefWatcher(WatchExecutor watchExecutor, DebuggerControl debuggerControl, GcTrigger gcTrigger,
    HeapDumper heapDumper, HeapDump.Listener heapdumpListener, ExcludedRefs excludedRefs) {
    this.watchExecutor = checkNotNull(watchExecutor, "watchExecutor");
    this.debuggerControl = checkNotNull(debuggerControl, "debuggerControl");
    this.gcTrigger = checkNotNull(gcTrigger, "gcTrigger");
    this.heapDumper = checkNotNull(heapDumper, "heapDumper");
    this.heapdumpListener = checkNotNull(heapdumpListener, "heapdumpListener");
    this.excludedRefs = checkNotNull(excludedRefs, "excludedRefs");
    retainedKeys = new CopyOnWriteArraySet<>();//保存key的集合
    queue = new ReferenceQueue<>();//引用队列
    }

可以看到build()方法主要是根据之前设置的一些参数来构建RefWatcher对象，如果有参数没有初始化，则使用默认的配置参数来初始化。在RefWatcher中，还有两个比较关键的成员，分别是retainedKeys和queue，他们在后面判断一个对象是否泄漏将会被用到。</br>
3.6 ActivityRefWatcher.install()

    /*
    * 注册监听
    */
    public static void install(Application application, RefWatcher refWatcher) {
    new ActivityRefWatcher(application, refWatcher).watchActivities();//见3.7
    }

    该方法首先是创建一个ActivityRefWatcher对象，该对象用来确保Activity被销毁的时候不会泄漏。接着调用watchActivities()来监控Activity实例是否泄漏了。
    public ActivityRefWatcher(Application application, RefWatcher refWatcher) {
    this.application = checkNotNull(application, "application");
    this.refWatcher = checkNotNull(refWatcher, "refWatcher");
    }

3.7 ActivityRefWatcher.watchActivities()

    public void watchActivities() {
    // Make sure you don't get installed twice.
    stopWatchingActivities();//避免两次重复注册
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);//调用Application的registerActivityLifecycleCallbacks()方法，该方法只在Android 4.0以后才支持，见3.8
    }

watchActivities()方法主要是调用Application的registerActivityLifecycleCallbacks()方法来注册生命周期回调监听，在Android 4.0以后，Application才提供registerActivityLifecycleCallbacks()方法来注册监听Activity的生命周期函数，因此在Android 4.0以后，则可以通过LeakCanary.install(Context)就完成注册监听过程。如果是Android 4.0以前，则需要先获取RefWatcher对象，然后在Activity的onDestroyed()方法里，主要去调用监听RefWatcher.watch()方法。</br>
3.8 Application.registerActivityLifecycleCallbacks()

    /*
    * 注册Activity生命周期方法调用回调接口
    */
    public void registerActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback) {
    synchronized (mActivityLifecycleCallbacks) {
    mActivityLifecycleCallbacks.add(callback);
    }
    }

    public interface ActivityLifecycleCallbacks {
    void onActivityCreated(Activity activity, Bundle savedInstanceState);//Activity执行onCreate()时回调
    void onActivityStarted(Activity activity);//Activity执行onStart()时回调
    void onActivityResumed(Activity activity);//Activity执行onResume()时回调
    void onActivityPaused(Activity activity);//Activity执行onPause()时回调
    void onActivityStopped(Activity activity);//Activity执行onStop()时回调
    void onActivitySaveInstanceState(Activity activity, Bundle outState);//Activity执行onSaveInstanceState(()时回调
    void onActivityDestroyed(Activity activity);//Activity执行onDestroy()时回调
    }

    在ActivityRefWatcher中，定义了ActivityLifecycleCallbacks回调接口，具体如下：
    private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
    new Application.ActivityLifecycleCallbacks() {
    @Override 
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
    }

    @Override 
    public void onActivityStarted(Activity activity) {
    }

    @Override 
    public void onActivityResumed(Activity activity) {
    }

    @Override 
    public void onActivityPaused(Activity activity) {
    }

    @Override 
    public void onActivityStopped(Activity activity) {
    }

    @Override 
    public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
    }

    @Override 
    public void onActivityDestroyed(Activity activity) {
    ActivityRefWatcher.this.onActivityDestroyed(activity);//见3.9
    }
    };

可以看到，在ActivityRefWatcher中的定义的lifecycleCallbacks，只实现了onActivityDestroyed的周期回调方法，该方法是在Activity执行onDestroy()方法时，被回掉的。因此ActivityRefWatcher是在Activity被销毁时，开始工作的，监控Activity是否被销毁了。</br>
3.9 ActivityRefWatcher.onActivityDestroyed()

    void onActivityDestroyed(Activity activity) {
    refWatcher.watch(activity);
    }

可以看到，最终的监控还是通过RefWatcher的watch()的方法来监控是否发生了泄漏。监控的时机是Activity被销毁的时候，此时开始监控该引用对象是否被回收了，从而判断是否泄漏了。</br>
小结</br>
从LeakCanary的install()方法可以看到，它主要初始化一些参数，为监控对象是否发生了泄漏做准备工作。其主要的工作有：</br>

通过Builder模式，构建一个RefWatcher对象，RefWatcher对象里面包含了很多内存泄漏相关的辅助类。</br>
通过Application的registerActivityLifecycleCallbacks()方法注册生命周期回调接口，可以让RefWatcher在Activity被销毁的时候,开始监控Activity引用是否发生了内存泄漏。</br>

需要说明的是，Application的registerActivityLifecycleCallbacks()方法是在Android 4.0以后才提供的，因此在Android 4.0以后，可以直接使用LeakCanary.install()来监听Activity实例是否泄漏了。如果是Android 4.0以前，或者是需要监听其他有生命周期的对象是否发生了内存泄漏，则需要先通过LeakCanary.install()方法返回一个RefWatcher对象，然后通过RefWatcher对象的watch()方法来监控对象是否发生了内存泄漏。例如：</br>

    RefWatcher refWatcher = LeakCanary.install(context);
    // 监控
    refWatcher.watch(objReference);


## 4.监控内存泄漏</br>

从上面的分析可以知道，内存泄漏监控是通过RefWatcher的watch()方法开始的，因此，接下来将从RefWatcher的watch()方法进行分析监控内存泄漏的原理。</br>
4.1 RefWatcher.watch()

    public void watch(Object watchedReference) {
    watch(watchedReference, "");
    }

    /*
    * 监控提供的引用对象，检查它是否能被垃圾回收；
    * 该方法是非阻塞的，检查操作是由WatchExecutor执行器来执行异步检查
    */
    public void watch(Object watchedReference, String referenceName) {
    // 如果RefWatcher被禁用了，则直接返回
    if (this == DISABLED) {
    return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();//记录开始监控的时间
    String key = UUID.randomUUID().toString();//生成一个唯一的key，后续判断一个对象是否泄漏了需要使用到该可以
    retainedKeys.add(key);//将可以添加到一个set集合中，retainedKeys在初始化RefWatcher的时候构建的，可以见3.5
    final KeyedWeakReference reference =
    new KeyedWeakReference(watchedReference, key, referenceName, queue);//构建一个KeyedWeakReference对象，见4.2

    ensureGoneAsync(watchStartNanoTime, reference);//异步检查引用对象是否被回收了，见4.3

    }
在RefWatcher的watch()方法中：</br>

首先随机生成一个唯一的key，这个key用来在后面确定一个对象是否泄漏了；</br>
接着创建一个KeyedWeakReference对象，该对象包含了key，引用对象队列queue以及引用对象，KeyedWeakReference继承了WeakReference，也是弱引用对象；</br>
最后调用异步方法ensureGoneAsync()检查引用对象是否被回收了，从而判断是否发生了内存泄漏；</br>

4.2 KeyedWeakReference()

    final class KeyedWeakReference extends WeakReference<Object> {
    public final String key;
    public final String name;

    KeyedWeakReference(Object referent, String key, String name,
    ReferenceQueue<Object> referenceQueue) {
    super(checkNotNull(referent, "referent"), checkNotNull(referenceQueue, "referenceQueue"));
    this.key = checkNotNull(key, "key");//保存key
    this.name = checkNotNull(name, "name");
    }
    }

    KeyedWeakReference继承了WeakReference类，调用WeakReference类的构造方法将引用对象referent和引用队列referenceQueue传递给WeakReference。
    /*
    * 创建一个新的弱引用，该弱引用指向指定的对象和注册到引用对象队列中
    */
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
    super(referent, q);//调用Reference的构造方法
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
    this.referent = referent;
    this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }

引用(Reference)为垃圾收集提供了更大的灵活性，根据不同的适用场景，引用可以分为4类：</br>

强引用(Strong Reference)</br>

强引用就是指在程序代码中普遍存在的，类似于“Object obj = new Object()”这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象；</br>

软引用(Soft Reference)——SoftReference</br>

软引用是用来描述一些还有用但并非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。(只有内存不足时，才会回收软引用对象)。</br>

弱引用(Weak Reference)——WeakReference</br>

弱引用也是用来描述非必需对象的，但它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存释放足够，都会回收掉只被弱引用关联的对象。(只要发生了垃圾收集，弱引用对象都会被回收)</br>

虚引用(Phantom Reference)——PhantomReference</br>

虚引用也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用获取一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。
当使用弱引用(WeakReference)时，只要发生了垃圾回收GC，则不管内存是否充足时，都会将该对象进行回收。如果WeakReference定义了引用队列(ReferenceQueue)，则还会将这个WeakReference放入与之关联的ReferenceQueue中。因此，每次KeyedWeakReference对象被回收时，都会被放入到与之关联的引用队列queue中。</br>
KeyedWeakReference类是一个弱引用，指向了需要监控的引用对象watchedReference，并且还关联了一个ReferenceQueue引用队列，这个引用队列是在构建RefWatcher类的时候创建的，保存在queue变量中。另外，在KeyedWeakReference类中，还保存了一个key，这个key是与被监控的引用对象watchedReference相关联的，因为这个key是一个随机生成的一个唯一的key，因此可以通过key定位到引用对象watchedReference。</br>
4.3 RefWatcher.ensureGoneAsync()

    private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {//见4.4
    @Override 
    public Retryable.Result run() {
    return ensureGone(reference, watchStartNanoTime);//见4.9
    }
    });
    }

    Retryable是一个接口，定义了一个可以重试的工作单元。
    public interface Retryable {

    enum Result {
    DONE, RETRY
    }

    Result run();
    }

WatchExecutor也是一个接口，里面定义了execute()方法，主要是用来执行Retryable。

    public interface WatchExecutor {
    //空实现
    WatchExecutor NONE = new WatchExecutor() {
    @Override 
    public void execute(Retryable retryable) {
    }
    };

    void execute(Retryable retryable);//定义了execute()接口
    }

    WatchExecutor在构建RefWatcher的时候被初始化了，默认被初始化为AndroidWatchExecutor。
    @Override 
    protected WatchExecutor defaultWatchExecutor() {
    return new AndroidWatchExecutor(DEFAULT_WATCH_DELAY_MILLIS);
    }

    public AndroidWatchExecutor(long initialDelayMillis) {
    mainHandler = new Handler(Looper.getMainLooper());//主线程的Handler
    HandlerThread handlerThread = new HandlerThread(LEAK_CANARY_THREAD_NAME);
    handlerThread.start();//启动一个LeakCanary-Heap-Dump线程
    backgroundHandler = new Handler(handlerThread.getLooper());//子线程Handler
    this.initialDelayMillis = initialDelayMillis;//初始化延迟时间，默认为5s
    maxBackoffFactor = Long.MAX_VALUE / initialDelayMillis;
    }

在AndroidWatchExecutor中初始化了两个Handler，一个是主线程相关的Handler，另外是一个子线程的Handler。</br>
4.4 AndroidWatchExecutor.execute()

    @Override 
    public void execute(Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
    waitForIdle(retryable, 0);// 如果是主线程，则等待消息队列空闲时调用，见4.5
    } else {
    postWaitForIdle(retryable, 0);// 如果是子线程，则延迟指定延迟执行，见4.8
    }
    }

AndroidWatchExecutor会根据线程是在主线程还是子线程，决定调用waitForIdle()还是postWaitForIdle()方法。</br>
因为在主线程的话，需要等待消息队列空闲的时候才会调用，这样不会干扰应用的正常运行。如果是在子线程中，则可以等待一段时间后再执行，不管是通过哪种方法，最后都是通过子线程延迟一段时间来执行。</br>
4.5 AndroidWatchExecutor.waitForIdle()

    void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    // 创建一个IdleHandler，将其放入MessageQueue中
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override 
    public boolean queueIdle() {
    postToBackgroundWithDelay(retryable, failedAttempts);//见4.6
    return false;//返回false，移除该IdleHandler，确保只执行一次
    }
    });
    }

当通过MessageQueue注册了一个IdleHandler，那么当消息队列空闲时，则会回调其queueIdle()方法，然后调用postToBackgroundWithDelay()方法延迟执行。</br>
4.6 AndroidWatchExecutor.postToBackgroundWithDelay()

    void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);//指数回退因素
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;//延迟时间
    backgroundHandler.postDelayed(new Runnable() {//将消息放入后台线程中执行
    @Override 
    public void run() {
    Retryable.Result result = retryable.run();//调用Retryable的run方法
    // 如果是需要重试的话，则尝试再次等待主线程空闲时，再由子线程尝试执行
    if (result == RETRY) {
    postWaitForIdle(retryable, failedAttempts + 1);
    }
    }
    }, delayMillis);
    }

在postToBackgroundWithDelay()方法中，主要是计算延迟执行时间，然后执行Retryable中定义的run()方法，如果执行结果是需要重新执行的话，则再通过postWaitForIdle()方法，等待主线程空闲后，调用子线程延迟执行。</br>
每次重试延迟的时间都不一样，延迟时间是呈指数增长的，例如第一次延迟时间为5s = 1 * 5，那么第二次延迟的时间就为10s = 2 * 5，第三次延迟时间就是为20s = 4 * 5;</br>
4.7 AndroidWatchExecutor.postWaitForIdle()

    void postWaitForIdle(final Retryable retryable, final int failedAttempts) {
    mainHandler.post(new Runnable() {
    @Override 
    public void run() {
    waitForIdle(retryable, failedAttempts);//等待主线程空闲后，调用子线程延迟执行。
    }
    });
    }

可以看到，postWaitForIdle()方法，主要是向主线程的消息队列发送一个消息，然后等待主线程的消息队列空闲，当主线程消息队列空闲时，则调用子线程延迟一段时间来执行任务。</br>
回调4.3中，我们可以知道，ensureGone()方法是在子线程中执行，并且根据执行结果来决定是否需要稍后重试执行，并且稍后重试执行的时间每次都不一样。</br>
4.8 RefWatcher.ensureGone()

    Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();// GC开始的时间
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);//记录监控的时长

    removeWeaklyReachableReferences();//尝试从集合中移除弱引用，见4.9

    // 如果是在调试，则直接返回
    if (debuggerControl.isDebuggerAttached()) {
    // The debugger can create false leaks.
    return RETRY;
    }

    if (gone(reference)) {//判断引用是否被回收了，见4.10
    return DONE;
    }

    gcTrigger.runGc();//主动触发垃圾回收，见4.11
    removeWeaklyReachableReferences();//再一次尝试从集合中移除弱引用

    if (!gone(reference)) {//弱引用没有被回收，说明发生了内存泄漏
    long startDumpHeap = System.nanoTime();//记录Dump Heap的开始时间
    long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);//记录GC所花费的时间

    File heapDumpFile = heapDumper.dumpHeap();//开始Dump Heap，见4.12
    if (heapDumpFile == RETRY_LATER) {//如果当前不能Dump Heap，则稍后重试
    // Could not dump the heap.
    return RETRY;
    }
    long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);//记录Dump Heap所花费的时间
    // 分析Dump Heap的结果
    heapdumpListener.analyze(
    new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
    gcDurationMs, heapDumpDurationMs));//见4.15
    }
    return DONE;//返回执行完成
    }

ensureGone()方法是监控内存泄漏的关键方法，其主要的逻辑如下：</br>

记录GC开始的时间，并且计算监控了多长时间；</br>
第一次调用removeWeaklyReachableReferences()方法，尝试从集合中移除弱引用，如果弱引用在引用队列中，则说明弱引用对象被回收了；</br>
第一次调用gone()方法，判断弱引用是否被回收了，如果被回收了，则说明没有发生内存泄漏，执行完成，直接返回； </br>
如果判断弱引用还没有被回收，则调用gcTrigger.runGc()方法主动触发一次GC操作；</br>
第二次调用removeWeaklyReachableReferences()方法，再次尝试从集合中移除弱引用；</br>
第二次调用gone()方法，判断弱引用是否被回收了；</br>
如果弱引用对象没有被回收，则说明了发生了内存泄漏，则开始处理内存泄漏的流程：</br>

记录堆转储(Dump Heap)的起始时间，以及GC所花费的时间；</br>
获取堆转储文件HeapDumpFile;</br>
记录堆转储所花费的时间；</br>
调用heapdumpListener中的analyze()方法来分析堆转储文件HeapDumpFile；</br>



可以看到，关键是通过判断引用队列中是否有弱引用对象，来判断是否发生了内存泄漏。如果发生了内存泄漏，则通过Dump Heap来获取堆转储文件，然后再通过分析堆转储文件来得出内存泄漏的引用链路。</br>
4.9 RefWatcher.removeWeaklyReachableReferences()

    private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    // 当弱引用对象可以被回收时，它会被放入引用队列中
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
    retainedKeys.remove(ref.key);//根据key值移除set集合中存在的对应key
    }

    }
在该方法中，首先从引用队列中获取一个弱引用对象，如果存在弱引用对象，则说明弱引用对象指向的引用对象是可以被回收的。因为retainedKeys是一个set集合，在构建RefWatcher类的时候进行了初始化，并且在调用RefWatcher的watch()方法时将需要监控的引用对象对应的key添加到retainedKeys集合中。因此，在调用retainedKeys的remove()方法之前，该引用对象对应的key将一直存在。当引用队列中存在弱引用对象，则将在retainedKeys中移除该弱引用对象对应的key。</br>
我们可以把retainedKeys集合看作是还没有被垃圾回收的引用对象的key集合，因此接下来的gone()方法主要是通过判断
retainedKey集合中是否还保存有引用对象对应的key。</br>
4.10 RefWatcher.gone()

    private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
    }

gone()方法主要是判断retainedKeys集合中是否还包含引用对象对应的key，如果不存在，则说明该引用对象以及被回收了，没有发生内存泄漏。如果存在，则说明发生了内存泄漏。  </br>
4.11 GcTrigger.gc()

    public interface GcTrigger {
    // 默认实现
    GcTrigger DEFAULT = new GcTrigger() {
    @Override 
    public void runGc() {
    // Code taken from AOSP FinalizationTest:
    // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
    // java/lang/ref/FinalizationTester.java
    // System.gc() does not garbage collect every time. Runtime.gc() is
    // more likely to perfom a gc.
    Runtime.getRuntime().gc();//调用Runtime.gc()来执行GC操作
    enqueueReferences();//等待100ms中，等待弱引用对象进入引用队列中
    System.runFinalization();//执行对象的finalize()方法
    }

    private void enqueueReferences() {
    // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
    // references to the appropriate queues.
    try {
    Thread.sleep(100);
    } catch (InterruptedException e) {
    throw new AssertionError();
    }
    }
    };

    void runGc();
    }

GcTrigger是一个接口类，里面定义而来runGc()方法，GcTrigger接口里面还定义了一个默认实现DEFAULT。通过GcTrigger的runGc()方法可以主动触发GC操作，这样可以被回收的弱引用对象将会被放入引用队列中，这样后续就可以检查引用队列来判断是否回收成功了。</br>
在GcTrigger的默认实现中，是通过Runtime.gc()方法将执行垃圾回收的，因为System.gc()不能保证每次都执行垃圾收集。另外，为了确保弱引用对象被放入引用队列中，需要每次垃圾收集后，等待100ms，让弱引用有足够时间放入到引用队列中。最后在通过System.runFinalization()方法执行引用对象的finalize()方法。</br>
4.12 AndroidHeapDumper.dumpHeap()

    @Override 
    public File dumpHeap() {
    File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();//创建一个dump文件，文件名是以_pending.hporf为结尾的

    if (heapDumpFile == RETRY_LATER) {
    return RETRY_LATER;//稍后重试
    }

    FutureResult<Toast> waitingForToast = new FutureResult<>();
    showToast(waitingForToast);//发送一个Dump Heap的等待Toast提示，见4.13

    if (!waitingForToast.wait(5, SECONDS)) {//等待5s种，如果5s内没有收到Toast，则说明当前还不能Dump Heap，需要稍后重试。
    CanaryLog.d("Did not dump heap, too much time waiting for Toast.");
    return RETRY_LATER;
    }

    Toast toast = waitingForToast.get();//获取Toast结果
    try {
    Debug.dumpHprofData(heapDumpFile.getAbsolutePath());//调用Debug类的dumpHprofData()方法，来Dump Heap，见4.14
    cancelToast(toast);//取消通知
    return heapDumpFile;// 返回HeapDumpFile文件
    } catch (Exception e) {
    CanaryLog.d(e, "Could not dump heap");
    // Abort heap dump
    return RETRY_LATER;
    }
    }

    heapDumper是在构造RefWatcher的时候初始化的，默认使用的AndroidHeapDumper类，通过defaultHeapDumper()方法来获取的。
    @Override 
    protected HeapDumper defaultHeapDumper() {
    LeakDirectoryProvider leakDirectoryProvider = new DefaultLeakDirectoryProvider(context);
    return new AndroidHeapDumper(context, leakDirectoryProvider);
    }

其中LeakDirectoryProvider主要是负责Dump Heap文件的创建以及文件的命名等工作。所有Dump Heap文件都是以_pending.hporf为结尾的，并且最多可以存储7个Dump Heap文件。</br>
在开始执行dumpHeap()之前，会先通过发送一个Toast提示需要执行Dump Heap的操作，待主线程的消息队列空闲后，在将结果通知子线程，可以开始Dump Heap操作，子线程会等待5s中，等待主线程通知是否可以Dump Heap操作。具体是showToast()方法和FutureResult来实现的。</br>
4.13 AndroidHeapDumper.showToast()

    private void showToast(final FutureResult<Toast> waitingForToast) {
    mainHandler.post(new Runnable() {
    @Override 
    public void run() {
    final Toast toast = new Toast(context);
    toast.setGravity(Gravity.CENTER_VERTICAL, 0, 0);
    toast.setDuration(Toast.LENGTH_LONG);
    LayoutInflater inflater = LayoutInflater.from(context);
    toast.setView(inflater.inflate(R.layout.leak_canary_heap_dump_toast, null));
    toast.show();//显示Toast，提示等待Dump Heap操作

    // Waiting for Idle to make sure Toast gets rendered.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override 
    public boolean queueIdle() {
    waitingForToast.set(toast);//当主线程的消息队列空闲时，则将结果通知给子线程，具体是通过FutureResult来实现的
    return false;
    }
    });
    }
    });
    }

在showToast()方法中，主要是发送一个Toast提示需要等待执行Dump Heap操作，当主线程的消息队列空闲时，则将此结果通知给主线程，具体是通过FutureResult来实现的。</br>
FutureResult主要是通过闭锁CountDownLatch来协调主线程与子线程间的同步操作。FutureResult类的定义如下：</br>

    public final class FutureResult<T> {

    private final AtomicReference<T> resultHolder;//原子引用
    private final CountDownLatch latch;//闭锁

    public FutureResult() {
    resultHolder = new AtomicReference<>();
    latch = new CountDownLatch(1);//初始化值为1
    }

    public boolean wait(long timeout, TimeUnit unit) {
    try {
    return latch.await(timeout, unit);//在闭锁上超时等待
    } catch (InterruptedException e) {
    throw new RuntimeException("Did not expect thread to be interrupted", e);
    }
    }

    public T get() {
    if (latch.getCount() > 0) {
    throw new IllegalStateException("Call wait() and check its result");
    }
    return resultHolder.get();//获取原子引用指向的对象
    }

    public void set(T result) {
    resultHolder.set(result);//用原子引用指向对象
    latch.countDown();//闭锁计数减1
    }
    }

可以看到，在dumpHeap()的过程中，首先构建一个FutureResult对象，该对象指向Toast对象，当调用showToast()方法时，会在主线程空闲时，通过FutureResult.set()方法来保存指向的引用对象，并且将闭锁计数减1，这样阻塞在FutureResult.wait()方法的子线程，将会被唤醒。后续就可以通过FutureResult.get()方法来获取保存的Toast对象，并通过cancelToast()方法来取消通知。</br>
4.14 Debug.dumpHprofData()</br>

    public static void dumpHprofData(String fileName) throws IOException {
    VMDebug.dumpHprofData(fileName);
    }

Dump Heap操作主要是通过Debug类的dumpHprofData()方法来完成的，把Dump出来的数据保存在一个hprof文件中。</br>
可以看到，dumpHeap()操作主要是借助系统的Debug类将内存快照Dump到一个hprof文件中。接下来看看，如何来解析hprof文件。</br>
4.15 ServiceHeapDumpListener.analyze()

    @Override 
    public void analyze(HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);//调用HeapAnalyzerService服务的runAnalysis方法，见4.16
    }

    heapdumpListener是在构建RefWatcher类的时候初始化的，默认实现是ServiceHeapDumpListener。
    @Override protected HeapDump.Listener defaultHeapDumpListener() {
    return new ServiceHeapDumpListener(context, DisplayLeakService.class);
    }

    public ServiceHeapDumpListener(Context context,
    Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabled(context, listenerServiceClass, true);
    setEnabled(context, HeapAnalyzerService.class, true);
    this.listenerServiceClass = checkNotNull(listenerServiceClass, "listenerServiceClass");//保存了解析结果处理类
    this.context = checkNotNull(context, "context").getApplicationContext();
    }

    在analyze方法中，解析的是一个HeapDump对象，该对象包含了前面操作中收集的各种参数。例如Heap Dump文件，引用对象对应的key，忽略的引用对象列表，以及监控时间、垃圾收集时间和Dump Heap时间。
    public HeapDump(File heapDumpFile, String referenceKey, String referenceName,
    ExcludedRefs excludedRefs, long watchDurationMs, long gcDurationMs, long heapDumpDurationMs) {
    this.heapDumpFile = checkNotNull(heapDumpFile, "heapDumpFile");
    this.referenceKey = checkNotNull(referenceKey, "referenceKey");
    this.referenceName = checkNotNull(referenceName, "referenceName");
    this.excludedRefs = checkNotNull(excludedRefs, "excludedRefs");
    this.watchDurationMs = watchDurationMs;
    this.gcDurationMs = gcDurationMs;
    this.heapDumpDurationMs = heapDumpDurationMs;
    }

    analyze()方法将调用HeapAnalyzerService类的runAnalysis()来执行具体的解析hprof文件操作。
    4.16 HeapAnalyzerService.runAnalysis()
    public static void runAnalysis(Context context, HeapDump heapDump,
    Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    context.startService(intent);
    }

HeapAnalyzerService类继承自IntentService类，实现了onHandleIntent()方法。在runAnalysis()方法中主要是启动其自身服务HeapAnalyzerService，并将参数listenerServiceClass和heapDump传递过程。IntentService的onHandleIntent()方法是在子线程中执行的，因此解析hprof文件也是在子线程中执行的。
</br>
4.17 HeapAnalyzerService.onHandleIntent()

    @Override 
    protected void onHandleIntent(Intent intent) {
    if (intent == null) {
    CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
    return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);//创建一个HeapAnalyzer，来自Square的HaHa开源项目

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);//见4.18
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);//4.21
    }

在子线程中，主要是借助Square开源项目的HaHa来解析hprof文件。具体是通过创建一个heapAnalyzer来解析hprof文件。
4.18 HeapAnalyzer.checkForLeak()

    /*
    * 通过key来搜索对应的KeyedWeakReference弱引用对象是否存在于 Heap Dump文件中，如果存在，则计算该引用对象到GC Roots的最短强引用路径
    */
    public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
    long analysisStartNanoTime = System.nanoTime();//记录分析的起始时间

    if (!heapDumpFile.exists()) {
    Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
    return failure(exception, since(analysisStartNanoTime));
    }

    try {
    HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
    HprofParser parser = new HprofParser(buffer);
    Snapshot snapshot = parser.parse();//把.hprof文件转为Snapshot，这个Snapshot对象包含了对象引用的所有路径
    deduplicateGcRoots(snapshot);//去除重复的路径

    Instance leakingRef = findLeakingReference(referenceKey, snapshot);//找出泄漏的对象，见4.19

    // False alarm, weak reference was cleared in between key check and heap dump.
    // 如果没有找到引用对象，则说明没有泄漏
    if (leakingRef == null) {
    return noLeak(since(analysisStartNanoTime));
    }

    return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef);//找出泄漏对象的最短路径，见4.20
    } catch (Throwable e) {
    return failure(e, since(analysisStartNanoTime));
    }
    }

可以看到，checkForLeak()方法的主要作用是解析.hprof，输出结果，主要分为以下几个步骤：</br>

把.hprof文件转换为Snapshot，这个Snapshot包含了引用对象的所有路径；</br>
去除重复的引用路径，减少解析hprof文件时的压力；</br>
找出泄漏的引用对象；</br>
如果没有找到泄漏的引用对象，则直接返回；</br>
如果存在泄漏的引用对象，则找出泄漏对象到GC Root的最短强引用路径；</br>

4.19 HeapAnalyzer.findLeakingReference()

    private Instance findLeakingReference(String key, Snapshot snapshot) {
    ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName());
    List<String> keysFound = new ArrayList<>();
    for (Instance instance : refClass.getInstancesList()) {
    List<ClassInstance.FieldValue> values = classInstanceValues(instance);
    String keyCandidate = asString(fieldValue(values, "key"));
    if (keyCandidate.equals(key)) {//找到了，返回内存泄漏对象
    return fieldValue(values, "referent");
    }
    keysFound.add(keyCandidate);
    }
    throw new IllegalStateException(
    "Could not find weak reference with key " + key + " in " + keysFound);
    }

该方法的主要是在Snapshot快照中，查找所有的KeyedWeakReference弱引用对象实例，然后比较key值是否相等，如果与参数中传递的key相等，则说明找到了内存泄漏的对象，并将该泄漏对象返回。</br>
4.20 HeapAnalyzer.findLeakTrace()

    private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot,
    Instance leakingRef) {

    ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
    ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);//查找最短引用路径

    // False alarm, no strong reference path to GC Roots.
    if (result.leakingNode == null) {//如果不存在强引用路径到GC Root，则说明没有内存泄漏
    return noLeak(since(analysisStartNanoTime));
    }

    LeakTrace leakTrace = buildLeakTrace(result.leakingNode);//构建对象泄漏路径

    String className = leakingRef.getClassObj().getClassName();//获取泄漏对象的类名

    // Side effect: computes retained size.
    snapshot.computeDominators();

    Instance leakingInstance = result.leakingNode.instance;//获取泄漏对象的实例

    long retainedSize = leakingInstance.getTotalRetainedSize();

    // TODO: check O sources and see what happened to android.graphics.Bitmap.mBuffer
    if (SDK_INT <= N_MR1) {
    retainedSize += computeIgnoredBitmapRetainedSize(snapshot, leakingInstance);
    }

    return leakDetected(result.excludingKnownLeaks, className, leakTrace, retainedSize,
    since(analysisStartNanoTime));//返回分析结果
    }

在findLeakTrace()方法中，主要借助ShortestPathFinder类查找最短引用路径，如果不存在最短引用路径，则说明没有内存泄漏。如果存在最短路径，则将查找结果返回。</br>
hprof文件主要是借助开源项目HAHA来完成解析工作，并找出内存泄漏对象到GC Root的最短引用路径。当HeapAnalyzer返回了解析结果后，在通过AbstractAnalysisResultService将解析结果发送给展示内存泄漏分析结果的DisplayLeakService服务。</br>
4.21 AbstractAnalysisResultService.sendResultToListener()

    public static void sendResultToListener(Context context, String listenerServiceClassName,
    HeapDump heapDump, AnalysisResult result) {
    Class<?> listenerServiceClass;
    try {
    listenerServiceClass = Class.forName(listenerServiceClassName);
    } catch (ClassNotFoundException e) {
    throw new RuntimeException(e);
    }
    Intent intent = new Intent(context, listenerServiceClass);
    intent.putExtra(HEAP_DUMP_EXTRA, heapDump);
    intent.putExtra(RESULT_EXTRA, result);
    context.startService(intent);//通过startService()方法启动DisplayLeakService服务
    }

    listenerServiceClassName在调用LeakCanary的install()方法时，传递了DisplayLeakService类名，保存在ServiceHeapDumpListener的listenerServiceClass成员变量中，因此该方法主要是启动DisplayLeakService服务。DisplayLeakService服务继承自AbstractAnalysisResultService类，而AbstractAnalysisResultService又继承自IntentService服务，因此最终会调用到AbstractAnalysisResultService的onHandleIntent()方法。
4.22 AbstractAnalysisResultService.onHandleIntent()

    @Override 
    protected final void onHandleIntent(Intent intent) {
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAP_DUMP_EXTRA);
    AnalysisResult result = (AnalysisResult) intent.getSerializableExtra(RESULT_EXTRA);
    try {
    onHeapAnalyzed(heapDump, result);//调用DisplayLeakService类的onHeapAnalyzed方法
    } finally {
    //noinspection ResultOfMethodCallIgnored
    heapDump.heapDumpFile.delete();
    }
    }

    在AbstractAnalysisResultService的onHandleIntent()方法中，主要是调用onHeapAnalyzed()方法，该方法是一个抽象方法，由AbstractAnalysisResultService的子类DisplayLeakService类来实现。
4.23 DisplayLeakService.onHeapAnalyzed()

    @Override 
    protected final void onHeapAnalyzed(HeapDump heapDump, AnalysisResult result) {
    String leakInfo = leakInfo(this, heapDump, result, true);//构造内存泄漏信息
    CanaryLog.d("%s", leakInfo);

    boolean resultSaved = false;
    boolean shouldSaveResult = result.leakFound || result.failure != null;
    if (shouldSaveResult) {
    heapDump = renameHeapdump(heapDump);
    resultSaved = saveResult(heapDump, result);//保存内存泄漏结果
    }

    PendingIntent pendingIntent;
    String contentTitle;
    String contentText;

    if (!shouldSaveResult) {//没有内存泄漏
    contentTitle = getString(R.string.leak_canary_no_leak_title);
    contentText = getString(R.string.leak_canary_no_leak_text);
    pendingIntent = null;
    } else if (resultSaved) {
    pendingIntent = DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);

    if (result.failure == null) {
    String size = formatShortFileSize(this, result.retainedHeapSize);
    String className = classSimpleName(result.className);
    if (result.excludedLeak) {
    contentTitle = getString(R.string.leak_canary_leak_excluded, className, size);
    } else {
    contentTitle = getString(R.string.leak_canary_class_has_leaked, className, size);
    }
    } else {
    contentTitle = getString(R.string.leak_canary_analysis_failed);
    }
    contentText = getString(R.string.leak_canary_notification_message);
    } else {
    contentTitle = getString(R.string.leak_canary_could_not_save_title);
    contentText = getString(R.string.leak_canary_could_not_save_text);
    pendingIntent = null;
    }
    // New notification id every second.
    int notificationId = (int) (SystemClock.uptimeMillis() / 1000);
    showNotification(this, contentTitle, contentText, pendingIntent, notificationId);//发送一个内存泄漏的通知
    afterDefaultHandling(heapDump, result, leakInfo);//子类可以覆写该方法，来上传内存泄漏的hprof文件。
    }

可以看到，在onHeapAnalyzed()方法中，主要是发送一个通知，告诉用户内存泄漏了，具体的内存泄漏信息在DisplayLeakActivity中展示，最后调用了afterDefaultHandling()方法，该方法默认是空实现，用户可以继承DisplayLeakService类然后覆写该方法，在该方法中实现将内存泄漏的hprof文件上传到网络服务器上。</br>
小结</br>
监控内存泄漏可以分为思大部分：</br>

一是检测是否发生了内存泄漏，具体是通过弱引用对象KeyedWeakReference指向需要被监控的引用对象，然后通过主动GC来判断该对象是否被回收了，如果该引用对象被回收了，则在引用队列中会保存该弱引用对象，然后通过判断引用队列中是否存在该弱引用对象来得出是否发生了内存泄漏。具体的过程见4.1~4.12。</br>
二是生成Dump Heap文件，具体是通过Debug类的dumpHprofData()方法来生成hprof格式的堆转储文件。具体的过程见4.13~4.15。</br>
三是解析hprof文件，找出内存泄漏对象到GC Roots的最短引用路径。具体是通过开源项目HaHa来完成解析工作，具体的过程见4.16~4.21。</br>
四是将内存泄漏解析结果展示给用户，具体是通过DisplayLeakService服务将内存泄漏结果以通知的形式发送给用户，并且在DisplayLeakActivity中展示内存泄漏信息。具体过程见4.22~4.24。</br>

## 5.总结
LeakCanary的使用非常简单，只要通过LeakCanary的install()方法，获取RefWatcher对象，然后调用RefWatcher的watch()方法进行监控需要被观察的引用对象。如果只是需要监控Activity引用对象，则更简单，只需在Application的onCreate()的方法中调用LeakCanary.install()方法，就可以完成监控操作(备注：针对Android 4.0以上，如果是Android 4.0以下，则还需要主动调用RefWatcher的watch()方法进行监控)。</br>
RefWatcher监控内存泄漏的原理主要是借助弱引用对象KeyedWeakReference与被监控的引用对象进行关联，当被监控的引用对象被回收时，则与之关联的弱引用对象也会被回收，并放入到一个引用队列中。通过在引用队列中查找是否存在该引用对象，来判断对象是否泄漏了。</br>
当对象发生了内存泄漏，则通过Debug类的dumpHprofData()方法获取堆内存快照hprof文件，然后通过开源项目HaHa分析hprof文件，并将分析结果(内存泄漏对象到GC Roots的最短引用链路)通过DisplayLeakService服务发送给用户查看。</br>
内存泄漏的监控过程都是在异步线程中处理的，主要体现在以下几个方面：</br>

在RefWatcher.ensureGoneAsync()方法中，如果是在主线程，则等到主线程消息队列空闲时，再由子线程延迟执行；如果是在子线程中，则直接由子线程延迟执行。每次延迟的时间都不一样，回退的时间呈指数增长。在该方法中，主要是检测是否发生了内存泄漏。</br>
AndroidHeapDumper.dumpHeap()操作是在子线程执行的，该方法也是等到主线程消息队列空闲后才执行的，具体是通过闭锁CountDownLatch来协调主线程与子线程间的同步操作。在该方法中，主要是Dump内存快照，生成hprof文件。</br>
HeapAnalyzerService.runAnalysis()操作是在子线程中执行的，因为HeapAnalyzerService继承了IntentService服务，在onHandleIntent()方法中执行具体的解析hprof文件操作。</br>
DisplayLeakService.onHeapAnalyzed()操作是在线程中执行的，因为DisplayLeakService继承了AbstractAnalysisResultService服务，而AbstractAnalysisResultService服务又继承了IntentService，并实现了onHandleIntent()方法。在onHeapAnalyzed()方法中，主要是发送一个通知告诉用户内存泄漏，并将内存泄漏结果在DisplayLeakActivity中展示。</br>


