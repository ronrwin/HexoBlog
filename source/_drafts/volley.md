---
title: volleys源码解析
date: 2018-01-29 22:19:41
tags:
- Android
categories:
- 开发
---

使用参考项目地址：[github](https://github.com/google/volley) 


# 主流程源码解析
（分析版本号为：tag 1.1.1）
先上简单使用
```java
RequestQueue queue = Volley.newRequestQueue(this);  // 1
String url = "http://www.google.com";

StringRequest stringRequest = new StringRequest(Request.Method.GET, url, new Response.Listener<String>() {
    @Override
    public void onResponse(String response) {
         mTextView.setText("Response is: "+ response.substring(0,500));
    }
}, new Response.ErrorListener() {
    @Override
    public void onErrorResponse(VolleyError error) {
        mTextView.setText("That didn't work!");
    }
});
queue.add(stringRequest);
```

<!-- more -->

##  Volley.newRequestQueue

```java
private static RequestQueue newRequestQueue(Context context, Network network) {
    File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
    RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);   // 2
    queue.start();  // 3
    return queue;
}
```

创建一个RequestQueue，并马上start()。我们看下这个RequestQueue做了啥。

##  new RequestQueue

```java
    /**
     * Creates a default instance of the worker pool and calls {@link RequestQueue#start()} on it.
     *
     * @param context A {@link Context} to use for creating the cache dir.
     * @param stack A {@link BaseHttpStack} to use for the network, or null for default.
     * @return A started {@link RequestQueue} instance.
     */
public static RequestQueue newRequestQueue(Context context, BaseHttpStack stack) {
    BasicNetwork network;
    if (stack == null) {
        if (Build.VERSION.SDK_INT >= 9) {
            network = new BasicNetwork(new HurlStack());
        } else {
            // Prior to Gingerbread, HttpUrlConnection was unreliable.
            // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
            // At some point in the future we'll move our minSdkVersion past Froyo and can
            // delete this fallback (along with all Apache HTTP code).
            String userAgent = "volley/0";
            try {
                String packageName = context.getPackageName();
                PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
                userAgent = packageName + "/" + info.versionCode;
            } catch (NameNotFoundException e) {
            }

            network = new BasicNetwork(
                    new HttpClientStack(AndroidHttpClient.newInstance(userAgent)));
        }
    } else {
        network = new BasicNetwork(stack);
    }

    return newRequestQueue(context, network);
}

public RequestQueue(Cache cache, Network network) {
    // DEFAULT_NETWORK_THREAD_POOL_SIZE = 4
    this(cache, network, DEFAULT_NETWORK_THREAD_POOL_SIZE);
}

public RequestQueue(Cache cache, Network network, int threadPoolSize) {
    this(cache, network, threadPoolSize,
            new ExecutorDelivery(new Handler(Looper.getMainLooper())));
}

public RequestQueue(Cache cache, Network network, int threadPoolSize,
        ResponseDelivery delivery) {
    // 缓存，是一个接口
    mCache = cache;
    // Network也是一个接口，这里实现为BasicNetwork
    mNetwork = network;
    // NetworkDispatcher是一个Thread，这里说明要起4个网络处理线程
    mDispatchers = new NetworkDispatcher[threadPoolSize];
    // 这里为ExecutorDelivery，处理网络结果回调
    mDelivery = delivery;
}
```

##  queue.start()
```java
public void start() {
    stop();  // Make sure any currently running dispatchers are stopped.
    // Create the cache dispatcher and start it.
    // 缓存线程
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery); 
    mCacheDispatcher.start();       // 4

    // Create network dispatchers (and corresponding threads) up to the pool size.
    for (int i = 0; i < mDispatchers.length; i++) {
        NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();      // 6
    }
}
```
由此可知，Volley是创建了一个缓存线程和四个网络线程来实现网路请求。下面分开来看线程做了什么。

## mCacheDispatcher.start()
CacheDispatcher也是一个Thread。我们看下run的实现
```java
// CacheDispatcher.java
public void run() {
    if (DEBUG) VolleyLog.v("start new dispatcher");
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

    // Make a blocking call to initialize the cache.
    mCache.initialize();

    while (true) {
        try {
            processRequest();   // 5
        } catch (InterruptedException e) {
            // We may have been interrupted because it was time to quit.
            if (mQuit) {
                return;
            }
        }
    }
}
```

## processRequest()
```java
// Extracted to its own method to ensure locals have a constrained liveness scope by the GC.
// This is needed to avoid keeping previous request references alive for an indeterminate amount
// of time. Update consumer-proguard-rules.pro when modifying this. See also
// https://github.com/google/volley/issues/114
private void processRequest() throws InterruptedException {
    // Get a request from the cache triage queue, blocking until
    // at least one is available.
    final Request<?> request = mCacheQueue.take();
    request.addMarker("cache-queue-take");

    // If the request has been canceled, don't bother dispatching it.
    if (request.isCanceled()) {
        request.finish("cache-discard-canceled");
        return;
    }

    // Attempt to retrieve this item from cache.
    Cache.Entry entry = mCache.get(request.getCacheKey());
    if (entry == null) {
        request.addMarker("cache-miss");
        // Cache miss; send off to the network dispatcher.
        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
            // 加入到网络队列
            mNetworkQueue.put(request);
        }
        return;
    }

    // If it is completely expired, just send it to the network.
    if (entry.isExpired()) {
        request.addMarker("cache-hit-expired");
        request.setCacheEntry(entry);
        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
            // 加入到网络请求队列，由网络线程获取，从而进行网络访问
            mNetworkQueue.put(request);
        }
        return;
    }

    // We have a cache hit; parse its data for delivery back to the request.
    request.addMarker("cache-hit");
    Response<?> response = request.parseNetworkResponse(
            new NetworkResponse(entry.data, entry.responseHeaders));
    request.addMarker("cache-hit-parsed");

    if (!entry.refreshNeeded()) {
        // Completely unexpired cache hit. Just deliver the response.
        // 命中缓存，不需要更新，不需重新请求
        mDelivery.postResponse(request, response);
    } else {
        // Soft-expired cache hit. We can deliver the cached response,
        // but we need to also send the request to the network for
        // refreshing.
        request.addMarker("cache-hit-refresh-needed");
        request.setCacheEntry(entry);
        // Mark the response as intermediate.
        response.intermediate = true;

        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
            // Post the intermediate response back to the user and have
            // the delivery then forward the request along to the network.
            mDelivery.postResponse(request, response, new Runnable() {
                @Override
                public void run() {
                    try {
                        // 到网络请求队列
                        mNetworkQueue.put(request);
                    } catch (InterruptedException e) {
                        // Restore the interrupted status
                        Thread.currentThread().interrupt();
                    }
                }
            });
        } else {
            // request has been added to list of waiting requests
            // to receive the network response from the first request once it returns.
            mDelivery.postResponse(request, response);
        }
    }
}
```

## networkDispatcher.start()
我们看下网络线程的逻辑
```java
// NetworkDispatcher.java
@Override
public void run() {
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    while (true) {
        try {
            processRequest();       // 7
        } catch (InterruptedException e) {
            // We may have been interrupted because it was time to quit.
            if (mQuit) {
                return;
            }
        }
    }
}
```
NetworkDispatcher也有一个processRequest().

## NetworkDispatcher.processRequest()
```java
// Extracted to its own method to ensure locals have a constrained liveness scope by the GC.
// This is needed to avoid keeping previous request references alive for an indeterminate amount
// of time. Update consumer-proguard-rules.pro when modifying this. See also
// https://github.com/google/volley/issues/114
private void processRequest() throws InterruptedException {
    // Take a request from the queue.
    // 获取请求对象
    Request<?> request = mQueue.take();

    long startTimeMs = SystemClock.elapsedRealtime();
    try {
        request.addMarker("network-queue-take");

        // If the request was cancelled already, do not perform the
        // network request.
        if (request.isCanceled()) {
            // 如果请求已取消，则不做网络请求
            request.finish("network-discard-cancelled");
            request.notifyListenerResponseNotUsable();
            return;
        }

        addTrafficStatsTag(request);

        // Perform the network request.
        // 执行网络请求
        NetworkResponse networkResponse = mNetwork.performRequest(request);     // 8
        request.addMarker("network-http-complete");

        // If the server returned 304 AND we delivered a response already,
        // we're done -- don't deliver a second identical response.
        if (networkResponse.notModified && request.hasHadResponseDelivered()) {
            request.finish("not-modified");
            request.notifyListenerResponseNotUsable();
            return;
        }

        // Parse the response here on the worker thread.
        Response<?> response = request.parseNetworkResponse(networkResponse);
        request.addMarker("network-parse-complete");

        // Write to cache if applicable.
        // TODO: Only update cache metadata instead of entire record for 304s.
        if (request.shouldCache() && response.cacheEntry != null) {
            // 缓存结果
            mCache.put(request.getCacheKey(), response.cacheEntry);
            request.addMarker("network-cache-written");
        }

        // Post the response back.
        request.markDelivered();
        // 回调结果
        mDelivery.postResponse(request, response);
        request.notifyListenerResponseReceived(response);
    } catch (VolleyError volleyError) {
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        parseAndDeliverNetworkError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    } catch (Exception e) {
        VolleyLog.e(e, "Unhandled exception %s", e.toString());
        VolleyError volleyError = new VolleyError(e);
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        mDelivery.postError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    }
}
```
mQueue是一个阻塞队列，通过mQueue.take()来拿到请求，因此在前面，只需要调用mNetworkQueue.put(request)往队列加请求，在这里的线程，就会从中取出来执行网络访问。


## mNetwork.performRequest(request)
这里执行网络请求，默认实现类是BasicNetwork.java
```java
@Override
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
    long requestStart = SystemClock.elapsedRealtime();
    while (true) {
        HttpResponse httpResponse = null;
        byte[] responseContents = null;
        List<Header> responseHeaders = Collections.emptyList();
        try {
            // Gather headers.
            Map<String, String> additionalRequestHeaders =
                    getCacheHeaders(request.getCacheEntry());
            httpResponse = mBaseHttpStack.executeRequest(request, additionalRequestHeaders);        // 9  这里是HurlStack
            int statusCode = httpResponse.getStatusCode();

            responseHeaders = httpResponse.getHeaders();
            // Handle cache validation.
            if (statusCode == HttpURLConnection.HTTP_NOT_MODIFIED) {
                Entry entry = request.getCacheEntry();
                if (entry == null) {
                    return new NetworkResponse(HttpURLConnection.HTTP_NOT_MODIFIED, null, true,
                            SystemClock.elapsedRealtime() - requestStart, responseHeaders);
                }
                // Combine cached and response headers so the response will be complete.
                List<Header> combinedHeaders = combineHeaders(responseHeaders, entry);
                return new NetworkResponse(HttpURLConnection.HTTP_NOT_MODIFIED, entry.data,
                        true, SystemClock.elapsedRealtime() - requestStart, combinedHeaders);
            }

            // Some responses such as 204s do not have content.  We must check.
            InputStream inputStream = httpResponse.getContent();
            if (inputStream != null) {
              responseContents =
                      inputStreamToBytes(inputStream, httpResponse.getContentLength());
            } else {
              // Add 0 byte response as a way of honestly representing a
              // no-content request.
              responseContents = new byte[0];
            }

            // if the request is slow, log it.
            long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
            logSlowRequests(requestLifetime, request, responseContents, statusCode);

            if (statusCode < 200 || statusCode > 299) {
                throw new IOException();
            }
            return new NetworkResponse(statusCode, responseContents, false,
                    SystemClock.elapsedRealtime() - requestStart, responseHeaders);
        } catch (SocketTimeoutException e) {
            attemptRetryOnException("socket", request, new TimeoutError());
        } catch (MalformedURLException e) {
            throw new RuntimeException("Bad URL " + request.getUrl(), e);
        } catch (IOException e) {
            int statusCode;
            if (httpResponse != null) {
                statusCode = httpResponse.getStatusCode();
            } else {
                throw new NoConnectionError(e);
            }
            VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
            NetworkResponse networkResponse;
            if (responseContents != null) {
                networkResponse = new NetworkResponse(statusCode, responseContents, false,
                        SystemClock.elapsedRealtime() - requestStart, responseHeaders);
                if (statusCode == HttpURLConnection.HTTP_UNAUTHORIZED ||
                        statusCode == HttpURLConnection.HTTP_FORBIDDEN) {
                    attemptRetryOnException("auth",
                            request, new AuthFailureError(networkResponse));
                } else if (statusCode >= 400 && statusCode <= 499) {
                    // Don't retry other client errors.
                    throw new ClientError(networkResponse);
                } else if (statusCode >= 500 && statusCode <= 599) {
                    if (request.shouldRetryServerErrors()) {
                        attemptRetryOnException("server",
                                request, new ServerError(networkResponse));
                    } else {
                        throw new ServerError(networkResponse);
                    }
                } else {
                    // 3xx? No reason to retry.
                    throw new ServerError(networkResponse);
                }
            } else {
                attemptRetryOnException("network", request, new NetworkError());
            }
        }
    }
}
```
mBaseHttpStack.executeRequest(request, additionalRequestHeaders)的实现是HurlStack（newRequestQueue中创建）。

## HurlStack.executeRequest
```java
@Override
public HttpResponse executeRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError {
    String url = request.getUrl();
    HashMap<String, String> map = new HashMap<>();
    map.putAll(request.getHeaders());
    map.putAll(additionalHeaders);
    if (mUrlRewriter != null) {
        String rewritten = mUrlRewriter.rewriteUrl(url);
        if (rewritten == null) {
            throw new IOException("URL blocked by rewriter: " + url);
        }
        url = rewritten;
    }
    URL parsedUrl = new URL(url);
    // 网络请求
    HttpURLConnection connection = openConnection(parsedUrl, request);      // 10
    for (String headerName : map.keySet()) {
        connection.addRequestProperty(headerName, map.get(headerName));
    }
    setConnectionParametersForRequest(connection, request);
    // Initialize HttpResponse with data from the HttpURLConnection.
    int responseCode = connection.getResponseCode();
    if (responseCode == -1) {
        // -1 is returned by getResponseCode() if the response code could not be retrieved.
        // Signal to the caller that something was wrong with the connection.
        throw new IOException("Could not retrieve response code from HttpUrlConnection.");
    }

    if (!hasResponseBody(request.getMethod(), responseCode)) {
        return new HttpResponse(responseCode, convertHeaders(connection.getHeaderFields()));
    }

    return new HttpResponse(responseCode, convertHeaders(connection.getHeaderFields()),
            connection.getContentLength(), inputStreamFromConnection(connection));
}
```

## openConnection(parsedUrl, request)
```java
/**
     * Opens an {@link HttpURLConnection} with parameters.
     * @param url
     * @return an open connection
     * @throws IOException
     */
private HttpURLConnection openConnection(URL url, Request<?> request) throws IOException {
    HttpURLConnection connection = createConnection(url);

    int timeoutMs = request.getTimeoutMs();
    connection.setConnectTimeout(timeoutMs);
    connection.setReadTimeout(timeoutMs);
    connection.setUseCaches(false);
    connection.setDoInput(true);

    // use caller-provided custom SslSocketFactory, if any, for HTTPS
    if ("https".equals(url.getProtocol()) && mSslSocketFactory != null) {
        ((HttpsURLConnection)connection).setSSLSocketFactory(mSslSocketFactory);
    }

    return connection;
}
```

到这里，算是把请求的流程走了一遍。

其他的请求，例如JsonRequest、StringRequest、ImageRequest，就是对于Request抽象类的具体实现了，在此就不细述。





