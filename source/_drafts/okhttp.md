---
title: 关于okhttp
date: 2016-06-09 14:39:58
tags:
- 网络
categories: 
- Java
---
项目github地址：[OkHttp](https://github.com/square/okhttp)
官方简介：OkHttp是一款优秀的HTTP框架，它支持get请求和post请求，支持基于Http的文件上传和下载，支持加载图片，支持下载文件透明的GZIP压缩，支持响应缓存避免重复的网络请求，支持使用连接池来降低响应延迟问题。

<!--more-->

## 配置方式
- 导入jar包：地址：https://search.maven.org/remote_content?g=com.squareup.okhttp3&a=okhttp&v=LATEST
- 通过构建方式导入 

Maven:
```xml
<dependency>
  <groupId>com.squareup.okhttp3</groupId>
  <artifactId>okhttp</artifactId>
  <version>3.3.1</version>
</dependency>
```
Gradle：
```xml
compile 'com.squareup.okhttp3:okhttp:3.3.1'
```

## 基本使用
#### Synchronous Get（同步Get）
下载一个文件，打印他的响应头，以string形式打印响应体。 
响应体的 string() 方法对于小文档来说十分方便、高效。但是如果响应体太大（超过1MB），应避免适应 string()方法 ，因为他会将把整个文档加载到内存中。对于超过1MB的响应body，应使用流的方式来处理body。
```java
private final OkHttpClient client = new OkHttpClient();
  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    Headers responseHeaders = response.headers();
    for (int i = 0; i < responseHeaders.size(); i++) {
      System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
    }

    System.out.println(response.body().string());
  }
```
Request是OkHttp中访问的请求，Builder是辅助类，Response即OkHttp中的响应。 

#### Asynchronous Get（异步Get）
在一个工作线程中下载文件，当响应可读时回调Callback接口。读取响应时会阻塞当前线程。OkHttp现阶段不提供异步api来接收响应体。
```java
private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    client.newCall(request).enqueue(new Callback() {
      @Override public void onFailure(Call call, IOException e) {
        e.printStackTrace();
      }

      @Override public void onResponse(Call call, Response response) throws IOException {
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

        Headers responseHeaders = response.headers();
        for (int i = 0, size = responseHeaders.size(); i < size; i++) {
          System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
        }

        System.out.println(response.body().string());
      }
    });
  }
```
#### Accessing Headers（获取头部信息）
典型的HTTP头像是一个`Map<String, String>`
当写请求头时，使用`header(name, value)`设置键值对。如果原来已经存在有该键值对，则会被覆盖。使用`addHeader(name, value)`来添加添加键值对，而不用覆盖原来的数值。
```java
private final OkHttpClient client = new OkHttpClient()

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://api.github.com/repos/square/okhttp/issues")
        .header("User-Agent", "OkHttp Headers.java")
        .addHeader("Accept", "application/json; q=0.5")
        .addHeader("Accept", "application/vnd.github.v3+json")
        .build()

    Response response = client.newCall(request).execute()
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response)

    System.out.println("Server: " + response.header("Server"))
    System.out.println("Date: " + response.header("Date"))
    System.out.println("Vary: " + response.headers("Vary"))
  }
```

#### Posting a String（Post方式提交String）
使用HTTP POST提交请求到服务。这个例子提交了一个markdown文档到web服务，以HTML方式渲染markdown。因为整个请求体都在内存中，因此避免使用此api提交大文档（大于1MB）。
```java
public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    String postBody = ""
        + "Releases\n"
        + "--------\n"
        + "\n"
        + " * _1.0_ May 6, 2013\n"
        + " * _1.1_ June 15, 2013\n"
        + " * _1.2_ August 11, 2013\n";

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, postBody))
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```

#### Post Streaming（Post方式提交流）
以流的方式POST提交请求体。请求体的内容由流写入产生。这个例子是流直接写入Okio的BufferedSink。你的程序可能会使用OutputStream，你可以使用BufferedSink.outputStream()来获取。.
```java
public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    RequestBody requestBody = new RequestBody() {
      @Override public MediaType contentType() {
        return MEDIA_TYPE_MARKDOWN;
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        sink.writeUtf8("Numbers\n");
        sink.writeUtf8("-------\n");
        for (int i = 2; i <= 997; i++) {
          sink.writeUtf8(String.format(" * %s = %s\n", i, factor(i)));
        }
      }

      private String factor(int n) {
        for (int i = 2; i < n; i++) {
          int x = n / i;
          if (x * i == n) return factor(x) + " × " + i;
        }
        return Integer.toString(n);
      }
    };

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(requestBody)
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```

#### Posting a File（Post方式提交文件）
```java
public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    File file = new File("README.md");

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, file))
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```
使用FormEncodingBuilder来构建和HTML标签相同效果的请求体。键值对将使用一种HTML兼容形式的URL编码来进行编码。
```java
private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    RequestBody formBody = new FormBody.Builder()
        .add("search", "Jurassic Park")
        .build();
    Request request = new Request.Builder()
        .url("https://en.wikipedia.org/w/index.php")
        .post(formBody)
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```

#### Posting a multipart request（Post方式提交分块请求）
MultipartBuilder可以构建复杂的请求体，与HTML文件上传形式兼容。多块请求体中每块请求都是一个请求体，可以定义自己的请求头。这些请求头可以用来描述这块请求，例如他的Content-Disposition。如果Content-Length和Content-Type可用的话，他们会被自动添加到请求头中。
```java
private static final String IMGUR_CLIENT_ID = "...";
  private static final MediaType MEDIA_TYPE_PNG = MediaType.parse("image/png");

  private final OkHttpClient client = new OkHttpClient();

  public void run() throws Exception {
    
    RequestBody requestBody = new MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart("title", "Square Logo")
        .addFormDataPart("image", "logo-square.png",
            RequestBody.create(MEDIA_TYPE_PNG, new File("website/static/logo-square.png")))
        .build();

    Request request = new Request.Builder()
        .header("Authorization", "Client-ID " + IMGUR_CLIENT_ID)
        .url("https://api.imgur.com/3/image")
        .post(requestBody)
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```

#### Parse a JSON Response With Gson（使用GSON解析JSON响应）
Gson是一个在JSON和Java对象之间转换非常方便的api。这里我们用Gson来解析Github API的JSON响应。 
> 注意：ResponseBody.charStream()使用响应头Content-Type指定的字符集来解析响应体。默认是UTF-8。
```java
private final OkHttpClient client = new OkHttpClient();
  private final Gson gson = new Gson();

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("https://api.github.com/gists/c2a7c39532239ff261be")
        .build();
    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    Gist gist = gson.fromJson(response.body().charStream(), Gist.class);
    for (Map.Entry<String, GistFile> entry : gist.files.entrySet()) {
      System.out.println(entry.getKey());
      System.out.println(entry.getValue().content);
    }
  }

  static class Gist {
    Map<String, GistFile> files;
  }

  static class GistFile {
    String content;
  }
```

#### Response Caching（响应缓存）
为了缓存响应，你需要一个你可以读写的缓存目录，和缓存大小的限制。这个缓存目录应该是私有的，不信任的程序应不能读取缓存内容。 

一个缓存目录同时拥有多个缓存访问是错误的。大多数程序只需要调用一次new OkHttp()，在第一次调用时配置好缓存，然后其他地方只需要调用这个实例就可以了。否则两个缓存示例互相干扰，破坏响应缓存，而且有可能会导致程序崩溃。 

响应缓存使用HTTP头作为配置。你可以在请求头中添加Cache-Control: max-stale=3600 ,OkHttp缓存会支持。你的服务通过响应头确定响应缓存多长时间，例如使用Cache-Control: max-age=9600。
```java
private final OkHttpClient client

  public CacheResponse(File cacheDirectory) throws Exception {
    int cacheSize = 10 * 1024 * 1024
    Cache cache = new Cache(cacheDirectory, cacheSize)

    client = new OkHttpClient.Builder()
        .cache(cache)
        .build()
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build()

    Response response1 = client.newCall(request).execute()
    if (!response1.isSuccessful()) throw new IOException("Unexpected code " + response1)

    String response1Body = response1.body().string()
    System.out.println("Response 1 response:          " + response1)
    System.out.println("Response 1 cache response:    " + response1.cacheResponse())
    System.out.println("Response 1 network response:  " + response1.networkResponse())

    Response response2 = client.newCall(request).execute()
    if (!response2.isSuccessful()) throw new IOException("Unexpected code " + response2)

    String response2Body = response2.body().string()
    System.out.println("Response 2 response:          " + response2)
    System.out.println("Response 2 cache response:    " + response2.cacheResponse())
    System.out.println("Response 2 network response:  " + response2.networkResponse())

    System.out.println("Response 2 equals Response 1? " + response1Body.equals(response2Body))
  }
```
为了防止使用缓存的响应，可以用CacheControl.FORCE_NETWORK。为了防止它使用网络，使用CacheControl.FORCE_CACHE。需要注意的是：如果您使用FORCE_CACHE和网络的响应需求，OkHttp则会返回一个504提示，告诉你不可满足请求响应。 

#### Canceling a Call（取消一个Call） 
使用Call.cancel()可以立即停止掉一个正在执行的call。如果一个线程正在写请求或者读响应，将会引发IOException。当call没有必要的时候，使用这个api可以节约网络资源。例如当用户离开一个应用时。不管同步还是异步的call都可以取消。 
你可以通过tags来同时取消多个请求。当你构建一请求时，使用RequestBuilder.tag(tag)来分配一个标签。之后你就可以用OkHttpClient.cancel(tag)来取消所有带有这个tag的call。
```java
 private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(1)
  private final OkHttpClient client = new OkHttpClient()

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/2") // This URL is served with a 2 second delay.
        .build()

    final long startNanos = System.nanoTime()
    final Call call = client.newCall(request)

    // Schedule a job to cancel the call in 1 second.
    executor.schedule(new Runnable() {
      @Override public void run() {
        System.out.printf("%.2f Canceling call.%n", (System.nanoTime() - startNanos) / 1e9f)
        call.cancel()
        System.out.printf("%.2f Canceled call.%n", (System.nanoTime() - startNanos) / 1e9f)
      }
    }, 1, TimeUnit.SECONDS)

    try {
      System.out.printf("%.2f Executing call.%n", (System.nanoTime() - startNanos) / 1e9f)
      Response response = call.execute()
      System.out.printf("%.2f Call was expected to fail, but completed: %s%n",
          (System.nanoTime() - startNanos) / 1e9f, response)
    } catch (IOException e) {
      System.out.printf("%.2f Call failed as expected: %s%n",
          (System.nanoTime() - startNanos) / 1e9f, e)
    }
  }
```

#### Timeouts（超时）
没有响应时使用超时结束call。没有响应的原因可能是客户点链接问题、服务器可用性问题或者这之间的其他东西。OkHttp支持连接，读取和写入超时。
```java
private final OkHttpClient client;

  public ConfigureTimeouts() throws Exception {
    client = new OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .writeTimeout(10, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/2") 
        .build();

    Response response = client.newCall(request).execute();
    System.out.println("Response completed: " + response);
  }
```

#### Per-call Configuration（每个Call的配置）
使用OkHttpClient，所有的HTTP Client配置包括代理设置、超时设置、缓存设置。当你需要为单个call改变配置的时候，clone 一个 OkHttpClient。这个api将会返回一个浅拷贝（shallow copy），你可以用来单独自定义。下面的例子中，我们让一个请求是500ms的超时、另一个是3000ms的超时。
```java
private final OkHttpClient client = new OkHttpClient()

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://httpbin.org/delay/1") // This URL is served with a 1 second delay.
        .build()

    try {
      // Copy to customize OkHttp for this request.
      OkHttpClient copy = client.newBuilder()
          .readTimeout(500, TimeUnit.MILLISECONDS)
          .build()

      Response response = copy.newCall(request).execute()
      System.out.println("Response 1 succeeded: " + response)
    } catch (IOException e) {
      System.out.println("Response 1 failed: " + e)
    }

    try {
      // Copy to customize OkHttp for this request.
      OkHttpClient copy = client.newBuilder()
          .readTimeout(3000, TimeUnit.MILLISECONDS)
          .build()

      Response response = copy.newCall(request).execute()
      System.out.println("Response 2 succeeded: " + response)
    } catch (IOException e) {
      System.out.println("Response 2 failed: " + e)
    }
  }
```

#### Handling authentication（处理验证）
OkHttp会自动重试未验证的请求。当响应是401 Not Authorized时，Authenticator会被要求提供证书。Authenticator的实现中需要建立一个新的包含证书的请求。如果没有证书可用，返回null来跳过尝试。
```java
 private final OkHttpClient client;

  public Authenticate() {
    client = new OkHttpClient.Builder()
        .authenticator(new Authenticator() {
          @Override public Request authenticate(Route route, Response response) throws IOException {
            System.out.println("Authenticating for response: " + response);
            System.out.println("Challenges: " + response.challenges());
            String credential = Credentials.basic("jesse", "password1");
            return response.request().newBuilder()
                .header("Authorization", credential)
                .build();
          }
        })
        .build();
  }

  public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/secrets/hellosecret.txt")
        .build();

    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

    System.out.println(response.body().string());
  }
```
为了防止处理验证失败产生的多次重试，可以返回`null`来取消。例如，当凭据已经被尝试过，可以跳过重试。
```java
if (credential.equals(response.request().header("Authorization"))) {
  return null; // If we already failed with these credentials, don't retry.
}
```
也可以在超过应用规定的重试次数后取消：
```java
if (responseCount(response) >= 3) {
    return null; // If we've failed 3 times, give up.
  }
```
以上代码依赖`responseCount()`方法：
```java
private int responseCount(Response response) {
    int result = 1;
    while ((response = response.priorResponse()) != null) {
      result++;
    }
    return result;
  }
```

## 拦截器
观察，修改以及可能短路的请求输出和响应请求的回来。通常情况下拦截器用来添加，移除或者转换请求或者回应的头部信息。 拦截器接口中有intercept(Chain chain)方法，同时返回Response。看一个简单的例子：
```java
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```
chain.proceed(request)是拦截器的关键部分。这个看似简单的方法是所有的HTTP工作发生的地方，产生满足要求的反应。 拦截器可以链接。假设你有一个压缩的拦截和校验拦截器：你需要决定数据是否被压缩，或者校验或校验然后压缩。okhttp使用列表来跟踪和拦截，拦截器会按顺序调用。
![拦截器](https://raw.githubusercontent.com/mxn21/mxn21.github.io/master/public/img/img99.png)

#### Application Interceptors
拦截器可以注册为应用程序或网络拦截。我们将使用上面定义的logginginterceptor说明差异。 可以在OkHttpClient.interceptors()返回的list中调用add(),来注册一个application interceptor 。
```java
OkHttpClient client = new OkHttpClient();
client.interceptors().add(new LoggingInterceptor());
Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();
Response response = client.newCall(request).execute();
response.body().close();
```
URL http://www.publicobject.com/helloworld.txt会重定向到https://publicobject.com/helloworld .txt，OkHttp会自动执行这些重定向。application interceptor执行之后，chain.proceed()返回的response会重定向到 下面的response：
```java
    INFO: Sending request http://www.publicobject.com/helloworld.txt on null
    User-Agent: OkHttp Example
    INFO: Received response for https://publicobject.com/helloworld.txt in 1179.7ms
    Server: nginx/1.4.6 (Ubuntu)
    Content-Type: text/plain
    Content-Length: 1759
    Connection: keep-alive
```
可以看到，url重定向来，因为response.request().url()和request.url()不同，两个不同的日志可以看出这一点。

#### Network Interceptors
注册一个网络拦截器是很相似的。添加到networkinterceptors()的list来代替interceptors()的list：
```java
OkHttpClient client = new OkHttpClient();
client.networkInterceptors().add(new LoggingInterceptor());
Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();
Response response = client.newCall(request).execute();
response.body().close();
```
当我们运行这个代码的时候，拦截程序运行了两次。一次为http://www.publicobject.com/helloworld.txt初始请求， 和另一个重定向到https://publicobject.com/helloworld.txt。
```java
INFO: Sending request http://www.publicobject.com/helloworld.txt on Connection{www.publicobject.com:80, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=none protocol=http/1.1}
User-Agent: OkHttp Example
Host: www.publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip
INFO: Received response for http://www.publicobject.com/helloworld.txt in 115.6ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/html
Content-Length: 193
Connection: keep-alive
Location: https://publicobject.com/helloworld.txt
INFO: Sending request https://publicobject.com/helloworld.txt on Connection{publicobject.com:443, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA protocol=http/1.1}
User-Agent: OkHttp Example
Host: publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip
INFO: Received response for https://publicobject.com/helloworld.txt in 80.9ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```
网络要求还包含更多的数据，如Accept-Encoding: gzip头来支持压缩。网络拦截的Chain有一个非空Connection可以用来询问IP地址和用于连接到Web服务器的TLS配置。

#### application和network interceptors的选择
每一个拦截链都有相对的优点。
- Application interceptors
  + 不必担心中间的responses，例如重定向和重连。 
  + 总是调用一次，即使是从缓存HTTP响应。
  + 观察应用程序的原始意图。不关心OkHttp的注入headers，例如If-None-Match 
  + 允许短路和不执行Chain.proceed().
  + 允许重连，多次调用proceed()。
- Network Interceptors
  + 能够操作中间反应，例如重定向和重连。
  + 不能被缓存响应，例如短路网络调用。 
  + 观察数据，正如它将在网络上传输。
  + 有权使用携带request的Connection
  
#### 重写Requests
拦截器可以添加，删除，或替换请求报头。他们还可以将这些请求的body转换。例如，你可以使用一个应用程序拦截来增加request body压缩， 如果你连接的服务器支持这种操作的话。
```java
/** This interceptor compresses the HTTP request body. Many webservers can't handle this! */
final class GzipRequestInterceptor implements Interceptor {
  @Override public Response intercept(Chain chain) throws IOException {
    Request originalRequest = chain.request();
    if (originalRequest.body() == null || originalRequest.header("Content-Encoding") != null) {
      return chain.proceed(originalRequest);
    }
    Request compressedRequest = originalRequest.newBuilder()
        .header("Content-Encoding", "gzip")
        .method(originalRequest.method(), gzip(originalRequest.body()))
        .build();
    return chain.proceed(compressedRequest);
  }
  private RequestBody gzip(final RequestBody body) {
    return new RequestBody() {
      @Override public MediaType contentType() {
        return body.contentType();
      }
      @Override public long contentLength() {
        return -1; // We don't know the compressed length in advance!
      }
      @Override public void writeTo(BufferedSink sink) throws IOException {
        BufferedSink gzipSink = Okio.buffer(new GzipSink(sink));
        body.writeTo(gzipSink);
        gzipSink.close();
      }
    };
  }
}
```

#### 重写Responses
对应的，拦截器可以重写response headers和转换response body。这是一般比重写请求标头更危险因为它可能违反了服务器的期望！ 如果你是在一个棘手的情况，并准备处理的后果，重写response headers是一个强大的方式来解决问题。例如，你可以将服务器的错误配置的缓存控制 响应头修改以便更好地响应缓存：
```java
    /** Dangerous interceptor that rewrites the server's cache-control header. */
    private static final Interceptor REWRITE_CACHE_CONTROL_INTERCEPTOR = new Interceptor() {
      @Override public Response intercept(Chain chain) throws IOException {
        Response originalResponse = chain.proceed(chain.request());
        return originalResponse.newBuilder()
            .header("Cache-Control", "max-age=60")
            .build();
      }
    };
```
作为补充一个网络服务器上的相应的修复，通常这种方法效果最好。
- 在某些情况下，如用户单击“刷新”按钮，就可能有必要跳过缓存，并直接从服务器获取数据。要强制刷新，添加无缓存指令：”Cache-Control”： “no-cache”。
- 有时你会想显示可以立即显示的资源。这是可以使用的，这样你的应用程序可以在等待最新的数据下载的时候显示一些东西， 重定向request到本地缓存资源，添加”Cache-Control”：”only-if-cached”。
- 有时候过期的response比没有response更好，设置最长过期时间来允许过期的response响应： int maxStale = 60 * 60 * 24 * 28; // tolerate 4-weeks stale
“Cache-Control”：”max-stale=” + maxStale。
> okhttp拦截器需要okhttp 2.2或更高。不幸的是，拦截器在OkUrlFactory和基于OkUrlFactory的库上不工作，或者基于okhttp library的，包括 Retrofit ≤1.8和Picasso≤2.4。





******
## 参考文献
[OkHttp官方教程解析-彻底入门OkHttp使用](http://gold.xitu.io/entry/5728441d128fe1006058b6b9)
[Okhttp缓存浅析](http://souly.cn/%E6%8A%80%E6%9C%AF%E5%8D%9A%E6%96%87/2015/09/08/okhttp%E7%BC%93%E5%AD%98%E6%B5%85%E6%9E%90/)