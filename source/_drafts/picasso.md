---
title: Picasso源码解析
date: 2016-06-17 10:53:14
tags:
- 框架
categories:
- Android
---
> 本篇文章分析针对picasso:2.5.2版本

## 简介
picasso是Square公司开源的一个Android图形缓存库，
地址http://square.github.io/picasso/，可以实现图片下载和缓存功能。
仅仅只需要一行代码就能完全实现图片的异步加载：
```java
Picasso.with(context).load("http://i.imgur.com/DvpvklR.png").into(imageView);
```
好处：
Picasso不仅实现了图片异步加载的功能，还解决了android中加载图片时需要解决的一些常见问题：
1.在adapter中需要取消已经不在视野范围的ImageView图片资源的加载，否则会导致图片错位，Picasso已经解决了这个问题。
2.使用复杂的图片压缩转换来尽可能的减少内存消耗
3.自带内存和硬盘二级缓存功能
```
1. 自带统计监控功能   
 支持图片缓存使用的监控，包括缓存命中率，已使用内存大小，节省的流量等
2. 支持优先级处理
  每次任务调度前会选择优先级高的任务，比如app页面的Banner的优先级高于Icon时很使用。
3. 支持延迟到图片尺寸计算完成加载
4. 支持飞行模式，并发线程根据网络类型而改变 
   这里Picasso 根据网络类型来决定最大并发数，而不是cpu核数。
5. "无"本地缓存
  并不是说没有本地缓存，而是picasso没有实现，而是交给了super的okhttp实现，好处是可以通过请求Response Header的cache-control以及Expired控制图片的过期时间。
6. 使用复杂的图片压缩转换来尽可能的减少内存消耗
```
<!--more-->
## 集成
Gradle
```xml
compile 'com.squareup.picasso:picasso:2.5.2'
```
Maven
```xml
<dependency>
  <groupId>com.squareup.picasso</groupId>
  <artifactId>picasso</artifactId>
  <version>2.5.2</version>
</dependency>
```

## 特性及示例代码
```java
// 加载一张图片
Picasso.with(mContext).load("url").into(imageView);
// 声明加载时显示的占位图片
Picasso.with(mContext).load("url").placeholder(R.drawable.placeholder).into(imageView);
// 声明加载失败时显示的错误图片。 如果加载发生错误会重复三次请求，三次都失败才会显示错误占位图片。
Picasso.with(mContext).load("url").error(R.drawable.error).into(imageView);
// 设定优先级
Picasso.with(mContext).load("url").priority(Picasso.Priority.HIGH).into(imageView);
// 加载一张图片并设置一个回调接口
Picasso.with(mContext).load("url").placeholder(R.mipmap.ic_default).into(imageView, new Callback() {
    @Override
    public void onSuccess() {
 
    }
 
    @Override
    public void onError() {
 
    }
});
// 预加载一张图片
Picasso.with(mContext).load("url").fetch();
// 同步加载一张图片,注意只能在子线程中调用并且Bitmap不会被缓存到内存里.
new Thread() {
    @Override
    public void run() {
        try {
            final Bitmap bitmap = Picasso.with(getApplicationContext()).load("url").get();
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    imageView.setImageBitmap(bitmap);
                }
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}.start();
 
// 除了加载网络图片picasso还支持加载Resources, assets, files, content providers中的资源文件。
Picasso.with(context).load(R.drawable.landing_screen).into(imageView1);
Picasso.with(context).load(new File(...)).into(imageView2);
//加载一张图片并设置tag,可以通过tag来暂定或者继续加载,可以用于当ListView滚动是暂定加载.停止滚动恢复加载.
Picasso.with(this).load("url").tag(mContext).into(imageView);
Picasso.with(this).pauseTag(mContext);
Picasso.with(this).resumeTag(mContxt);.priority(Picasso.Priority.HIGH)

```

## 框架结构
![](https://skykai521.github.io/image/Picasso-classes-relation.png)
从类图上我们可以看出Picasso的核心类主要包括:__Picasso__、__RequestCreator__、__Action__、__Dispatcher__、__Request__、__RequestHandler__、__BitmapHunter__等等.一张图片加载可以分为以下几步:
```
创建->入队->执行->解码->变换->批处理->完成->分发->显示(可选)
```

## 流程
简单的讲就是 Picasso 收到加载及显示图片的任务，创建 Request 并将它交给 Dispatcher，Dispatcher 分发任务到具体 RequestHandler，任务通过 MemoryCache 及 Handler(数据获取接口) 获取图片，图片获取成功后通过 PicassoDrawable 显示到 Target 中。

需要注意的是上面 Data 的 File system 部分，Picasso 没有自定义本地缓存的接口，默认使用 http 的本地缓存，API 9 以上使用 okhttp，以下使用 Urlconnection，所以如果需要自定义本地缓存就需要重定义 Downloader。

## 源码解析
我们就先从简单的使用开始看起：
```java
Picasso.with(this).load(url).into(imageView);
```
让我们先来看看Picasso.with()做了什么:
```java
public static Picasso with(Context context) {
  if (singleton == null) {
    synchronized (Picasso.class) {
      if (singleton == null) {
        singleton = new Builder(context).build();
      }
    }
  }
  return singleton;
}
```
这是一个单例模式的应用，整个应用维护一个Picasso单例。
如果还未实例化就通过new Builder(context).build()创建一个singleton。我们看build()的代码：
```java
public static class Builder {
     
  public Builder(Context context) {
    if (context == null) {
      throw new IllegalArgumentException("Context must not be null.");
    }
    this.context = context.getApplicationContext();
  }
     
  /** Create the {@link Picasso} instance. */
  public Picasso build() {
    Context context = this.context;
     
    if (downloader == null) {
       //创建默认下载器
      downloader = Utils.createDefaultDownloader(context);
    }
    if (cache == null) {
       //创建Lru内存缓存
      cache = new LruCache(context);
    }
    if (service == null) {
       //创建线程池,默认有3个执行线程,会根据网络状况自动切换线程数
      service = new PicassoExecutorService();
    }
    if (transformer == null) {
       //创建默认的transformer,并无实际作用
      transformer = RequestTransformer.IDENTITY;
    }
    //创建stats用于统计缓存,以及缓存命中率,下载数量等等
    Stats stats = new Stats(cache);
    //创建dispatcher对象用于任务的调度
    Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);
     
    return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
        defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
  }
}
```
上面的代码里省去了部分配置的方法,当我们使用Picasso默认配置的时候(当然也可以自定义),最后会调用build()方法并配置好我们需要的各种对象,最后实例化一个Picasso对象并返回。最后在Picasso的构造方法里除了对这些对象的赋值以及创建一些新的对象,例如清理线程等等.最重要的是初始化了requestHandlers,下面是代码片段:

```java
int builtInHandlers = 7; // Adjust this as internal handlers are added or removed.
int extraCount = (extraRequestHandlers != null ? extraRequestHandlers.size() : 0);
List<RequestHandler> allRequestHandlers =
    new ArrayList<RequestHandler>(builtInHandlers + extraCount);
 
// ResourceRequestHandler needs to be the first in the list to avoid
// forcing other RequestHandlers to perform null checks on request.uri
// to cover the (request.resourceId != 0) case.
allRequestHandlers.add(new ResourceRequestHandler(context));
if (extraRequestHandlers != null) {
  allRequestHandlers.addAll(extraRequestHandlers);
}
allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
allRequestHandlers.add(new MediaStoreRequestHandler(context));
allRequestHandlers.add(new ContentStreamRequestHandler(context));
allRequestHandlers.add(new AssetRequestHandler(context));
allRequestHandlers.add(new FileRequestHandler(context));
allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));
requestHandlers = Collections.unmodifiableList(allRequestHandlers);
```
到这里，使用了建造者模式创建的一个Picasso实例就完成了。

默认初始化了以下的参数：

Downloader
DownLoader就是下载用的工具类，在Picasso当中，如果OKHttp可以使用的话，就会默认使用OKHttp，如果无法使用的话，就会使用UrlConnectionDownloader(默认使用HttpURLConnection实现)。

Cache
默认实现为LruCache，就是使用LinkedHashMap实现的一个Cache类，注意的一个地方就是，在其他的地方，我们一般默认的是限制的capacity，但是这个地方我们是限制的总共使用的内存空间。因此LruCache在实现的时候，其实简单理解就是将LinkedHashMap封装，然后基于LinkedHashMap的方法实现Cache的方法，在Cache的set()方法的时候，会不断计算当前还可以使用的空间大小，要是超出范围，则删除之前保存的数据。

ExecutorService
默认的实现为PicassoExecutorService，该类也比较简单，其实就是ThreadPoolExecutor，在其功能的基础上继续封装，在其中有一个比较细心的功能就是，Picasso通过PicassoExecutorService设置线程数量，来调整在2G/3G/4G/WiFi不同网络情况下的不同表现。

RequestTransformer
ReqeustTransformer是一个接口，用来预处理Reqeust，在之前写的《Picasso的使用：使用Transformation，下载后预处理图片并显示》中写过，可以用来将请求进行预先处理，比如改个域名啥的。

Stats
主要是一些统计信息，比如cache hit/miss，总共下载的文件大小，下载过的图片数量，转换的图片数量等等。

Dispatcher
Picasso当中，分发任务的线程，这是我们以后要重点研究的一个类，先标记一下，这个Dispatcher主要做了以下的事情：
启动了一个DispatcherThread线程
初始化了一个用来处理消息的DispatcherHandler，注意，根据Dispatcher中默认配置，该Handler所有数据的处理是在DispatcherThread之上。
初始化并注册了一个网络状态广播接收器。

之后我们看下load()的实现：
```java
public RequestCreator load(String path) {
  if (path == null) {
    return new RequestCreator(this, null, 0);
  }
  if (path.trim().length() == 0) {
    throw new IllegalArgumentException("Path must not be empty.");
  }
  return load(Uri.parse(path));
}
 
public RequestCreator load(Uri uri) {
  return new RequestCreator(this, uri, 0);
}
```
load方法返回了RequestCreator 对象：
```java
RequestCreator(Picasso picasso, Uri uri, int resourceId) {
  if (picasso.shutdown) {
    throw new IllegalStateException(
        "Picasso instance already shut down. Cannot submit new requests.");
  }
    // 持有一个Picasso对象
  this.picasso = picasso;
    // 请求的图片信息都保存在data当中
  this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);
}
```
这个图片请求中，data是包含了我们所要的图片的信息。在我们通过centerCrop()或者transform()等等方法的时候实际上也就是改变data内的对应的变量标识，再到处理的阶段根据这些参数来进行对应的操作。
所以在我们调用into()方法之前，所有的操作都是在设定我们需要处理的参数，真正的操作都是有into()方法引起的。

那么into方法做了什么呢？
```java
public void into(ImageView target) {
  // 传入空的callback
  into(target, null);
}
 
public void into(ImageView target, Callback callback) {
  long started = System.nanoTime();
  // 检查调用是否在主线程，如果不是则抛出异常
  checkMain();
 
  if (target == null) {
    throw new IllegalArgumentException("Target must not be null.");
  }
     
  // return uri != null || resourceId != 0;如果没有设置需要加载的uri,或者resourceId
  if (!data.hasImage()) {
    picasso.cancelRequest(target);
    //如果设置占位图片,直接加载并返回
    if (setPlaceholder) {
      setPlaceholder(target, getPlaceholderDrawable());
    }
    return;
  }
  // 如果是延时加载,也就是选择了fit()模式
  if (deferred) {
    // fit()模式是适应target的宽高加载,所以并不能手动设置resize,如果设置就抛出异常
    if (data.hasSize()) {
      throw new IllegalStateException("Fit cannot be used with resize.");
    }
    int width = target.getWidth();
    int height = target.getHeight();
    // 如果目标ImageView的宽或高现在为0
    if (width == 0 || height == 0) {
      // 先设置占位符
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      //监听ImageView的ViewTreeObserver.OnPreDrawListener接口,一旦ImageView
      //的宽高被赋值,就按照ImageView的宽高继续加载.
      picasso.defer(target, new DeferredRequestCreator(this, target, callback));
      return;
    }
    // 如果ImageView有宽高就设置设置
    data.resize(width, height);
  }
 
  // 构建Request
  Request request = createRequest(started);
  // 构建requestKey，用于缓存图片
  String requestKey = createKey(request);
 
  // 根据memoryPolicy来决定是否可以从内存里读取
  if (shouldReadFromMemoryCache(memoryPolicy)) {
    // 通过LruCache来读取内存里的缓存图片
    Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);
    // 如果读取到
    if (bitmap != null) {
      // 取消target的request
      picasso.cancelRequest(target);
      // 设置图片
      setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
      if (picasso.loggingEnabled) {
        log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
      }
      //如果设置了回调接口就回调接口的方法.
      if (callback != null) {
        callback.onSuccess();
      }
      return;
    }
  }
  //如果缓存里没读到,先根据是否设置了占位图并设置占位
  if (setPlaceholder) {
    setPlaceholder(target, getPlaceholderDrawable());
  }
  //构建一个Action对象,由于我们是往ImageView里加载图片,所以这里创建的是一个ImageViewAction对象.
  Action action =
      new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
          errorDrawable, requestKey, tag, callback, noFade);
 
  //将Action对象入列提交
  picasso.enqueueAndSubmit(action);
}
```
整个流程看下来应该是比较清晰的,最后是创建了一个ImageViewAction对象并通过picasso提交。
其实Picasso实现了三种类型的Action对象。
第一种是ImageViewAction：当我们需要往ImageView里加载图片的时候会创建ImageViewAction对象。
第二种是TargetAction：如果是往实现了Target接口的对象里加载图片是则会创建TargetAction对象,
第三种是RemoteViewsAction：派生了NotificationAction和AppWidgetAction，即Picasso加载图片支持这些控件对象。
这些Action类的实现类不仅保存了这次加载需要的所有信息,还提供了加载完成后的回调方法.也是由子类实现并用来完成不同的调用的。

下面我们看下enqueueAndSubmit方法：
```java
void enqueueAndSubmit(Action action) {
  Object target = action.getTarget();
  //取消这个target已经有的action.
  if (target != null && targetToAction.get(target) != action) {
    // This will also check we are on the main thread.
    cancelExistingRequest(target);
    targetToAction.put(target, action);
  }
  //提交action
  submit(action);
}
//调用dispatcher来派发action
void submit(Action action) {
  dispatcher.dispatchSubmit(action);
}
```
转到了dispatcher来处理了。

```java
void dispatchSubmit(Action action) {
  handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
}
```
看到通过一个handler对象发送了一个REQUEST_SUBMIT的消息，那么这个handler是存在与哪个线程的呢?
```java
Dispatcher(Context context, ExecutorService service, Handler mainThreadHandler,
    Downloader downloader, Cache cache, Stats stats) {
  this.dispatcherThread = new DispatcherThread();
  this.dispatcherThread.start();    
  this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
  this.mainThreadHandler = mainThreadHandler;
}
 
  static class DispatcherThread extends HandlerThread {
  DispatcherThread() {
    super(Utils.THREAD_PREFIX + DISPATCHER_THREAD_NAME, THREAD_PRIORITY_BACKGROUND);
  }
}
```
上面是Dispatcher的构造方法(省略了部分代码),可以看到先是创建了一个HandlerThread对象,然后创建了一个DispatcherHandler对象,这个handler就是刚刚用来发送REQUEST_SUBMIT消息的handler,这里我们就明白了原来是通过Dispatcher类里的一个子线程里的handler不断的派发我们的消息。

```java
case REQUEST_SUBMIT: {
  Action action = (Action) msg.obj;
  dispatcher.performSubmit(action);
  break;
}
```
这里收到REQUEST_SUBMIT消息,最终调用了 dispatcher.performSubmit(action);方法:
```java
void performSubmit(Action action) {
  performSubmit(action, true);
}
 
void performSubmit(Action action, boolean dismissFailed) {
  //是否该tag的请求被暂停
  if (pausedTags.contains(action.getTag())) {
    pausedActions.put(action.getTarget(), action);
    if (action.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
          "because tag '" + action.getTag() + "' is paused");
    }
    return;
  }
  //通过action的key来在hunterMap查找是否有相同的hunter,这个key里保存的是我们
  //的uri或者resourceId和一些参数,如果都是一样就将这些action合并到一个
  //BitmapHunter里去，由于请求的对象时一致的，执行了去重的操作。
  BitmapHunter hunter = hunterMap.get(action.getKey());
  if (hunter != null) {
    hunter.attach(action);
    return;
  }
 
  if (service.isShutdown()) {
    if (action.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_IGNORED, action.request.logId(), "because shut down");
    }
    return;
  }
 
  //创建BitmapHunter对象
  hunter = forRequest(action.getPicasso(), this, cache, stats, action);
  //通过service执行hunter并返回一个future对象。hunter继承了runnable，从这里开始网络访问。
  hunter.future = service.submit(hunter);
  //将hunter添加到hunterMap中
  hunterMap.put(action.getKey(), hunter);
  // 从失败队列中移除
  if (dismissFailed) {
    failedActions.remove(action.getTarget());
  }
 
  if (action.getPicasso().loggingEnabled) {
    log(OWNER_DISPATCHER, VERB_ENQUEUED, action.request.logId());
  }
}
```

我们分析下forRequest方法：
```java
static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats,
    Action action) {
  Request request = action.getRequest();
  List<RequestHandler> requestHandlers = picasso.getRequestHandlers();
 
  //从requestHandlers中检测哪个RequestHandler可以处理这个request,如果找到就创建
  //BitmapHunter并返回.
  for (int i = 0, count = requestHandlers.size(); i < count; i++) {
    RequestHandler requestHandler = requestHandlers.get(i);
    if (requestHandler.canHandleRequest(request)) {
      return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
    }
  }
 
  return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
}
```
里就体现出来了责任链模式（[《JAVA与模式》之责任链模式][a]）,通过依次调用requestHandlers里RequestHandler的canHandleRequest()方法来确定这个request能被哪个RequestHandler执行,找到对应的RequestHandler后就创建BitmapHunter对象并返回.再回到performSubmit()方法里,通过service.submit(hunter);执行了hunter,hunter实现了Runnable接口,所以run()方法就会被执行,所以我们继续看看BitmapHunter里run()方法的实现:
[a]:http://www.cnblogs.com/java-my-life/archive/2012/05/28/2516865.html
```java
@Override public void run() {
  try {
    //更新当前线程的名字
    updateThreadName(data);
 
    if (picasso.loggingEnabled) {
      log(OWNER_HUNTER, VERB_EXECUTING, getLogIdsForHunter(this));
    }
 
    //调用hunt()方法并返回Bitmap类型的result对象.
    result = hunt();
 
    //如果为空,调用dispatcher发送失败的消息,
    //如果不为空则发送完成的消息
    if (result == null) {
      dispatcher.dispatchFailed(this);
    } else {
      dispatcher.dispatchComplete(this);
    }
  //通过不同的异常来进行对应的处理
  } catch (Downloader.ResponseException e) {
    if (!e.localCacheOnly || e.responseCode != 504) {
      exception = e;
    }
    dispatcher.dispatchFailed(this);
  } catch (NetworkRequestHandler.ContentLengthException e) {
    exception = e;
    dispatcher.dispatchRetry(this);
  } catch (IOException e) {
    exception = e;
    dispatcher.dispatchRetry(this);
  } catch (OutOfMemoryError e) {
    StringWriter writer = new StringWriter();
    stats.createSnapshot().dump(new PrintWriter(writer));
    exception = new RuntimeException(writer.toString(), e);
    dispatcher.dispatchFailed(this);
  } catch (Exception e) {
    exception = e;
    dispatcher.dispatchFailed(this);
  } finally {
    Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
  }
}
 
Bitmap hunt() throws IOException {
  Bitmap bitmap = null;
  //是否可以从内存中读取
  if (shouldReadFromMemoryCache(memoryPolicy)) {
    bitmap = cache.get(key);
    if (bitmap != null) {
      //统计缓存命中率
      stats.dispatchCacheHit();
      loadedFrom = MEMORY;
      if (picasso.loggingEnabled) {
        log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache");
      }
      return bitmap;
    }
  }
 
  //如果未设置networkPolicy并且retryCount为0,则将networkPolicy设置为
  //NetworkPolicy.OFFLINE
  data.networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
  //通过对应的requestHandler来获取result，网络请求
  RequestHandler.Result result = requestHandler.load(data, networkPolicy);
  if (result != null) {
    loadedFrom = result.getLoadedFrom();
    exifOrientation = result.getExifOrientation();
    bitmap = result.getBitmap();
 
    // If there was no Bitmap then we need to decode it from the stream.
    if (bitmap == null) {
      InputStream is = result.getStream();
      try {
        bitmap = decodeStream(is, data);
      } finally {
        Utils.closeQuietly(is);
      }
    }
  }
 
  if (bitmap != null) {
    if (picasso.loggingEnabled) {
      log(OWNER_HUNTER, VERB_DECODED, data.logId());
    }
    stats.dispatchBitmapDecoded(bitmap);
    //处理Transformation
    if (data.needsTransformation() || exifOrientation != 0) {
      synchronized (DECODE_LOCK) {
        if (data.needsMatrixTransform() || exifOrientation != 0) {
          bitmap = transformResult(data, bitmap, exifOrientation);
          if (picasso.loggingEnabled) {
            log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId());
          }
        }
        if (data.hasCustomTransformations()) {
          bitmap = applyCustomTransformations(data.transformations, bitmap);
          if (picasso.loggingEnabled) {
            log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId(), "from custom transformations");
          }
        }
      }
      if (bitmap != null) {
        stats.dispatchBitmapTransformed(bitmap);
      }
    }
  }
  //返回bitmap
  return bitmap;
}
```
在run()方法里调用了hunt()方法来获取result然后通知了dispatcher来处理结果,并在try-catch里通知dispatcher来处理相应的异常,在hunt()方法里通过前面指定的requestHandler来获取相应的result,我们是从网络加载图片,自然是调用NetworkRequestHandler的load()方法来处理我们的request,这里我们就不再分析NetworkRequestHandler具体的细节.获取到result之后就获得我们的bitmap然后检测是否需要Transformation,这里使用了一个全局锁DECODE_LOCK来保证同一个时刻仅仅有一个图片正在处理。我们假设我们的请求被正确处理了,这样我们拿到我们的result然后调用了dispatcher.dispatchComplete(this);最终也是通过handler调用了dispatcher.performComplete()方法:
```java
void performComplete(BitmapHunter hunter) {
  //是否可以放入内存缓存里
  if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {
    cache.set(hunter.getKey(), hunter.getResult());
  }
  //从hunterMap移除
  hunterMap.remove(hunter.getKey());
  //处理hunter
  batch(hunter);
  if (hunter.getPicasso().loggingEnabled) {
    log(OWNER_DISPATCHER, VERB_BATCHED, getLogIdsForHunter(hunter), "for completion");
  }
}
 
private void batch(BitmapHunter hunter) {
  if (hunter.isCancelled()) {
    return;
  }
  batch.add(hunter);
  if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
    handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
  }
}
 
void performBatchComplete() {
  List<BitmapHunter> copy = new ArrayList<BitmapHunter>(batch);
  batch.clear();
  mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
  logBatch(copy);
}
```
首先是添加到内存缓存中去,然后在发送一个HUNTER_DELAY_NEXT_BATCH消息,实际上这个消息最后会出发performBatchComplete()方法,performBatchComplete()里则是通过mainThreadHandler将BitmapHunter的List发送到主线程处理,所以我们去看看mainThreadHandler的handleMessage()方法:
```java
static final Handler HANDLER = new Handler(Looper.getMainLooper()) {
  @Override public void handleMessage(Message msg) {
    switch (msg.what) {
      case HUNTER_BATCH_COMPLETE: {
        @SuppressWarnings("unchecked") List<BitmapHunter> batch = (List<BitmapHunter>) msg.obj;
        //noinspection ForLoopReplaceableByForEach
        for (int i = 0, n = batch.size(); i < n; i++) {
          BitmapHunter hunter = batch.get(i);
          hunter.picasso.complete(hunter);
        }
        break;
      }
      default:
        throw new AssertionError("Unknown handler message received: " + msg.what);
    }
  }
};
```
很简单,就是依次调用picasso.complete(hunter)方法:
```java
void complete(BitmapHunter hunter) {
   //获取单个Action
   Action single = hunter.getAction();
   //获取被添加进来的Action
   List<Action> joined = hunter.getActions();
 
   //是否有合并的Action
   boolean hasMultiple = joined != null && !joined.isEmpty();
   //是否需要派发
   boolean shouldDeliver = single != null || hasMultiple;
 
   if (!shouldDeliver) {
     return;
   }
 
   Uri uri = hunter.getData().uri;
   Exception exception = hunter.getException();
   Bitmap result = hunter.getResult();
   LoadedFrom from = hunter.getLoadedFrom();
 
   //派发Action
   if (single != null) {
     deliverAction(result, from, single);
   }
 
   //派发合并的Action
   if (hasMultiple) {
     //noinspection ForLoopReplaceableByForEach
     for (int i = 0, n = joined.size(); i < n; i++) {
       Action join = joined.get(i);
       deliverAction(result, from, join);
     }
   }
 
   if (listener != null && exception != null) {
     listener.onImageLoadFailed(this, uri, exception);
   }
 }
  
private void deliverAction(Bitmap result, LoadedFrom from, Action action) {
   if (action.isCancelled()) {
     return;
   }
   if (!action.willReplay()) {
     targetToAction.remove(action.getTarget());
   }
   if (result != null) {
     if (from == null) {
       throw new AssertionError("LoadedFrom cannot be null.");
     }
     //回调action的complete()方法
     action.complete(result, from);
     if (loggingEnabled) {
       log(OWNER_MAIN, VERB_COMPLETED, action.request.logId(), "from " + from);
     }
   } else {
     //失败则回调error()方法
     action.error();
     if (loggingEnabled) {
       log(OWNER_MAIN, VERB_ERRORED, action.request.logId());
     }
   }
 }
```
可以看出最终是回调了action的complete()方法,从前文知道我们这里是ImageViewAction,所以我们去看看ImageViewAction的complete()的实现:
```java
@Override public void complete(Bitmap result, Picasso.LoadedFrom from) {
  if (result == null) {
    throw new AssertionError(
        String.format("Attempted to complete action with no result!\n%s", this));
  }
 
  //得到target也就是ImageView
  ImageView target = this.target.get();
  if (target == null) {
    return;
  }
 
  Context context = picasso.context;
  boolean indicatorsEnabled = picasso.indicatorsEnabled;
  //通过PicassoDrawable来将bitmap设置到ImageView上
  PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);
 
  //回调callback接口
  if (callback != null) {
    callback.onSuccess();
  }
}
```
很显然通过了PicassoDrawable.setBitmap()将我们的Bitmap设置到了我们的ImageView上,最后并回调callback接口,这里为什么会使用PicassoDrawable来设置Bitmap呢?使用过Picasso的都知道,Picasso自带渐变的加载动画,所以这里就是处理渐变动画的地方,由于篇幅原因我们就不做具体分析了,感兴趣的同学可以自行研究,所以到这里我们的整个Picasso的调用流程的源码分析就结束了.



## 细节阐述
#### Bitmap的变换
```java
static Bitmap transformResult(Request data, Bitmap result, int exifRotation) {
    int inWidth = result.getWidth();
    int inHeight = result.getHeight();
    boolean onlyScaleDown = data.onlyScaleDown;

    int drawX = 0;
    int drawY = 0;
    int drawWidth = inWidth;
    int drawHeight = inHeight;

    Matrix matrix = new Matrix();

    if (data.needsMatrixTransform()) {
      int targetWidth = data.targetWidth;
      int targetHeight = data.targetHeight;

      float targetRotation = data.rotationDegrees;
      if (targetRotation != 0) {
        if (data.hasRotationPivot) {
          matrix.setRotate(targetRotation, data.rotationPivotX, data.rotationPivotY);
        } else {
          matrix.setRotate(targetRotation);
        }
      }

      if (data.centerCrop) {
        float widthRatio = targetWidth / (float) inWidth;
        float heightRatio = targetHeight / (float) inHeight;
        float scaleX, scaleY;
        if (widthRatio > heightRatio) {
          int newSize = (int) Math.ceil(inHeight * (heightRatio / widthRatio));
          drawY = (inHeight - newSize) / 2;
          drawHeight = newSize;
          scaleX = widthRatio;
          scaleY = targetHeight / (float) drawHeight;
        } else {
          int newSize = (int) Math.ceil(inWidth * (widthRatio / heightRatio));
          drawX = (inWidth - newSize) / 2;
          drawWidth = newSize;
          scaleX = targetWidth / (float) drawWidth;
          scaleY = heightRatio;
        }
        if (shouldResize(onlyScaleDown, inWidth, inHeight, targetWidth, targetHeight)) {
          matrix.preScale(scaleX, scaleY);
        }
      } else if (data.centerInside) {
        float widthRatio = targetWidth / (float) inWidth;
        float heightRatio = targetHeight / (float) inHeight;
        float scale = widthRatio < heightRatio ? widthRatio : heightRatio;
        if (shouldResize(onlyScaleDown, inWidth, inHeight, targetWidth, targetHeight)) {
          matrix.preScale(scale, scale);
        }
      } else if ((targetWidth != 0 || targetHeight != 0) //
          && (targetWidth != inWidth || targetHeight != inHeight)) {
        // If an explicit target size has been specified and they do not match the results bounds,
        // pre-scale the existing matrix appropriately.
        // Keep aspect ratio if one dimension is set to 0.
        float sx =
            targetWidth != 0 ? targetWidth / (float) inWidth : targetHeight / (float) inHeight;
        float sy =
            targetHeight != 0 ? targetHeight / (float) inHeight : targetWidth / (float) inWidth;
        if (shouldResize(onlyScaleDown, inWidth, inHeight, targetWidth, targetHeight)) {
          matrix.preScale(sx, sy);
        }
      }
    }

    if (exifRotation != 0) {
      matrix.preRotate(exifRotation);
    }

    Bitmap newResult =
        Bitmap.createBitmap(result, drawX, drawY, drawWidth, drawHeight, matrix, true);
    if (newResult != result) {
      result.recycle();
      result = newResult;
    }

    return result;
  }
```
我们设定的像rotate，pivot等参数，是在获取到原始的bitmap之后，在transformResult方法里执行变换的。

#### PicassoBitmap的渐变显示
```java
@Override public void draw(Canvas canvas) {
    if (!animating) {
      super.draw(canvas);
    } else {
      float normalized = (SystemClock.uptimeMillis() - startTimeMillis) / FADE_DURATION;
      if (normalized >= 1f) {
        animating = false;
        placeholder = null;
        super.draw(canvas);
      } else {
        if (placeholder != null) {
          placeholder.draw(canvas);
        }

				// 
        int partialAlpha = (int) (alpha * normalized);
        super.setAlpha(partialAlpha);
        super.draw(canvas);
        super.setAlpha(alpha);
        if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.GINGERBREAD_MR1) {
          invalidateSelf();
        }
      }
    }

    if (debugging) {
      drawDebugIndicator(canvas);
    }
  }
```




## 参考文献
[Picasso源代码分析](https://skykai521.github.io/2016/02/25/Picasso%E6%BA%90%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90/)












