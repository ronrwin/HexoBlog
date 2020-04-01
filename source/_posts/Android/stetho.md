---
title: Stetho---在Chrome中调试Android程序
date: 2016-06-18 17:39:31
tags:
- Android
---
官网链接：http://facebook.github.io/stetho/

Stetho是Facebook出品的一个可以在Chrome上调试Android的工具，使用起来非常方便，一定程度上减少了我们通过使用adb命令的麻烦。

<!--more-->
## 特性
#### Chrome调试
在Chrome地址栏中输入`chrome://inspect`，即可看到可调试的对象：
![Chrome DevTools](http://facebook.github.io/stetho/static/images/inspector-discovery.png)

#### 网络监控
可预览图片，json数据，trace等
![Network Inspection](http://facebook.github.io/stetho/static/images/inspector-network.png)

#### 数据监控
可查看或修改数据库以及SharedPrerenced数据
![Database Inspection](http://facebook.github.io/stetho/static/images/inspector-sqlite.png)

#### View Hierarchy
方便查看页面层级，类似HierarchyViewer的功能但更加方便使用。
![View Hierarchy](http://facebook.github.io/stetho/static/images/inspector-elements.png)

#### Dumpapp
可输出app的数据状态
![Dumpapp](http://facebook.github.io/stetho/static/images/dumpapp-prefs.png "Dumpapp ")

#### JavaScript控制台
可支持JavaScript的使用
![Javascript Console](http://facebook.github.io/stetho/static/images/inspector-js.png)


## 集成
```xml
  // Gradle dependency on Stetho 
  dependencies { 
    compile 'com.facebook.stetho:stetho:1.3.1' 
  } 
```
```xml
  <dependency>
    <groupid>com.facebook.stetho</groupid> 
    <artifactid>stetho</artifactid> 
    <version>1.3.1</version> 
  </dependency> 
```
如果使用网络监测，可能还需要添加以下依赖：
```xml
dependencies { 
    compile 'com.facebook.stetho:stetho-okhttp3:1.3.1' 
  }
```
或者
```xml
  dependencies { 
    compile 'com.facebook.stetho:stetho-okhttp:1.3.1' 
  } 
```
或者
```xml
  dependencies { 
    compile 'com.facebook.stetho:stetho-urlconnection:1.3.1' 
  } 
```

添加依赖之后，只需要在Application中添加以下代码就可以了。
```java
public class MyApplication extends Application {
  public void onCreate() {
    super.onCreate();
    Stetho.initializeWithDefaults(this);
  }
}
```

__允许网络监测__
如果使用了OkHttp网络框架，版本在2.2+或3.x，则可以使用下面语句添加网络监测
```java
OkHttpClient client = new OkHttpClient();
client.networkInterceptors().add(new StethoInterceptor());
```
或者
```java
new OkHttpClient.Builder()
    .addNetworkInterceptor(new StethoInterceptor())
    .build();
```
要使得okhttp2.x也可以运行，我们还需要用到`stetho-okhttp`(不是`stetho-okhttp3`)。
如果使用的是`HttpURLConnection`,我们可以使用`StethoURLConnectionManager`实现这个功能。需要在请求头声明Accept-Encoding: gzip并且手动解决压缩响应确保Stetho能打印压缩负荷大小。

__使用dumpapp插件__
```java
Stetho.initialize(Stetho.newInitializerBuilder(context)
    .enableDumpapp(new DumperPluginsProvider() {
      @Override
      public Iterable<DumperPlugin> get() {
        return new Stetho.DefaultDumperPluginsBuilder(context)
            .provide(new MyDumperPlugin())
            .finish();
      }
    })
    .enableWebKitInspector(Stetho.defaultInspectorModulesProvider(context))
    .build())
```




