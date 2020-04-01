---
title: Android性能优化
date: 2016-04-27 11:14:48
description: 涉及到的点比较细，都是在工作学习中记录下来的，养成良好习惯。
tags:
- 性能
---

关于Android性能优化的文章，网上有很多，涉及的点也很多，包括内存方面，界面方面，布局方面，代码编写方面。
基本遵循的规则有两个：
1. **不做多余的工作**
2. **尽量少分配内存**

除去编写代码时需注意的规范之外，在应用运行中也会有很多需要优化的地方，这部分的性能优化需要结合性能分析，有很多种性能分析的工具，对性能作系统的观测与研究。

性能分析方面会另外开辟，这里只是记录在优化上面的方式，还需花时间慢慢做到细化。不能说大而全，但也算是切中要点，以养成良好的习惯，在平时码代码的时候能够尽量避开，仅供所有人参考。

### 日志
- 应保守使用`Log`。
- 正确使用`Log`级别。
  + `ERROR`：用于严重错误场景。
  + `WARNNING`：用于记录非预期场景发生时的信息，例如处理非法消息类型时使用。
  + `INFO`：用于记录关键路径的关键信息。
  + `DEBUG`：用于调试程序。DEBUG级别的日志最好不要出现在release版本当中，使用if(DEBUG)开关控制。
  + `VERBOSE`：用于临时调试。

### UI相关
- 采用硬件加速，在`androidmanifest.xml`中application添加`android:hardwareAccelerated="true"`。不过这个需要在android 3.0才可以使用。android4.0这个选项是默认开启的。
- 使用`merge`优化布局层数，采用`include`共享布局。
- View中设置缓存属性`.setDrawingCache`为true。
- png图片采用压缩工具压缩。
- 没有透明度要求的图片，应使用**jpg**，加快decode/compress速度。
- 不要通过静态变量缓存Drawable对象。
- 不要将同一个Drawable对象设置给多个控件。
- 对于复杂的窗口，不要在顶层使用RelativeLayout、LinearLayout等[multi-pass layout](/2016/04/27/study/multi-pass-viewgroup)。
- 谨慎使用`requestLayout()`，会执行measure, layout, draw等操作。
- 文字颜色尽量不设置透明度。
- 自绘的View，`invalidate`应当计算最小的重绘区域，避免重绘整个View。
- 使用**EditText**时，最好指定**inputType**，追求更高体验。
- 不要在**onAnimationEnd()**中把自己从view tree中移除，AC下这个View还会继续绘制最后一帧，移除会导致parent的dispatchDraw出现崩溃。
- View的框架事件中不能修改布局，否则会导致显示/响应错乱甚至崩溃，如：onMessure、onLayout、onDraw、onTouchEvent、dispatchKeyEvent、onInterupteXxx等。
- View的框架事件中不宜创建对象，因其触发频度高，可能引起频繁GC进而影响性能。
- 按钮尽量都设为`singleLine(true)`，以防止按钮文本过长时出现折行。
- 对于字体使用sp还是dp，看实际需要。Android官方推荐使用sp，设置为sp的文字会随系统文字大小变化而变化，dp则不会。
- ListView中使用convertView复用控件；背景色与cacheColorHint设置相同颜色，提高滑动时渲染性能；item尽可能减少布局。

### 数据/内存相关
- 避免在UI线程执行耗时操作
- 尽量不在主线程做I/O、联网操作。
- Service.onCreate()/onStart()，BroadcastReceiver.onReceive()都是在UI线程执行，因此不要在这些方法里面执行耗时操作。
- 优先使用线程池管理线程。
- 注意Cursor，文件I/O的关闭。
- Message必须采用`Message.Obtain()`获取，Android2.x存在bug，使用new Message()获取的Message对象将会泄漏。
- Activity的Context在Activity范围内使用，即UI相关的操作，其他情况使用Application Context。
- 使用`View.post()`接口时，应确保当前对象处于View Tree上。
- 有些能用文件操作的，尽量采用文件操作，文件操作的速度比数据库的操作要快10倍左右。

### 编码相关
- 用`static final`声明常量。
- 用`移位`代替乘除运算。
- 使用`StringBuffer`或`StringBuilder`替代String的构造操作。
- StringBuilder没有同步保护，因此比StringBuffer要快；如果构建的字符串是局部变量，则没有线程安全问题，用StringBuilder效率高。

- 字符串比较使用`"html".equals(targetString)`这种方法，避免空指针异常；
- 谨慎使用`synchronized`；调用一个synchronized方法比没有synchronized的方法要慢。
- **慎用反射**。通过反射调用方法，访问成员变量要慢很多。
- 尽量避免创建对象，使用引用。
- 使用`clone()`方法避免构造一个新的对象。例如数组也是一个对象，可以用clone()方法拷贝数组。
- 变量引用新对象之前先把变量置`null`；特别注意，在变量引用的旧对象占用大量内存时，及时把变量置null使得GC有机会及时把内存回收。
- 如果不需要访问对象的属性，把方法声明为static，这样方法的调用会快15%~20%。
- 如果不希望方法被重载，把方法声明为`final`。
- 类的内部不要用getter/setter访问成员变量，Android下虚函数的调用代价高昂。把成员设为public，直接访问。
- 使用增强for循环`foreach`或`for (obj : objs)`。
- 私有内部类要访问外部类的field或方法时,其成员变量不要用private，因为在编译时会生成setter/getter,影响性能。可以把外部类的field或方法声明为包访问权限。
- 容器使用完先用clear()或removeAllElements()清空然后再置null；数组直接置null就可以，而不用每个数组元素都置null。
- 少使用`浮点数`。`浮点数`比`整形`慢两倍。
- 尽量原生数据类型替代包装类。使用`int`替代`Interger`。
- **慎用异常**，异常对性能不利。抛出异常首先要创建一个新的对象。Throwable接口的构造函数调用名为fillInStackTrace()的本地（Native）方法，fillInStackTrace()方法检查堆栈，收集调用跟踪信息。只要有异常被抛出，VM就必须调整调用堆栈，因为在处理过程中创建了一个新的对象。
异常只能用于错误处理，不应该用来控制程序流程。
- 少用inner class，在主类实现相应的Listener接口，处理event；inner class和anonymous class都是另外一个类，运行时都要加载到内存。但这是以牺牲main class的重用性为代价。
```java
public class MyActivity extends Activity { 
    protected void onCreate(Bundle icicle) { 
        super.onCreate(icicle); 
        setContentView(R.layout.content_layout_id);  
        final Button button = (Button) findViewById(R.id.button_id); 
        button.setOnClickListener(new View.OnClickListener() { 
            public void onClick(View v) { 
                 // Perform action on click 
            }
        });
    }
}
  
// 编译时会多出一个类来，如果改成如下实现，则只有一个类。
public class MyActivity extends Activity implements View.OnClickListener
    // …
    button.setOnClickListener(this);
```
- Vector，HashTable和Stack的访问都是同步的，因此每次访问都有同步开销；不过不需要同步访问，考虑HashMap，ArrayList。
- 尽量使用索引去操作容器。例如用ArrayList. remove(int index)比ArrayList. remove(Object object)要快，因为它不用去查找索引。
- 尽量使用有快速hashcode()方法的类型做hashtable的key，推荐Number类型，例如Integer, Short等，避免使用String类型的key。
- 创建容器(Map、List、Set等)时指定适当的capacity，以减少扩容时的性能和内存损耗。
