---
title: FileObserver文件监听研究
date: 2016-06-19 19:25:35
tags:
- Android
---
在Linux系统中，使用inotify监控文件的变更。Android在这个基础上封装了一个FileObserver类来方便使用inotify机制。FileObserver是一个抽象类，需要定义子类实现该类的onEvent抽象方法，当被监控的文件或者目录发生变更事件时，将回调FileObserver的onEvent()函数来处理文件或目录的变更事件。

<!--more-->
## 实例
首先，我们创建一个类MyFileOberserver继承FileObserver：
```java
public class MyFileObserver extends FileObserver {
    public MyFileObserver(String path) {
        super(path, ALL_EVENTS);
    }
 
    @Override
    public void onEvent(int event, String path) {
        switch (event) {
            case FileObserver.ACCESS:
                Log.i("MyFileObserver", "ACCESS: " + path);
                break;
            case FileObserver.ATTRIB:
                Log.i("MyFileObserver", "ATTRIB: " + path);
                break;
            case FileObserver.CLOSE_NOWRITE:
                Log.i("MyFileObserver", "CLOSE_NOWRITE: " + path);
                break;
            case FileObserver.CLOSE_WRITE:
                Log.i("MyFileObserver", "CLOSE_WRITE: " + path);
                break;
            case FileObserver.CREATE:
                Log.i("MyFileObserver", "CREATE: " + path);
                break;
            case FileObserver.DELETE:
                Log.i("MyFileObserver", "DELETE: " + path);
                break;
            case FileObserver.DELETE_SELF:
                Log.i("MyFileObserver", "DELETE_SELF: " + path);
                break;
            case FileObserver.MODIFY:
                Log.i("MyFileObserver", "MODIFY: " + path);
                break;
            case FileObserver.MOVE_SELF:
                Log.i("MyFileObserver", "MOVE_SELF: " + path);
                break;
            case FileObserver.MOVED_FROM:
                Log.i("MyFileObserver", "MOVED_FROM: " + path);
                break;
            case FileObserver.MOVED_TO:
                Log.i("MyFileObserver", "MOVED_TO: " + path);
                break;
            case FileObserver.OPEN:
                Log.i("MyFileObserver", "OPEN: " + path);
                break;
            default:
                Log.i("MyFileObserver", "DEFAULT(" + event + "): " + path);
                break;
        }
    }
}
```
我们只需要调用startWatching方法，就可以开始监听文件了：
```java
String path = Environment.getExternalStorageDirectory().getAbsolutePath();
listener = new MyFileObserver(path);
//开始监听
listener.startWatching();
```
但是，FileObserver只能支持一个文件或文件夹的变化，对于子文件不具备监听功能。那么我们需要逐个对其下的子文件各个再设置监听，这里使用到了递归的使用：
```java
public class RecursiveFileObserver extends FileObserver {
    /**
     * Only modification events
     */
    public static int CHANGES_ONLY = CREATE | DELETE | CLOSE_WRITE | MOVE_SELF | MOVED_FROM | MOVED_TO;
 
    List<SingleFileObserver> mObservers;
    String mPath;
    int mMask;
 
    public RecursiveFileObserver(String path) {
        this(path, ALL_EVENTS);
    }
 
    public RecursiveFileObserver(String path, int mask) {
        super(path, mask);
        mPath = path;
        mMask = mask;
    }
 
    @Override
    public void startWatching() {
        if (mObservers != null) return;
 
        mObservers = new ArrayList<SingleFileObserver>();
        Stack<String> stack = new Stack<String>();
        stack.push(mPath);
 
        while (!stack.isEmpty()) {
            String parent = stack.pop();
            mObservers.add(new SingleFileObserver(parent, mMask));
            File path = new File(parent);
            File[] files = path.listFiles();
            if (null == files) continue;
            for (File f : files) {
                if (f.isDirectory() && !f.getName().equals(".") && !f.getName().equals("..")) {
                    stack.push(f.getPath());
                }
            }
        }
 
        for (SingleFileObserver sfo : mObservers) {
            sfo.startWatching();
        }
    }
 
    @Override
    public void stopWatching() {
        if (mObservers == null) return;
 
        for (SingleFileObserver sfo : mObservers) {
            sfo.stopWatching();
        }
        mObservers.clear();
        mObservers = null;
    }
 
    @Override
    public void onEvent(int event, String path) {
        switch (event) {
            case FileObserver.ACCESS:
                Log.i("RecursiveFileObserver", "ACCESS: " + path);
                break;
            case FileObserver.ATTRIB:
                Log.i("RecursiveFileObserver", "ATTRIB: " + path);
                break;
            case FileObserver.CLOSE_NOWRITE:
                Log.i("RecursiveFileObserver", "CLOSE_NOWRITE: " + path);
                break;
            case FileObserver.CLOSE_WRITE:
                Log.i("RecursiveFileObserver", "CLOSE_WRITE: " + path);
                break;
            case FileObserver.CREATE:
                Log.i("RecursiveFileObserver", "CREATE: " + path);
                break;
            case FileObserver.DELETE:
                Log.i("RecursiveFileObserver", "DELETE: " + path);
                break;
            case FileObserver.DELETE_SELF:
                Log.i("RecursiveFileObserver", "DELETE_SELF: " + path);
                break;
            case FileObserver.MODIFY:
                Log.i("RecursiveFileObserver", "MODIFY: " + path);
                break;
            case FileObserver.MOVE_SELF:
                Log.i("RecursiveFileObserver", "MOVE_SELF: " + path);
                break;
            case FileObserver.MOVED_FROM:
                Log.i("RecursiveFileObserver", "MOVED_FROM: " + path);
                break;
            case FileObserver.MOVED_TO:
                Log.i("RecursiveFileObserver", "MOVED_TO: " + path);
                break;
            case FileObserver.OPEN:
                Log.i("RecursiveFileObserver", "OPEN: " + path);
                break;
            default:
                Log.i("RecursiveFileObserver", "DEFAULT(" + event + "): " + path);
                break;
        }
    }
 
    /**
     * Monitor single directory and dispatch all events to its parent, with full path.
     *
     * @author uestc.Mobius <mobius@toraleap.com>
     * @version 2011.0121
     */
    class SingleFileObserver extends FileObserver {
        String mPath;
 
        public SingleFileObserver(String path) {
            this(path, ALL_EVENTS);
            mPath = path;
        }
 
        public SingleFileObserver(String path, int mask) {
            super(path, mask);
            mPath = path;
        }
 
        @Override
        public void onEvent(int event, String path) {
            String newPath = mPath + "/" + path;
            RecursiveFileObserver.this.onEvent(event, newPath);
        }
    }
}
```

